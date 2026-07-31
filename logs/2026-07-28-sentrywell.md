# Build Log: Sentrywell
**Date:** 2026-07-28
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty and every existing `tools/` dir mapped to a
deployed registry entry (no undeployed backlog; collate/pagewell are the
known-dead duplicates). Applied principle #6's input question — recent runs
leaned on file-in/file-out and mic; the camera-as-a-*monitor* (motion detection,
MediaRecorder, wake lock) was an untouched combination. Picked a **private,
on-device motion-detecting security camera**: the honest answer to "use my old
laptop/phone as a security camera without uploading my home to a free app's
cloud", plus the inverse — "find the movement in this CCTV/dashcam video". Both
leave the user with real artefacts (clips, snapshots, contact sheet, CSV/JSON).

## Tool Details
- **Name:** Sentrywell
- **Repo:** ben-gy/sentrywell
- **Category:** sensor-tools (index category: privacy)
- **Audience:** Someone repurposing a spare device as a private room monitor
  (parcel watch, pet/baby cam, workshop) who won't upload home footage to a cloud
  app — and, secondarily, anyone hunting the moving moments in existing footage.
- **Stack:** Vite 6 + vanilla TypeScript + Vitest. No runtime dependencies.
- **Browser APIs:** getUserMedia (camera + optional mic), MediaRecorder, Canvas 2D
  frame differencing, HTMLVideoElement seek-stepping, Screen Wake Lock,
  Notifications + Web Audio, IndexedDB, File System Access / Web Share / Clipboard,
  Service Worker (PWA).
- **Worker strategy:** main-thread `setInterval` detection loop (a hidden tab
  throttles rAF to a halt, which would kill a background monitor); the grid diff is
  O(cells) and cheap, and MediaRecorder encodes off-thread. No dedicated worker.

## Privacy Model
- **Protected:** camera/mic frames are compared and discarded — nothing is recorded
  unless motion is detected; clips live only in this browser's IndexedDB on this
  device; dropped videos are scanned in-tab; every track is stopped on
  disarm/pagehide; "Clear all" wipes stored events.
- **Not protected:** GitHub Pages' CDN sees the initial page-load IP; a
  notification's text shows on the user's own lock screen; detection is pixel-based
  (light changes fool it, and it can't tell a person from a curtain); while armed,
  monitoring continues in the background by design.
- **Trust surface:** the hash-pinned static bundle + TLS to Pages; a cookie-less
  Cloudflare Web Analytics beacon (anonymous page views, footage never sent to it);
  the feedback widget only when the user opens it and presses Send.

## Architecture Decisions
- **First-party DSP, unit-tested.** `motion.ts` reduces a frame to a 20×15 luma
  grid and diffs cells within an optional zone mask; `gate.ts` holds both the live
  `MotionGate` state machine (debounce → start, cooldown/maxClip → stop) and the
  `segmentsFromScores` grouper for the file path. Keeping these pure made them
  testable without a camera and is what the automated dry-run leans on.
- **Two modes, one DSP.** Live mode (camera sensor) and file mode (upload a video)
  share the exact grid-diff code; file mode is also the first-class fallback when
  the camera is denied, and the only path an automated harness can drive end-to-end.
- **Seek-stepping the file path** rather than playback keeps it deterministic. Each
  seek waits one presented frame via `requestVideoFrameCallback`, **always raced
  against a timeout** — rvfc (like rAF) is paused on a hidden tab, and without the
  fallback a backgrounded scan hangs forever (caught and fixed during the dry-run).
- **A fresh MediaRecorder per event** so every clip is a self-contained,
  immediately-playable file; codec chosen via `isTypeSupported` (vp9/vp8/webm, mp4
  on Safari); snapshot-only degradation if MediaRecorder is unavailable.
- **Dark monitoring-console aesthetic** — a live feed reads best on dark, dark cuts
  glare on a device left watching a room, cyan accent with amber=armed/red=recording.

## Test Results
- Tests written: 29 (2 files: motion DSP + gate/segmentation/formatting)
- Tests passed: 29
- Tests failed: 0

## Build Status
- npm install: pass
- npm test: pass (29/29)
- npm run build: pass (tsc + vite; 51 KB JS gzip 17.75 KB)
- Local preview: pass (HTTP 200)
- Production workflow dry-run: pass — see below

## Dry-run (local production preview, in-browser)
- **File-scan path driven end-to-end** on the shipped sample: 1 motion segment
  detected (`00:03.0 – 00:05.6`, the crossing bar), thumbnail rendered, and the
  **timeline CSV, segments JSON, and contact-sheet PNG artefacts were all produced**
  (verified by intercepting the blob outputs: 546 B CSV, 301 B JSON, 37 KB PNG).
- **Live artefact path** proven via a synthetic event (`captureSynthetic`):
  snapshot → gallery card → IndexedDB → contact-sheet PNG (1.6 KB JPEG in, PNG out).
- **DSP + gate** verified through the test hook (sensitivity mapping monotonic;
  gate fires start after 3 consecutive over-frames and stops after the cooldown).
- Modals open (Privacy button labelled "Privacy"; beacon disclosure present); event
  drawer closes via × and Escape at 375 px; `[hidden]` guard holds (drawer/modal
  `display:none` on load, page centre is real content); PWA manifest resolves with a
  maskable icon; apple-touch-icon 200; no horizontal overflow at 375 px; no console
  errors.
- **No real-device camera/microphone check was possible** in this pipeline
  (getUserMedia + MediaRecorder can't be driven headlessly). The live sensor wiring
  is covered by unit-tested pure functions and the synthetic hook; the file path is
  the fully-exercised end-to-end path. This is stated in the PR too.

## Deployment
- Repo created: yes (ben-gy/sentrywell)
- GitHub Pages enabled: yes (workflow build); deploy run succeeded
- DNS: Cloudflare CNAME sentrywell → ben-gy.github.io (created)
- Custom domain live: https://sentrywell.benrichardson.dev serves the app (HTTP 200);
  `https_enforced` was still flipping at ship time (GitHub finishes it server-side).
- PR created: https://github.com/ben-gy/sentrywell/pull/1
- Registry + public index updated and pushed; IndexNow pinged; lab deploy triggered.

## Errors & Resolutions
- **Sample generation:** no ffmpeg/cv2/imageio on the box. Authored the sample MP4
  in Node with a WASM H.264 encoder (`h264-mp4-encoder`) installed in scratch, from
  hand-drawn RGBA frames (a dark room with an "intruder" bar crossing 2.4–5.4 s).
- **Seek snapping to keyframes:** the first sample (GOP = fps) produced motion
  scores only at 1 s intervals — seeking snapped to keyframes, so the gate never saw
  consecutive motion frames (0 segments). Re-encoded the sample **all-intra**
  (`groupOfPictures = 1`) so every 0.2 s seek lands on a distinct frame.
- **Hidden-tab scan hang:** adding a `requestVideoFrameCallback` frame-wait to the
  seek made a backgrounded scan hang (rvfc is paused when the document is hidden).
  Raced every wait against a timeout so the scan always progresses. Both fixes
  verified: the sample scans to exactly 1 segment.
- **Third-party notices:** the app bundle is fully first-party; the only shipped
  third-party code is the Workbox SW runtime (MIT, via vite-plugin-pwa). Notices
  assembled mechanically from the on-disk `workbox-core/LICENSE`.
