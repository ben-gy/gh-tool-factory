# Build Log: Backslide
**Date:** 2026-08-04
**Status:** deployed

## Idea Source

`IDEAS.md`, entry **backslide** — but it is the *fifth* entry in the queue, and the four skipped
above it were skipped under standing-note rule 2 (never ship two tools from the same engine group in
the same week), not by preference:

- **runledger** and **epochcost** — `@huggingface/jinja` + `@huggingface/tokenizers` group, deferred
  behind **chatprint** (2026-08-02) until 2026-08-09 by a note already in the file.
- **chunkforge** and **leakspot** — text-embedding group, named in the standing note alongside
  **searchpack** (2026-08-02).

backslide's own spec named `static-retrieval-mrl-en-v1` for a *demoted* semantic-drift column, which
would have collided twice over: the same model and the same byte count as searchpack two days
earlier, and the same transformers.js + onnxruntime-web + WebGPU + cosine-boot-self-check shape as
**loraprep**, which shipped the same day. Rather than reorder the queue again, the collision was put
to an adversarial review, which found the neural path was not merely avoidable but **wrong** — see
below. Shipping with no model at all cleared the conflict and improved the product.

## Tool Details

- **Name:** Backslide
- **Repo:** ben-gy/backslide
- **Category:** developer-tools (indexed under `data-explorers`)
- **Audience:** an engineer with a PR open, two eval output files, and a pass rate that moved
- **Stack:** Vite 7 + vanilla TypeScript + Vitest
- **Browser APIs:** Web Workers (module), File API + drag-and-drop + file input, File System Access,
  Web Share (level 2), Clipboard, Service Worker
- **Worker strategy:** one module worker holding the entire pipeline **and the parsed rows**, so a
  settings change recomputes over memory rather than re-parsing
- **Bundled dependencies:** three — `@cfworker/json-schema`, `diff`, `fastest-levenshtein`

## Privacy Model

- **Protected:** both files are read, joined, scored and exported in the tab; no code path puts file
  content into a network request. Verified in the dry-run — every request was same-origin.
- **Not protected:** GitHub Pages logs the page request; the downloaded artefacts quote model output
  verbatim and are as sensitive as the inputs.
- **Trust surface:** the static bundle and its TLS chain, plus the Cloudflare beacon and the feedback
  widget — both named in the UI, and the only two origins the CSP permits. The landing copy
  deliberately does **not** claim "no network requests"; it claims none of them can carry your files
  and points at the policy as the thing to check.

## Architecture Decisions

**Why no model, in one line:** the join key is the *input*, and A and B are the same prompts run
twice — so the strings are identical or near-identical by construction, semantic similarity buys
nothing on identical text, and a lexical score can be shown to the user in a way a cosine cannot.
The measured decomposition on an adversarial set was 0 genuine measure failures; every error was a
tie between byte-identical inputs, which an embedding model ties on identically.

**The join is the product.** Four stages, none trusted blind, because if the two rows were never the
same case then every number afterwards is fiction.

**Scorer precedence over independence.** Making the checks disjoint is what lets the headline be
generated from the summary instead of transcribed — which is the difference between a demo whose
arithmetic holds on the first click and one that does not.

**Trajectory alignment is the differentiator and was treated as such.** Every agent-eval library
scores against a hand-written expected sequence; nothing diffs two real runs.

## Adversarial review — 9 blockers, all fixed before a line of UI was written

A 10-agent workflow (5 verification lenses → architect → 4 refutation lenses, all Opus 5) ran before
the build. Everything below was **executed against the real libraries**, not reasoned about.

1. `jsonrepair` throws on `"Sure! Here's the JSON:"` and succeeds on truncated JSON → replaced with a
   first-party structural classifier; jsonrepair is no longer a dependency.
2. `is-json` and `errored` overlap → strict precedence; columns now sum to the total.
3. `js-rouge` throws on empty input, scores identical text 0.833, has no ESM default export, and
   drops 0.50 on a harmless paraphrase → cut entirely.
4. `cost-regression`'s floor was a tautology and `promptfoo import` writes `latencyMs: 0` → both
   numeric scorers require a non-zero baseline; cached rows are `n/a`.
5. Fanout identity contained the prompt text → ordinal identity, text is a label.
6. Positional case ids silently mis-join a shifted dataset → the key join is verified and discarded
   when the ids and the inputs disagree.
7. Ambiguous groups made the count bijection-dependent → permutation-invariant group delta.
8. Unbounded quadratic join and 3× memory amplification → hard caps stated in preflight.
9. The landing copy claimed no server while loading two third-party scripts → the copy names them.

Plus: `shortCircuit: false`, `diffJson` on parsed objects, ordinal-keyed carried assertions, jointly
resolved field mapping, CSV formula-injection guard, and Markdown for the PR artefact because
GitHub's sanitiser strips `<style>`.

## Test Results

- Tests written: **70** across 4 files
- Tests passed: **70**
- Tests failed: 0 at ship

Four failures during development were **real bugs, fixed in the code rather than the assertions**:

1. The stage-4 margin test rejected correct pairings when computed in-pass, and accepted
   `case 2` ↔ `case 22` when computed only against unmatched rows. Settled on a margin against every
   other candidate for either row, which abstains readily — the intended trade, since stages 1–2
   resolve the ordinary cases by hashing.
2. `verifyKeyJoin` did not catch a shifted `testIdx` because formulaic inputs are 90% alike. Replaced
   the similarity median with a comparison of identical inputs *available* against identical inputs
   the ids actually *lined up*.
3. `is-json` reported a confident pass/fail on prose-only files → now `na` when neither run was
   attempting JSON, which is also what makes the capability check fire correctly.
4. The pinned `is-json = 14` was transcribed from the idea file; the pipeline measures **16** (14
   wrapper rows plus the 2 inserted-step rows that also broke). The test now reads the number from
   the pipeline and asserts the arithmetic reconciles.

## Build Status

- npm install: pass
- npm test: pass (70/70)
- npm run build: pass — 48 kB JS gzipped 17.5 kB, 891 KiB precached including samples
- Local preview: pass
- Production workflow dry-run: **pass**

## Browser dry-run (localhost:4196, production build)

- Sample click → headline in **653 ms**: *"20 rows got worse. 16 of them stopped returning valid
  JSON."* Chips 16 + 4 = 20 regressions, 5 errors reported separately. Figures reconcile to 300.
- Trajectory drill-down renders `==+=============` with the divergence-not-regression sentence.
- Second sample (no ids anywhere): *"Joined on full text · 36 exact · 2 by similarity"*, 2 dropped,
  2 added, and the truncated-JSON row correctly found. One paraphrase deliberately left unmatched.
- All four artefacts captured through the **real** download path (`URL.createObjectURL` intercepted),
  all well-formed.
- Modal escape-hatch test at 375px: all four exits pass on all four modals **and** the drawer, using
  real touch-type `PointerEvent`s. Baseline `[hidden]` scan clean; `elementFromPoint` at the centre
  returns page content before and after.
- PWA: manifest resolves, maskable icon present, `apple-touch-icon.png` 200 and **verified opaque by
  pixel** (min alpha 255) rather than by colour-type guess.
- No console errors. **Every network request same-origin** — which is the privacy claim, checked.

## Errors & Resolutions

- **A stale service worker from `loraprep` was serving the shared preview port** and returned
  loraprep's HTML for backslide's URL. Unregistered and cleared caches before testing. (The known
  fleet trap; caught immediately because the title was checked first.)
- **Background-tab timer throttling** made a `setTimeout`-heavy verification script exceed the 30 s
  budget. Rewritten with synchronous handlers and no timers.
- **Two sample-generator bugs found by the tests, not by inspection:** the no-ids sample reused only
  10 prompts across 40 rows, so the join correctly refused to pair them and the demo showed
  abstention instead of matching; and the JSON-flip rows carried an `is-json` assertion that already
  caught them, so `carried-assertion` owned every row and the tool's actual point was hidden. The
  sample now models an eval that only asserts `contains: category` — which most real evals do — so
  the user's own eval passes all 16 broken rows and Backslide catches them.
- **`jsonrepair` was in `dependencies` but absent from the build**, because the structural classifier
  replaced it. Moved to `devDependencies` (where it pins the measured defects) and the About modal
  corrected — it had claimed a repair-preview feature that does not exist.

## Deployment

- Repo created: yes
- GitHub Pages enabled: yes
- Cloudflare DNS: yes — `backslide` CNAME → `ben-gy.github.io`
- TLS certificate: **approved** (polled `.https_certificate.state`, approved on the 2nd check)
- Deploy workflow: completed success
- Live check: `https://backslide.benrichardson.dev/` → 200, correct title, samples and manifest 200
- PR created: https://github.com/ben-gy/backslide/pull/1
