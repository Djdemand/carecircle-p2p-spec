# CareCircle P2P — Architecture Specification

## Vision

A fully independent mobile app. No cloud database, no subscription, no server. Fifteen caregivers coordinate medication administration for a patient using direct phone-to-phone communication. The app works across different cellular networks, different carriers, and spotty internet. QR codes establish trust. WebRTC carries the data. SQLite keeps the history.

---

## Table of Contents

1. [Peer Discovery & QR Handshake](#1-peer-discovery--qr-handshake)
2. [WebRTC Mesh Topology](#2-webrtc-mesh-topology)
3. [State Synchronization](#3-state-synchronization)
4. [Conflict Resolution](#4-conflict-resolution)
5. [Offline & Reconnection](#5-offline--reconnection)
6. [Security Model](#6-security-model)
7. [Notification Strategy](#7-notification-strategy)
8. [React Native Implementation Plan](#8-react-native-implementation-plan)
9. [Data Flow Diagrams](#9-data-flow-diagrams)
10. [Migration Path from Supabase](#10-migration-path-from-supabase)

---

## 1. Peer Discovery & QR Handshake

### Concept

One caregiver (the "host") generates a QR code. Everyone else scans it. After scanning, WebRTC negotiates a direct encrypted connection. No server mediates the data exchange after the initial handshake.

### The QR Code

The QR code encodes a JSON object — NOT a server URL, NOT an IP address. It encodes cryptographic material for WebRTC signaling:

```json
{
  "v": 1,
  "cid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "sig": "eyJhbGciOiJFZERTQSJ9...",
  "ice": [
    "candidate:842163049 1 udp 16777215 73.12.45.67 54321 typ srflx",
    "candidate:842163049 1 udp 16777215 192.168.1.5 54321 typ host"
  ],
  "fp": "sha-256 AB:CD:EF:12:34:56:78:90:...",
  "ts": 1719000000
}
```

| Field | Meaning |
|-------|---------|
| `cid` | Unique CareCircle ID for this patient/care-circle |
| `sig` | Ed25519 signature of the connection offer |
| `ice` | ICE candidates — public and local IP:ports |
| `fp` | DTLS certificate fingerprint (identity verification) |
| `ts` | Timestamp to prevent replay |

### Handshake Sequence

```
Host (Phone A)                      Guest (Phone B)
─────────────────                   ─────────────────
1. Opens app
2. Generates ECDH keypair
3. Creates WebRTC offer + ICE           
4. Encodes offer → QR code
5. Shows QR on screen   
                                    6. Opens app → "Join Circle"
                                    7. Scans QR code
                                    8. Decodes offer
                                    9. Reads fingerprint
                                   10. User confirms: "Connect to Wallace's circle?"
                                   11. Generates own ECDH keypair
                                   12. Creates WebRTC answer
                                   13. Sends answer via data channel
14. Receives answer
15. DTLS handshake completes
16. Secure P2P channel established
```

### ICE Candidate Exchange via QR

Since QR codes are scanned from screens (one-directional), the host pre-generates ICE candidates and embeds them in the QR. The guest uses these to connect. The guest's candidates flow back through the newly established WebRTC data channel.

If NAT traversal fails with just the QR-embedded candidates, fallback: both phones connect to a free public STUN server (`stun:stun.l.google.com:19302`) to discover their public IPs, then re-exchange candidates over the already-partially-established channel.

### Reconnection

After the first pairing, each device stores the peer's WebRTC credentials locally:

```
SQLite table: peers
  id            TEXT PRIMARY KEY
  cid           TEXT (circle ID)
  fingerprint   TEXT (DTLS fingerprint)
  last_ice      TEXT (last known ICE candidates, JSON)
  last_seen     INTEGER (unix timestamp)
  display_name  TEXT
```

On subsequent app launches, the device tries to reconnect to all known peers automatically using stored credentials. If DNS/IPs have changed, STUN rediscovery resolves it. Only if ALL peers are unreachable does the app fall back to showing a QR code again.

---

## 2. WebRTC Mesh Topology

### Why Mesh (not Star)

With 15 caregivers, a star topology (one server, 14 clients) means if the server's phone goes offline, everyone loses sync. In a mesh, every device connects to every other device — no single point of failure.

```
      A ──────── B
      │ ╲       ╱ │
      │  ╲     ╱  │
      │   ╲   ╱   │
      │    ╲ ╱    │
      C ─── D ─── E
      │ ╲   │   ╱ │
      │  ╲  │  ╱  │
      ... up to 15 peers
```

### Connection Count

15 devices, full mesh = 105 P2P connections. In practice, WebRTC handles this efficiently — each connection is just a data channel, not a separate bandwidth pipe. For medication sync (small JSON payloads, not streaming video), 105 connections is negligible overhead.

### Mesh Maintenance

Each device maintains a list of connected peers. When a message arrives from peer X, it's processed locally and NOT re-broadcast (no flooding). Instead, each device is responsible for sending its own state changes to all connected peers directly.

```
Phone A logs a dose → sends to B, C, D, E, F...
Each recipient stores locally and updates UI
No peer re-broadcasts — avoids loops
```

### Signaling Server (Optional, Free)

For the initial handshake when QR scanning isn't convenient, a minimal signaling relay can help:

- **Free option**: A single Cloudflare Worker (100K requests/day free) that forwards WebRTC offers/answers between peers for 30 seconds during initial pairing, then drops out
- **Zero-server option**: QR code only (no relay at all)
- **SMS option**: Encode the initial offer in a text message as a deep link

---

## 3. State Synchronization

### CRDT-Inspired Approach

The app doesn't need full CRDT libraries. Medication state is simple enough for a practical hybrid:

**Data types and how they sync:**

| Data Type | Sync Strategy |
|-----------|--------------|
| Medication definition (add/edit/delete) | Last-write-wins by timestamp |
| Dose log (medication taken) | Append-only, immutable once logged |
| Hydration log | Append-only, immutable |
| BM log | Append-only, immutable |
| Medication position (drag reorder) | Last-write-wins by timestamp |
| User profile | Last-write-wins by timestamp |

### Sync Protocol

Every state change broadcasts a message to all connected peers:

```json
{
  "type": "med_log",
  "action": "create",
  "payload": {
    "id": "uuid",
    "medication_id": "uuid",
    "taken_at": "2026-06-23T14:00:00Z",
    "taken_by": "uuid",
    "notes": ""
  },
  "sender": "uuid",
  "timestamp": 1719000000000,
  "hash": "sha256-of-payload"
}
```

### Tombstones for Deletes

Deleted items aren't removed — they're marked with `deleted_at`. All peers sync the deletion. After 30 days, a background cleanup removes tombstones older than that from all devices simultaneously.

### Sync on Connect

When a device comes online and connects to peers, it requests a "catch-up":

```
Phone B (rejoining): → "I last synced at timestamp 1719000000"
Phone A:             → sends all changes since that timestamp
Phone B:             → applies changes in timestamp order
                     → Phone B is now in sync
```

---

## 4. Conflict Resolution

### The Rule: Timestamp Order

When two caregivers log a dose for the same medication window simultaneously on different devices, both log entries exist. The UI shows the **earliest** one as the authoritative dose for that window. The duplicate is kept in the log for audit but flagged as `duplicate: true`.

### Example

```
Phone A: logs Hydrocodone dose at 14:00:01 (offline)
Phone B: logs Hydrocodone dose at 14:00:02 (offline, no network)
Both come online at 14:01
Phone A sends its log timestamped 14:00:01
Phone B sends its log timestamped 14:00:02
Both devices receive both logs
Both devices mark 14:00:02 as { duplicate: true }
UI shows "Taken at 14:00 by Wallace"
Log shows both entries for audit
```

### Medication Edits

If two caregivers edit the same medication simultaneously (e.g., one changes dosage to 10mg, another to 15mg), last-write-wins by timestamp. The "losing" edit is retained in a `medication_history` table as a side log for audit.

---

## 5. Offline & Reconnection

### Offline Mode

Every device has a full copy of all data in local SQLite. The app works identically online or offline — the only difference is whether changes propagate to other devices.

### Reconnection Flow

```
App starts / comes to foreground
  └→ Check SQLite peers table
      └→ For each known peer:
          ├─ Try stored ICE candidates
          ├─ If fail: query STUN server for new candidates
          ├─ If fail: mark peer unreachable
          └─ If success: establish WebRTC, request catch-up
  └→ If zero peers reached after 15 seconds:
      └→ Show QR code / "Waiting for connection" indicator
```

### Background Sync (minimal)

iOS and Android aggressively suspend background apps. WebRTC connections survive in the background for a limited time (iOS ~30 seconds, Android varies by OEM). Strategy:

1. When a dose is logged, immediately attempt to broadcast to all connected peers
2. If some peers are unreachable, queue the message
3. When the app comes to the foreground (or on a timer), flush the queue
4. Scheduled local notifications wake the app briefly on both platforms — use this window to attempt reconnection

---

## 6. Security Model

### Encryption

WebRTC mandates DTLS-SRTP — all data channels are encrypted end-to-end by the browser/OS WebRTC stack. No additional encryption layer needed.

### Trust Model

The QR code establishes trust physically — a caregiver must be in the same room to scan the host's screen. After pairing, the DTLS fingerprint is stored and verified on every reconnection. If a fingerprint changes, the app alerts: "Caregiver device identity changed. Re-scan QR to verify."

### No External Auth

No usernames, no passwords, no email. Each device generates a random UUID on first launch. Display names are set locally (e.g., "Wallace", "Nurse Sarah") and shared with the circle. This is a private care team, not a public service.

### Data at Rest

SQLite database is stored in the app's sandboxed storage. On iOS, this is already encrypted via Data Protection. On Android, add SQLCipher (or use `expo-secure-store` for the most sensitive fields).

### Expulsion

If a caregiver leaves the team, the host can "forget" their device. This removes their peer entry from the mesh. Their stored fingerprint is invalidated. They cannot reconnect without re-scanning a fresh QR code (which the host would not provide).

---

## 7. Notification Strategy

### Scheduled Local Notifications

Each device independently schedules local notifications for every medication in its database. No server needed.

```
Medication: Hydrocodone every 4 hours
→ Device schedules: 04:00, 08:00, 12:00, 16:00, 20:00, 00:00
→ When a dose is logged at 08:00, cancel the 08:00 notification
→ When peer syncs a dose at 08:00, cancel local 08:00 notification too
```

### Cross-Device Notification (WebRTC)

When caregiver A logs a dose, all connected peers receive the WebRTC message and immediately show a local notification: "Wallace administered Hydrocodone at 08:00." This is NOT a push notification — it only works when the app is in the foreground or recently backgrounded.

### SMS Fallback Alert

For critical missed doses when all peers are unreachable, the app can optionally send an SMS to designated emergency contacts. This is a configurable per-circle setting, off by default.

---

## 8. React Native Implementation Plan

### Required Packages

```json
{
  "dependencies": {
    "react-native-webrtc": "^124.0.0",
    "expo-sqlite": "~14.0.0",
    "expo-camera": "~15.0.0",
    "expo-notifications": "~0.28.0",
    "react-native-qrcode-svg": "^6.3.0",
    "expo-crypto": "~13.0.0",
    "react-native-tcp-socket": "^6.2.0"
  }
}
```

### Module Architecture

```
src/
├── p2p/
│   ├── WebRTCManager.ts      // create offers, answers, data channels
│   ├── PeerRegistry.ts        // track connected peers, store credentials
│   ├── SignalingProtocol.ts   // encode/decode QR & WebRTC signaling messages
│   ├── MeshSync.ts            // broadcast state changes, handle catch-up
│   └── ConnectionRecovery.ts  // reconnection logic, STUN fallback
├── db/
│   ├── schema.ts              // SQLite schema (mirrors current Supabase)
│   ├── medications.ts          // CRUD for medications
│   ├── logs.ts                // dose/hydration/bm logs
│   └── sync.ts                // track sync state, apply deltas
├── sync/
│   ├── OutgoingQueue.ts       // queue messages when offline
│   ├── ConflictResolver.ts    // timestamp-based resolution
│   └── CatchUpProtocol.ts     // request/respond with change sets
├── auth/
│   ├── DeviceIdentity.ts      // generate/load device UUID, keypair
│   └── TrustStore.ts          // store/verify peer fingerprints
├── notifications/
│   ├── ScheduleManager.ts     // local notification scheduling
│   └── PeerNotifier.ts        // show notifications for peer actions
├── screens/
│   ├── HostCircle.tsx         // generate QR, show peer list
│   ├── JoinCircle.tsx         // scan QR, confirm handshake
│   └── CircleStatus.tsx       // show connected peers, sync status
```

### Migration from Current Codebase

The existing `src/screens/` and `src/utils/` are well-structured. The P2P layer is an ADDITION, not a rewrite:

| Existing Module | P2P Equivalent | Change |
|----------------|----------------|--------|
| `utils/supabase.ts` | `db/` + `p2p/MeshSync.ts` | Replace Supabase calls with local DB + broadcast |
| `hooks/useRealtimeMeds.ts` | `db/sync.ts` | Listen to local DB changes instead of Supabase channel |
| `hooks/useAuth.ts` | `auth/DeviceIdentity.ts` | Generate local UUID instead of Supabase login |
| `utils/teamService.ts` | `p2p/PeerRegistry.ts` | Manage WebRTC peers instead of Supabase users |
| `screens/Login.tsx` | `screens/HostCircle.tsx` | Show QR code instead of login form |
| `supabase/migrations/` | `db/schema.ts` | SQLite migration instead of PostgreSQL |

---

## 9. Data Flow Diagrams

### Dose Logging (Normal Operation)

```
Phone A                          Phone B, C, D...
────────                         ────────────────
1. Caregiver taps "Take Dose"
2. SQLite INSERT med_log
3. UI updates to "Taken"
4. MeshSync.broadcast(log)
   └→ WebRTC → B             5. Receive message
   └→ WebRTC → C             6. Verify sender fingerprint
   └→ WebRTC → D             7. SQLite INSERT med_log
                             8. Cancel local notification
                             9. Show "Wallace administered X at 14:00"
                            10. UI updates
```

### New Caregiver Joining

```
Host (Phone A)                  New Guest (Phone E)
─────────────                   ───────────────────
1. "Add Caregiver" pressed
2. Generates WebRTC offer
   + ICE candidates
3. Shows QR code               4. Scans QR
                               5. Verifies fingerprint
                               6. "Connect to Wallace?" → Yes
                               7. Creates WebRTC answer
                               8. Sends answer via data channel
9. Receives answer
10. DTLS handshake
11. Sends full medication list 12. Receives list → populates SQLite
    + recent logs                  + recent logs
13. PeerRegistry.add(E)        14. PeerRegistry.add(A)
    └→ broadcast to B,C,D:         └→ now fully synced
       "New peer E joined"
15. B,C,D connect to E         16. Accept connections from B,C,D
```

---

## 10. Migration Path from Supabase

### Phase 1: Shadow Mode (safe, reversible)

- Add SQLite + WebRTC module alongside existing Supabase code
- All writes go to BOTH Supabase AND local SQLite
- WebRTC broadcasts are sent but not acted on
- Verify local DB matches Supabase after 1 week of real use
- **Supabase is still the source of truth — no risk**

### Phase 2: P2P Primary

- Reads come from SQLite (fast, offline-capable)
- Writes go to SQLite + broadcast via WebRTC
- Supabase becomes the fallback sync when WebRTC can't connect
- **Supabase is the backup, P2P is primary**

### Phase 3: Full Independence

- Supabase code removed
- All sync via WebRTC mesh
- QR codes for initial pairing
- **No cloud dependency remains**

### Rollback

At any phase, reverting is a single toggle: switch the data source back to Supabase. No data is lost because every write touches both stores during Phase 1.

---

## Appendix: QR Code Size & Encoding

A WebRTC offer with ICE candidates is roughly 2-4 KB. At QR code version 40 (max), this fits in a single QR code with binary encoding. At typical screen brightness and phone camera quality, scanning takes ~0.5-1 seconds.

For older phones or poor lighting, the app can split the offer across 2-3 sequential QR codes (animated QR). The scanner captures each frame and reassembles automatically. This is a solved problem — `react-native-qrcode-svg` handles animated QR generation.

---

*Specification v1.0 — Ready for implementation*
