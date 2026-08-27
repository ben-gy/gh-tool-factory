# Build Log: unflatten
**Date:** 2026-08-27
**Status:** deployed

## Idea Source

IDEAS.md, the top live queue entry. **Queue order was followed — the fourth run in a row.**

One housekeeping finding first: the entry above it, **cuepoint**, was still in the queue with its
leading `-` intact despite having been built, deployed and registered on 2026-08-26. The 2026-08-26
run missed its own strike. This run applied it, so the queue parser no longer sees it.

Standing-note rule 2 was checked by a dedicated judge and **does not block**. The two named engine
groups are the text-embedding/cosine set and the jinja/tokenizer set; unflatten produces no
embedding, computes no cosine, ranks nothing and loads no tokenizer. cuepoint at one day old shares
transformers.js, ONNX Runtime, WebGPU-with-WASM-fallback, module workers and mediabunny — all of
which are fleet infrastructure already in bgwipe, scribewell, blurwell, vidwell, embiggen and
wavewell — but not the algorithmic core (cross-modal retrieval versus dense depth regression plus
geometry), and not even the codec direction (cuepoint decodes and trims footage the user already had;
unflatten encodes frames that never existed). The nearest engine siblings are **wavewell** (30 days,
the canvas→VideoEncoder→MP4 path) and **embiggen** (32 days, dense image→image on WebGPU), not
cuepoint.

**Sensor debt: NOT paid, deliberately.** The last sensor tool is still tiltwell, 2026-08-02. The file
records that the 2026-08-11 hunt spent three adversarial rounds killing all eight candidates and asks
for a hand-added idea rather than another speculative hunt. That remains the right next move, and the
debt is now the oldest open item in IDEAS.md.

## Tool Details

- **Name:** unflatten
- **Repo:** ben-gy/unflatten
- **Category:** images
- **Audience:** someone with a flatbed scan of a grandparent who saw a 3D-photo clip years ago and
  has wanted it for this one image ever since. Secondary and deliberately demoted: a ControlNet user
  who wants a depth map.
- **Stack:** vanilla TypeScript + Vite 6; `@huggingface/transformers` 4.2.0; `mediabunny` 1.55.3
- **Browser APIs:** WebGPU / ONNX-WASM inference, WebGL2 (first-party renderer, no three.js),
  OffscreenCanvas, WebCodecs `VideoEncoder`, module Web Workers ×2, `CompressionStream('deflate')`,
  `createImageBitmap` with `imageOrientation`, Web Share (files), `navigator.wakeLock`, Cache API
- **Worker strategy:** two dedicated module workers — the model in one, the WebGL2 render and encode
  loop with an `OffscreenCanvas` in the other

## Privacy Model

- **Protected:** the photograph, the depth field, every frame and every output file — all in-tab.
  Nothing written to disk (no cookies, no localStorage, no IndexedDB, no OPFS); the build gate reads
  `dist/` and fails if any of those appears in first-party code. No account, no counter, no limit.
- **Not protected:** the first-run model download from `huggingface.co` (weights via `*.hf.co`),
  before a photograph has been chosen; GitHub Pages' own request log; the fact that the output
  invents pixels; and — stated explicitly rather than glossed — that iOS clears cached data for sites
  unopened for about a week, so the model may re-download.
- **Trust surface:** the bundle and its TLS chain, `huggingface.co`/`*.hf.co`,
  `static.cloudflareinsights.com`, `feedback.benrichardson.dev`. ONNX Runtime's WebAssembly is served
  from this origin, not jsDelivr. The gate matches the CSP's `connect-src` against that list exactly.

## Architecture Decisions

**The travel limit is the product.** The width of the invented region at a depth edge is exactly
`excursion × depth_jump`, independent of how soft the edge is (measured over a 1-D warp of a 30 px
jump spread across 1–32 pixels: 6.00 px of invention in every case, while the per-pixel gradient fell
32×). So the cap is driven by a windowed **jump** map rather than a gradient — on Middlebury ground
truth, blurring the field to imitate a network's own smoothing collapsed the p99.5 gradient 5× while
the windowed jump moved 15%, so a gradient-driven cap would permit about five times too much travel
with the smear unchanged. `cap_px = min(tol/J99.5, 4·tol/Jmax, 0.12·W)` with `tol = 2.5` output px.

**No projection matrix.** Off-axis frustum-shear parallax is exactly `f·t·(1/Zc − 1/Z)` — affine in
inverse depth, which is what the network returns — so a vertex is displaced in screen space by
`shift × (depth − pivot)` and that is the exact answer, verified numerically to 5.6e-17 against the
true projection. A look-at camera moves points that sit on the convergence plane and adds vertical
parallax; both are visible, and the second is what makes an anaglyph hurt to look at.

**Two normalisations, on purpose.** Robust p2/p98 percentiles for the geometry (one hot pixel must
not decide how far the camera travels) and true min/max for the exported 16-bit PNG (clipping four
per cent of a depth map flat is not acceptable in a file somebody opens elsewhere).

## Corrections to the queue spec (all measured this run, by eleven agents)

- **Ship fp16, not q4f16.** The spec's headline was the 19.1 MB q4f16 build. It is a `com.microsoft`
  `MatMulNBits` graph (confirmed by reading the file's bytes at offset 5505), no public deployment
  uses it for this model, and two open transformers.js issues document q4/q8 weights producing
  **silently wrong output** on the WebGPU execution provider. A wrong depth field does not throw — it
  renders as a plausible but incorrect parallax, the one failure a person could not see. Shipped
  fp16 (49,642,442 B) on WebGPU and int8 (27,258,801 B) on the processor.
- **The first-run download is ~56 MB, not 19 MB.** The spec omitted ONNX Runtime's WebAssembly, which
  is fetched on every path including WebGPU. GitHub Pages serves it **gzipped, not brotlied** —
  5,862,143 B, measured against a live fleet tool on the real host, not the 4,732,131 B a brotli host
  would send. The consent panel says 56 MB.
- **The spec's "491 ms on an Apple Metal-3 adapter" is unverifiable.** No such measurement exists in
  any public source. **No latency figure appears anywhere in the UI.**
- **mediabunny is 1.55.3**, not the spec's 1.52.2 — and 1.53.0 carries a fix to Safari encodability
  checks, i.e. to the exact call that gates the whole export on the browser most likely to break it.
- **Capping the processor's `size` does not cap the work.** The DPT preprocessor pins whichever axis
  has a scale factor closest to 1 — always the shorter one — and applies it to both, so the network's
  input is `target² × aspect` **at any source resolution**; shrinking the source does nothing.
  `processorSize()` lowers the target for a long picture, and is unit-tested against exactly that.
- **transformers.js's Safari detection cannot work in a worker** — it reads `navigator.vendor`, which
  the HTML specification exposes to Window only, so Safari would be handed the 23.6 MB asyncify
  runtime instead of the 12.9 MB build meant for it. This build decides from `navigator.userAgent`.
- **The ControlNet audience is contested.** The demand judge found at least six free, no-signup,
  client-side depth-map generators already on that query. `depth.png` ships as a listed artefact with
  no SEO copy built around it.
- **There is one same-model competitor the spec never mentioned:** mondniles.com runs
  `onnx-community/depth-anything-v2-small` client-side and meters it at two watermarked renders a
  month anonymously. The About copy was rewritten around what is actually true — unmetered,
  unbranded, and a real depth model rather than the brightness heuristic the free browser tools use.

## Defects found by the adversarial review and the browser run

- **A "dolly" that only zooms produces zero parallax.** Design review. A dolly moves along the view
  axis, so its parallax is a change of magnification with distance, not a translation — modelling it
  as a uniform zoom renders a flat Ken Burns with no depth effect *and* drives the travel cap to
  infinity, because a path with no excursion has nothing to bound. Fixed in the vertex shader
  (`zoom + dz·(depth − pivot)`), with a test that every registered path has a non-zero extent.
- **`evenise()` could return 0.** Its own test caught it: `Math.round(0.4)` is 0, which is even, and
  a zero-width encode fails at the very end of a long render.
- **The depth PNG was built from the clipped field**, throwing away ~4% of the image. Two separate
  normalisations now, both tested.
- **No EXIF orientation and no texture-size cap.** A phone photograph would have been fed in sideways,
  and the stated hero input — a 600–1200 dpi flatbed scan — exceeds `MAX_TEXTURE_SIZE`, where
  `texImage2D` does not throw but leaves a **black** texture.
- **The codec probe can hang for ever.** Found in the browser: mediabunny's
  `getFirstEncodableVideoCodec` never settled inside the worker, and because the export is one long
  `await` chain that presented as a progress bar frozen at zero with **no error, indefinitely**. It
  now has a 12 s deadline and falls back to asking `VideoEncoder.isConfigSupported` directly. The
  fallback was then observed working: the run selected MP4/H.264 after the library probe timed out.
- **The clip card was deleted the instant it was created.** `enterStudio()` cleared the artefacts
  panel, and the clip handler called it one line after adding the card — so a person waited out a
  whole render and was handed nothing. Nothing short of running an export end to end would have
  found it; every scripted DOM check passes, because the card is built correctly and then removed.
- **A stale service worker masqueraded as a hang.** Rebuilding changed the worker chunk hash while a
  precached SW served the old one, and the module worker failed to evaluate with an empty error
  message. The recorded `preview-stale-service-worker` trap, in a new costume.
- The consent panel rendered a 38 KB file as "0.0 MB".
- Both model tiers were being tried with a full 180 s timeout each on an unreachable network — six
  minutes of a progress bar at zero. They fetch from the same host, so the first network failure now
  stops the chain.

## Test Results

- Tests written: **73**, across `png16`, `depth` and `camera`/`plan`
- Tests passed: **73**
- Tests failed: 0 (two failed on first run and both were real defects — `evenise` returning 0, and a
  jump-map claim that only holds when the window is wider than the edge; the code was fixed and the
  second test was rewritten to state the true property plus the condition on it)

The load-bearing ones: `excursionCap` gives a hard-edged photograph less travel than a soft one; the
round-trip identity `cap × J99.5 ≤ tol`; every camera path closes its loop in **velocity** as well as
position; the 16-bit PNG round-trips byte-exactly with **big-endian** samples.

## Build Status

- npm install: pass
- npm test: pass (73/73)
- npm run build: pass
- Build gate: pass — no storage, no sending, no console; CSP, manifest, runtime and samples sound
- Local preview: pass
- Production workflow dry-run: pass, with one exception recorded below

## Browser verification (what was and was not proven)

Run against the production build at `localhost:5217` and, for the parts needing a test hook, a
development build at `:5218`.

**Verified in a real browser:**
- Page renders, correct dark palette, stylesheet linked, no leaked `[hidden]` elements, viewport
  centre returns page content rather than an overlay host.
- **All six overlays × all four exits** (× control, outside pointerdown, Escape, and a real
  `pointerType:'touch'` sequence), re-opened between each. Every close control measures 44×44.
- Both sample pills ingest through the real ingestion path and do **not** also open the file picker.
- Consent panel: correct tier, correct 56 MB figure, names the host, never auto-downloads.
- Model-download failure → a clear user-visible error and a working Retry.
- Depth → normalise → jump map → travel cap → studio. **The headline claim demonstrated end to end:
  a hard-edged subject capped at 0.53% of the frame, a gently graded scene at 5.60% — a ten-fold
  difference derived from the picture with no setting.**
- All four artefacts produced and inspected: the 16-bit depth map (valid signature, `IHDR` bit depth
  16, colour type 0, non-interlaced, decoded by the browser at 472×531), the lens blur, the red/cyan
  still (centre pixel `[163,179,96]` against the blur's `[164,137,94]`, confirming the Dubois channel
  separation is genuinely applied), and **the clip: 288 KB of AVC in an MP4**.

**NOT verified, stated plainly:**
- **The depth model itself never ran.** The automated browser has no outbound route to
  `huggingface.co` (`ERR_CONNECTION_REFUSED`), so the weights could not be downloaded. Every stage
  downstream of the depth field was exercised with a synthetic field injected through a
  development-only test hook, which is identical code from that point on — but no real inference
  happened, no fp16-versus-int8 comparison was possible, and **no throughput figure from this build
  is a measurement**. Nothing in the UI quotes one.
- No real iOS device was tested. The Safari-specific claims (WebGPU floor, the 7-day cache
  eviction, Share as the delivery path) come from compatibility data, not from a device.
- The two shipped samples were driven with synthetic depth, so the travel limits quoted above
  characterise those synthetic fields, not what Depth Anything returns for those photographs.

## Deployment

- Repo created: yes — https://github.com/ben-gy/unflatten
- GitHub Pages enabled: yes (workflow build)
- Cloudflare CNAME created: yes (zone at 233 records; no reclamation needed)
- TLS certificate: **approved on the first poll**
- Directory entry live on main: yes
- Workflow triggered: yes

## Errors & Resolutions

Every defect above was fixed in this run except the two verification gaps, which are limits of the
environment rather than of the tool and are recorded as such. One process note worth keeping: the
preview browser's hidden tab clamps `setTimeout` to ~1 s, so a scripted check with two dozen `await
sleep()` calls exhausts the 30-second tool budget and reports a timeout that looks like an
application hang. The overlay matrix had to be rewritten synchronously, and its first (failed) run
was measuring debris left by the timed-out attempt before it — not a real defect.
