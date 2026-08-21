# Build Log: quotewitness
**Date:** 2026-08-22
**Status:** deployed

## What this run actually was

**A resumed build, not a fresh one.** The queue's top entry, `veracite`, was built on 2026-08-21 in a
sibling worktree under the name **quotewitness**. That run got as far as `gh repo create`, pushed, and
then died — the deploy workflow ran green and the TLS certificate issued, so the tool has been LIVE at
quotewitness.benrichardson.dev since 2026-08-20T20:24Z, but it was never reviewed, never verified in a
browser, and never registered. It appeared in no `registry.json`, no `index/tools.json`, no
`index/tools.txt` and no log. Live and invisible.

This is the deploy-backlog failure mode, and the memory note on it is exact: a worktree run hides its
build in `.claude/worktrees/*/tools/`, which a main-repo `tools/` diff misses entirely. It was found by
listing `.claude/worktrees/*/tools/` before researching anything new.

**The resumed build was reviewed as hard as a fresh one, and that was the right call.** Six independent
review lenses raised 36 findings; each went to a separate agent whose only job was to refute it.
**34 survived**, 5 of them critical. 42 agents, 3.2M subagent tokens.

## Idea Source

`IDEAS.md`, top queue entry, `veracite`. Standing-note rule 2 was checked: the nearest engine relative
is evalspill (2026-08-18, four days), which is also a text-overlap tool — but evalspill measures
asymmetric n-gram *containment* between two corpora to find contamination, with no alignment, no diff
and no page model. quotewitness confirms candidates with a banded semi-global aligner to produce a
word-level diff and attributes each match to a printed page. Different job, different engine. The name
`veracite` was dropped during the 08-21 vetting.

**The input question (principle #6) was asked and the answer is honest: this tool has to take files,**
because the whole premise is checking a draft against the sources on your own disk. **The sensor debt
recorded on 2026-08-10 is therefore still open and is now the oldest item in IDEAS.md** — the last
sensor tool remains tiltwell, 2026-08-02, twenty days ago. The 2026-08-11 hunt killed all eight of its
candidates and asked for a hand-added idea rather than another speculative hunt; that is still the right
next move and nothing in this run pays the debt down.

## Tool Details
- **Name:** quotewitness
- **Repo:** ben-gy/quotewitness
- **Category:** documents
- **Audience:** a postgraduate at 11pm with a chapter draft and a folder of PDFs; also journalists
  checking a piece against transcripts, and editors checking a submission. Deliberately not litigators.
- **Stack:** vanilla TypeScript + Vite 7. Three runtime dependencies: `pdfjs-dist`, `mammoth`, `fflate`.
  The normaliser, tokeniser, five-gram index, aligner, word differ, quotation extractor, folio
  harvester, CSV writer and HTML report writer are all first-party.
- **Browser APIs:** pdf.js text layer with per-item geometry; module Web Worker for index + alignment +
  DOCX/EPUB; `Intl.Segmenter`; drag-and-drop with `webkitGetAsEntry` directory recursion; clipboard
  paste ingest; Blob download; Cache API via vite-plugin-pwa; `fetch` for identifier resolution, off by
  default.
- **Worker strategy:** pdf.js on the main thread (it owns its own worker; nesting measured 2.3–3.4x
  slower), one dedicated module worker for everything else.
- **Storage:** none. A build gate asserts IndexedDB and OPFS never appear in `dist/**/*.js`.

## Privacy Model
- **Protected:** the draft and every source file. Read with `arrayBuffer()`, parsed in the tab, never
  uploaded in any mode. Every artefact is generated in the tab.
- **Not protected:** with "Check identifiers online" ticked (off by default), the DOIs, PMIDs and arXiv
  ids *from the draft* go to four public registries, which learn which papers you cite. Nothing else is
  sent. The page load itself is served by GitHub Pages, which sees your IP.
- **Trust surface:** the static bundle and its TLS chain; the Cloudflare Web Analytics beacon; the
  feedback widget when used; and those four registries when the switch is on.
- **Two privacy overclaims were found and corrected this run** — see findings 33 and 34 below. The
  privacy copy is the one place a privacy-first tool cannot afford to be approximately right.

## Architecture Decisions

Carried from the 08-21 plan and re-verified: word 5-grams over 32-character shingles (6.06 MB index vs
44.23 MB, recall 0.998 vs 0.983, 13.6x lower alignment cost); banded semi-global alignment so a leading
gap is free on the source side only, which stops a local alignment truncating a quote at its first
mismatch and calling the truncated span exact; and the printed-folio harvester as PRIMARY with
`getPageLabels()` as corroboration only — the natural "trust /PageLabels, fall back to harvesting"
policy is wrong on 75 of 181 ground-truth pages.

New this run: pdf.js is given `standardFontDataUrl`. The original comment recorded that extracted
strings are byte-identical with and without it, which is true on the sample — but this tool derives its
word separators from `item.width`, and that width comes out of the font program, so a failed font load
is a geometry failure waiting to happen. Measured six warnings before, zero after.

## The 34 confirmed findings

Five critical, all of which produced the tool's declared worst output — a false NOT FOUND, which reads
as an accusation that someone fabricated a quotation:

1. **`tokens.ts`** sized its token arrays at one token per two characters while emitting one token per
   character for CJK/kana/Hangul. Writes past a typed array are silent, so from the 17th character of
   any spaceless-script passage every token read back `undefined` and hashed to the bare FNV basis.
   Measured: a verbatim 25-character Chinese quotation returned `not-found` in one source and `near`
   with two fabricated edits in another, and an altered quotation scored identically to a correct one —
   the tool could not distinguish them at all. The corruption threshold (17) sits directly above the
   CJK adjudication floor (16), so effectively every adjudicated CJK quotation was affected.
2. **`quotes.ts`** accepted a word-internal U+2019 as a closing quotation mark, so every quotation in
   British single-quote style was cut at its first contraction or possessive. The missing tail was then
   rendered as an `<ins>` — the report told the author they had inserted a word that is in fact in the
   source.
3. **`layout.ts`** lost gutter detection whenever one full-width item sat on a two-column page (a title
   block, a spanning figure, a centred running head), then read the two columns line-by-line across the
   gutter, making every multi-line quotation on that page unmatchable.
4. **`pdf.ts`** only probed for an image-only page when it yielded exactly zero characters, so a
   photographed page carrying a stamp or a page number was recorded as fully searched. Worse: a wholly
   stamped scan cleared the median-character floor, then had its repeating stamp stripped as running
   furniture, leaving a "searchable" source whose text was literally empty.
5. **`match.ts`** dropped a source that failed to open from the corpus entirely, so the "one of your
   files is unreadable, so I will not guess" abstention could not see it and the tool asserted NOT FOUND
   about a file it had never read.

Twelve high, twelve medium, five low. The rest, briefly: an ellipsis segment shorter than five tokens
was deleted from the check while the quotation was still reported Exact; an unfound ellipsis segment was
charged a flat 2 edits regardless of length, so a half-invented quotation read as "Near match"; a
blockquote containing any inner quotation was discarded and only the inner fragment reported; plain-text
sources got no repair for words the typesetter broke across a line, so a verbatim quotation read as
"Substantially altered"; the pinpoint was scraped from 160 unanchored characters of following prose, so
"p53" parsed as page 53; an empty sheet stole the previous sheet's text and shifted every quotation one
page late; the global paste handler swallowed pastes into the feedback widget and silently replaced the
loaded draft; **none of the four header overlays could be opened from the keyboard at all**; there was no
live region anywhere, so errors and completion were never announced; a network failure was reported as
the verdict "unresolvable" and painted the same red as a fabricated quotation, so every real DOI in a
bibliography was accused when the user was offline; the exported report dropped the per-page "no text
layer" warning the screen showed and described a file that would not open as a scan; the CSV had no
UTF-8 BOM, so Excel mojibaked every curly quote; the drop zones were `role="button"`, which hides the
accepted-file list from screen readers; "Drag the whole folder of them here" was impossible and failed
silently; the near-match slider was destroyed mid-drag by a full re-render; six palette pairs failed
WCAG AA; the privacy modal claimed "Nothing is written to disk" while the service worker precaches
2.5 MB; and `api.semanticscholar.org` was in the CSP, the privacy modal and the README as one of "five
public registries" although no code path ever contacted it.

**Two findings were refuted** and correctly not acted on.

**Three defects were found by the fixers themselves, beyond the review:** a gap of exactly zero between
ellipsis segments was treated as disorder, demoting a correctly-copied passage to "Near match" with zero
edits and an empty diff; the abstention fired only on a wholly-missed quotation rather than on any
unplaced segment; and two verdict badges — the EXACT and NEAR MATCH chips, i.e. the headline output —
measured 3.57:1 and 3.73:1 against their own backgrounds.

**Two further items were found during integration:** the vendored pdf.js standard-14 fonts ship their own
`LICENSE_FOXIT` and `LICENSE_LIBERATION` (the Liberation faces are OFL, not Apache), so redistributing
them as site assets needs those notices carried — `make-notices.mjs` now assembles them mechanically
from the on-disk files, per the no-model-transcribed-licenses rule. And the page-level scan flag never
reached a verdict: flagging the page made the *preflight* honest while a NOT FOUND over a book with a
photographed plate stayed bare and confident. That qualification now travels with the verdict, in both
the screen and the exported report. It is deliberately a qualification and not an abstention — a
400-page book with one plate would otherwise never be able to answer anything.

**One improvement beyond the findings:** a cited page range (`pp. 411-12`) was made to abstain by the
fix for the pinpoint defect, which gave up the check on exactly the quotation a range exists for — the
one that crosses the page break. Ranges are now carried and checked across the span they name, with
Chicago-style elision handled and a notation change or descending pair refusing to invent a range.

## Test Results
- Tests before this run: **191**
- Tests after: **318** (127 added: 17 + 21 + 39 + 29 + 21 across five regression files)
- Failed at the end: **0**
- Every regression test was written failing first. The extraction suite was explicitly run against the
  pre-fix code: **25 of its 37 tests fail there**, and the 12 that pass on both sides are labelled
  `GUARD` and exist to pin behaviour the fixes had to leave alone.

## Build Status
- npm install: pass
- npm test: pass (318/318)
- tsc --noEmit: pass
- npm run build: pass, including `scripts/gate.mjs` (no `console.*`, no IndexedDB, no OPFS in `dist/`,
  and a new closed check that every `https://` host in the CSP `connect-src` has a real call site)
- Local preview: pass
- Production workflow dry-run: pass

## Browser verification (localhost preview, real production build)

- Sample loads through the real ingestion path; full run completes: 10 quotations, 5 verified, 2 near
  match with word-level diffs, 3 correctly abstained on because one bundled source is an image-only scan.
- The abstention is honest in the UI: "reading-notes-scan.pdf has no text layer, so I cannot tell you
  whether this passage is in there... and I will not guess."
- **All four overlays open on a real touch tap and stay open** (the double-fire trap the activation fix
  could have introduced), **and now open from the keyboard**, which none of them did before.
- All four dismissal exits pass on all four overlays; close controls 44x44; focus moves inside on open
  and returns to the trigger on close.
- Glossary terms open on Enter and close on Escape.
- Near-match slider survives a retune: same node, still focused, zero scroll jump.
- CSV carries the UTF-8 BOM (verified on the raw bytes — `Blob.text()` strips a BOM by spec, so the
  obvious check is a false negative). Report is self-contained.
- 375px: no horizontal document overflow; wide tables scroll inside their own container.
- Zero console errors and, after the font fix, zero pdf.js warnings — confirmed by tagging the console
  and re-running, because the earlier warnings were stale history from the pre-fix session.
- Manifest complete with a maskable icon; apple-touch-icon 200 and opaque; CSP carries the beacon and
  the feedback widget in both `script-src` and `connect-src`.
- **Not verified:** no real touch device was available, so the tap sequences were synthesised
  (`pointerdown`/`pointerup`/`click` with `pointerType: 'touch'`). Folder drag-and-drop could not be
  driven in the automation and is covered by unit tests of its predicates plus code review only. No
  screen reader was run; the ARIA work is verified structurally, not aurally.

## Deployment
- Repo created: yes (2026-08-20, by the aborted run)
- GitHub Pages enabled: yes, cert `approved`, expires 2026-11-19
- Fix commit pushed: yes — `1de6acb`
- Directory entry live on main: yes
- Workflow triggered: yes

## Errors & Resolutions

- **One of the five fixer agents was terminated mid-task by a content-filtering error.** It had already
  written all four of its fixes to disk; what it lost was its regression tests and its report. A
  replacement agent was given its diffs and wrote the tests, verified the fixes against pre-fix code,
  and probed both critical fixes for over-correction.
- **That probe found the one over-correction and it was deliberately left in place.** The widened
  low-text page rule now declares a template-heavy slide deck unsearchable: the repeating header and
  footer are correctly stripped as furniture, leaving 471 characters of body text but a median page of
  58, under the floor of 100. A quotation verbatim on slide 2 goes from `exact` to
  `not-checked: "deck.pdf has no text layer"`, which is false of it. Every available loosening weakens
  the guard that stops a thin scanned book getting a confident NOT FOUND, and abstention is the safe
  error, so the test asserts the safe property (never `not-found`) and records the number instead. This
  is a real cost and it is written down rather than hidden.
- **A second known limit:** `rowSpansGutter` uses a fixed 3% of page width as its minimum break, so on a
  layout whose physical gutter is narrower, rows justified flush to the gutter are read across it
  (measured: 4 of 28 rows). Still strictly better than pre-fix, which read all 28 across. The honest
  repair changes `findGutter`'s return type; flagged, not done.
- **Cost of the widened image probe:** +87 ms over a 60-page stamped scan (112 ms vs 25 ms), scaling
  with page count.
- **The first deploy of the fixes FAILED, on a test that passes locally**, and chasing it properly was
  worth the round trip. `tests/regress-c.test.ts` asserted that a title page carrying a small
  publisher's device is flagged image-only. That is true on macOS/Node 22 and false on Linux/Node 20.
  Rather than guess, the test was pushed with a diagnostic line and CI's own numbers read back: the
  per-page character counts are **byte-identical in both environments** (66, 1585 x4, 61, 8), so the
  difference is purely that pdf.js does not report a 70x70 image in its operator list consistently
  across Node versions — it always reports a full-page one. The test now asserts the stable and
  important half (a full-page plate IS flagged, in both) and records why the other half is not
  asserted. The direction matters and is benign: the instability can only ADD a preflight warning,
  never remove one, so it cannot produce a false NOT FOUND, and the case that would be serious — a real
  scanned sheet going undetected — is asserted in both environments. Second deploy green.
- The sample ships 10 quotations rather than the 41 the queue spec imagined. It demonstrates exact,
  near-match-with-diff, and the unsearchable abstention, which is what the tool needs to show; the
  smaller number is a deliberate reduction, not an oversight.
