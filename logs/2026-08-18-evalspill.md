# Build Log: evalspill
**Date:** 2026-08-18
**Status:** deployed — https://evalspill.benrichardson.dev

## Idea Source

IDEAS.md, the top queue entry, taken verbatim as `leakspot`:

> **leakspot** — Drop your training file and your eval file side by side and get the exact percentage of
> your benchmark that leaked out of training, plus a decontaminated eval set and an honest, corrected
> score. […] Stage 1 lexical, reproducing the published GPT-3/Llama decontamination recipe: normalise →
> 13-gram character shingles → MinHash → LSH banding → exact Jaccard on candidate pairs only […]
> the headline "your model scores 91%, but 12% of the eval set leaked; on the clean subset it is 84%"

**Queue order was followed.** Standing-note rule 2 had blocked this entry while chunkforge (2026-08-10)
shared its static-embedding engine; on 2026-08-18 that is eight days, so it was unblocked and taken.

**The sensor debt was considered and deliberately not paid.** IDEAS.md records that the 2026-08-11 hunt
spent three adversarial rounds and killed all eight candidates, and asks for a hand-added idea rather
than another speculative hunt. That remains the right next move; the debt is still open and is still the
oldest item in the file.

**The input question was asked, as principle #6 requires.** The answer is that this tool's input has to
be two files: the entire premise is comparing a training corpus against an eval set that both live on the
user's disk. The last eleven registry entries in a row now ingest a file and emit a file.

## What the research changed, before a line was written

Thirteen agents ran — six research lenses, each attacked by an independent refuter, plus a synthesist;
1.36M subagent tokens. **The pitch's engine, its headline and its market claim were each independently
falsified.** Two further gate agents then ran on the two questions that decide whether to build at all.

### 1. MinHash is the wrong primitive. Deleted.

Contamination is **asymmetric containment**, not symmetric similarity. Reproduced digit-for-digit by
three agents:

| eval words contained in a train doc of L words | Jaccard (13-word grams) | Containment |
|---|---|---|
| 100 in 150 | 0.638 | 1.00 |
| 100 in 400 | 0.227 | 1.00 |
| 100 in 1000 | **0.089** | 1.00 |
| 100 in 10000 | 0.0088 | 1.00 |

At J=0.089 the LSH candidate probability at b=8, r=4 is **0.0003** — the pair never becomes a candidate,
so no downstream stage can rescue it. Every published recipe (GPT-3 TOKEN-MATCH, PaLM NGRAM-MATCH,
Llama 2 TOKEN-EXTEND, ConTAM LONGEST-MATCH, Llama 3 §5.1.4) normalises by the evaluation sample. The
one-permutation-hashing and densification work the spec mandated was solving a problem that does not
exist: 128 permutations over 100k documents measured 11.2–26.2 s, about 5.6× over budget rather than the
claimed two to three orders of magnitude.

### 2. Character shingles were the most dangerous line in the spec

Measured on real text: 13-**character** shingles are shared by between 1.93% and 58% of *completely
unrelated* document pairs depending on the corpus; 8-**word** grams by 0.0020%; 13-**word** grams by
0.0000%. Roughly a thousandfold difference in noise. It also destroys the tool's main defence against a
false positive — showing the reader the shared passage — because at thirteen characters that passage is a
stopword phrase.

### 3. N must be adaptive, and this is load-bearing

GPT-3's own rule is the 5th-percentile eval row length in words, floor 8, ceiling 13. A **fixed N=13
reports "0% of your eval set is contaminated" on TruthfulQA**, 84% of whose rows are shorter than that —
a confident false negative on exactly the claim the tool exists to protect.

### 4. The flagship headline is arithmetically impossible

With overall score `o` and contaminated fraction `c`, the clean-subset mean is pinned to
`[(o−c)/(1−c), min(1, o/(1−c))]`. At o=0.91, c=0.12 that is [89.77%, 100.00%] — the largest possible fall
is **1.23 points**, and 91→84 requires c = 43.75%. The tool prints the bound in **both** directions
(GPT-3 Appendix C anticipates the upward one verbatim), gated on the metric being an unweighted mean of
per-row scores in [0,1], with a Wilson-interval gate that suppresses the comparison when the two subsets
are within noise.

### 5. A free client-side incumbent exists, and I read its code

The spec's "nobody sells this specifically" is false — SemHash (MIT) names this exact job in its README.
More seriously, **jsonlkit.com ships a free, no-signup, fully client-side train/eval leakage detector**.

The first gate agent died on an API error, so I ran this check directly and **read the shipped
`tool.js` (318,594 chars, fetched 2026-08-18)** rather than trusting a report:

- `initLeakageDetector` matches by **whole-key exact string equality** — `index[k]` on a normalised key,
  with optional `toLowerCase`, whitespace collapse and punctuation strip. Four modes: whole record, one
  field, first user prompt, whole conversation. **No n-grams, no containment, no partial or fuzzy
  matching of any kind.** An eval question inside a longer training document is invisible to it; so is
  one changed word.
- Output is `leakage-report.txt` only — **never a cleaned file** — while its own last line tells the user
  "Remove these from the evaluation set — or from training — before you report a score."
- `initNearDuplicates` is real MinHash (H=64, 16 bands × 4 rows, word shingles k=5, union-find) but
  **single-pane**, i.e. intra-file dedup, a different job.

So browser MinHash is *not* unattempted, and the shipped copy never claims it is. The narrow true claim —
the one the interface makes — is that no free client-side tool does **cross-file n-gram containment with
a cleaned-file output**.

### 6. The name changed

`leakspot` collides with a published academic JS memory-leak detector (doi 10.1002/spe.2406) and sits in
the breach-lookup SEO namespace where the ML meaning cannot rank. `evalspill` cleared four checks: npm
404, `evalspill.com` unregistered per Verisign RDAP, zero GitHub repos, empty SERP. Two proposed
alternatives were killed on inspection — `overlapcheck.com` is a live ETF overlap calculator and
`spancheck` is a golangci-lint linter with 83 stars.

## Tool Details

- **Name:** evalspill
- **Repo:** ben-gy/evalspill
- **Category:** developer-tools
- **Audience:** someone who fine-tuned a model this month and has a number they are about to repeat
- **Stack:** vanilla TypeScript + Vite 7, one runtime dependency (`fflate`)
- **Browser APIs:** `File.stream()`, fflate `Gunzip`, module Web Worker, typed-array open addressing,
  `File.slice()`, Service Worker (vite-plugin-pwa)
- **Worker strategy:** one module worker holding the whole scan; the eval index is in memory, the
  training file is streamed past it and discarded

## Privacy Model

- **Protected:** neither file is uploaded; no training bytes retained beyond byte counts and a bounded
  number of 240-character excerpts; no IndexedDB, OPFS or cookies; one `localStorage` key holding two
  booleans; works offline after first visit.
- **Not protected:** the exported report and CSV quote the user's own rows by design; absence of a
  finding is not a clean bill of health, because the base model's pretraining corpus is not in the
  comparison.
- **Trust surface:** the GitHub Pages bundle and its TLS chain; the Cloudflare Web Analytics beacon;
  feedback.benrichardson.dev only when the user opens the form and presses Send.

## Architecture Decisions

- **Word n-gram containment, streamed.** The eval side is indexed; the training file streams past it.
- **Two passes when the boilerplate filter is on.** The first counts how many distinct training records
  contain each eval run; the second applies only those under GPT-3's cap of `max(10, 0.1%)`. It cannot be
  done in one pass without either keeping every hit — one boilerplate phrase produces millions — or being
  unable to withdraw a phrase that turns out to be boilerplate near the end of the file.
- **Pure-JS rolling hash, not hash-wasm.** Measured 104–211× faster per shingle; the JS/WASM boundary
  dominates at ~50-byte payloads. verisum's hash-wasm precedent is whole-file hashing, the opposite
  workload.
- **The scan core has no `File` and no `self`,** so the tests drive the real engine over the real shipped
  sample rather than a second implementation that can agree with the first while both are wrong.
- **Stage 2 was cut on scope, not on doubt** — see "Honest gaps".

## The sample took three attempts, and that is the interesting part

1. **A slot grammar of ten openers, twenty subjects and ten qualifiers.** Reads fine, useless as a
   sample: **487 of 500 eval rows matched**, every one of them a *correct* detection of a genuinely
   shared eight-word run. A demo whose background noise drowns its signal teaches the reader that the
   tool cries wolf.
2. **The same grammar with rejection sampling.** Could not finish: a single constraint phrase is itself
   eight words drawn from twelve options, so after twelve rows every one of those runs is claimed and no
   further row can be generated.
3. **Structural rather than statistical.** A per-row reference is woven through each sentence so that no
   run of eight words can avoid one, and the generator *asserts* that invariant rather than trusting it.
   Two further collisions were then found by measurement and fixed: runs straddling the question/answer
   join (the answer now carries a leading reference) and runs straddling the boilerplate-footer junction
   (the footer is attached behind a fresh reference, so its own runs stay byte-identical across rows —
   which is what gives them a document frequency the filter can see).

The sample now yields **exactly the fifty planted rows and nothing else**, and plants twenty rewordings
that the tool provably cannot find, named by id in `public/samples/README.md`.

## Test Results

- Tests written: **170**
- Tests passed: **170**
- Tests failed: 0

Failures found and fixed along the way:

- Three unit tests were wrong rather than the code (a miscounted n-gram window; an affix fixture whose
  "varying" part was itself shared; a fixture whose `completion` was constant, which the near-constant
  detector correctly flagged).
- **The source-hygiene gate caught a literal NUL byte in a test file written during this run** — exactly
  the failure it exists for.
- **The structural no-verdict gate fired on the sentence "It does not tell you a model cheated"** — the
  denial the whole tool is built around. Narrowed to identifier-shaped matches.
- The `mark` contrast gate matched `.brand-mark {` by substring; anchored.
- `third-party-notices.txt` was absent from `dist/` because notices are generated *from* the build; the
  order is now build → notices → build.

## Build Status

- npm install: pass
- npm test: pass (170/170)
- npm run build: pass
- Local preview: pass
- Production workflow dry-run: pass

## Browser verification — measured, not asserted

Run against the production build on `localhost:5199`. **The stale-service-worker trap fired
immediately**: the first load served epochcost's shell, because tools share the port. Unregistered and
hard-reloaded before anything was believed.

- The shipped sample yields **exactly 50 of 488 scorable rows (10.2%)** through the real worker, matching
  the pinned unit test. N chosen as **13** by the GPT-3 rule; **12 rows reported too short to score**
  rather than counted as clean.
- The bound renders as **90.0% → 88.9% … 100.0%, at most 1.1 points down**, and the measured clean subset
  lands **exactly on the bound's floor** — because the flagged rows all scored 1, which is the extreme
  case, and which makes the sample the most instructive possible demo.
- **9 word runs dropped as boilerplate** — exactly the number of 13-grams in the 21-word planted footer.
- **3 exact duplicate training rows** reported as their own fact.
- `eval.clean.jsonl`: 450 rows, **zero byte-exact mismatches** against the original file, dropping
  precisely the 50 plants. Checked by comparing the literal line strings, not the parsed objects.
- `contamination-report.csv`: 10 manifest lines + header + 50 rows, **UTF-8 BOM verified on the BYTES**
  because `Blob.text()` strips it by spec. `too-short-to-score.csv`: exactly 12 data rows.
- `overlap-report.html`: 40 KB, **0 scripts, 0 links, 0 `src` attributes, 0 `on*` handlers**, 50 marks.
- **All three modals pass all five dismissal exits at 375px** — × control, backdrop click, Escape, a real
  touch-type `PointerEvent` sequence, and the keyboard path — 15/15, at 44×44 hit areas. The event-log
  drawer passes its own exits and there is still exactly **one** drawer node after repeated re-renders.
- **The feedback widget's button survives four full re-renders**, proving the footer memoisation is
  load-bearing rather than decorative.
- A full run issues requests for the page's own assets and its own samples and **nothing else**.

### A mobile defect found in the browser and fixed

At 375px the four numeric columns ate the width and pushed the shared passage — the one thing on the page
the reader must be able to read — into a horizontal scroll they had to discover. The match table now
stacks into cards below 40rem. Re-verified: the page body does not scroll horizontally at 375px and the
passage is fully legible in both the eval row and the training excerpt.

## Honest gaps

- **Stage 2 — the semantic detector for reworded rows — is not in this build.** Its asset was extracted
  and **verified against the real ONNX runtime to 3.62e-8** and calibrated (AUC 0.953, precision@5 62.5%,
  median rank 3 of 247), and the work turned up a finding the research pass missed: mean-pooling
  converges to the corpus centroid, so background cosine reaches **0.789 at 4,000 characters** and
  **chunking is mandatory** — whole-row top-5 collapses to 5/40 on long rows against 25/40 chunked. But
  it needs a 5.8 MB vendored table and a second engine, and shipping both half-finished serves nobody.
  Recorded in EXPANSION_IDEAS.md with every measurement intact. **The interface states plainly that
  reworded rows are not found**, rather than implying coverage it does not have.
- **Removing rows from the training side is not offered**, and the interface says why: it needs another
  full pass over a file that may be gigabytes, and rewriting it inside a tab is not something to do
  casually.
- **The analytics beacon could not be observed delivering a request** from the sandboxed preview, so it
  is verified structurally (present in `dist/index.html`, both origins in both CSP directives, asserted
  by a gate). The feedback widget *was* observed mounting.
- **Desktop screenshots came back blank whenever the page was scrolled** — a Browser-pane capture
  artefact, since the same content captured correctly at 375px and at scroll position 0. Desktop layout
  below the fold was therefore verified by DOM geometry and text extraction rather than by eye.
- **No real multi-gigabyte file was scanned.** Throughput figures in the research (≈150 MB/s single
  worker) are from node/V8 and are not quoted to the user; the interface measures and shows its own
  MB/s and ETA from the file in front of it.
- The Unicode separator set is the punctuation that turns up in prose, not the full `\p{P}` category — a
  `\p{P}` test per code unit is far too slow for this loop. Stated in the source rather than hidden.

## Deployment

- Repo created: yes — https://github.com/ben-gy/evalspill
- GitHub Pages enabled: yes (build_type=workflow)
- DNS record created: yes (CNAME evalspill → ben-gy.github.io)
- TLS certificate: **approved on the first poll**
- Live and serving 200 with the correct title, samples, manifest and og.png
- Workflow: completed/success
- Directory entry live on main: yes

## Errors & Resolutions

| Error | Resolution |
|---|---|
| Two subagents died with "API Error: Connection closed mid-response" (the stage-2 extractor and the incumbent gate) | Relaunched the extractor; ran the incumbent check directly with curl and read jsonlkit's shipped `tool.js` myself, which produced better evidence than the agent would have |
| Sample generator: 487/500 rows matched | Rebuilt with per-row references woven through every sentence and the invariant asserted |
| Sample generator: could not finish under rejection sampling | Same fix — structural, not statistical |
| `tsc` error: `Json | undefined` not assignable | Explicit undefined check in `resolvePath` |
| Source-hygiene gate: literal NUL in `tests/stats.test.ts` | Replaced with `String.fromCharCode(0)` |
| No-verdict gate fired on the tool's own disclaimer | Narrowed to identifier-shaped matches |
| Stale service worker served epochcost's shell on `:5199` | Unregistered all workers and cleared caches before verifying anything |
| Match table unreadable at 375px | Stacks into cards below 40rem |
| A 1-character "shared suffix" reported as a finding | Minimum affix length of 8, with tests both ways |
