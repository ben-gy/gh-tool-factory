# Build Log: Redactwell
**Date:** 2026-07-28
**Status:** deployed

## Idea Source
Researched. `IDEAS.md` was empty, so I looked for a gap in the fleet. The registry has PDF tools for
signing (inkwell), page-ops (pagesmith), compression (pdf-crush) and OCR (textlift) — but **nothing
redacts**. "Redact PDF free" / "black out text in PDF permanently" is a high-demand query with a
razor-sharp privacy story: most online redactors just draw a black rectangle over selectable text
that anyone can copy from underneath (a recurring real-world leak), *and* they upload the sensitive
file. A client-side tool that (a) never uploads and (b) physically removes the text is a strong,
non-duplicate fit. Recent runs leaned on live sensors (tunewell, noisewell, shakewell, swatchwell), so
a document tool was also a good rotation.

## Tool Details
- **Name:** Redactwell
- **Repo:** ben-gy/redactwell
- **Category:** document-tools
- **Audience:** paralegals / journalists / HR / anyone sharing a sensitive PDF who needs a name or number *gone*, not covered
- **Stack:** Vite 6 + vanilla TypeScript + Vitest
- **Browser APIs:** pdf.js (render + text layer), Canvas 2D (burn-in), pdf-lib in a Web Worker (rebuild + strip metadata), Pointer Events, IntersectionObserver, File System Access, Web Share, Service Worker
- **Worker strategy:** pdf.js runs its own parse/render worker; page rasterisation happens on the main thread (pdf.js must render to a canvas) yielding between pages with determinate progress; the pdf-lib re-assembly runs in a dedicated module worker.

## Privacy Model
- **Protected:** the PDF is opened, redacted and rebuilt entirely on-device and never uploaded; redacted pages are flattened to images so the text under each black bar is physically removed; document metadata (author/title/producer) is dropped from the output.
- **Not protected:** GitHub Pages' CDN sees the request for the *page* (never the document); pages the user doesn't redact keep their original selectable text unless "flatten every page" is ticked; exported files are then in the user's hands.
- **Trust surface:** the static bundle, the TLS chain, a cookie-less Cloudflare Web Analytics beacon (the document is never sent to it), and the hosted feedback widget (sends only what the user types, on Send).

## Architecture Decisions
- **Vanilla TS**, not React — a single document view + a box overlay doesn't need component state orchestration.
- **Safe redaction by flattening.** Rather than attempting selective text-stream removal (fragile, error-prone with pdf-lib), any page carrying a redaction is rasterised whole and embedded as a flat image, which guarantees no text layer remains under the bars. Untouched pages are `copyPages`-ed verbatim to preserve quality/selectability; an opt-in "flatten every page" strips all text. This is the industry-standard safe client-side approach and its guarantee is testable.
- **Find & redact** uses pdf.js `getTextContent` + `Util.transform` to place a reviewable box over each match (proportional horizontal slicing for axis-aligned text, full-item AABB otherwise). Boxes are reviewed before export, so a slightly-loose box over-covers rather than under-covers — the safe failure mode.
- **Boxes stored in normalized 0..1 coordinates** relative to the rotated (scale-1) viewport, so drawing and export share one orientation-agnostic representation.
- **Draw-vs-Scroll mode toggle** so `touch-action:none` (needed to draw) doesn't block scrolling on a phone.
- **rAF shim** (native when visible, setTimeout when `document.hidden`): pdf.js schedules render continuations on requestAnimationFrame, which browsers throttle/pause in a backgrounded tab — the shim keeps an in-progress export rendering if the user switches tabs. (Discovered during the preview dry-run; kept as a genuine production hardening.)

## Test Results
- Tests written: 23 (redact geometry/planning core + formatBytes)
- Tests passed: 23
- Tests failed: 0
- Covered: drag→box normalization + clamping, meaningful-box threshold, pad/over-cover, box→pixel mapping, point-in-box, flatten planning, box counting, DPI→scale, output-name derivation, %PDF magic, case-insensitive non-overlapping search ranges.

## Build Status
- npm install: pass
- npm test: pass (23/23)
- npm run build: pass
- Local preview: pass
- Production workflow dry-run: pass — full end-to-end in the in-app browser: loaded the sample, used Find & redact on two names (5 boxes), exported a 214 KB redacted PDF, then **reopened the exported file and searched for the redacted names → 0 matches** (the text is genuinely gone). Also confirmed the production rAF shim lets an export complete with the tab hidden. Modals (How / Privacy w/ beacon disclosure), event drawer (× + Esc at 375px), maskable + opaque apple-touch icons, no console errors, mobile layout.

## Deployment
- Repo created: yes (ben-gy/redactwell)
- GitHub Pages enabled: yes (workflow build)
- DNS: Cloudflare CNAME redactwell → ben-gy.github.io created
- TLS: https_enforced=true (cert live after one CNAME cycle)
- Live: https://redactwell.benrichardson.dev (200), sample + third-party-notices served (200)
- PR created: https://github.com/ben-gy/redactwell/pull/1
- Workflow triggered: yes (Deploy to GitHub Pages — success)

## Errors & Resolutions
- **Stale service worker on `localhost:5199`** — the shared preview port served a previous tool's cached shell (title showed "Tunewell", then "Squirm"). Resolved by running the preview on a fresh port (5233/5244), the reliable escape from the shared-SW pollution.
- **pdf.js render stalled during export when the preview pane was hidden** — background tabs throttle rAF, on which pdf.js schedules render continuations. Verified the cause, then added `src/raf-shim.ts` (native rAF when visible, setTimeout when hidden) as a production fix and re-verified export completes with `document.hidden === true`.
- **Offline gap** — the pdf.js worker is a `.mjs` file and wasn't matched by the initial `globPatterns`, so it wouldn't be precached (breaking offline PDF opening). Added `mjs` to the workbox glob so the ~1.4 MB worker is precached for genuine offline use.
