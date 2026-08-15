# Build Log: crowdsize
**Date:** 2026-08-16
**Status:** deployed

## Idea Source

`IDEAS.md` queue entry **crowdsize** — *"Before you email that spreadsheet of people to
someone, find out how many rows identify exactly one person…"*

**Queue order was not followed, for the reason the file itself supplies and for the third
run running.** The top three entries — `epochcost`, `leakspot`, `topicgap` — are all still
blocked by standing-note rule 2: `epochcost`'s exact-token-count engine is chunkforge's
(2026-08-10, six days) and its streaming-NDJSON-through-a-worker-pool-to-a-cost-figure
architecture is runledger's (2026-08-09, seven days); `leakspot` and `topicgap` both rest on
chunkforge's static-embedding engine. `crowdsize` is the first unblocked entry, which is
exactly the `ghostdep` (08-11) and `unfence` (08-15) precedent.

**The sensor debt was considered and deliberately not paid.** IDEAS.md records that the
2026-08-11 hunt spent three adversarial rounds and ~2M tokens killing all eight candidates,
and asks for a hand-added idea rather than another speculative hunt. That remains the right
next move; the debt is now the oldest item in the file (last sensor tool: tiltwell,
2026-08-02).

**The input question was asked.** crowdsize takes a file, and it has to: the whole premise
is measuring re-identification risk in *your* spreadsheet of people. There is no sensor
version of that question.

## THIS RUN RESUMED AN ABANDONED BUILD

A near-complete `tools/2026-08-14-crowdsize` was found in another worktree
(`.claude/worktrees/gracious-roentgen-53920b`). That run wrote the engine, the interface,
the sample generator and 93 tests, produced a clean `dist/`, and **died at step 8b** — no
README, no third-party notices, no repo, no registry entry, and `package.json` referencing
two scripts that were never written. This is the `docxray` precedent (2026-08-12/13): the
directory keeps its original date, the registry records the ship date.

Nothing was taken on trust. The resumed build was put through the same adversarial review a
fresh one would get, and 33 findings came back confirmed.

## Tool Details

- **Name:** crowdsize
- **Repo:** ben-gy/crowdsize
- **Category:** privacy
- **Audience:** a research coordinator at six in the evening with a participant list a
  collaborator needs tonight, who has been told "make sure it's anonymised" by someone who
  did not say what that means, and who is the only person who will be blamed if it is
  wrong. Also DPOs, IRB submitters, public-health analysts, council FOI officers.
- **Stack:** vanilla TypeScript + Vite 7, one module Web Worker, **zero runtime dependencies**
- **Browser APIs:** `File.stream()` + `TextDecoderStream`, module Web Worker, `Int32Array`
  throughout, `URL.createObjectURL`, `navigator.share`, Service Worker (vite-plugin-pwa)
- **Worker strategy:** ONE worker, not a pool — see below

## Privacy Model

- **Protected:** the file is read, scored and rewritten in the tab; no upload endpoint
  exists in the code and no network request is made while it runs. No IndexedDB, no OPFS,
  no cookies, no sessionStorage; one `localStorage` key holding two clamped integers.
- **Not protected:** the exported files — `suppressed-rows.csv` is *by design* the rows that
  identify somebody and is labelled as such; anything typed into the feedback form; and the
  released file once released.
- **Trust surface:** the static bundle on GitHub Pages and its TLS chain, the Cloudflare Web
  Analytics beacon (anonymous page views), and `feedback.benrichardson.dev` on Send.
  Nothing else — no model, no CDN, no font, no remote data.

## Architecture Decisions

**One worker, not a pool — for correctness, not cost.** The obvious optimisation is to slice
the file into N byte ranges. It cannot be done safely: from an arbitrary offset you cannot
know whether the next `"` opens a field or closes one, because quote parity depends on every
byte before it, and you cannot resynchronise on a newline because inside a quoted field a
newline is ordinary data — which is exactly how a free-text notes column behaves. (The
research lens proposed a quote-parity prescan to make the parse shardable, measured at
5.53× on 48 MB. It was not adopted: the shipped ceiling is far below where that pays, and it
trades the one property this engine sells — exactness — for throughput nobody has asked for.)

**No `Map<string, count>` anywhere in the counting path.** The tool's worst case *is* its
headline case — a mostly-unique file — and a million distinct groups is a million retained
string keys. Dictionary-encoded columns plus open addressing over three `Int32Array`s: 24
bytes per row, allocated once. **The hash is never the key**; a match is confirmed by
comparing integer codes, so class counts are exact and two groups can never merge into a `k`
that is too high.

**Rows are not people, and crowdsize says so before it says any number.** A claims,
encounter or trial-visit extract has many rows per person; five rows sharing a birth year
and postcode district may be one population-unique person, and a tool counting rows prints
k=5 over a singleton. The error runs in the fatal direction.

## Test Results

- Tests at start of run (inherited): **93**
- Tests written this run: **65**
- **Tests passing: 158 / 158**
- New files: `tests/build-gates.test.ts` (28), `tests/csv.test.ts` (18),
  `tests/sample.test.ts` (8), plus additions to `ladder`, `parse`, `classes` and `score`

## Build Status

- npm install: pass
- npm run build: pass
- npm test: **pass (158)**
- Local preview: pass
- Production workflow dry-run: **pass** — see below

## The review, and what it changed

Two adversarial workflows ran: **five research lenses** each attacked by an independent
refuter (11 agents, 1.67M tokens), and **five code-review dimensions** each verified by a
skeptic that had to reproduce or refute every finding (10 agents, 1.73M tokens). **33
findings were confirmed by reproduction.** The refuters correctly *rejected* two findings
that had already been fixed mid-run, which is the loop working.

### The four HIGH findings

1. **The tool asserted a person basis it had guessed.** Until the user answers *"does each
   row describe a different person?"*, `oneRowPerPerson` is null — and the score request sent
   the auto-guessed person key anyway. The worker reported `basis:'person-key'`, the
   interface withdrew its own "these are rows, not people" warning, and READ-THIS-FIRST.txt
   stated *"counted in PEOPLE"* as fact in a file destined for an ethics committee. The fix
   is one character (`=== false`, not `!== true`). Verified in the browser: the headline now
   reads "47 **rows** are each the only one…" until the question is answered.

2. **The homogeneity pass allocated one JS array per equivalence class** — on a mostly-unique
   file, one per row. Reproduced at **302 MB** of retained heap at a million rows, re-running
   on every drag of a rung, in the one file whose sibling's header comment explains why that
   allocation was designed out. Replaced with a prefix-sum + single `Int32Array` layout and a
   bounded top-50. **Re-measured: 0.1 MB, 114 ms** — and a test proves the *strongest*
   findings survive compaction rather than the first fifty seen.

3. **Every interaction destroyed keyboard focus.** `render()` opens with
   `app.replaceChildren()` and every control calls it, so tabbing to a column checkbox and
   pressing Space left focus on `<body>` with the operated element detached. Selecting a
   quasi-identifier set — the tool's central task — was effectively impossible by keyboard.

4. **The result was never announced.** Zero `aria-live` regions existed. A sighted user
   watched 12,474 become 47; a screen-reader user was told nothing. The region now lives
   *outside* `#app`, because one created inside it is destroyed by the same
   `replaceChildren` that makes it necessary.

### Notable others

- **The feedback link died on the first interaction** — the memory note's exact trap,
  reproduced by planting a marker in the footer and watching it vanish after one click.
  `footer()` is memoised. **The gate meant to catch this was itself wrong**: it counted call
  sites and passed, because the single call sits inside `render()`.
- **The formula-injection guard corrupted every negative number**, turning `-3.5` into text
  in Excel. Complete numeric literals are now exempt (safe: Excel evaluates `-3.5` to
  `-3.5`), while `-1+1` and `@SUM(1+1)` are still guarded. `src/csv.ts` — which writes every
  artefact — had **no test file at all**; it now has 18.
- **The ZIP3 list was six prefixes short.** Safe Harbor tests against *"the current publicly
  available data"*, and HHS's citable seventeen are from the 2000 census. `369` (rural
  Alabama, ~17,300 residents, stable across three ACS vintages) and five others were being
  released. The conservative union of 23 ships, with each list's provenance stated.
- **l-diversity counted a blank as a value**, so a group where everybody shares one diagnosis
  and one record is empty scored l=2 and passed — the precise attack the metric names.
- A blank line emitted a phantom all-empty row inflating both the row count and the all-blank
  class; `ageBand` filed sentinel ages (999/888/150) into the statutory "90 or older"
  category; `dateParts` accepted `25/13/2019`; `pickPersonKey` sorted **ascending**, so an
  organisational id beat the patient id; the year rung claimed to produce "a Safe Harbor
  output" when the rule also removes the year where it indicates an age over 89;
  `--text-tertiary` measured **3.60:1** and failed WCAG AA on every background; close buttons
  were `pointerdown`-only so Enter did nothing; glossary terms were focusable spans that
  ignored the keyboard; Enter on "Try a sample" opened the file picker instead; and *"the
  previous file is gone from memory"* was false — the worker kept the whole encoded dataset.
- **The source-hygiene gate caught two real control bytes**: a NUL keying the reference
  implementation the counter is verified against, and — once widened from `src/` to `tests/`
  — one written during this run.

### Research findings that did *not* require a change

The kill check survived, reproduced twice: **no free, no-signup, client-side browser tool
computes a k-distribution on a local CSV.** Amnesia's demo POSTs to its own server; Redivis
requires the data already sit on its platform behind OAuth; the free "anonymise a csv" field
is per-column *maskers*, which is a different operation.

Three claims in the queue spec were refuted and **none had reached the shipped code**: the
fabricated "$125,000 penalty", "there is no browser option at any price", and the proposed
electoral-roll headline (which asserts population uniqueness from a sample measurement
against a register holding neither birth year nor sex). The 08-14 build had already declined
all three, and had already cut t-closeness for the right reason.

## Verified in the browser, not asserted

- The shipped sample yields **exactly 47 singletons** on {birth year, sex, postcode
  district}, measured through the real parser and counter — and pinned by a test.
- A hostile CSV with `<img src=x onerror=…>` as a column header produces **zero** live
  elements and **zero** `onerror` attributes; the payload renders as text.
- The exported report is 8 KB with no `script`, `link` or `src`, carries all three
  limitations, and contains **no record-level row**.
- Each artefact is **one download per click**.
- **All four overlays pass all four dismissal exits at 375px** with real touch-type
  PointerEvents, plus the new keyboard path — 20/20, at 44×44 hit areas, with `[hidden]`
  clean at baseline and the page reachable after every close.
- A full run issues requests for the page's own assets and its own sample, **and nothing
  else**.
- State resets cleanly on a second file.

## Honest gaps

- **The sample's l-diversity demo is weak at the default threshold.** Its largest homogeneous
  group is **three**; there is none at k=5, so somebody who never moves the slider sees the
  panel find nothing on the demo file. Measured, pinned by a test with the number in it, and
  left alone rather than regenerating the sample and risking the 47-singleton guarantee that
  the headline and every browser check rest on.
- **The analytics beacon and the feedback widget could not be exercised.** The sandboxed
  preview browser has no external network (`ERR_CONNECTION_REFUSED`), so both are verified
  *structurally* — the script tags ship, the CSP names both origins in both directives, no
  module import double-mounts the widget, and the footer node survives a re-render — but
  neither was seen to load. They can only be confirmed on the live domain.
- **No real-device check.** All browser verification ran in the preview browser at 1280×900
  and 375×812. Nothing was tested on an actual phone.
- The remaining LOW findings were left: the stat row prints `k = rowCount` when no
  quasi-identifiers are selected, exposed records are labelled "row N" by data index rather
  than spreadsheet row, and the homogeneity card mixes people and rows in one sentence.

## Deployment

- Repo created: yes — https://github.com/ben-gy/crowdsize
- GitHub Pages enabled: yes (`build_type: workflow`)
- Cloudflare DNS record created: yes — `crowdsize` → `ben-gy.github.io`
- Pages custom domain set: yes — `crowdsize.benrichardson.dev`
- Certificate: polled on `.https_certificate.state` (never `https_enforced` — see
  `ssl-root-cause-analysis.md`, Cause 1)
- Directory entry live on main: yes
- Workflow triggered: yes

## Errors & Resolutions

- **`preview_start` found port 5199 held by another process.** Moved to a dedicated port
  (5231), which also sidesteps the shared-port stale-service-worker trap in the memory notes.
  The stale SW *did* bite once mid-run — a fix appeared not to work until the registration
  was unregistered and caches cleared.
- **The preview tab is hidden, so `setTimeout` is throttled to ~1s.** Two verification
  scripts timed out at 30s and one left an overlay open, producing a spurious "Privacy modal
  fails every exit" reading. Re-run cleanly one overlay at a time: all pass.
- **`console.log` is stripped from tests** by the `esbuild.drop` config, so a measurement
  script silently produced no output; the numbers were written to a file instead.
- Review agents left scratch test files behind (`tests/_probe`, `zz-*.test.ts`), including one
  asserting the a11y defect as intended behaviour. Removed; the useful half was rewritten as
  a real regression test.
