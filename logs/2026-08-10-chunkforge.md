# Build Log: chunkforge
**Date:** 2026-08-10
**Status:** deployed

## Idea Source

IDEAS.md queue head, taken top-down as the standing note requires. The entry opened:

> **chunkforge** — Drop a folder of PDFs and docs and watch a real query's retrieval ranks move as you
> drag the chunk-size slider, then export the chunks and the exact splitter config.

**The input question (principle #6), asked and answered.** What if the input were not a file? For this
tool it has to be: the whole premise is measuring retrieval over *your* corpus, and a corpus is files.
But the honest observation is that the last nine registry entries in a row all ingest a file and emit
a file. **The sensor debt is real and should be paid within the next two or three runs** — the queue
has no sensor idea in it, so that means reaching past the queue or adding one to it.

**Standing-note timing check.** chunkforge belongs to the text-embedding engine group (searchpack,
2026-08-02). Eight days, so the "never two from the same engine group in the same week" rule is clear.
The engine is explicitly REUSED rather than presented as new work — see below.

## Tool Details
- **Name:** chunkforge
- **Repo:** ben-gy/chunkforge
- **Category:** developer-tools
- **Audience:** an engineer debugging a RAG pipeline that cites the wrong page, holding documents they
  are not allowed to upload
- **Stack:** vanilla TypeScript + Vite 7, no framework
- **Browser APIs:** Web Workers, Cache API, File API + drag-and-drop, Clipboard, Web Share, Service
  Worker (vite-plugin-pwa)
- **Worker strategy:** one dedicated module worker owning the embedding table, the token indices and
  every re-score

## Reuse, stated plainly

The embedding engine is **searchpack's**, not new. `src/onnxtable.ts` (the ~200-line protobuf reader
that lifts the quantised table out of the ONNX file) and the WordPiece tokenizer are ports of
searchpack's, with searchpack's own findings carried over: `add_special_tokens=False`, never subset a
WordPiece vocabulary, `scale` cancels under L2 so only `zero_point` matters. What is new here is the
boundary tokenizer (byte-level BPE), the three splitters, the metric, and the artefacts.

## What the research changed

43 agents across six lenses, each finding attacked by an independent skeptic. Six load-bearing
inversions, every one of which changed the build:

1. **The spec's embedding path cannot run.** `@huggingface/transformers` needs `config.json`,
   `tokenizer.json`, `tokenizer_config.json` and `special_tokens_map.json` at the model's repo root;
   all four are 404. Even given them, the graph's only output is `sentence_embedding` while the
   feature-extraction pipeline reads `last_hidden_state ?? logits ?? token_embeddings`. The
   runtime-free table reader is not an optimisation, it is the only path.
2. **IDF-weighted pooling is a measured regression.** It was in the design on the strength of one
   anecdote from searchpack. On BEIR SciFact (300 queries, harness validated by reproducing this
   model's published MTEB numbers to four decimal places) plain mean scores nDCG@10 **0.5938** against
   **0.5648** weighted, paired bootstrap over 10,000 resamples giving P(weighted better) = **0.007**.
   Worse for this tool specifically: weights drawn from the chunking move with the slider — eight
   points of drift, the same size as the effect being measured. Pooling is now plain mean; the weights
   survive only as the no-signal threshold, which is not a score.
3. **The modern pdf.js build is a white page on iOS below 18.4.** `build/pdf.mjs` contains
   `if (typeof Iterator.prototype.join !== "function")` unguarded at module top level — `typeof`
   protects a bare identifier, not a property access on one. chunkforge ships the legacy build. This
   would have shipped undetected; the fleet's existing tools use the modern import.
4. **Without `cMapUrl` a CJK PDF returns `items: []` and no error.** The 169 predefined CMaps are now
   served from this origin, fetched only by documents that need one, and deliberately not precached.
5. **Sorting lines by y breaks two-column PDFs.** Real two-column producers emit each column
   contiguously, so document order IS reading order and a y-sort interleaves them. Column detectors
   built to repair the damage were measured finding zero columns on a genuine two-column paper and up
   to sixteen spurious ones on pages with tables. No column detector ships; lines keep content-stream
   order with a geometric fallback when the stream is visibly shuffled.
6. **`transform[3]` is not the font size.** It is exactly 0 for text rotated 90 degrees. Size is
   `hypot(transform[2], transform[3])`.

And the one that reshaped the product rather than the code: **k must be derived from a context budget,
not fixed.** At a fixed top-k a bigger chunk retrieves more text for free, so "recall rises with chunk
size" is arithmetic. k is now `floor(budget ÷ size)`, a zero-skill chance level is drawn beside every
score in closed form, a hit is binary character overlap rather than a fraction, and no winner is
declared — separating two close settings needs about six one-way discordant queries and nobody types
six.

## Privacy Model
- **Protected:** documents and queries never leave the tab; nothing written to disk (no IndexedDB, no
  OPFS, no cookies); exports assembled locally.
- **Not protected:** which model you pick is visible to Hugging Face; the feedback form when you press
  Send; your own exports.
- **Trust surface:** the GitHub Pages bundle and its TLS chain, `huggingface.co` for two public files,
  the Cloudflare Web Analytics beacon, the feedback widget.
- The claim on the page is **"your documents never leave the tab"**, deliberately not "no network".

## Architecture Decisions

Runtime-free scoring, because GitHub Pages cannot send COOP/COEP, so onnxruntime-web would be pinned
to a single WASM thread — and because a static embedding is a weighted sum of lookup-table rows, which
is 26 KB of JavaScript instead of 11 MB of WASM. Measured: **3,000 chunks of ~800 characters embedded
in 220 ms**, which is what makes a live slider possible.

Byte-level BPE was implemented first-party rather than pulling in `gpt-tokenizer`'s 53 MB barrel. It
covers OpenAI's cl100k and o200k, Qwen3 and gte-modernbert, and it is exact: identical ids to
`gpt-tokenizer` across ten edge cases and an 11,003-token document, first token to last.

## Test Results

- **Tests written: 258** in the new suites (extract 87, exports 67, score 54, split 38, build gates
  12), plus 26 in the hand-written suites (LangChain fidelity 9, PDF sample 7, sample demo 7, model
  integration 5) and the tokenizer suites added at the end.
- **Tests passed: all.** Tests failed: none.
- **The fidelity test is the important one:** chunkforge's recursive splitter is compared against
  `langchain-text-splitters` 0.3.11 itself — a fixture generated by running the library — and matches
  **chunk for chunk, string for string, across all seven size and overlap settings**, including the
  degenerate 80-token case where the reference emits chunks larger than the limit.

### Bugs the tests and the dry-run caught
- `chanceOfHit` returned 1 instead of 0 for k = 0 when every chunk touched the span (found by checking
  the closed form against brute-force enumeration of every k-subset). Latent, but it is exactly the
  number that stops an arithmetic 1.0 being read as retrieval.
- A document row printed `[object Object],…` instead of the page count — `doc.pages` where
  `doc.pages.length` was meant. Caught by the browser dry-run, invisible to every unit test.
- The sweep carried the overlap as an ABSOLUTE token count, so a 200-token overlap at size 96 clamped
  to 95 and made the stride one token: **11,014 chunks from an 11,000-token document**, crowding out
  every real row in the comparison table. The sweep now holds the overlap RATIO.
- Two literal NUL bytes were written into `src/` by the authoring tool, one of them keying the BPE
  merge table. Both sides of the lookup agreed so every test passed; it would have broken the first
  time either line was edited. A source-hygiene gate now fails the build on any literal control byte
  in `src/`.
- The running-header stripper used index-based bands, which assume the line list is in visual order —
  it is not, since lines keep content-stream order. One "Page 1" leaked through. Bands are now
  geometric.
- The paragraph threshold of 1.55x the line pitch merged four consecutive paragraphs into one
  1,537-character block on the shipped sample. It is 1.3x.

### The sample's own plan was wrong, twice
It was written so one query would favour SMALL chunks. Measured, that query favoured large ones — the
900 tokens around the planted answer are all on the query's topic, so the big chunk deserved to win.
Then the metric changed to a fixed context budget and nearly everything retrieved all three answers,
moving the interesting variation to cost. What the sample now demonstrates is measured rather than
assumed:

| setting | answers | retrieved tokens/query | chance level |
|---|---|---|---|
| recursive 1024/200 — the most-copied config anywhere | 3 of 3 | 3,724 | 29% |
| heading-aware 1024 | 3 of 3 | **1,408** | **10%** |
| fixed 96 | 3 of 3 | 4,028 | 43% |

## Build Status
- npm install: pass
- npm test: pass
- npm run build: pass
- Local preview: pass
- Production workflow dry-run: pass — sample loaded through the real ingestion path, PDF extracted (19
  pages, 53 headings, 137 paragraphs, 37 running-header lines removed), model downloaded and cached,
  3 of 3 answers retrieved, slider recomputed live, 24-row sweep completed, export produced a
  31 KB zip containing chunks.jsonl, splitter-config.md, comparison.csv, report.html and README.txt.
- **Overlay escape-hatch test: 20/20.** All five overlays (three modals, the event-log drawer, the
  reader) close via the × control, a backdrop pointerdown, Escape, and a real touch-type PointerEvent
  sequence, at 375px, each with a 44x44 hit area, with the `[hidden]` baseline clean and
  `elementFromPoint` returning page content after every close.
- PWA: manifest resolves with name, short_name, start_url, scope, display, a 512x512 maskable icon and
  the 180x180 apple-touch-icon; the sample is precached; the cmaps deliberately are not.

## Deployment
- Repo created: yes
- GitHub Pages enabled: yes
- Directory entry live on main: yes
- Workflow triggered: yes

## Errors & Resolutions

- The first test-writing fan-out was launched against source that was about to be rewritten by the
  research findings. Stopped it, did the rework, relaunched. Correct call — the tests would have
  encoded the pre-correction behaviour.
- One of the four test agents never started (three of four launched); the tokenizer suites were
  written in a follow-up run.
- The preview served a stale bundle from the service worker and the sweep looked unfixed after a real
  fix. Unregistering the worker and clearing the non-model caches showed the corrected numbers. This
  is the recurring localhost trap and it cost about ten minutes.

## Not done, and stated rather than implied

- **No DOCX or HTML ingestion.** PDF, Markdown and plain text only.
- **No real-device check.** Every timing in the log is desktop V8. iOS behaviour is inferred from the
  measured feature-detection facts, not observed — including the pdf.js legacy-build fix, which is the
  right fix for the right reason but has not been run on an iPhone.
- **No live provider comparison.** The scorer is a static embedding model and is a weaker retriever
  than anything anyone deploys; the tool says so wherever it reports a number.
