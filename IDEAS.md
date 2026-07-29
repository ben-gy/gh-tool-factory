# Tool Ideas

Add client-side browser tool ideas here. The scheduled task will pick the first idea from this list, build it, and remove it. One idea per line — be as brief or detailed as you like, the task will flesh it out.

**The bar every idea must clear:**

- **Zero runtime backend.** GitHub Pages only. GitHub Actions at build time is fine — nothing at runtime.
- **Pushes a browser API to do something genuinely useful.** Web Crypto, WebCodecs, WebRTC/DataChannel, WebTorrent, File System Access, Origin Private File System, WASM (ffmpeg.wasm, pdf-lib, jsquash, onnxruntime-web, transformers.js), Web Audio, Web Speech, Web Share, WebUSB, WebBluetooth, WebSerial, WebHID, WebGPU, OffscreenCanvas, Streams, Compression Streams. The more ambitious the capability, the better.
- **Privacy-first.** Data never leaves the device. The UI must say so explicitly and the "How it works" / "Threat model" modals must back it up.
- **Solves a real problem someone would Google for.** Not a novelty demo. Someone is stuck, they type a query, this tool is the best answer. **This tests demand, not seriousness** — "voice changer online" and "spirit level app" are real, high-volume searches. Playful is fine; unwanted is not.
- **The user leaves with an artefact.** A file, an image, a number they can act on, a link they can send. This is the line between a playful *tool* and a toy.
- **Not CSS-only.** Must have real logic — compression, encryption, codec work, P2P transfer, ML inference, signal processing. No pure-CSS utilities.

**The input does not have to be a file.** The microphone, camera, gyroscope and accelerometer are all
local, zero-backend and a perfect fit for the privacy promise — a live-sensor tool is the most honest
version of "your data never leaves the device". Ideas in that register are very welcome.

The north-star is **[Dropwell](examples/dropwell/)** — end-to-end encrypted peer-to-peer file transfer with no server, using WebTorrent + Web Crypto. That level of ambition is the baseline, not the ceiling.

## Ideas

**Shared note for every P2P idea below (read once, applies to all eight).** These all follow the Dropwell pattern so "zero runtime backend" holds honestly: **peer discovery/signalling rides public infrastructure** (WebTorrent trackers or nostr relays via a trystero-style layer) — no dedicated server, nothing we host at runtime. WebRTC gives DTLS on the wire, but the **hard privacy win is an application-layer E2E layer on top**: derive a key with Web Crypto (PBKDF2/HKDF) from a room passphrase that is **never sent to any relay**, encrypt media frames with **Insertable Streams / SFrame** (WebCodecs) and data with AES-GCM, and show a **Short Authentication String** (a few words both sides read aloud) so a malicious relay cannot MITM the handshake. Be honest in the Threat Model modal about the **one real limitation**: NAT traversal uses public STUN, and a symmetric-NAT pair with no TURN relay may fail to connect — we will not add a TURN relay because that would be a backend and would see the (encrypted) traffic. Every idea must still leave an **artefact** (a shareable room link, a local-only recording, a saved file, a delivered payload) — a call with no takeaway is a toy.

- **P2P encrypted collaborative whiteboard/doc** — real-time co-editing of a whiteboard (or a plain-text/markdown doc) with a CRDT synced over an encrypted DataChannel — no Google account, nothing stored anywhere. Genuinely useful "collaborative whiteboard no signup" / "shared scratchpad private" demand, and the CRDT + E2E combo is ambitious. Artefact: export the board as PNG/SVG or the doc as markdown, and a room link to invite others live.
- **P2P encrypted secret hand-off (burn-after-read)** — send a password, API key, or recovery phrase directly device-to-device over a one-time encrypted channel, verified by an out-of-band Short-Authentication-String, then **burned** — no "paste your secret into a website that emails a link" middleman, no copy persisted anywhere. The safe answer to "how do I securely send someone a password". Distinct from the clipboard hand-off: single-use, MITM-verified, self-destructing. Artefact: confirmed one-time delivery, and proof it's gone.
