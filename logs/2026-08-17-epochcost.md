# Build Log: epochcost

**Date:** 2026-08-17
**Status:** deployed — https://epochcost.benrichardson.dev

## Idea Source

`IDEAS.md`, the **top queue entry**, followed in order for the first time since 2026-08-09.

epochcost had been blocked on 08-15 and 08-16 by standing-note rule 2 ("never ship two tools
from the same engine group in the same week"): chunkforge (2026-08-10) shares the exact-token-count
engine, and runledger (2026-08-09) shares the streaming-NDJSON-through-a-worker-pool-to-a-cost-figure
architecture. On 2026-08-17 those are **7 and 8 days old**, so the week has passed and the entry was
unblocked. No queue-skipping was needed.

**The sensor debt was considered and deliberately not paid.** It remains the oldest item in the file
(last sensor tool: tiltwell, 2026-08-02). IDEAS.md records that the 2026-08-11 hunt spent three
adversarial rounds and ~2M tokens killing all eight candidates, and explicitly asks for a
hand-added idea rather than another speculative hunt. That remains the right next move.

## Tool Details

- **Name:** epochcost
- **Repo:** ben-gy/epochcost
- **Category:** developer-tools
- **Audience:** someone about to spend money and a day of wall-clock on a fine-tune — an ML engineer
  at a small company, a solo builder on Together or Fireworks, a researcher with a scripted dataset.
  Honest size: low hundreds of thousands who could use it, a few thousand recurring.
- **Stack:** vanilla TypeScript + Vite 7
- **Browser APIs:** `File.stream()`, fflate `Gunzip`, module Web Workers + `MessageChannel` with
  credit-based backpressure, `gpt-tokenizer` 4.0.0 ESM subpaths, `@huggingface/tokenizers`,
  Service Worker (vite-plugin-pwa)
- **Worker strategy:** one reader worker cutting at line boundaries, feeding N counter workers, two
  credits per lane. Pool size bounded by `deviceMemory`, not core count.

## What the research changed (12 agents, 2.09M subagent tokens)

Six investigators, each attacked by an independent refuter. **Six findings changed the product
rather than the code.**

1. **OpenAI is winding down fine-tuning** — its own pricing page carries the banner, and the
   deprecations page dates it: 7 May 2026 no new orgs, 2 Jul 2026 no org without fine-tuned
   inference in 60 days, **6 Jan 2027 nobody**. The spec's OpenAI-first framing and its target
   query would have shipped dead. The tool is provider-neutral; OpenAI is one labelled legacy row.
2. **A direct incumbent exists** — `jsonlkit.com/token-counter` is free, client-side and takes a
   file, so the spec's "the free counters cannot take a file" is false and appears nowhere in the
   shipped copy. It is beatable on four measured points, which became the whole product thesis.
3. **The truncation premise is confirmed and stronger** — OpenAI truncates *from the end*, and the
   training example limit (65,536) is exactly half the inference window (128,000).
4. **The billing formula is capped per record** — `Σ min(record, limit) × epochs`. The obvious
   implementation overestimates on exactly the files this tool exists to find.
5. **`gpt-tokenizer` 3.4.0's `dist/cl100k_base.js` is a mislabelled o200k_base build** —
   reproduced independently. Every GPT-4/3.5/embedding count on it is silently wrong.
6. **The padding-waste headline is refuted and cut** — providers bill tokens in the file, and
   padding is not in the file.

Further corrections inherited from the refuters: `DecompressionStream('gzip')` is unsound on
multi-member gzip (and Node's is non-conformant, so a vitest test of it returns a **false pass**);
`davinci-002`/`babbage-002` fine-tuning was shut down 2024-10-28 and `o4-mini` is billed per hour,
so none may appear in a token-cost table; the Azure Retail Prices API has no CORS headers.

## Measurements taken rather than assumed

| | |
|---|---|
| End-to-end throughput | **14.8 MB/s** single-threaded (split + `JSON.parse` + BPE); BPE is **97%** of it |
| 1.2 GB projection | ~1.3 min on one thread, under a minute on a pool — the spec's "5–10 minutes" was pessimistic |
| o200k_base chunk | 2,012,571 B raw / 1,028,715 gz |
| cl100k_base chunk | 951,254 / 442,560 — the spec's "2.05 MB" was the mislabelled build |
| p50k_base chunk | 474,572 / 203,221 |
| o200k_harmony | 1,125 B — reuses the o200k rank chunk |
| base64 pathology | **did not reproduce**: 16.8 MB/s on random base64, *faster* than prose, against an agent's claimed 0.43 MiB/s |
| HF gating | `meta-llama/*` and `google/gemma-*` return **401** anonymously; Qwen/Mistral/DeepSeek and the `unsloth/*` mirrors return 200 |
| Precache | 2.95 MB (shell + sample + default vocabulary only) |

## Bugs the build caught on itself

- **`encode()` throws on real training content** containing `<|im_start|>` — one such record would
  end a scan of two million. And the obvious fix (`allowedSpecial: 'all'`) collapses a content
  string that *is* the marker to one token instead of six. Shipped option: an empty
  disallowed-special set, which does both jobs correctly.
- **`costCents` rounded to whole cents**, printing "$0.00" for a real sub-cent charge — which reads
  as *free* rather than as *cheap*. Different claims.
- **The event-log drawer stopped opening after the first render** — found in the browser, not in
  the unit tests. `mountEventLog` is called from `header()`, which `render()` rebuilds on every
  interaction, so it appended a **new drawer to `document.body` each time**: three were live, the
  toggle controlled the newest, and `getElementById` returned the first. Same class of bug as the
  memoised footer, same fix, now gated.
- **Two build gates were themselves wrong.** The tokenizer-import gate missed *dynamic* imports, so
  it did not see the file it most needed to see. And the banned-API scan fired on the **rank
  tables**, which genuinely contain `sessionStorage` and `XMLHttpRequest` as vocabulary tokens — the
  gate was scanning data as if it were code. The exemption is now narrow and itself asserted.
- Three unit-test expectations were wrong rather than the code (CSV quoting after escaping, the
  `cut === 0` carry contract, a sub-cent rounding case) and were corrected in the direction of the
  measured behaviour.

## Test Results

- Tests written: **115**
- Tests passed: **115**
- Tests failed: 0

The suite pins the shipped sample end-to-end: **exactly 8 of 612 records over the 65,536-token
limit**, every one losing assistant tokens, and `capped < raw` by exactly the truncated amount.

## Build Status

- npm install: pass
- npm run build: pass
- npm test: pass (115/115)
- Local preview: pass
- Production workflow dry-run: pass
- CI deploy workflow: **pass on first run**

## Deployment

- Repo created: yes
- GitHub Pages enabled: yes
- Cloudflare DNS record: yes — **the 200-record zone cap that blocked ghostdep on 2026-08-11 has
  since cleared**; the record was created without needing to reclaim a slot
- TLS certificate: `approved` (second poll)
- Live: https://epochcost.benrichardson.dev — HTTP 200, sample/manifest/icon/notices/og all 200
- Directory entry live on main: yes

## Verified in the browser, not asserted

- The shipped sample yields **exactly 8 over-limit records of 612** through the real gzip reader,
  worker pool and tokenizer — matching the pinned unit test.
- `over-limit.jsonl` is 4.75 MB and exactly **8 parseable JSON records** (24 messages).
- `over-limit.csv` shows record 442 losing 16,845 tokens of which **all 16,845 are assistant
  tokens** — the product thesis on real data.
- The exported report is 5.2 KB with **no script, link or src**, and carries its own caveat.
- The capped-vs-naive figures are both shown: 1,884,876 billable against 2,196,051 raw.
- All three modals pass **all four dismissal exits at 375px** with real touch-type PointerEvents
  (12/12) at 44×44 hit areas, plus the drawer's own exits after three forced re-renders.
- A full run issues requests for the page's own assets and its own sample and **nothing else**.

## Honest gaps

- **The open-model (Hugging Face) tokenizer path was not exercised end-to-end in a browser.** It is
  wired, typed and unit-covered at the registry level, but no run was made against a live
  `tokenizer.json`. A refuter measured that path at **0.24 MiB/s** on SentencePiece models (~79×
  slower than gpt-tokenizer), which is why the interface measures throughput and shows an ETA
  before committing — but the end-to-end behaviour is unverified.
- **The multi-member gzip claim rests on fflate's documented behaviour**, not on a differential test
  against a concatenated archive in a browser. Node's `DecompressionStream` is non-conformant, so a
  unit test of the alternative would have returned a false pass — which is why the choice is
  argued from the spec rather than from a local measurement.
- **The analytics beacon and feedback widget were verified structurally** (the footer link mounts
  and survives re-render; both origins are in both CSP directives) rather than by observing a
  delivered request — the sandboxed preview has no external network.
- One research agent returned a degenerate placeholder result, so the honesty lens was re-derived
  from scratch by its refuter rather than by an investigator. Its findings still landed.

## Errors & Resolutions

Listed above under "Bugs the build caught on itself". Nothing outstanding.
