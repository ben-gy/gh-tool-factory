# Build Log: Noisewell
**Date:** 2026-07-27
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty. Applied principle #6's "what if the input were
not a file?" prompt: the last two runs (vectorwell, embiggen) were both
file-in/file-out image tools, a strong signal to reach for a live sensor. The mic
had one prior tool (voxwell, a voice modulator) but no *measurement* tool. A
sound level meter — "decibel meter online" / "sound level meter" — is a real,
high-volume query with a genuine gap for a privacy-first web version (results are
almost all app-store downloads). It clears every gate: zero backend, a live-mic
sensor input, a privacy story with teeth (a meter, not a recorder), and — the
line that separates a tool from a toy — the user leaves with an artefact: an
exportable noise report (chart PNG + CSV + summary numbers).

## Tool Details
- **Name:** Noisewell
- **Repo:** ben-gy/noisewell
- **Category:** sensor-tools (indexed as live-dashboards, matching shakewell)
- **Audience:** anyone needing to put a number on how loud something is — a
  tenant documenting a noisy neighbour, a parent at a gig, someone checking an
  office/AC/fan against a comfortable level.
- **Stack:** Vite 6 + vanilla TypeScript, zero runtime dependencies.
- **Browser APIs:** getUserMedia (live mic sensor), Web Audio (AnalyserNode
  time-domain tap + OfflineAudioContext decode), Web Workers, Canvas 2D, File
  System Access, Clipboard, Web Share, Service Worker (PWA).
- **Worker strategy:** live path uses AnalyserNode on the main thread (cheap,
  native) driven by a setInterval tick (hidden-tab safe); the uploaded-file path
  runs frame-by-frame FFT in a dedicated Web Worker.

## Privacy Model
- **Protected:** mic audio reduced to numbers slice-by-slice in memory and
  discarded — never recorded, saved or uploaded; uploaded files decoded/analysed
  locally; a dB time-series is only assembled while recording and stays on-device
  until exported; no cookies/fingerprinting/accounts.
- **Not protected:** absolute accuracy — a browser can't read the mic's hardware
  gain, so readings are a *relative estimate* with a calibration offset; not a
  certified meter, not for legal/occupational compliance. Initial page load is
  served by GitHub Pages (sees IP).
- **Trust surface:** static bundle + TLS to GitHub Pages; anonymous cookie-less
  Cloudflare Web Analytics beacon; feedback widget (only on Send).

## Architecture Decisions
- **No runtime library.** The FFT (radix-2 Cooley–Tukey), IEC 61672 A/C-weighting
  curves, Leq/Ln statistics, Fast/Slow ballistics and 1/1-octave banding are all
  first-party, which keeps the bundle tiny (~18 KB gzipped JS) and the privacy
  story clean.
- **Level anchored to broadband RMS; weighting applied as a spectral
  *correction*.** The absolute level comes from the frame's RMS (robust); A/C
  weighting is the dB difference between the weighted and flat total power, so FFT
  window/normalisation constants cancel in the ratio. One pure `analyseFrame` is
  the single code path for both the live meter and the file worker — so the part
  the automation can't drive with a real mic is exactly the part that is
  unit-tested hard.
- **Default calibration offset of +90 dB** so out-of-the-box numbers read like
  plausible SPL, with the UI and Privacy modal explicit that it is an estimate and
  adjustable. "Calibrated" in the report means the user changed the offset.
- **Vanilla TS** — a four-state workflow (start → live → analyzing → report) is
  simple enough not to need React.

## Test Results
- Tests written: 61 (dsp: 33, analysis: 9, report: 8, format: 10, platform: 1)
- Tests passed: 61
- Tests failed: 0
- Coverage: weighting curves vs IEC reference values, FFT, RMS/detrend/Hann,
  analyseFrame (RMS dBFS, Z=no-correction, +20 dB for 10× amplitude), LevelSmoother
  convergence + tau ordering, Leq energy-weighting, Ln percentile ordering,
  analyseRecording on synthetic tones (flat series, offset application, octave
  localisation, progress), CSV/summary builders, formatting.
- One initial test encoded a wrong assumption (a 100-sample buffer returning [])
  and was corrected to the real contract.

## Build Status
- npm install: pass
- npm test: pass (61/61)
- npm run build: pass (worker split out; 24 PWA precache entries, ~348 KiB)
- Local preview: pass
- Production workflow dry-run: pass — GitHub Actions "Deploy to GitHub Pages"
  completed with conclusion=success.

## Deployment
- Repo created: yes (ben-gy/noisewell)
- GitHub Pages enabled: yes (workflow build type)
- DNS: Cloudflare CNAME noisewell → ben-gy.github.io created; CNAME cycled to
  trigger the TLS cert.
- Live: https://ben-gy.github.io/noisewell/ serves 200. Custom domain
  https://noisewell.benrichardson.dev was still propagating (DNS/cert) at
  hand-off — expected within the hour, matching prior tools.
- PR created: https://github.com/ben-gy/noisewell/pull/1

## Verification notes (live sensor)
No real-device microphone check was possible in this pipeline — the automation
cannot make a sound. The live sensor path was verified only through synthetic
frames via a test hook (a −20 dBFS 1 kHz tone read 67.0 dB(A); injected
500/2k/4k Hz tones localised to the correct octave bands) and through the
file-upload path, which was driven end-to-end via "Try a sample" to a full report
artefact (Leq 67.3 dB(A), chart + octave spectrum + CSV). The reported dB numbers
are therefore not from a real acoustic source. Also verified: drawer ×/Escape
close, modals, [hidden] guard (centre of viewport returns real content, drawer
display:none on load), mobile 375px (no horizontal scroll), PWA manifest with
maskable + 192/512 icons, apple-touch-icon 200/opaque, sample precached.

## Errors & Resolutions
- Preview browser served a stale service worker from a previous tool on the shared
  localhost:5199 — unregistered the SW + cleared caches, hard-reloaded (a known,
  recurring trap).
- Browser pane policy blocks arbitrary localhost ports; had to use preview_start
  on the allowlisted :5199 rather than an alternate port.
- Fixed a real bug found in review: showReport always hid the Share button, so it
  never appeared even where Web Share works — now gated on navigator.share.
- Minor cosmetic: end-of-scale labels on the live level bar clipped at the canvas
  edges; aligned edge labels inward.
