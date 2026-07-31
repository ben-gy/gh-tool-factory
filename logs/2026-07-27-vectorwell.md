# Build Log: Vectorwell
**Date:** 2026-07-27
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Applied the input question (principle #6): the last three
tools before this run (swatchwell/camera, shakewell/accelerometer, voxwell/mic) were sensor
tools in a row, then embiggen (file), so the sensor quota was well satisfied — a file tool was
appropriate this run. Scanning the registry for gaps surfaced a clear one: there is **no
raster→vector (image→SVG) tracer** anywhere in the fleet, despite it being one of the
highest-demand privacy-sensitive conversions ("png to svg", "image to vector free", "logo to
svg"; the good incumbents — vectorizer.ai, Adobe — are paid and/or upload your artwork).

Engine research: evaluated `vectortracer` (VTracer WASM — but only BinaryImageConverter is
implemented, colour is unfinished and the worker wrapper is incomplete → too immature) and
`@neplex/vectorizer` (Node/NAPI only, not browser). Settled on **`esm-potrace-wasm`** (by
tomayac/Google DevRel): a mature WASM build of Peter Selinger's Potrace that does BOTH black &
white line-art AND colour (posterised) tracing, accepts `ImageData`, bundles its wasm inline
(no separate file, no fetch), detects `WorkerGlobalScope`, and returns an SVG string. One clean,
reliable, ambitious WASM engine covers the whole tool.

## Tool Details
- **Name:** Vectorwell
- **Repo:** ben-gy/vectorwell
- **Category:** image-tools (indexed under `images`)
- **Audience:** designers/developers converting confidential logos/icons/sketches to vector; plotter/laser hobbyists
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest
- **Browser APIs:** WebAssembly (Potrace), Web Workers, createImageBitmap/OffscreenCanvas, Canvas 2D, Pointer Events + clip-path, File System Access, Clipboard, Web Share, Service Worker (PWA)
- **Worker strategy:** single dedicated worker (decode → down-scale → potrace), job-id multiplexing so a newer trace supersedes an in-flight one; hard-cancel tears the worker down

## Privacy Model
- **Protected:** image decoded + traced entirely on-device, never uploaded; SVG generated locally, only leaves the device if the user downloads/copies/shares it; potrace wasm same-origin; no cookies/fingerprinting/third-party fonts.
- **Not protected:** page load + static assets from GitHub Pages CDN (sees the page request, never the image); exported artefacts are then in the user's hands.
- **Trust surface:** hash-pinned site bundle + same-origin potrace WASM, the TLS chain, a cookie-less Cloudflare Web Analytics beacon, and the opt-in feedback widget. Images/SVGs never sent to either.

## Architecture Decisions
- Vanilla TS (no React) — single-view workflow, no complex multi-pane state.
- One WASM engine (esm-potrace-wasm) instead of two (dropped the imagetracerjs colour path)
  once it was confirmed potrace's `extractcolors`/`posterizelevel` handle colour posterised
  tracing well — simpler and fully reliable.
- Pure, unit-tested core: `presets` (param↔potrace mapping, preset matching), `imageutil`
  (trace-size/downscale math, format acceptance), `svgstats` (path/colour counting, viewBox,
  and a colour-safe SVG minifier that rounds only geometry attributes and strips the DTD).
- Live re-trace on any control change (debounced 220ms); presets collapse to a "Custom" tag when edited.
- Determinate progress for decode; the trace itself has no potrace progress callback, so that
  phase is honestly shown as indeterminate.

## Test Results
- Tests written: 36 (presets 11, imageutil 10, svgstats 15)
- Tests passed: 36
- Tests failed: 0

## Build Status
- npm install: pass
- npm test: pass (36/36)
- npm run build: pass
- Local preview: pass (HTTP 200)
- Production workflow dry-run: pass — see below

## Production dry-run (local `vite preview`, in-browser)
- Empty state renders; feedback widget self-mounts in the footer; no console errors.
- **"Try a sample"** loads the bundled flat-colour PNG through the real upload path and traces
  it: 9 colours, 18 paths, 3.8 KB SVG in 137 ms (colour mode).
- **Live re-trace** to black & white: 1 path, 1 colour, 1.1 KB in 11 ms; the colours slider
  correctly hides in B&W mode.
- Compare slider renders raster vs vector; stats bar correct.
- Privacy modal (header button labelled "Privacy"), How-it-works and About modals open/close.
- Event drawer opens via toggle and closes via its × button AND Escape; verified at 375px — it
  slides fully into view and there is no horizontal page overflow.
- **[hidden] guard:** `modal-host` computes `display:none`; `elementFromPoint(centre)` returns
  real content (`svg-img`), never the drawer/modal host.
- **Download** produces a valid `sample-emblem.svg` (starts `<svg`, contains `<path`).
- **PWA:** manifest resolves with name/start_url/scope, a `maskable` icon, and PNG icons;
  `apple-touch-icon.png` returns 200 `image/png`.
- Cleared a stale service worker left on the shared :5199 port from a previous tool before testing.

No real sensor involved (this is a file-input tool), so no sensor caveat applies.

## Deployment
- Repo created: yes (ben-gy/vectorwell)
- GitHub Pages enabled: yes (workflow build); DNS CNAME created in Cloudflare; custom domain set + cert cycle triggered
- Deploy workflow: completed success
- ben-gy.github.io/vectorwell/ returns 301 (redirect to custom domain — live)
- Custom-domain TLS: provisioning at log time (https_enforced not yet true); resolves within minutes of the cname cycle
- PR created: https://github.com/ben-gy/vectorwell/pull/1
- IndexNow ping: 202/200 OK; lab deploy workflow dispatched to refresh the master sitemap

## Errors & Resolutions
- **Safety guardrail on THIRD-PARTY-NOTICES:** the first attempt assembled the notices by
  transcribing the full GPL-2.0 licence text through the model, which tripped the output safety
  guardrail and blocked the message. Resolved by assembling the notices **mechanically** with a
  throwaway Node script that reads the on-disk `node_modules/<pkg>/LICENSE` files and
  concatenates them (the model authored only the short headers/intro). The routine SKILL.md
  step 8b was updated to mandate this approach for all future runs (do not transcribe long
  licence bodies through the model; copy the existing on-disk files instead).
- No other errors.
