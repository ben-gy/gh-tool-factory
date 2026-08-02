# Build Log: Searchpack
**Date:** 2026-08-02
**Status:** deployed

## Idea Source

IDEAS.md, **second** entry — deliberately not the first.

The queue's head was **runledger**. The same file's standing note binds harder than its ordering
rule: *"the `@huggingface/jinja` + `@huggingface/tokenizers` + ungated-mirror-registry renderer
underlies chatprint, runledger and epochcost. Never ship two tools from the same group in the same
week."* **chatprint shipped earlier today (2026-08-02).** Shipping runledger now is precisely the
reskin failure that note exists to prevent, so runledger was left at the head of the queue for the
next run and searchpack was taken instead — its engine-group siblings (chunkforge, leakspot,
topicgap) are all unbuilt. Only the searchpack line was deleted from IDEAS.md.

## Tool Details
- **Name:** Searchpack
- **Repo:** ben-gy/searchpack
- **Category:** developer-tools
- **Audience:** developers running docs sites on free static hosting, whose options today are
  application-gated Algolia, a vector-DB bill, or keyword-only Lunr
- **Stack:** Vite 6 + vanilla TypeScript + Vitest; one runtime dependency (`fflate`)
- **Worker strategy:** one dedicated module worker (model fetch, table parse, chunk, embed, pack,
  self-test, and the live playground). HTML extraction stays on the main thread because Workers have
  no `DOMParser`, batched with yields.

## Privacy Model
- **Protected:** every file is parsed, chunked and embedded in the tab and never uploaded. The pack
  contains no beacon, callback or network code beyond fetching its own two static files, so the
  user's *readers'* queries stay on their machines too.
- **Not protected:** one anonymous pinned GET to huggingface.co for the model (IP visible, no
  content); iOS Safari clears the cache after 7 days unless installed to the Home Screen; the pack
  contains chunk text for snippets, so it is as sensitive as the folder it came from.
- **Trust surface:** the GitHub Pages bundle + TLS, huggingface.co for the one-time fetch, the
  Cloudflare Web Analytics beacon, and feedback.benrichardson.dev only when the user submits.

## Architecture Decisions

**The decisive finding, established by direct measurement before any code was written.**
`sentence-transformers/static-retrieval-mrl-en-v1` is not really a neural network. Downloading
`onnx/model_int8.onnx` and parsing its graph showed the whole computation is
`Gather(embedding.weight_quantized, ids) → DequantizeLinear(scale, zero_point) → ReduceMean`, with
the table a `uint8[30522, 1024]` initializer at byte offset 3825, `scale = 1.249721884727478` and
`zero_point = 121`, both per-tensor scalars. An embedding is the mean of some table rows.

So the build reads the table directly and skips ONNX Runtime, `@huggingface/transformers` and WASM
entirely — which the IDEAS.md sketch had specified. That is not a shortcut, it is the *only* way the
product works: the shipped engine is 26 KB of JavaScript instead of 11 MB of WASM, and a search
engine you hand to a third party cannot ask their readers to download a runtime. The positive scalar
`scale` factors out of the mean and is annihilated by L2 normalisation, so only `zero_point` is
needed.

**Deviations from the IDEAS.md sketch, all deliberate:**
- No `@huggingface/transformers` / `onnxruntime-web` (above). An adversarial agent independently
  confirmed the sketch's stated reason for preferring this model — that it works "through the
  STANDARD feature-extraction pipeline" — is **false**: the repo has no root `tokenizer.json`, the
  pipeline hard-defaults `add_special_tokens: true`, and the ONNX emits `sentence_embedding` rather
  than any tensor name the pipeline reads.
- No `@mozilla/readability` / `turndown`. Readability finds prose and discards heading `id`
  attributes — exactly what a docs search needs to deep-link to `page.html#section`. A first-party
  structural extractor is smaller and strictly better for this job.
- `fflate` for ZIP in *and* out (the sketch already caught that `CompressionStream` cannot write a
  ZIP container).

**The shared-runtime trick.** The idea's stated hard part was making the build-time embedder and the
shipped shim agree. They agree *by construction*: the tokenizer and engine live in plain `.js` files
under `src/shim/` with hand-written `.d.ts` siblings; the app imports them normally and the packer
imports the same files with Vite's `?raw` suffix and concatenates their literal source into
`search.js`. Measured round-trip cosine is exactly **1.000000**.

## Adversarial review (7 agents, Opus 5, all pinned)

Five refutation lenses plus duplication and demand. **All five claims were refuted**, three
decisively, and each changed the build:

1. **`[CLS]`/`[SEP]` — my claim was backwards.** `sentence_transformers/.../static_embedding.py`
   line 114 calls `encode_batch(inputs, add_special_tokens=False)`. Adding them "to match the
   reference" is the deviation: measured cosine 0.9994 on prose but **0.763 on a one-word query** —
   and a search box receives one-word queries. Fixed: ids 101/102 are never emitted.
2. **Subsetting the tokenizer vocabulary is unsound.** WordPiece is greedy longest-match, so removing
   entries *re-segments* unknown words instead of silencing them. On a real bert-base-uncased vocab,
   **41% of held-out words** re-segment into pieces that are still in the index — `carpet` →
   `["car", "##pet"]`, so the query would confidently retrieve every page about cars. Worse, it is
   invisible to any test that only tokenizes corpus text (which passes vacuously). Fixed: all 30,522
   token *strings* ship (~230 KB) and the subset applies only to embedding rows, as a post-filter.
   Confirmed the corpus-side lemma still holds for this vocabulary.
3. **Parity with the fp32 reference cannot be claimed.** The uint8 table is lossy: per-row cosine vs
   the float weights has median 0.945, and sentence-level deviation reaches ~0.013 at 256 dims — four
   orders of magnitude larger than "float rounding". All copy now states the real figure.
4. **MRL ordering** — the agent confirmed truncate-then-normalise is canonical and that
   normalize-then-truncate emits non-unit vectors. Searchpack truncates the *rows* before summing,
   which is already truncate-then-normalise, so it was correct; the concern did not apply.
5. **ONNX robustness** — flagged that `scale`/`zero_point` live in `float_data`/`int32_data` rather
   than `raw_data` (a raw-only reader returns silent garbage, measured cosine 0.0046 vs 0.9999) and
   that a QInt8 re-export flips the dtype. Both were already handled; the run pins an immutable
   revision SHA as recommended.
6. **Model choice** — published MTEB Retrieval: potion-base-8M 31.11, static-retrieval-mrl-en-v1
   34.95, potion-retrieval-32M 35.06, all-MiniLM-L6-v2 42.92, bge-small 51.68. The chosen model is
   the best static option at a sane size (potion-retrieval-32M is marginally better but 129 MB). The
   agent's recommendation to use a transformer was **rejected with reasons**: it needs an 11 MB WASM
   runtime *on every reader's site*, which destroys the artefact. Its iOS 7-day cache-eviction point
   was accepted and is disclosed in the UI.
7. **Duplication: BUILD_NEW.** The reviewer fetched tokenpeek, textlift, gridwell, pagesmith and
   chatprint and failed to refute novelty on every axis — chatprint emits a diagnosis, searchpack
   emits a redistributable software artefact.

## Findings from the browser dry-run that changed the build

Three problems only visible end-to-end, all fixed:

- **Plain mean-pooling was badly wrong for retrieval.** "how do I stop it retrying forever" scored
  **0.0325** against the page answering it and ranked an unrelated page first, because nine of its
  twelve tokens are function words that dominate the average. Fixed with **IDF-weighted pooling**
  using document frequencies computed from the user's own corpus (one float per shipped row, and the
  BM25 postings were already being built). The same query then scored **0.46** and ranked correctly.
  The weights also give a principled "no signal" test — nonsense does *not* become `[UNK]`, because
  bert-base-uncased has every letter, so `zzqqxwv` shatters into ubiquitous single-character
  subwords whose weights are ~0.
- **The sample corpus poisoned its own demo.** The zip shipped a README listing the suggested demo
  queries verbatim, so it ranked first for every one of them — the index was working perfectly and
  the demo looked broken. The README was removed from the corpus.
- **A stale-response race in the playground.** `searchSeq` was generated but never checked, so a
  short query started later could finish first and overwrite fresher results. Fixed.

## Test Results
- Tests written: **144** across 5 files
- Tests passed: **144**
- Tests failed: 0
- Two genuine bugs were caught by the tokenizer parity fixture before they could ship:
  - JavaScript's `\s` matches U+FEFF but Rust's `is_whitespace` does not, so format characters must
    be filtered *before* the whitespace fold or a byte-order mark splits a word.
  - Non-ASCII symbols (`€`, `→`, `©`) are not BERT punctuation, though ASCII `$ + < = > ^ | ~` are.

The tokenizer fixture is generated from the **real HuggingFace tokenizer** via
`scripts/gen-tokenizer-fixture.mjs` and committed with the full 30,522-entry vocabulary, so the suite
tests against the reference rather than against itself, and covers OOV words, accents, CJK,
astral-plane emoji and the 100-code-point word limit.

## Build Status
- npm install: pass
- npm test: pass (144/144)
- npm run build: pass (`tsc` clean, 380 KiB precache)
- Local preview: pass (port 4207 — 4196 was occupied by an earlier chatprint preview, which served
  the wrong app until spotted)
- Production workflow dry-run: **pass** — sample → 18 docs → 66 chunks → verified pack → live queries

## Verification measured in the browser
- Round-trip cosine **1.000000** across 20 chunks (the emitted `search.js` loaded from a Blob URL and
  run against the emitted binaries)
- Binary pre-pass recall **100%** of exhaustive · self-retrieval **100%** · 8/8 adversarial queries
- Vocabulary subset: **921 of 30,522 rows** shipped — 609 KB instead of 7.5 MB
- `search.zip` 356 KB · `search.js` 26.6 KB · rebuild 0.2 s once the table is cached
- Mobile 375 px: drawer closes via × and Escape; `getComputedStyle(drawer).display === 'none'` on
  load and `elementFromPoint` at the centre returns real content, so the `[hidden]` trap is absent;
  no horizontal overflow

## Honest limits recorded in the UI, README and PR
Static embeddings score ~35 on MTEB Retrieval vs ~43 for all-MiniLM-L6-v2 and ~52 for bge-small.
**Measured on the bundled 18-page sample, five conversational questions deliberately phrased to avoid
the answering page's wording put the right page in the top three twice (2/5).** Hybrid RRF exists
because BM25 rescues exactly the queries pooled embeddings handle worst. This is the ceiling of the
architecture, not a defect — and it is stated in the Privacy modal, the README, the generated pack's
README and the PR rather than buried.

## Licensing decision flagged for review
`ADDITIONAL-TERMS.md` releases the **generated pack under MIT** while Searchpack itself stays
AGPL-3.0-or-later. Without that carve-out the AGPL's network-use/source-disclosure obligation could be
read as reaching the user's own website, which would make the tool useless for its only purpose. This
was a judgement call made during the build and is called out explicitly in `REVIEW.md` and the PR.

## Deployment
- Repo created: yes — https://github.com/ben-gy/searchpack
- GitHub Pages enabled: yes (build_type=workflow)
- Cloudflare DNS `searchpack` → `ben-gy.github.io`: created
- TLS certificate: **approved** (second poll); `https://searchpack.benrichardson.dev/` returns 200
- Deploy workflow: completed success
- PR created: https://github.com/ben-gy/searchpack/pull/1

## Errors & Resolutions
- Four TypeScript strictness errors on first build (`webkitGetAsEntry` typing, `Uint8Array` as
  `BodyInit`, two always-defined function checks) — all fixed.
- Destructuring-assignment-at-statement-start bugs in the ONNX reader (`[a, b] = f()` parses as an
  array literal) — caught by inspection before running and rewritten to return an object.
- The first preview served **chatprint**: port 4196 was still held by an earlier run's server, so
  `--strictPort` did not save it. Moved to 4207 and re-verified. Worth remembering: always check
  `lsof` before trusting a preview.
