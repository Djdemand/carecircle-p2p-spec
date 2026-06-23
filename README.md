# CareCircle P2P — WebRTC + QR Handshake Demo

Working demo of the CareCircle P2P architecture — medication sync between phones with no cloud database.

## Quick Start

Open `demo.html` in any modern browser.

### To test cross-device sync:

1. **Open two browser tabs** pointing to the same `demo.html`
2. **Tab A** (Host): A QR code appears automatically
3. **Tab B** (Guest): Switch to "Paste Offer" tab → copy the offer text from Tab A's textarea
4. **Tab B**: Paste into the textarea → click "Connect to Peer"
5. **Tab B**: An answer JSON appears in the textarea
6. **Tab A**: Switch to "Paste Offer" tab → paste Tab B's answer JSON → click "Connect to Peer"
7. **Both tabs are now connected!** Add medications in either tab — they sync instantly.

### To test with QR scanning:

1. Open `demo.html` on a desktop (Host) — QR code appears
2. Open `demo.html` on a phone (Guest) 
3. On the phone, switch to "Paste Offer" tab
4. Use a QR scanner app on the phone to read the desktop's QR → copy the decoded text
5. Paste into the phone's textarea → click "Connect to Peer"
6. Copy the answer from phone → paste into desktop's "Paste Offer" tab → click "Connect to Peer"
7. Connected! Add/take medications — they sync in real time.

## What This Demo Shows

- **WebRTC peer connection** using public STUN servers (no server needed after handshake)
- **QR code encoding** of the full WebRTC offer (ICE candidates, session description)
- **Data channel messaging** for medication sync
- **Multi-peer state synchronization** — add/delete medications, they appear everywhere
- **Offline resilience** — LocalStorage persists medications even when peers disconnect
- **Real-time log** of all sync events across peers

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full production architecture spec covering:

- WebRTC mesh topology for 15 caregivers
- QR handshake protocol with DTLS fingerprint verification
- CRDT-inspired sync with conflict resolution
- Offline queue and reconnection logic
- Security model (no external auth, physical trust via QR)
- React Native implementation plan
- Migration path from Supabase to fully independent

## Files

| File | Description |
|------|-------------|
| `demo.html` | Self-contained working demo |
| `ARCHITECTURE.md` | Full production architecture specification |
