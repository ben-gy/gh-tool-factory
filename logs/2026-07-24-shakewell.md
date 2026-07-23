# Build Log: Shakewell
**Date:** 2026-07-24
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty, so I applied the principle-#6 "input question": scanning the last
several registry entries showed the fleet has mic tools (voxwell) and camera tools (papershot, qrforge,
blurwell) but **zero gyroscope/accelerometer tools** — a whole sensor category the guidance explicitly
names (spirit level, seismograph, vibration analyser). The last sensor build was voxwell (mic, 07-23),
so reaching for motion this run was well-aligned with the ~1-in-3-4 sensor target. Chose a **vibration
analyser**: accelerometer → FFT → dominant frequency/RPM. It clears all five idea criteria — zero
backend, an essential non-trivial API (DeviceMotion + a hand-written FFT), a strong privacy narrative
(a live sensor stream is the most literal "your data never leaves the device"), real Googleable demand
("measure vibration with phone", "washing machine vibration app", "phone seismograph"), and a concrete
artefact (a frequency/RPM number, a spectrum PNG, a recording CSV).

## Tool Details
- **Name:** Shakewell
- **Repo:** ben-gy/shakewell
- **Category:** sensor-tools (index category: live-dashboards)
- **Audience:** DIY-ers / makers with a vibrating appliance or motor; plus the pocket-seismograph curious
- **Stack:** Vite 6 + vanilla TypeScript + Vitest; no runtime dependencies
- **Browser APIs:** DeviceMotion + iOS `requestPermission`, first-party radix-2 FFT / Hann window /
  resampler, Web Worker, Canvas 2D, File/Blob, Clipboard, Web Share, Service Worker
- **Worker strategy:** dedicated worker for large-CSV parse + full-recording FFT; lightweight live
  windowed FFT (~4×/s) on the main thread; setInterval render loop (a hidden tab throttles rAF)

## Privacy Model
- **Protected:** the accelerometer stream is processed in-frame and discarded; nothing recorded until
  Record is pressed; no sample, recording, CSV or image ever uploaded — every FFT runs on-device.
- **Not protected:** the initial page load is an ordinary GitHub Pages request (sees IP/UA); exported
  files the user then sends are on the user.
- **Trust surface:** the static bundle + its TLS; the Cloudflare Web Analytics beacon (anonymous page
  views, no cookies/fingerprinting); feedback.benrichardson.dev only if the user presses Send.

## Architecture Decisions
- **Vanilla TS, no React** — a single input→process→output workflow, no multi-pane state.
- **First-party FFT** rather than a library — keeps the bundle dependency-free (52 kB JS / 18 kB gzip)
  and the DSP fully unit-testable. Iterative Cooley–Tukey with bit-reversal.
- **Summed per-axis power spectra**, not the magnitude of the acceleration vector — taking `|a|`
  frequency-doubles a pure tone; summing power captures the vibration whichever way the phone lies.
- **Linear resampling before the FFT** — DeviceMotion timestamps jitter, and an FFT assumes even
  spacing. `resampleUniform` is a heavily-tested pure function.
- **CSV-upload fallback as a first-class path** — it makes the tool useful on desktops and for
  permission-deniers, and it is the input that makes the whole pipeline drivable by the test harness.
- **Honest ceiling in the UI** — a phone samples ~60 Hz, so Nyquist ~30 Hz / 1800 RPM is stated, and
  harmonic hints are labelled informational, not a diagnosis.

## Test Results
- Tests written: 51 (dsp 22, csv 9, motion 9, format 11)
- Tests passed: 51
- Tests failed: 0 (one initial failure was a wrong test expectation — `23.45.toFixed(1)` rounds to
  23.4 due to float representation; corrected the expected value)

## Build Status
- npm install: pass
- npm test: pass (51/51)
- npm run build: pass (52 kB JS / 18 kB gzip; worker split to its own 6 kB chunk)
- Local preview: pass (HTTP 200 on :5212)
- Production workflow dry-run: pass — see below

## Browser Dry-Run (production build, Browser pane)
- Page loads, title/app/footer correct, no console errors.
- **CSV fallback path** (first-class): synthetic 20 Hz tone → **20.0 Hz / 1198 RPM**; 23 Hz tone →
  **23.0 Hz / 1379 RPM**; columns read, rate/Nyquist shown.
- **Live sensor path** (synthetic samples via a guarded `window.__shakewell` test hook): 15 Hz feed →
  **15.0 Hz / 900 RPM**, amplitude meter at 28 %, spectrum canvas rendering (3168 non-bg px).
- **Artefacts produced and verified well-formed:** spectrum **PNG** 64 kB with a valid PNG signature;
  recording **CSV** 9983 bytes / 361 lines (header + 360 samples).
- Modals: How it works, **Privacy** (label correct; Trust Surface present; discloses Cloudflare
  Analytics + feedback.benrichardson.dev), About — all open; Escape and × close.
- Event drawer at 375 px: opens, and closes via both the in-drawer × and Escape; no horizontal overflow.
- Feedback widget mounted in the footer.

**No real-device sensor check was possible** — the automation cannot shake a phone. The sensor path was
exercised only with synthetic samples through the test hook; permission handling (tap-gated
`requestPermission`, denied → CSV fallback, track/listener teardown on stop/hide) is implemented and
unit-tested at the pure level but not verified against a physical accelerometer.

## Deployment
- Repo created: yes (ben-gy/shakewell)
- GitHub Pages enabled: yes (workflow build type)
- Cloudflare DNS CNAME shakewell → ben-gy.github.io: created
- Deploy workflow: completed successfully (run 30034450214)
- TLS cert: provisioning at hand-off (github.io serves and 301-redirects to the custom domain;
  `https_enforced` had not flipped true yet — cert issuance commonly takes 5–15 min and cycles were
  triggered; no action needed)
- PR created: https://github.com/ben-gy/shakewell/pull/1

## Errors & Resolutions
- **TS strict build errors** (unused `AppRefs` import, unused `liveSampleRate`, `navigator.share` used
  as a truthiness condition → TS2774): removed the dead import/variable and switched to
  `'share' in navigator`.
- **CSV round-trip test failure**: `toCsv` writes a `t_ms` header, which didn't match the time-column
  key list; added `tms`/`ts`/`timems` to `T_KEYS`.
- **Format test expectation**: `formatHz(23.45)` returns `23.4 Hz` (float rounding); corrected the test
  to `23.46`.
- **Browser pane blocks localhost `navigate`**: used `preview_start` with a launch.json entry instead,
  which is the sanctioned opener.
