# Build Log: Ditherwell
**Date:** 2026-07-27
**Status:** deployed

## Idea Source
IDEAS.md was empty, so this was a researched pick. Surveying the fleet, the last several builds
clustered hard on **live sensors** (noisewell, swatchwell, shakewell, voxwell) and the
**admin-console register** (encrypt / compress / convert / trace). Per the skill's principle #6
("useful is not the same as serious"), I deliberately chose a **playful-but-useful** tool that still
produces a real artefact and pushes genuine signal-processing logic: a retro image ditherer.

Googleable demand is high and specific — "floyd steinberg online", "image to 1 bit", "gameboy camera
filter", "dithering tool", "convert image for thermal printer", "pixel art converter". The user leaves
with a PNG. It's distinct from vectorwell (tracing), pixpress (compression) and embiggen (upscaling),
so not an expansion of an existing tool.

## Tool Details
- **Name:** Ditherwell
- **Repo:** ben-gy/ditherwell
- **Category:** image-tools
- **Audience:** zine makers, e-ink/thermal-printer users, indie game devs prepping sprites, retro-aesthetic hobbyists
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest
- **Browser APIs:** Web Workers, createImageBitmap + OffscreenCanvas, OffscreenCanvas.convertToBlob, File System Access, Clipboard (ClipboardItem), Web Share, Pointer Events + clip-path, Service Worker (PWA)
- **Worker strategy:** single dedicated worker; job-id multiplexing so a newer render supersedes an in-flight one

## Privacy Model
- **Protected:** image decoded, downsampled and dithered entirely on-device; never uploaded. Output PNG only leaves if the user downloads/copies/shares it.
- **Not protected:** GitHub Pages CDN sees the page request (never the image); exported files are then in the user's hands.
- **Trust surface:** static bundle + TLS to GitHub Pages; anonymous cookie-less Cloudflare Web Analytics beacon (no image data); feedback widget (only sends on explicit Send).

## Architecture Decisions
- **All DSP is first-party, no runtime libraries.** `dither.ts` implements error diffusion
  (Floyd–Steinberg, Atkinson, Jarvis–Judice–Ninke, Stucki) with serpentine scanning, ordered Bayer
  4×4/8×8 dithering, median-cut adaptive palette (2–32 colours), fixed palettes (1-bit, Game Boy DMG,
  CGA, grayscale, sepia), squared-Euclidean nearest-colour, and brightness/contrast. This keeps the
  bundle tiny (~34 KB JS) and makes the whole engine unit-testable without a DOM.
- **Ordered dithering to arbitrary palettes** uses a `paletteSpread()` heuristic — the average
  nearest-neighbour distance in the palette, converted to a per-channel amplitude — so the Bayer
  perturbation auto-adapts to mono vs a 32-colour palette.
- **Pixel-size model:** the source is downsampled by `pixelSize` (capped at 1400 px long side to
  bound error-diffusion cost), dithered, then nearest-neighbour upscaled back (`convertToBlob`) so the
  output reads as crisp square pixels at roughly the original size. `planSize()` is a pure, tested
  function.
- **Worker returns a ready PNG Blob** (encoded via `OffscreenCanvas.convertToBlob`) rather than raw
  pixels, simplifying the main thread — it just makes an object URL for the compare `<img>` and the
  download.
- Reused the vectorwell UI/eventlog/glossary/compare-slider shell for a consistent, polished look;
  amber CRT-phosphor accent to differentiate and suit the retro theme; light-first, dark via
  prefers-color-scheme.

## Test Results
- Tests written: 50 (dither DSP 29, imageutil/sizing 10, presets 11)
- Tests passed: 50
- Tests failed: 0
- Coverage highlights: every dithering algorithm asserted to emit only palette colours + preserve
  alpha; median-cut bounded by requested count and safe on all-transparent input; Bayer matrices
  distinct and in [0,1); kernel weights sum to divisor; planSize caps + aspect preserved; preset
  round-trip.

## Build Status
- npm install: pass
- npm test: pass (50/50)
- npm run build: pass (one TS fix — construct ImageData via ctx.createImageData rather than the
  Uint8ClampedArray<ArrayBufferLike> constructor overload)
- Local preview: pass (production dist served on a fresh port to avoid the shared-5199 stale-SW trap)
- Production workflow dry-run: pass

## Dry-run verification (local production preview, Browser pane)
- Sample loads through the real upload path → 1-bit Floyd–Steinberg dither produced (512×384, 2 colours, ~124 ms)
- Switched to Retro 16 → adaptive median-cut palette, 16 colours, dithered @ 256×192 (~176 ms)
- No console errors
- [hidden]/overlay guard: `elementFromPoint(centre)` returns `out-img` on both desktop (1280) and mobile (375); closed drawer sits fully off-screen (left = innerWidth); modal-host hidden
- Privacy modal labelled "Privacy", has Protected / Not protected / Trust surface incl. the analytics + feedback disclosures
- Modal and event-log drawer both close via their × button and Escape; verified the mobile drawer's × is reachable at 375 px
- Footer shows benrichardson.dev + lab backlink; hosted feedback widget self-mounts its "Feedback" link
- PWA manifest has name/start_url/scope and a maskable icon; apple-touch-icon + sample + third-party-notices all 200

_This is a file-input tool, not a live-sensor tool, so no device-sensor caveat applies._

## Deployment
- Repo created: yes (ben-gy/ditherwell)
- GitHub Pages enabled: yes (workflow build)
- Cloudflare DNS CNAME: created (ditherwell → ben-gy.github.io)
- TLS: cert_state approved; custom domain serves 200 over HTTPS; https_enforced finalising automatically
- PR created: https://github.com/ben-gy/ditherwell/pull/1
- Deploy workflow: completed successfully; https://ben-gy.github.io/ditherwell/ and https://ditherwell.benrichardson.dev both 200
- IndexNow pinged; lab hub deploy triggered

## Errors & Resolutions
- **TS2769 in worker.ts** — `new ImageData(Uint8ClampedArray, w, h)` failed under the current lib
  types (SharedArrayBuffer-vs-ArrayBuffer generic mismatch). Fixed by allocating with
  `ctx.createImageData(w, h)` and `.data.set(out)`.
- No other errors.
