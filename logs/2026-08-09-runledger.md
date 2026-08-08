# Build Log: runledger

**Date:** 2026-08-09
**Status:** deployed
**Live:** https://runledger.benrichardson.dev
**Repo:** https://github.com/ben-gy/runledger

## Idea Source

IDEAS.md, first queue entry. The **deferral note expired today**: the 2026-08-03 and 2026-08-04
runs had skipped runledger three times under standing-note rule 2 (never ship two tools from the
same engine group in the same week), because it was named in the `@huggingface/jinja` +
`@huggingface/tokenizers` group alongside **chatprint**, which shipped 2026-08-02. The note said
explicitly: *"On or after 2026-08-09, delete this note and build runledger as the first entry."*
Done. The note was replaced with a short record of why the queue order was not followed for six
days, and the queue is back in plain top-down order.

Ironically the engine-group collision that caused the deferral **did not survive contact with the
data**: runledger ships no tokenizer at all (see below), so it shares no engine with chatprint after
all.

## Tool Details

- **Name:** runledger
- **Category:** developer-tools (directory category: data-explorers)
- **Audience:** a developer who has just seen an agent bill they did not expect
- **Stack:** vanilla TypeScript + Vite 7, **zero runtime dependencies**
- **Worker strategy:** one dedicated module Worker; the file's `ReadableStream` is *transferred*
  into it, and only aggregates come back
- **Storage:** `localStorage` for theme and the price table only. No IndexedDB, no OPFS, no cookies
- **Bundle:** 32.6 kB JS (11.9 kB gzipped), 12.9 kB CSS

## How the research was run

A 21-agent Workflow was launched (7 research lenses over the real corpus, each with independent
adversarial refuters, then a synthesis). **It was still running when the build finished**, so it did
not gate the build — every decision below was instead measured directly in the main loop against the
ground truth available on this machine: **1,823 real Claude Code session `.jsonl` files**. That is
recorded honestly here rather than presented as a fan-out-driven result. The measurements are all
reproducible and are cited inline.

## The measurement that inverted the product

The idea file framed repeated content as an anomaly to hunt: *"you paid for the same MCP schema
fourteen times."* Pricing 25 random real sessions per model against the published rate card of
2026-08-09:

| bucket | tokens | dollars | share |
|---|---:|---:|---:|
| cache **read** | 2,902,793,387 | $749.26 | **73.2%** |
| cache write (1 h) | 41,163,012 | $153.37 | 15.0% |
| output | 12,190,188 | $120.09 | 11.7% |
| input (uncached) | 403,707 | $0.75 | 0.1% |

Repetition is not the anomaly — **repetition is the bill**. So the product became a **carry-cost
ledger**: every row is a turn, showing what it added and how many later turns re-read it.

(A first pass used the retired Opus 4.1 card at $15/$75 and overstated everything ~3x. Corrected
against the published list, and the same error in code is now pinned by a unit test.)

## Corrections the build made to the specification

1. **No tokenizer ships.** The spec named `gpt-tokenizer` (o200k_base, ~2 MB) and
   `@huggingface/tokenizers`. Both cut. `cache_read(N)` *is* the previous turn's prompt — measured
   over **12,475 turn pairs from 80 sessions**, the residual has a median of **exactly 2 tokens**
   (a fixed cache-breakpoint offset, not an error) and lands within ±64 for **96.9%**. So
   `input_tokens + cache_creation_input_tokens` is already the exact token count of everything
   appended since the previous turn. A tokenizer would replace a true number with an estimate of it
   — and there is no public local Anthropic tokenizer, so a tiktoken figure labelled "Claude" would
   be a fabrication.

2. **Dropping unreconciled turns was measurably wrong.** The first implementation excluded them.
   That breaks the `p_i = p_(i-1) + d_i` chain either side of the gap: coverage of the reported
   cache-read column fell to **78.25%** on a session with 9 resets in 29 turns and **83.61%** on one
   with 19 in 109. They are now **segment boundaries reported as context resets** — the cache
   expired and the whole context was bought again at write prices, which in the shipped sample is
   the single most expensive line at 38.2% of the run. Adding the segment head's own cache read
   closed the last gap. Coverage across 40 real sessions is now **min 100.00%, median 100.05%, max
   100.09%**, and the tool prints that figure rather than asking to be believed.

3. **Per-item pricing refused outright.** Calibrating bytes-per-token against the sessions' own
   ground truth gives a session-level ratio spanning **0.45 to 11.45** (within-session per-turn
   p25–p75 1.88–4.26), and only **25.6% of turns** have a single block accounting for 80% of their
   bytes. Splitting a turn's exact token count between its blocks by size would be a fabrication
   wearing a decimal point. The ledger's unit is the turn; repeats are reported in **bytes**, never
   in dollars.

4. **MinHash was over-engineering by three orders of magnitude.** The spec wanted 64-bit MinHash
   over 5-gram shingles with LSH banding. Measured blocks per session: **median 466, p90 1,197, max
   1,230** — a plain `Map` costs microseconds. Exact repeats are **4.4% of appended bytes**
   (1,424,785 of 32,642,318 across 20 large sessions): a real secondary finding, not the headline it
   was specified to be. No near-duplicate pass ships — the obvious near-duplicate is the same file
   re-read after an edit, which is the job, not waste.

5. **The headline promise was unanswerable and was cut.** Naming *"which MCP server's schema block
   was re-sent fourteen times"* cannot be done: the schemas, the system prompt and CLAUDE.md are not
   in the log. What ships is the honest partial — the block is sized **exactly** (median **61,373**
   tokens before the user's first word across 118 sessions; p25 52,186 / p75 81,836 / max 107,958),
   priced exactly, and never itemised. MCP attribution comes from `attributionMcpServer` /
   `attributionMcpTool`, which the records do carry (257 of 1,823 files).

6. **Compaction handling cut as unreachable.** Zero compaction records exist across all 1,823 files
   (`isCompactSummary`, `compactMetadata`, `type:"summary"`, `compact_boundary` all absent — the one
   grep hit was this session's own log echoing the search terms). Prompt growth is monotonic in
   **77.1%** of 118 sessions; the rest are the cache resets above.

## Two real bugs caught by the tests

- **A 3× overcharge.** The obvious model regex `/opus-4(?:[-.]1)?(?:[^0-9]|$)/` **matches
  `claude-opus-4-8`**, because the next character is a hyphen — silently pricing every current Opus
  session on the retired $15/$75 card instead of $5/$25. A minor-version parser ships instead (a run
  of ≥6 digits is read as a release date, not a version), with every real model id pinned by test.
  The comment warning about this trap was already written above the broken regex.
- **Literal control bytes in source.** A control-character class had been written with raw bytes
  rather than escape sequences. A source-hygiene gate now fails the build on any literal control
  byte in `src/`.

## Test Results

- Tests written: **75** — all passing
- 4 suites: `engine` (37), `build-gate` (16), `sample` (14), `honesty` (8)
- One sample test initially failed asserting the baseline was the top ledger row. **The code was
  right and the expectation was wrong**: the turn-41 cache reset genuinely outranks it, because
  re-writing a 120k context at the 1-hour rate (2× base input) costs more than the baseline's carry.
  The headline now reports whichever is actually true rather than always leading with the baseline.

## Build Status

- npm install: pass
- tsc --noEmit: pass
- npm run build: pass (32.6 kB JS / 11.9 kB gz)
- npm test: pass (75/75)
- Local preview: pass
- Production workflow dry-run: pass

## Browser verification (local preview, real build)

- **Stale service worker hit again** — leakmap's SW was still serving `localhost:5199` and rendered
  leakmap's page under runledger's URL. Unregistered + caches cleared + reload. This is the third
  time this trap has appeared in the fleet; it is in memory.
- "Try a sample" → full run: 58 turns, $9.75, coverage **100.0%**, 1 context reset, artefacts shown.
- **Overlay dismissal: 26/26 assertions pass at 375px** across all four modals and the drawer —
  including exit 4 with real `pointerType:'touch'` PointerEvents, the `[hidden]` baseline, and
  `elementFromPoint` at the viewport centre after every close. Close controls measure 44×44.
- Backdrop covers 375×812 with `pointer-events: auto`. No sideways scroll.
- Both artefacts produced and read back: `run-report.html` (16.5 kB, starts `<!doctype html>`, no
  `<script>`, 72 rows) and `carry-ledger.csv` (4.8 kB, 59 lines = 58 rows + header). Asserted that
  neither contains a line of the session.
- PWA: manifest resolves, icons include a `maskable`, `apple-touch-icon.png` 200 and opaque.
- Network during analysis: **same-origin only**. The 3 console errors are the Cloudflare beacon and
  the feedback widget being unreachable from the offline preview host — expected locally, both
  resolve in production.
- **Sample tuned after verification:** the first sample made repeats **46.8%** of appended bytes
  against a measured real-world median of 4.4% (max 15.4%). A demo that overstates the finding it
  demonstrates teaches the wrong lesson, so the repeated block was resized; it now reads **14.6%**.
- A duplicated heading (panel title identical to the drop-zone title) was found in the screenshot
  and fixed.

## Deployment

- Repo created: yes — `ben-gy/runledger`
- GitHub Pages enabled: yes (workflow build type)
- Cloudflare DNS CNAME: yes — `runledger.benrichardson.dev` → `ben-gy.github.io`
- TLS certificate: **approved** (polled `.https_certificate.state`, not `.https_enforced`)
- Deploy workflow: **success**
- Live check: `https://runledger.benrichardson.dev/` → 200, plus sample, manifest,
  apple-touch-icon, og.png, robots.txt, sitemap.xml, third-party-notices.txt all 200
- Directory entry live on main: yes

## Licensing

AGPL-3.0-or-later + section 7(b) attribution terms, `CONTRIBUTING.md` copyright assignment, and
`THIRD-PARTY-NOTICES.md` / `public/third-party-notices.txt` **assembled mechanically** by
`scripts/make-notices.mjs` from on-disk licence files (4 bundled packages, all workbox, from the
service worker). No licence body was transcribed by the model. SPDX headers on every first-party
source file, asserted by test.

## Errors & Resolutions

| Error | Resolution |
|---|---|
| Workflow script parse error (backticks inside a template literal) | Replaced with plain quotes and relaunched |
| Coverage 74% — decomposition lost tokens | Root-caused to dropping unreconciled turns; replaced with the segment model (→ 100.0%) |
| Coverage still <100% on reset-heavy sessions | Segment head's own `readTokens` was unattributed; added |
| Opus 4.8 priced on the retired card | Regex replaced with a minor-version parser; pinned by test |
| Literal control bytes written into `csv.ts` | Rewritten with escape sequences; source-hygiene gate added |
| `localStorage` build gate passed vacuously | Minification hoists keys into variables; gate now checks string literals |
| Honesty gate fired on its own disclaimer | Made sentence- and negation-aware |
| Stale leakmap service worker on :5199 | Unregistered SW + cleared caches + reload |

## Notes for next time

- The research Workflow (21 agents) was still running at ship time. It cost real tokens and
  contributed nothing to this build. For a task where the ground truth is **local files**, direct
  measurement in the main loop was faster, cheaper and more certain than a fan-out — worth
  remembering before launching a large fan-out for a measurable question.
- `EXPANSION_IDEAS.md` candidate: runledger currently reads Claude Code session JSONL only. The
  deltaknit note in IDEAS.md (HAR / SSE ingest with `Authorization` / `x-api-key` / `cookie`
  redaction inside the Worker before rendering) remains unbuilt and is a genuine second dialect.
