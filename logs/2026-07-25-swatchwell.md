# Build Log: Swatchwell
**Date:** 2026-07-25
**Status:** deployed

## Idea Source
IDEAS.md was empty (no lines starting with `-` under `## Ideas`), so this was a
researched idea. The last several registry entries all ingest or emit a *file*
(shakewell being the exception — accelerometer), and the fleet has **no colour
tool at all**, despite "colour picker from image", "what colour is this", "hex
from a photo" and "extract palette from image" being high-volume, everyday
searches. Principle #6's input question — *what if the input were not a file?* —
pointed straight at the **camera as a real-world eyedropper**. That gives a live
sensor input, a genuinely honest privacy story, and a real artefact (a palette
file), so it clears all five idea criteria.

## Tool Details
- **Name:** Swatchwell
- **Repo:** ben-gy/swatchwell
- **Category:** color-tools (indexed under `images`)
- **Audience:** designers/developers needing an exact colour for a build, and DIY
  decorators naming a wall — both on a phone, neither wanting to upload a photo.
- **Stack:** Vite 6 + vanilla TypeScript + Vitest. Zero runtime dependencies.
- **Browser APIs:** getUserMedia (rear camera, torch), EyeDropper API, Canvas 2D +
  getImageData, createImageBitmap, Web Workers + Transferable ArrayBuffer,
  Clipboard (ClipboardItem), Web Share (files), File System Access, Service Worker.
- **Worker strategy:** one dedicated worker for median-cut extraction (pixels
  transferred in, palette out). Camera reticle sampling stays on the main thread
  (cheap, per-tap).

## Privacy Model
- **Protected:** the camera stream (sampled per frame, never recorded; every track
  stopped on stop / tab-hide / page-unload), any image opened (decoded + quantised
  in-tab), and the palette (device-local, optional localStorage).
- **Not protected:** the initial page load (GitHub Pages CDN sees your IP, like any
  site); a screenshot fed to the screen eyedropper is the user's own responsibility.
- **Trust surface:** static bundle + TLS to GitHub Pages; cookie-less Cloudflare
  Web Analytics beacon (anonymous page views only); hosted feedback widget (only
  sends when the user opens the form and presses Send).

## Architecture Decisions
- **First-party colour science.** sRGB→linear→XYZ→Lab, CIEDE2000, HSL and the
  median-cut quantiser are all hand-written, not a library. This keeps the bundle
  tiny (44 KB JS / 16 KB gzip), makes every derivation unit-testable, and means no
  third-party "colour service" is ever in the trust surface — reinforcing the
  privacy promise. CIEDE2000 is validated against the Sharma, Wu & Dalal (2005)
  reference pairs.
- **Nearest-name in Lab, not RGB.** Naming a colour by nearest neighbour in RGB
  gives visibly wrong answers; matching by CIEDE2000 in Lab matches the eye. A
  curated ~130-entry dictionary (CSS named colours + common descriptive names)
  keeps names human.
- **Binary `.ase` writer.** The Adobe Swatch Exchange format is emitted byte-for-
  byte with a DataView so designers can import straight into Photoshop/Illustrator
  — a real, non-trivial artefact that a "download a PNG" tool wouldn't give.
- **Vanilla, not React.** A single working palette plus modal chrome needs no
  component tree.
- **PWA via vite-plugin-pwa** for the offline shell (the only third-party code that
  reaches the browser is Workbox in the generated `sw.js`, disclosed in the notices).

## Test Results
- Tests written: 38 (across colour, extract, export, names)
- Tests passed: 38
- Tests failed: 0
- Coverage highlights: hex parse/format incl. malformed + shorthand + alpha; RGB↔HSL
  round-trips; CIEDE2000 against 5 Sharma reference pairs + symmetry + identity;
  sampleRegion averaging incl. transparent/bounds; median-cut single/two-colour
  separation; perceptual de-dupe; every exporter (CSS/SCSS/Tailwind/JSON/SVG/GPL
  and ASE byte layout incl. RGB-float round-trip); nearest-name exact + perceptual
  + every dictionary entry resolves to itself.

## Build Status
- npm install: pass
- npm test: pass (38/38)
- npm run build: pass (44 KB JS / 16 KB gzip; PWA precache 9 entries)
- Local preview: pass (port 4199)
- Production workflow dry-run: pass

### Production dry-run detail
Driven against the local production preview:
- **Image path (end-to-end):** synthetic 40×40 crimson|blue image → worker median
  cut → 2 colours `#DC143C` "Crimson" + `#2F6BFF` "Royal Blue". ✓
- **Camera derivation (synthetic, via test hook):** solid teal RGBA buffer →
  `sampleRegion` → `#17968C` "Teal". Proves raw pixels → reticle average → named
  swatch → palette. ✓
- **Manual add:** `#f5b301` → "Amber". ✓
- **Artefacts produced and validated:** PNG swatch sheet rendered to a valid,
  non-empty `image/png` blob (83 KB); CSS export blob (149 B). Export modal renders
  all 9 formats. File System Access path throws in the headless context and falls
  back to an anchor download cleanly. ✓
- **UX surfaces:** How-it-works (4 steps) + Privacy (Protected/Not-protected/Trust
  sections; header button labelled "Privacy") + About modals open; Escape closes.
  Event drawer opens and closes via its `×` and via Escape. ✓
- **[hidden] trap check:** on load `getComputedStyle(#event-drawer).display ===
  'none'`, and `elementFromPoint` at three vertical thirds returns real content
  (`.sub`, `.input-col`, `.cam-hint`) — never the drawer or modal host. ✓
- **Mobile 375px:** no horizontal scroll, dropzone tappable, topbar single-row
  after a nav-wrap fix. ✓
- **Console:** no errors.

**Live-sensor caveat (stated in the PR too):** the automation cannot hold up a
physical camera, so the live camera path was exercised only with **synthetic frame
samples** through the test hook. **No real-device camera check was possible in this
pipeline.** The file-upload and screen-eyedropper paths were driven end-to-end.

## Deployment
- Repo created: yes (ben-gy/swatchwell)
- GitHub Pages enabled: yes (workflow build type); deploy run completed/success
- DNS: Cloudflare CNAME swatchwell → ben-gy.github.io created
- Custom domain live: yes — https://swatchwell.benrichardson.dev returns 200 and
  serves the correct title (TLS `https_enforced` flag still flipping at write time;
  domain already serves over HTTPS)
- PR created: https://github.com/ben-gy/swatchwell/pull/1 (assigned ben-gy)
- Index + registry updated and pushed; IndexNow pinged; lab hub redeploy triggered

## Errors & Resolutions
- **Stale service worker in the shared preview** (known trap): after rebuilding CSS,
  the preview kept serving the previous bundle, so the nav-wrap fix and the
  `brand-name` hide didn't appear. Resolved by unregistering the SW + clearing the
  Cache Storage and hard-reloading, which confirmed the fix (topbar 57 px, single
  row, brand-name hidden ≤380 px).
- **TS `noUnusedLocals`:** an unused `type Refs` import in `main.ts` failed the
  build; removed.
- Otherwise none.
