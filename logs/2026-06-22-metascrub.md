# Build Log: metascrub
**Date:** 2026-06-22
**Status:** deploy_failed (built + fully verified locally; deployment blocked by missing GitHub auth)

## ⚠️ TOP-LEVEL BLOCKER — read first

**GitHub authentication is completely unavailable in this environment, so nothing could
be deployed this run.** This is the single most important finding.

- `gh auth status` → token for account `ben-gy` is **invalid**.
- `gh api user` → `HTTP 401: Requires authentication`.
- No `GH_TOKEN` / `GITHUB_TOKEN` env vars set.
- macOS keychain has **no** usable GitHub credential: `git ls-remote` against the
  **private** repo `ben-gy/dropwell` fails with *"could not read Username … Device not
  configured"*. (A public-repo `ls-remote` succeeds only because it needs no auth.)
- `gh repo create ben-gy/metascrub …` → `HTTP 401`.

Because this is a scheduled task, the user is not present to run `gh auth login`, which is
interactive. Per the task rules ("never halt", "producing a report is the correct output"),
I built and fully verified the tool locally, committed it on `main`, and left it ready to
deploy the instant auth is restored.

**To finish deployment once `gh auth login -h github.com` succeeds**, run Steps 9–13 of the
SKILL for both `tools/2026-06-22-metascrub` and `tools/2026-06-21-bgwipe` (repo create →
push → Pages → Cloudflare DNS → PR → index update). The metascrub repo is already
`git init`-ed and committed locally; just add the remote and push.

## Idea Source
IDEAS.md contained no actual ideas (only the documentation header). Researched/selected a
new idea against the criteria. Chose a **lossless EXIF/metadata scrubber & viewer** because:
- Strongest possible privacy narrative (photos leak home GPS) — core to this factory.
- Distinct from all existing tools: pdf-crush (PDF compression), dropwell (P2P transfer),
  bgwipe (ML background removal). Different domain, different browser-API pillar.
- Robust to deploy: no large model download, no COOP/COEP/SharedArrayBuffer headers (which
  GitHub Pages can't set), no runtime dependencies.
- Highly unit-testable via synthetic byte fixtures.
- Very Googleable: "remove exif online", "strip metadata from photo", "remove location
  from photo before posting".

## Tool Details
- **Name:** metascrub
- **Repo:** ben-gy/metascrub
- **Category:** encryption-privacy
- **Audience:** privacy-anxious sharer (marketplace seller, parent, journalist, lawyer)
  about to post a photo and worried about leaking location/device fingerprints.
- **Stack:** Vanilla TypeScript + Vite 6, vite-plugin-pwa, Vitest. Zero runtime deps.
- **Browser APIs:** File API + DataTransfer, ArrayBuffer/DataView byte parsing, Web Workers
  (ES module), Transferable objects, OffscreenCanvas + createImageBitmap, URL.createObjectURL,
  Clipboard API, Web Share API, Service Worker (PWA offline).
- **Worker strategy:** single dedicated ES-module worker owning scan + scrub + thumbnail;
  image bytes transferred zero-copy both directions.

## Privacy Model
- **Protected:** image never leaves the device; all parse/scrub in-tab; no analytics/cookies/
  fonts/telemetry; no account; works offline after load; removal is lossless (pixels copied
  byte-for-byte, no new fingerprint).
- **Not protected:** removes container metadata only — pixels (visible faces/signs) untouched;
  GitHub Pages/Cloudflare log the initial page load; HEIC unsupported (convert to JPEG first).
- **Trust surface:** static bundle pinned to commit; TLS to metascrub.benrichardson.dev; no
  third-party runtime code (hand-written parser ships in the bundle).

## Architecture Decisions
- **Vanilla TS, not React:** single linear workflow (drop → scan → scrub → download); no
  multi-pane state to justify React.
- **Hand-written parsers, zero runtime deps:** for a privacy tool the trust surface matters;
  shipping no third-party runtime code is itself a feature, and the JPEG/PNG/WebP container
  formats are simple enough to walk directly. Also keeps the bundle ~22 KB JS (~8 KB gzip).
- **Lossless segment surgery, not canvas re-encode:** the killer differentiator vs incumbents.
  We drop metadata byte-ranges and copy image data verbatim — JPEG entropy data, PNG IDAT,
  WebP bitstream — so quality is untouched. For WebP we also rewrite the RIFF size and clear
  the VP8X EXIF/XMP flag bits to keep the file valid.
- **Conservative-on-colour, aggressive-on-metadata policy:** JPEG keeps JFIF/ICC/Adobe
  markers (rendering + colour), strips EXIF/XMP/IPTC/MPF/COM/maker-notes.

## Test Results
- Tests written: 25 (2 files: `tests/scrub.test.ts`, `tests/exif.test.ts`)
- Tests passed: 25
- Tests failed: 0
- One bug found & fixed during testing: scrubbing a *truncated* JPEG threw RangeError
  (a metadata segment whose declared length ran past EOF). Fixed by bounds-checking the
  segment total against buffer length in `scanJpeg` before recording/removing it.
- Coverage: format detection (incl. empty/unknown), JPEG/PNG/WebP scan + lossless scrub,
  EXIF GPS/make/model decode, XMP-in-iTXt detection, idempotency (double-scrub removes
  nothing), already-clean detection, and malformed/truncated/empty inputs (no throw).

## Build Status
- npm install: pass
- npm test: pass (25/25)
- npm run build: pass (`tsc && vite build`; one TS5.7 fix: cast Uint8Array to BlobPart in
  worker.ts for the OffscreenCanvas thumbnail Blob). Output: dist/index.html 1.5 KB,
  index JS 22 KB (gzip 8.2 KB), worker 8.25 KB, CSS 11 KB (gzip 3 KB); PWA service worker
  + manifest generated.
- Local preview (vite preview): pass (HTTP 200)
- Production browser dry-run (Claude Preview MCP on built dist, port 5188): **pass** —
  - Page loads, no console errors.
  - Drop a synthetic JPEG (XMP + comment) → card shows "Metadata blocks (2) · save 69 B".
  - Click Scrub → "✓ Verified clean — removed 69 B … byte-for-byte identical otherwise";
    Download + Copy buttons wired; button reads "Scrubbed ✓".
  - Drop a synthetic JPEG with a real EXIF GPS TIFF → highlights render
    "📍 GPS location 51.5, -0.125 view on map ↗" + "📷 Camera Canon X9", with a correct
    OpenStreetMap link.
  - Threat-model modal opens with all 3 sections (Protected / Not protected / Trust surface);
    Escape closes it. Event drawer opens (7 events logged). Light + dark themes both apply.

## Deployment
- Repo created: **no** (gh auth 401)
- GitHub Pages enabled: no
- Cloudflare DNS: not attempted (repo doesn't exist yet)
- PR created: no
- Workflow triggered: no
- Local git: **yes** — `git init`, branch `main`, initial commit `96eae01`, ready to push.
- Public index (`index/tools.json` / `tools.txt`): **intentionally not updated** — those
  files drive live links on benrichardson.dev and would be broken until the sites actually
  deploy. Add metascrub (and bgwipe) to the index as part of the deployment finish-up.

## Side Finding: bgwipe (2026-06-21) is also orphaned & undeployed
`tools/2026-06-21-bgwipe` is a complete, tests-passing tool from a prior interrupted run
(in-browser background removal via RMBG-1.4 / WebGPU). It was never deployed or registered —
almost certainly the same auth blocker. I verified its tests pass and registered it in
`registry.json` as `deploy_failed` so it isn't lost. It should be deployed alongside
metascrub once auth is restored.

## Errors & Resolutions
1. **gh auth invalid (FATAL for deploy):** documented above; cannot self-resolve without an
   interactive `gh auth login`. Everything that doesn't need GitHub was completed.
2. **Chrome MCP not connected:** used the Claude Preview MCP instead for the browser dry-run.
3. **TS5.7 BlobPart type error in worker.ts:** cast `bytes as BlobPart`.
4. **RangeError on truncated JPEG:** added a segment-bounds check in `scanJpeg`.
