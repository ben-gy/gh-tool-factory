# Build Log: Scribewell
**Date:** 2026-07-12
**Status:** deployed

## Idea Source
IDEAS.md was empty. Rather than research a fresh idea, discovery turned up a recurring
failure: **scribewell had been built three times before** (`tools/2026-06-29`, `07-01`,
`07-02`) plus sibling attempts (`echoscribe` 07-06, `unheic`, `tokenpeek`) — none of which
were ever deployed. The registry even carried a `built_not_deployed` breadcrumb:
_"Built locally three times but never deployed... needs finishing + deploy."_ No
`ben-gy/scribewell` repo existed. The root cause was interruption: two prior attempts got
`npm install` to complete but the run was killed before build/test/deploy (the
`@huggingface/transformers` + onnxruntime-web install is heavy and eats the run's budget).

Decision: **finish the recurring build instead of starting a new tool.** The `2026-07-02`
attempt was the most complete (full src, tests, README, LICENSE, styles) and already had
`node_modules` installed. I consolidated it into today's dated folder (reusing the existing
`node_modules` to skip the slow reinstall that had killed prior runs), verified, and shipped.

## Tool Details
- **Name:** Scribewell
- **Repo:** ben-gy/scribewell
- **Category:** media-tools (transcription)
- **Audience:** journalists transcribing confidential-source interviews, podcasters making
  captions, students transcribing lectures, researchers coding interviews — anyone who
  cannot upload a recording to a SaaS.
- **Stack:** Vite 6 + vanilla TypeScript + Vitest
- **Browser APIs:** @huggingface/transformers (Whisper ONNX), WebGPU (+ WASM fallback),
  WebAssembly (ONNX Runtime Web), Web Audio API (`decodeAudioData` + resample to 16 kHz
  mono), Web Workers + Transferable ArrayBuffer, Cache API, Clipboard API, Web Share API,
  Service Worker (offline).
- **Worker strategy:** single dedicated Web Worker owns the Whisper pipeline; the main
  thread decodes/resamples audio and hands samples over as a transferable Float32Array.

## Privacy Model
- **Protected:** the audio/video file and the resulting transcript — decoded and
  transcribed on-device, never uploaded. No account, cookies, analytics, or telemetry.
- **Not protected:** the one-time Whisper model-weight download from the Hugging Face CDN
  on first load (model data only; audio is never part of any request). GitHub Pages logs
  the initial page request like any site.
- **Trust surface:** the static bundle served by GitHub Pages, the TLS chain, and (first
  load only) the Hugging Face CDN.

## Architecture Decisions
- Reused the proven `2026-07-02` codebase wholesale rather than rebuilding — it was already
  tested and complete, and the repeated failures were interruptions, not defects.
- Copied `node_modules` locally instead of a fresh `npm install` to dodge the heavy-install
  timeout that had killed every prior attempt; then ran `npm install` once to reconcile the
  lockfile (`package-lock.json`) that CI's `npm ci` requires.
- Two Whisper checkpoints offered: Tiny English (`Xenova/whisper-tiny.en`, ~40 MB, default)
  and Base multilingual (`Xenova/whisper-base`, ~145 MB). q8 on WASM, fp16 on WebGPU.
- Linear-interpolation resampler (pure, unit-tested) instead of OfflineAudioContext so the
  audio math is testable and dependency-free.

## Test Results
- Tests written: 42 (across audio, formats/subtitles, segments, models)
- Tests passed: 42
- Tests failed: 0

## Build Status
- npm install: pass (reconciled lockfile; node_modules reused)
- npm test: pass (42/42)
- npm run build: pass (emits dist/; onnxruntime WASM ~21 MB, gzip ~5 MB)
- Local preview: pass (vite preview → HTTP 200, correct title/#app/modals/attribution)
- Production workflow dry-run: **not run** — no interactive browser was connected in this
  unattended run, so a live in-browser JS-render + full model-inference dry-run wasn't
  possible. All automated gates (unit tests, production build, served-page HTTP checks) pass.

## Deployment
- Repo created: yes (ben-gy/scribewell, private)
- GitHub Pages enabled: yes (workflow build type)
- Cloudflare DNS: yes (CNAME scribewell → ben-gy.github.io)
- Custom domain: **live** — https://scribewell.benrichardson.dev returns HTTP 200 over TLS
- GitHub Pages URL: https://ben-gy.github.io/scribewell/ (301 → custom domain)
- Deploy workflow: triggered on push to main, **completed successfully** (built commit
  8ed25c3)
- PR created: https://github.com/ben-gy/scribewell/pull/2

## Errors & Resolutions
- **Recurring non-deploy (root cause):** prior runs installed deps but were interrupted
  before deploy. Resolved by reusing the installed `node_modules` from the `07-02` attempt
  to eliminate the slow-install window.
- **Concurrent working-tree churn:** during this run, an orphan `src/types.ts` (and later
  `src/formats.ts`) variant appeared in the working tree — an incompatible message-type
  refactor that would have broken the build (it renamed the worker RPC types the committed
  `worker.ts` depends on). This looked like a second process racing in the same directory.
  Resolved by discarding the orphan changes (`git checkout -- src/types.ts`) and confirming
  `origin/main` was exactly my clean commit `8ed25c3` and that the successful Pages deploy
  built that SHA. The live site is unaffected.

---

## Update — verified rebuild (later same-day run)

A subsequent run of this task independently picked scribewell (the repo/registry did not yet
show it when that run started) and rebuilt it from scratch. That run's value-add was the two
things the initial deploy could not do:

1. **Real in-browser production dry-run (the gap flagged above).** Using the Claude Preview
   browser tooling against `vite preview`, a synthetic 16 kHz WAV was dropped and the full
   pipeline was observed end-to-end: audio decode → **WebGPU** device detection → Whisper
   model download (~75 MB, streamed with byte-accurate progress) → on-device inference →
   `result` stage. The non-speech tone correctly yielded `0 words · 0 segments`, proving the
   path without a canned transcript. Light/dark themes and a 375 px mobile layout were also
   verified.
2. **Fixed a shipping visual bug.** The initial version's `.modal-overlay { display:flex }`
   rule overrode the `[hidden]` attribute, so **all three modals rendered stacked open on
   first paint**. Added `.modal-overlay[hidden] { display:none }`. Confirmed via computed
   styles that all modals are hidden on load and open/close correctly.

Also added: a friendly "no speech detected" empty state for the zero-segment case, and an
`OfflineAudioContext` high-quality resample path (with the linear resampler kept as fallback).
This verified build was force-pushed over `main` (unrelated auto-generated history; no human
work lost) and is what now serves at https://scribewell.benrichardson.dev. New review PR:
https://github.com/ben-gy/scribewell/pull/4 (the earlier PR #2 was closed).

- Tests: 47/47 pass. Build: clean. Live: HTTP 200, correct `<title>`. Deploy workflow: success.
