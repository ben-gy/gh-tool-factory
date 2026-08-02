# Build Log: GGUFScope
**Date:** 2026-08-02
**Status:** deployed

## Idea Source

`IDEAS.md`, first queue entry — **ggufscope**: *"Inspect and size-check any GGUF model over HTTP
Range before you download a byte of it, and get the exact llama-server flags that genuinely fit your
card."* The entry was detailed and had already been adversarially vetted as part of the twelve-idea
AI-builder batch. Removed from the queue at the start of the run.

Duplication check: no `ggufscope` in `registry.json` or in `gh repo list ben-gy --limit 200`. The
nearest catalogue entries (`tokenpeek` — JWTs; `verisum` — checksums) are unrelated domains. Not an
expansion of an existing tool: new problem, new file format, new browser capability as the pillar.

**Input question (principle #6):** the input here is neither a file nor a sensor — it is a *remote
byte range*. That is a third register the fleet had not used, and it is the whole product rather
than a detail.

## Tool Details
- **Name:** GGUFScope
- **Repo:** ben-gy/ggufscope
- **Category:** data-explorers (developer tooling)
- **Audience:** people running local LLMs on one consumer GPU, on a slow connection, who have been
  burned by a 20 GB download that OOM'd on launch
- **Stack:** Vite 6 + vanilla TypeScript + Vitest; one runtime dependency (`@huggingface/jinja`)
- **Browser APIs:** `fetch` + HTTP Range, CORS `Content-Range`, `Blob.slice`, Web Worker,
  `DataView`/`TextDecoder`, `AbortController`, Canvas 2D, File System Access, Web Share, Clipboard,
  Service Worker
- **Worker strategy:** one dedicated module worker doing fetch + parse, streaming determinate
  progress; main thread does the (pure, cheap) arithmetic so config changes recompute synchronously

## Ultracode execution

Ran a 13-agent Workflow (six research lenses, each adversarially refuted by an independent agent,
then a synthesis), all pinned to Opus 5. The research changed the design materially:

1. **The idea's central technical claim was wrong.** It specified a 4096-byte Range read. A GGUF's
   tensor table sits *after* all metadata, and the tokenizer vocabulary lives in that metadata, so
   real tensor tables start **5.7–7.5 MB in** (13 MB for gpt-oss). 4 KB gets you the magic, version
   and counts and nothing else. I measured this myself against four live models before writing any
   UI, and the ladder starts at 8 MB as a result — one request for the overwhelming majority.
2. **The ggml block-size table was verified twice independently** — by compiling `ggml-common.h` and
   printing `sizeof` for every block struct, and by diffing llama.cpp's `gguf-py/gguf/constants.py`.
   Both agree. I then reconciled it myself against six live models: **delta 0 bytes on every one.**
3. **The adversarial pass caught a factual error in shipped UI copy.** The widely-repeated "Ollama
   defaults to `num_ctx` 2048" is no longer true — the default is negotiated from available VRAM
   (262144 at ≥47 GiB, 32768 at ≥23 GiB, else 4096), clamped to trained context and stepped down on
   OOM. The Modelfile note was rewritten.
4. **`@huggingface/gguf` was rejected** on evidence, not taste: it cannot read a local `File`, it
   zero-pads past EOF so a truncated read yields 147 nameless zero-shape tensors and a confident
   **0 GB** answer, and it calls `arrayBuffer()` with no status check so a server ignoring `Range`
   downloads the whole model. A first-party parser avoids all three by construction.

## The design decision that matters

Everything hangs on one assertion: **sum the parsed tensor table and compare it against the file's
true byte size** from `Content-Range`. Exact match → every ggml type was sized correctly and the
VRAM figure is trustworthy; the UI says *verified*. Mismatch → say so, with a named diagnosis
(header-only fixture / truncated download / fork quantisation), rather than print a confident wrong
number. It catches fork quantisations, removed ggml types and cut-short downloads for free, and it
is checkable by the user rather than asserted.

## Privacy Model
- **Protected:** a local `.gguf` is read with `Blob.slice` in-tab and never uploaded — that path
  works fully offline. GPU size, context choice and every computed number stay on the device; all
  artefacts are generated locally.
- **Not protected:** inspecting a *remote* model is by definition a request to Hugging Face — its
  CDN sees the user's IP and which file they asked about, the same as opening the model page. Stated
  plainly in the UI with the local path offered as the alternative. This is the honest framing: the
  tool does not pretend the network request isn't happening.
- **Trust surface:** the static bundle + TLS, `huggingface.co`/`*.hf.co` (remote only), the
  Cloudflare Web Analytics beacon, and the feedback widget only if used.

## Architecture Decisions

- **First-party GGUF parser over `@huggingface/gguf`** — for the three defects above, and because
  one parser reading from an abstract byte source (`fetch`+Range remote, `Blob.slice` local) is less
  code than a library for one half and a hand-rolled parser for the other, and keeps both paths
  provably identical.
- **Huge arrays walked, not materialised.** The tokenizer vocab is 128k–260k strings; building them
  to throw them away costs more than the whole parse. Only the BOS/EOS/PAD/UNK entries are decoded,
  by index, for the chat-template preview.
- **Working memory is an allowance, not a formula.** llama.cpp's compute buffer depends on batch
  size, backend and flash attention and has no closed form. Inventing one would have been the easy,
  dishonest choice; it is a separate adjustable line item instead.
- **Metadata rewriting cut entirely**, per the idea's own instruction — it shifts every tensor offset
  and risks a corrupt 20 GB file discovered hours later.
- **`qrcode` dropped** — there is no link-based output here, so it was dead weight.
- **Ollama chat-template Levenshtein prediction not built** (logged to EXPANSION_IDEAS.md): it would
  need Ollama's 37-template index vendored in, and the run had a scope to hold.

## Test Results
- Tests written: **52** (35 VRAM/arithmetic, 17 parser/fixture)
- Tests passed: **52**
- Tests failed: 0

One failure during development was my own test's arithmetic — it forgot to subtract the KV cache
from expected headroom (the delta was exactly 262,144 bytes, i.e. the cache). Test corrected, not
the code.

The KV-cache suite asserts against numbers llama.cpp itself printed at startup: Llama-3.1-8B at
131072 f16 → 16384.00 MiB; TinyLlama at 4096 → 88.00 / 46.75 MiB; Llama-3.2-1B → 128.00 / 68.00 MiB;
Qwen3-32B at 32768 → 8192.00 / 4352.00 MiB. All exact.

## Build Status
- npm install: pass (5 advisories, all dev-only — vite/vitest/esbuild dev server, none shipped)
- npm test: pass (52/52)
- npm run build: pass
- Local preview: pass
- Production workflow dry-run: pass

## Live browser dry-run (Browser pane, real network)

- **Remote path (the headline):** `Qwen/Qwen3-8B-GGUF` → repo listed 5 quants with real sizes → picked
  Q4_K_M → **read 8.0 MiB of 4.68 GiB (0.17%) in 2.6 s**, verdict rendered, reconciliation
  **✓ verified**.
- `unsloth/Qwen3-32B-GGUF` Q4_K_M → 32.76 B params, 4.82 bpw, 64 layers, 64Q/8KV — matching the
  published fixture exactly — read 0.04% of an 18.40 GiB file, ✓ verified.
- **Exported `fit-report.json` matched published ground truth byte-for-byte**: 807,694,464 file
  bytes, 1,235,814,432 params, 5.1779 bpw, `deltaBytes: 0`.
- Report-card PNG valid (1200×630, correct magic bytes, real content drawn).
- Solver: dropping to an 8 GiB card rewrote the command to `-ngl 32`; a quantised KV cache correctly
  forced `-fa on`.
- Chat template rendered live with real `<|im_start|>` tokens resolved from the vocab by index.
- "Try a sample" ingests through the identical upload path and does **not** also open the file picker.
- `[hidden]` guard: drawer/modal/scrim `display:none` on load; `elementFromPoint` at the viewport
  centre returns real content.
- Mobile 375px: drawer closes via its `×` and via Escape; no horizontal overflow; all three modals
  open, fit and close.
- PWA manifest resolves with a `maskable` icon; `apple-touch-icon.png` 200 and opaque.
- No console errors.

## Errors & Resolutions

1. **The stylesheet was never imported — the first build shipped with no CSS at all.** Every
   structural DOM assertion still passed (`[hidden]` works from the UA stylesheet alone), so this was
   invisible to scripted checks and only the visual screenshot caught it. Fixed by importing
   `./styles/main.css` in `main.ts`; rebuilt and re-verified. Worth recording: this is exactly the
   class of bug that makes the screenshot step non-optional.
2. **A stale service worker from a previous tool was serving `localhost:5199`** — the recurring
   shared-port trap. Unregistered and cleared caches before each verification pass.
3. **Preview in a worktree:** `launch.json` requires a project-relative `cwd`, and the tool lives
   outside the worktree. Solved with `runtimeExecutable: "sh"` and a `-c "cd … && exec npx vite
   preview"` argument instead of a `cwd` field.
4. **Brand mark squeezed to 28×38** by its flex row; added `flex: 0 0 auto`.
5. **Dangling "Otherwise" in the advice copy** when no prior remedy applied; made conditional.
6. **F32 norm tensors rendered as "0.00 GiB"** in the quant table; switched that column to
   auto-scaling units.
7. **README accuracy:** two header-depth figures were estimates. I measured them rather than ship
   guesses — one was wrong (IQ4_XS is 1.05% of its file, not 0.66%) and was corrected.

## Deployment
- Repo created: yes — https://github.com/ben-gy/ggufscope
- GitHub Pages enabled: yes (workflow build type)
- Cloudflare DNS: `CNAME ggufscope → ben-gy.github.io` created, DNS only
- TLS certificate: `approved`
- Deploy workflow: completed **success**
- Live: https://ggufscope.benrichardson.dev — 200, `CN=ggufscope.benrichardson.dev`, Let's Encrypt
- PR created: https://github.com/ben-gy/ggufscope/pull/1

## Honest caveats shipped in the UI

- Working memory is an allowance, not a computed value.
- MLA architectures (DeepSeek-V2/V3) are flagged — the standard formula overstates their cache, so
  the figure is labelled an upper bound.
- Sliding-window models (Gemma) likewise get a safe upper bound and say so.
- Fork quantisations cannot be sized; the tool refuses rather than guessing.
- The remote path is disclosed as a real request to Hugging Face, not hand-waved.

## Post-deploy: synthesis findings applied

The 13-agent workflow's final synthesis landed after the first deploy. It confirmed both headline
capabilities and, importantly, confirmed the CSP requirement I had already shipped: `connect-src`
must include `https://*.hf.co`, because CSP is enforced against the **redirect target** and
huggingface.co 302s to its CDN. A policy naming only `huggingface.co` fails with `TypeError: Failed
to fetch`. (This contradicts the existing `transformers-image-to-image-superres` memory note that
"connect-src only needs huggingface" — that note does not generalise to GGUF LFS range reads.)

Two findings warranted changes, both shipped in `0d7c66c` and redeployed:

1. **The trust banner said "about 6 MB".** Observed header depths span **14 KB (stories260K) to
   13.07 MB (Llama-4-Scout, gpt-oss-20b)** with no proven ceiling, and the driver is vocab + BPE
   merge count rather than model size — a 770 MiB Llama-3.2-1B has a 7.8 MB header while a 19.8 GB
   Qwen3-32B has a 5.98 MB one. The banner now says "a few megabytes", which is true. The 8 MB → 24
   MB ladder already covers the 13 MB cases in two requests.
2. **Hosts that serve `.gguf` without CORS** (GitHub release assets, ollama.com — both return 206
   with no `access-control-allow-origin`) fail the fetch before any response object exists, which
   surfaced as a bare `TypeError` and sent the user looking for a fault on their end. That case is
   now named explicitly, with the local-file path offered as the remedy. Verified live against a
   real github.com URL.

Findings noted but deliberately not acted on: the synthesis suggests a 32 MiB hard abort cap where
mine is 160 MB (mine succeeds where theirs would abort, and is still bounded); and `x-error-code` /
`?expand[]=gated` could sharpen gated-repo errors, which the current 401 message already handles
acceptably.
