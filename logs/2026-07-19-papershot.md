# Build Log: Papershot
**Date:** 2026-07-19
**Status:** deployed

## Idea Source

Researched. `IDEAS.md` contained no queued ideas, so the space was surveyed against `registry.json` for gaps. The catalogue already covers images (pixpress, unheic, bgwipe, blurwell, metascrub), PDFs (pdf-crush, pagesmith, inkwell, textlift), media (gifsmith, scribewell, screenwell, clipwell, vidwell), crypto/privacy (sealbox, veilpix, tokenpeek, verisum, dropwell), archives (zipsafe) and dev utilities (qrforge).

The clear gap: **nothing captures from a camera to produce a document**. qrforge scans QR codes and screenwell records the screen, but no tool turns a photograph of a page into a scan. That is a different input modality, a different problem domain, and a different central browser capability — so a new tool rather than an `EXPANSION_IDEAS.md` entry.

A web search confirmed the space is dominated by **jscanify** (which requires an ~8 MB OpenCV.js WASM build) and commercial SDKs (Dynamsoft, Scanbot). Writing the four required operations directly — blur, gradient, contour/hull, `warpPerspective` — is genuinely differentiated: it keeps the bundle at ~200 KB and makes the pipeline unit-testable.

Googleable as: *"scan document to pdf with phone camera free"*, *"remove shadow from photo of document"*, *"photo to pdf scanner no upload"*.

## Tool Details
- **Name:** Papershot
- **Repo:** ben-gy/papershot
- **Category:** document-tools
- **Audience:** Consumer, usually on a phone — someone asked to "just send a scan" who doesn't own a scanner, often handling a document with a signature or an ID number on it. Light, warm, spacious UI (paper-and-ink palette, single teal accent) rather than the dark treatment used for the developer-facing tools.
- **Stack:** Vite 6 + vanilla TypeScript + Vitest
- **Browser APIs:** getUserMedia (+torch), Web Workers, Transferable ArrayBuffer, createImageBitmap, Canvas 2D, Pointer Events, pdf-lib, File System Access, Web Share, Clipboard, Service Worker
- **Worker strategy:** One dedicated module worker holding the source RGBA. It never touches a canvas — raw pixels in, raw pixels out — which keeps every function it calls pure and testable in Node.

## Privacy Model
- **Protected:** Photos and PDF never leave the device (no upload endpoint). Camera track stopped on capture/cancel/Escape/pagehide. Exports rebuilt from raw pixels, so EXIF and GPS don't survive. **PDF carries no timestamps** — pdf-lib writes `CreationDate`/`ModDate` by default, which would record exactly when a document was scanned; suppressed via `create({ updateMetadata: false })`. Nothing persisted: no IndexedDB, no OPFS, no history; `localStorage` holds UI preferences only.
- **Not protected:** The original photo remains in the camera roll with EXIF intact. Black & white discards colour (a stamp or highlighter can be flattened away). Scanning is not redacting. Detection can miss a corner on a page whose border is in deep shadow.
- **Trust surface:** GitHub Pages static bundle, the TLS chain, and the cookie-less Cloudflare Web Analytics beacon. No third-party requests at all at runtime.

## Architecture Decisions

**Vanilla TS, not React.** State is a flat page list plus one editor selection — nowhere near enough to justify a component tree.

**No OpenCV.** The decisive call. `src/cv/` is ~600 lines with zero dependencies:

| Module | Contents |
|--------|----------|
| `gray.ts` | Luma, box-average downscale, integral image, O(1) box mean, histogram percentile |
| `geometry.ts` | Convex hull (monotone chain), max-area inscribed quad, corner ordering, quad shrink/scale/clamp |
| `detect.ts` | Scharr magnitude → bounded threshold → outer edge points → hull → quad |
| `homography.ts` | 8×8 Gaussian elimination with partial pivoting for the exact 4-point transform |
| `warp.ts` | Inverse-mapped bilinear perspective warp with progress callback |
| `enhance.ts` | Shadow removal by division, percentile contrast stretch, Bradley–Roth adaptive threshold |

**PNG for black & white, JPEG otherwise.** Bilevel output compresses far better losslessly and avoids JPEG ringing around letterforms. Measured on the test document: 39 KB PNG vs 91 KB JPEG for the same page.

**Per-page settings live in the editor, not the export panel.** Encoding happens at "Add page", so a quality slider on the result screen would have been misleading — it couldn't affect already-encoded pages without keeping full-resolution ImageData for every page in memory.

## Test Results
- Tests written: **158** across 10 files
- Tests passed: **158**
- Tests failed: 0

Two initial failures were **wrong test expectations, not code bugs**, and both were corrected once the underlying maths was worked through by hand:
1. A homography test asserted a symmetric keystone's centre would shift; it doesn't — the quad chosen was symmetric about its own midline. Replaced with an asymmetric taper where the projected centre is provably at y = 500/7, plus a direct assertion that the matrix's bottom row is non-zero.
2. A shadow-removal test compared post-`contrastStretch` ranges, which are normalised to full scale in both branches and therefore always equal. Replaced with the assertion that actually matters: with a strong lighting gradient, ink at the lit end is *brighter* than paper at the shadowed end unless shadow removal runs.

## Build Status
- npm install: pass
- npm test: pass (158/158)
- npm run build: pass
- Local preview: pass
- Production workflow dry-run: pass — synthetic photographed document → detection → flatten → clean → valid multi-page PDF, run against both the local production build and the live site

## Bugs Found by Browser Verification

Three real defects that the unit tests did not catch, each fixed and then pinned by a regression test:

1. **Dense page text outvoted a shadowed page border.** With a global percentile threshold, a densely printed page produces far more and often stronger edges than its own border. With one corner in shadow, that border stopped qualifying — only three corners were found, and the flattened output came out rotated 90°. Two changes: `outerEdgePoints` keeps only the first and last qualifying pixel of each row and column (a letterform can never be the leftmost edge pixel of a row that also crosses the paper's edge, so interior content is excluded *structurally*), and the adaptive threshold is now capped at 22 % of the frame's peak so it can never climb into the text's own distribution. Verified across shadow strengths 0.0/0.30/0.45: max corner error fell from 38.9 % of frame width to 1.26 %.

2. **Output orientation used the wrong rule.** `orderCorners` started the corner cycle at the corner nearest the origin. For a page lying at an angle on a desk that is frequently not the top-left corner, and the scan lands sideways. It now starts at the rotation whose leading edge runs most nearly left-to-right — the top edge of a clockwise cycle by definition.

3. **`[hidden]` did not hide.** `.editor { display: grid }` silently beat the user-agent `[hidden]` rule, so the editor panel rendered beneath the intake panel on first paint (visible in the first mobile screenshot as stray corner handles below the drop zone). Fixed with an early `[hidden] { display: none !important }`. Page height dropped from 1676 px to 878 px. *This is the same trap recorded in the blurwell build.*

Two further quality fixes:

4. **The exported PDF was recording the scan time.** pdf-lib writes `CreationDate` and `ModDate` on save; for a tool whose promise is that the document stays private, silently stamping when it was scanned is the wrong default. Now suppressed. Found while writing an integration test — and that test initially gave a *false pass*, because the info dictionary sits inside a compressed object stream so grepping the raw bytes finds nothing either way. The assertion now re-loads with `updateMetadata: false`, since pdf-lib's own loader otherwise rewrites the metadata as it parses.

5. **A dark rim of desk around every scan.** Detected corners land a few pixels *outside* the paper — the pre-blur smears the border and `outerEdgePoints` then deliberately takes the outermost pixel of that smear. Corners are now pulled 2.5 % toward the centre. Measured on the synthetic worst case: dark-rim coverage of the output border fell from 66.6 % to 24.8 %, mean border brightness rose from 111 to 198, and all 198 text rows were retained.

## Deployment
- Repo created: yes (name re-checked immediately before `gh repo create`, per the concurrent-run guard)
- GitHub Pages enabled: yes (build_type=workflow)
- Cloudflare DNS: yes — CNAME `papershot` → `ben-gy.github.io`, unproxied
- TLS certificate: yes — issued after one CNAME cycle; `https_enforced=true`
- PR created: https://github.com/ben-gy/papershot/pull/1
- Workflow triggered: yes — completed successfully
- Live site verified: https://papershot.benrichardson.dev returns 200, full scan-to-PDF flow exercised in the browser against production, no console errors, service worker registered, all SEO assets (og.png, robots.txt, sitemap.xml, manifest, icons, IndexNow key) serving 200

## Errors & Resolutions

- **`preview_start` could not find the tool.** The harness reads `.claude/launch.json` from the session working directory (a worktree), not the tool directory, and runs `npm --prefix` relative to that root. Resolved by adding a `papershot` entry with an absolute prefix.
- **Stale service worker served old builds during verification.** Successive tools reuse port 5199, so a `vidwell-shell-v1` cache was still installed on `localhost:5199` and kept serving previous bundles — which initially made a genuine CSS fix look like it hadn't worked. Unregistering the SW and clearing caches between rebuilds was necessary throughout. Not a production concern (each tool has its own origin), but a real hazard for local verification.
- **TypeScript `Uint8ClampedArray<ArrayBufferLike>` rejected by the `ImageData` constructor.** Pinned the worker result type to `Uint8ClampedArray<ArrayBuffer>`; same fix for the base64 helper in the PDF test.
- **jsdom's `Blob` has no `arrayBuffer()`.** Added `tests/setup.ts` to shim it via `FileReader`, rather than stubbing the method the code under test depends on.
