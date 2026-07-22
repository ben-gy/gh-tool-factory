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

