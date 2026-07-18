# Build Log: Screenwell
**Date:** 2026-07-15
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Surveyed the existing catalog (15 tools) and found a clear, uncovered gap: **in-browser screen recording**. No existing tool touches the Screen Capture API / MediaRecorder, the privacy story is strong (screen recordings are highly sensitive and every "free online recorder" either installs software, watermarks, or uploads), and it's very Googleable ("free online screen recorder no watermark no download / no upload"). Confirmed `screenwell` was free both in registry.json and `gh repo list ben-gy` (and fits the -well naming family).

Checked the deploy backlog first (per memory): `pagewell` in tools/ is a known concurrent-run duplicate of `pagesmith` (repo exists, intentionally unindexed) — not a real backlog item. All genuine tools are deployed.

## Tool Details
- **Name:** Screenwell
- **Repo:** ben-gy/screenwell
- **Category:** media-tools (index category: media)
- **Audience:** Developers / support engineers who need a quick screen clip for a ticket or Slack without installing OBS or trusting a SaaS with their screen.
- **Stack:** Vite 6 + vanilla TypeScript + Vitest (no runtime dependencies)
- **Browser APIs:** Screen Capture API (getDisplayMedia), MediaRecorder + isTypeSupported, Web Audio (MediaStreamAudioDestinationNode mixing), getUserMedia (mic + webcam), Canvas 2D + captureStream (webcam compositor), Web Share API (files), Service Worker (PWA)
- **Worker strategy:** none. MediaRecorder performs the CPU-heavy video encoding natively, off the main thread, inside the browser. The optional webcam compositor is a lightweight requestAnimationFrame drawImage loop (with a document.hidden keep-alive fallback so a backgrounded tab doesn't freeze the composite). Documented honestly in README/plan.

## Privacy Model
- **Protected:** Screen capture, mic, system audio, webcam frames and the encoded video never leave the device — there is no upload endpoint anywhere in the code. No account, no cookies for user data, no third-party fonts. Fully offline once loaded (PWA).
- **Not protected:** The output is an ordinary, unencrypted video file the user handles/shares themselves; the screen-picker dialog is the browser's, not Screenwell's.
- **Trust surface:** Static bundle + GitHub Pages TLS; the browser's native Screen Capture / MediaRecorder; a cookie-less Cloudflare Web Analytics page-view beacon (recording never sent to it). CSP is `self`-only apart from the beacon host.

## Architecture Decisions
- **Vanilla TS, not React** — one screen driven by a small state machine; no framework overhead needed.
- **Pure reducer for the recorder lifecycle** (`reduce(model, action)`) so the full idle→arming→recording→paused→finalizing→ready/error graph is unit-testable without any media APIs. `RecorderController` wraps a real MediaRecorder around it.
- **Record the raw screen track directly when there's no webcam overlay** (sharper, no re-encode); only spin up the canvas compositor when the webcam bubble is enabled.
- **Web Audio mixing** only when >1 audio source exists; a single source passes its track straight through.
- **Codec picked per browser** via a preference list (MP4/H.264 first for Safari, then WebM/VP9…), injected `isTypeSupported` for testability.

## Test Results
- Tests written: 33 (2 files)
- Tests passed: 33
- Tests failed: 0
- `format.test.ts` — codec/MIME selection (incl. throwing isTypeSupported), extForMime, codecLabel, formatBytes/Duration/Throughput, timestampSlug + buildFilename.
- `recorder.test.ts` — full state-machine transition coverage incl. invalid transitions, fail-from-any-state, reset, isActive.
- One initial failure: formatBytes rounded 1536 → "2 KB" (a stray `i === 0` clause forced integer KB); fixed to `value >= 100 ? 0 : 1`.

## Build Status
- npm install: pass
- npm test: pass (33/33)
- npm run build: pass (tsc + vite; needed a `src/vite-env.d.ts` reference for `import.meta.env`)
- Local preview: pass (vite preview :5203)
- Production workflow dry-run: pass — verified idle/modal/event-log/mobile states in the preview browser, no console errors, and drove a real canvas→MediaRecorder→Blob pipeline (3 chunks, valid MP4 blob) to prove the recording engine works in-browser.

## Deployment
- Repo created: yes (ben-gy/screenwell)
- GitHub Pages enabled: yes (workflow build)
- DNS: Cloudflare CNAME screenwell → ben-gy.github.io created
- TLS: https_enforced=true after one CNAME cycle
- Live: https://screenwell.benrichardson.dev returns 200 with correct title + CF beacon
- PR created: https://github.com/ben-gy/screenwell/pull/1
- Workflow triggered: yes — "Deploy to GitHub Pages" completed successfully
- Index + registry pushed to ben-gy/gh-tool-factory (main)

## Errors & Resolutions
- **formatBytes rounding bug** — integer-rounded KB; fixed the decimal condition. (test caught it)
- **`import.meta.env` type error** under tsc — added `src/vite-env.d.ts` with the vite/client reference.
- **`[hidden]` overridden by `display:flex`** — the REC badge, stats and error box use `display:flex` in CSS, which beat the UA `[hidden]` rule, so they showed in the idle state. Added a global `[hidden] { display: none !important; }`. (caught in visual preview)
- **Stale preview server on :5199** — a previous tool's `vite preview` still held the port and served Pixpress; moved to :5203.
- **Mobile nav wrapped "How it works" across 3 lines** — added `white-space:nowrap` + topbar `flex-wrap` in the mobile media query.
