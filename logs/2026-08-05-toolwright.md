# Build Log: Toolwright
**Date:** 2026-08-05
**Status:** deployed (content live; custom-domain TLS certificate still pending — see Deployment)

## Idea Source

`IDEAS.md`, entry **toolwright**. It was NOT the first entry in the queue: the standing deferral set
on 2026-08-03 (expires 2026-08-09) puts `runledger` and `epochcost` in the
`@huggingface/jinja` + `@huggingface/tokenizers` group with **chatprint** (shipped 2026-08-02), and
`chunkforge`, `leakspot` and `topicgap` in the text-embedding group with **searchpack** (also
2026-08-02). All five were skipped per that rule, which is explicitly a rule and not a preference.
`toolwright` is named in neither group and was the first eligible entry.

**The input question (principle #6), asked and answered.** The last six tools all ingest a file and
emit a file; the last sensor tool was tiltwell on 2026-08-02. That is a real signal and it is
recorded here rather than ignored. It was not acted on this run because IDEAS.md takes priority over
research and its queue order is deliberate — but **the next run that reaches the research step should
reach for a live sensor**, and the queue's remaining entries are all file-shaped, so this will not
resolve itself.

## Tool Details

- **Name:** Toolwright
- **Repo:** ben-gy/toolwright
- **Category:** developer-tools
- **Audience:** an engineer shipping an agent against more than one provider, mid-debug, with a red
  400 in the terminal and a tool belt an MCP server generated for them
- **Stack:** Vite 7 + vanilla TypeScript. Two runtime dependencies (~9 kB gz between them)
- **Browser APIs:** module Web Worker (retains the belt), Clipboard paste as primary ingest, File API
  + drag/drop, File System Access / anchor download, Web Share level 2, Service Worker (PWA),
  localStorage (theme + selected targets only)
- **Worker strategy:** one dedicated module worker that retains the parsed belt so re-targeting
  recomputes from memory

## Research phase (ultracode)

A 15-agent workflow ran before any code: five research bundles (OpenAI strict, Anthropic + Gemini,
libraries, llama.cpp GBNF, duplication + demand), each attacked by two independent verifiers with a
correctness lens and a shippability lens. ~1.9M subagent tokens, 599 tool calls, zero agent errors.
A synthesis pass reconciled the ten refutations against the five bundles into `RESEARCH.md` (1,209
lines), which is committed to the tool repo and is the build spec.

**Eleven load-bearing claims in the IDEAS.md spec were refuted, including its headline.** All eleven
are enumerated in `RESEARCH.md` §0 and in the registry entry. The most consequential:

1. **The headline was false.** `post-parse-checks.ts` was justified by "pattern, minLength, minimum,
   uniqueItems, which strict mode silently lets through". `pattern` and `minimum` have been on
   OpenAI's supported list since the 2025-05-20 changelog; `uniqueItems` returns a literal 400. The
   file now rests on four different, defensible gaps.
2. **"Three providers" is a fiction** — the tool shape and the keyword subset vary independently, so
   eleven targets ship.
3. **`strict: false` is not an escape hatch** — the Responses API validates the keyword subset
   regardless, and one bad tool 400s the entire array.
4. **The traversal must recurse** into `type:"namespace"` groups or a linter clears a belt that 400s.
5. **The GBNF drop list was wrong in both directions**, and half the drops are conditional on sibling
   keywords — so the drop set is computed per node, and that computation is the product.
6. **The port is ~700 lines, not ~300**, needs BigInt, and must not fetch remote `$ref`s (llama.cpp's
   own C++ passes a stub fetcher, so implementing it would make us *disagree* with `llama-server`).
7. **The 2020-12 meta-schema cannot be used at all** — `@cfworker/json-schema` has zero `$dynamicRef`
   implementation and that meta-schema recurses exclusively through it, so it validates the root only.
   Measured: five nested-error cases all returned `valid: true`. Draft-07 caught all five.
8. **Both tokenizers cut** — see below.
9. **jsonrepair needed a first-party guard** in front of it.
10. **Anthropic does publish limits** (20/24/16), and they are request-wide aggregates.
11. The specced sample fixture was mis-calibrated (it highlighted `$ref` as an OpenAI-strict failure;
    `$ref` is explicitly supported). Fifteen genuinely discriminating fixtures ship instead.

Two build agents (also Opus 5) then wrote the two file-disjoint heavy modules in parallel against a
frozen type contract, and found a further eleven errors in RESEARCH.md itself when they checked it
against the actual upstream source — including that `Unbalanced parentheses` is dead code in
llama.cpp (8 reachable hard errors, not 9), that the real TestCase count is 75 not 73, and that
Vertex's `type` is documented Optional where generativelanguage's is Required.

## Architecture Decisions

- **Vanilla TS, no React.** The state is one belt and one selected cell.
- **The provider matrix is DATA** (63 keyword rows × 11 targets = 689 cells), each cell carrying a
  status, the literal provider error string where one exists, and a provenance badge. Gemini's
  allowed-key list is derived mechanically at build time from Google's own discovery documents rather
  than transcribed, and a test asserts every REJECT cell is absent from that list and every SUP cell
  present. Zero mismatches — which converts a transcribed table into a checkable one.
- **The GBNF port is first-party**, not `json2gbnf` — which was inspected rather than assumed absent:
  it is MIT, CSP-clean and vendorable, but was last published 2024-11-24, does not reproduce
  llama.cpp's output, and handles no `pattern`, `$ref`, `$defs`, `oneOf` or `anyOf`.
- **Both tokenizers cut.** Anthropic and Gemini have no client-side tokenizer, so two of three columns
  would have been fabrications; `gpt-tokenizer/functionCalling` speaks only the legacy 2023
  `functions[]` dialect and returns a wrong number for the modern `tools[]` shape **with no error**;
  and the chunk is ~1.0 MB gzipped and silently exceeds workbox's precache cap. Cutting both is what
  buys "nothing leaves the tab" — and the run verified in the browser that a full 14-tool analysis
  issues **zero network requests**. The cut is justified on methodology, weight and privacy, and
  explicitly **not** on chatprint already shipping the column, because both duplication verifiers
  traced chatprint's source and proved it does not.

## Privacy Model

- **Protected:** everything pasted or dropped. There is no upload code path to disable; `connect-src`
  names no data endpoint. localStorage holds theme and selected targets only.
- **Not protected:** anything deliberately sent through the feedback form; the page load itself.
- **Trust surface:** the GitHub Pages bundle over TLS, the Cloudflare Web Analytics beacon (anonymous
  page views, no cookies), and the feedback widget. No other origin is reachable.

## Test Results

- **Tests written: 277. Passed: 277. Failed: 0.**
  - `gbnf.test.ts` 124 — including the full llama.cpp golden corpus, **73/73 SUCCESS byte-exact**,
    no skips, no `it.fails`, no normalisation of the actual output
  - `targets.test.ts` 41 — including the discovery-document cross-check
  - `transform.test.ts` 40 · `lint.test.ts` 27 · `ingest.test.ts` 24 · `pipeline.test.ts` 21 (end to
    end over the shipped samples)
- Three failures during development, all fixed, and two of them found real bugs rather than bad tests:
  1. `sliceToStructure` reported "truncated" for `{"a":[1,2}` when "mismatched bracket at character 9"
     is the better diagnosis — reordered.
  2. **Nesting depth counted `anyOf` branches as levels**, which would have falsely tripped OpenAI's
     10-level limit on a union-heavy belt. Split into a separate `containerDepth` that counts only
     descent through `properties`/`items`/`prefixItems`/`additionalProperties`. A linter that invents
     a violation is worse than one that misses it.
  3. Two transform fixtures omitted `required`, so the transform correctly nullable-widened them and
     the assertions were wrong. Fixtures fixed, not the code.

## Build Status

- npm install: pass
- npm test: pass (277/277)
- tsc --noEmit (strict, noUnusedLocals, noUnusedParameters, noUncheckedIndexedAccess): pass
- npm run build: pass — 121.69 kB main JS (36.21 kB gz), 15.13 kB CSS (3.77 kB gz), 27 precache
  entries / 474 KiB
- Local preview: pass
- Production workflow dry-run: pass

## Browser dry-run (production build, local preview)

- Baseline: zero elements hidden via the `hidden` attribute are displayed; `elementFromPoint` at the
  viewport centre returns page content, not an overlay host.
- **"Try a sample" drives the real ingestion path** — 14 tools read in the mcp-tools-list shape,
  analysed in 83–97 ms, 111 findings, 5-column matrix, grammar compiled (11,616 chars), 9 artefacts
  offered. The sample control did **not** also open the file picker.
- Artefacts inspected by intercepting the object URL: the OpenAI export has 13 tools (the root-level
  union is refused, correctly), all `strict: true`, and **16 of 16 objects closed with every declared
  key required — zero violations**. The Gemini export contains no `additionalProperties` and no
  `$ref`, and uses uppercase proto type names. `post-parse-checks.ts` contains no `TODO`, carries the
  strict-fallback warning and a real assertion table. `tool-audit.csv` begins with a UTF-8 BOM
  (verified on the bytes — `Blob.text()` strips it and would have hidden this).
- **Network: zero requests beyond the page, its assets and the sample.** This is the privacy claim
  verified by observation rather than by policy.
- Console: zero errors.
- **Four-way dismissal on all five overlays** (four modals + the drawer) at mobile width: close
  control, backdrop `pointerdown`, Escape, and an outside-panel pointer — all pass, each re-opened
  between exits, with a real touch-type PointerEvent sequence including the close-then-reopen
  double-fire check. Close controls are 44×44. The drawer covers its own toggle on a phone, which is
  why the in-drawer control is not optional.
- PWA: manifest resolves with `name`/`start_url`/`scope`/`display`, PNG icons including a 512
  `maskable`; `apple-touch-icon.png` returns 200 and is a real opaque PNG.

**One real bug found by the dry-run and fixed:** four controls in the findings toolbar escaped a
375px viewport. The cause was a nested-flex chain — every link needs `min-width: 0`, not just the
innermost — and, underneath it, that a `<select>` will not shrink below its longest option
(`"OpenAI non-strict (Responses)"` is 379px on its own) however much `flex-shrink` you give it. Fixed
by stacking the label above the control on mobile and giving the control an explicit width.

**A stale service worker from another tool on the shared preview port served the pre-fix CSS twice**
and made a correct fix look broken. Caught by comparing the loaded stylesheet hash against the built
one. This is the recurring trap; it cost two cycles.

## Deployment

- Repo created: yes — `ben-gy/toolwright`, 77 files
- GitHub Pages enabled: yes (build_type=workflow)
- Workflow triggered: yes — **completed, conclusion success**
- Content live: yes — `https://ben-gy.github.io/toolwright/` returns 200 and redirects to the custom
  domain; `http://toolwright.benrichardson.dev/` returns 200
- **Custom-domain TLS certificate: NOT yet issued.** `https_certificate.state` is `null` after ten
  polls over five minutes and one cycle; `curl` to the HTTPS custom domain fails with exit 60. This is
  the same fleet-wide issuance stall recorded in `logs/ssl-2026-08-04.md`, which lists 15 other
  properties in the identical state. Per that log's own conclusion, **cycling further makes it worse
  and was deliberately stopped.** The next fleet SSL sweep should pick it up.
- Directory entry live on main: yes

### Cloudflare DNS record quota — resolved during this run

`POST /dns_records` failed with `81045 Record quota exceeded`: the `benrichardson.dev` zone was at
**exactly 200 records**, its cap. Rather than assume, the run enumerated the zone, found ten
rename-shaped pairs (a bare name plus a suffixed one), and checked each against its repo's Pages
`cname` and a live probe. Nine were genuinely orphaned — the repo had moved to the suffixed hostname
and the bare hostname served nothing:

`au-insolvency`, `bgwipe`, `castwell`, `emberwake`, `facet`, `skein`, `tiltwell`, `veilpix`,
`wavewell`

Those nine were deleted (191 records now, 9 slots free). **`metascrub` was deliberately excluded**
even though it matched the same pattern: its repo still claims the bare `metascrub.benrichardson.dev`
and its probe fails only because its certificate is itself stuck pending — deleting it would have
taken a live property offline. The nine deletions are trivially reversible.

## Errors & Resolutions

| # | Problem | Resolution |
|---|---|---|
| 1 | Cloudflare zone at its 200-record cap | Verified and removed nine orphaned records left by earlier hostname migrations; excluded the one ambiguous case |
| 2 | `anyOf` branches counted as nesting levels | Added a separate container depth; the raw walk depth would have invented OpenAI limit violations |
| 3 | Four toolbar controls escaped 375px | Fixed the whole nested-flex chain and gave the selects an explicit mobile width |
| 4 | Stale service worker served pre-fix CSS twice | Unregistered SW + cleared caches + cache-busted reload; verified by stylesheet hash |
| 5 | `package.json` referenced a script name the build agent did not use | Pointed `fetch:upstream` at both real scripts |
| 6 | Literal control bytes written into a regex character class | Replaced with explicit ` -` escapes |
| 7 | First draft of the checks generator emitted `TODO` comments into a shipped artefact | Rewritten as a declarative table plus a generic runtime that abstains, as a comment, on union branches |
| 8 | TLS certificate stalled | Not resolved. Reported honestly above; not cycled further, per the fleet SSL log's own finding |
