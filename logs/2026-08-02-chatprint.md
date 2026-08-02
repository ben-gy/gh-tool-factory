# Build Log: chatprint
**Date:** 2026-08-02
**Status:** deployed

## Idea Source

`IDEAS.md`, first queue entry (removed on selection). The idea proposed rendering any model's real
chat template in the browser, tokenizing it, splitting the context by segment, and packing it to fit,
with two deferred features folded in from killed candidates — a loss-mask legend (from *lossmask*) and
a cache-boundary prefix diff (from *cachehound*). Both were built.

## Execution mode

ultracode on Opus 5. A nine-agent Workflow was launched: five parallel verification agents (npm
reality, Hugging Face CORS/gating, jinja render harness, segmentation + cache + loss-mask research,
duplication + demand) followed by a design agent and three adversarial reviewers.

**The five verification agents completed. The design and three refutation agents failed with
"You've hit your session limit".** That is reported here rather than papered over: the synthesis and
adversarial review were therefore done inline by the main loop, against the five completed dossiers,
which is weaker than three independent refuters would have been. The verification evidence itself —
which is where all the load-bearing facts came from — is complete.

## Tool Details
- **Name:** chatprint
- **Repo:** ben-gy/chatprint
- **Category:** developer-tools
- **Audience:** an engineer whose agent just 400ed on context length
- **Stack:** vanilla TypeScript + Vite 6 + Vitest
- **Browser APIs:** @huggingface/jinja, @huggingface/tokenizers, gpt-tokenizer (dynamic), fetch+CORS,
  Cache API, Web Worker, Service Worker, File System Access, Web Share, Clipboard
- **Worker strategy:** one dedicated module worker: download → parse → render → encode → attribute

## What verification changed about the build

Five of the idea's own premises were **refuted** by agents that actually ran the commands, and the
build follows the measurements rather than the idea:

1. **tokenizer.json sizes.** Claimed 0.7–2.2 MB; measured 1.76 MB to 31.84 MB, median ~10.9 MB, and
   the large ones arrive uncompressed. Consequence: sizes are shown before download, the default model
   is a small one, and the Cache API is a headline rather than a footnote.
2. **Mirror equivalence.** The claim that ungated mirrors serve byte-identical tokenizers is false.
   NousResearch's Llama 3.1 mirror carries a stale Llama-3.0-era 348-char template and counts a
   two-message conversation at 29 tokens where the correct 4,614-char template counts 49. The registry
   uses the template three independent mirrors agree on, and badges every mirror.
3. **jinja globals.** `strftime_now` and `raise_exception` are PRE-DECLARED; passing them in the render
   context throws "Variable already declared" and kills every render. A reserved-name filter is
   mandatory, and the globals are injected through a hand-built Environment.
4. **Competitive positioning.** Hugging Face's own chat-template-playground (196 likes, same library,
   same version) already renders these templates — it just cannot count a token. tiktokenizer is not
   OpenAI-only. The "displaces PromptLayer/Agenta" framing was dropped: those are logging platforms.
   None of the idea's original competitive sentences were shipped.
5. **Llama 3.1 does not use `strftime_now`** (it hardcodes a date); Llama 3.2 does. The date-breaker
   example points at 3.2, where it was verified live.

## Bugs caught during the build

- **`encode()` adds a second beginning-of-text token.** The template already writes `<|begin_of_text|>`
  or `<s>` into the string and the post-processor adds its own — measured 4 ids for
  `<|begin_of_text|>Hello world` where the model sees 3. Every count now uses
  `add_special_tokens: false`. This would have inflated every Llama and Mistral figure by one, silently.
- **Collapsed attribution that still summed exactly.** Rendering prefixes *without* tools made every
  cut point land at the same position on Llama (which splices tool definitions into the first *user*
  message), piling all 6,114 tokens onto the final segment — while passing the "segments sum to total"
  invariant. Fixed by rendering prefixes with tools, and a `degenerate` guard was added because the
  arithmetic invariant provably cannot catch this class of failure.
- **Template raises misclassified.** Without a pinned clock, jinja's own `raise_exception` throws a
  bare Error, so a deliberate model-author rejection was reported as an internal render error. All
  renders now go through one Environment.
- **Cross-model comparison leak.** The cache view could diff a comparison captured under one model
  against another model's token ids, which is meaningless. Comparisons are now tagged with their model.
- **A label that lied.** The cache view's fallback was labelled "rendered an hour later" while
  re-using the same render. Replaced with a real second render at a shifted clock.

## Architecture decisions

- **Vanilla TS, not React** — one document, five views over one result object.
- **@huggingface/jinja + @huggingface/tokenizers, not @huggingface/transformers.** A verification agent
  recommended transformers (120 kB gzipped) for `apply_chat_template` and offsets. Neither is needed:
  the first-party harness gives a pinned clock, capability detection and verbatim error surfacing that
  `apply_chat_template` does not, and the LCP cut algorithm needs no offsets. The chosen pair is
  ~8.7 kB + ~14 kB gzipped.
- **`diff` was dropped** after the divergence engine turned out to be ten lines of first-party code —
  shipping an unused dependency would have made THIRD-PARTY-NOTICES claim something untrue.
- **GPT encodings split per encoding** and excluded from precache; the service-worker precache is
  307 KiB and o200k (1.02 MB gzip) loads only if the user picks a GPT model.

## Test Results
- Tests written: **124**, all offline and hermetic (synthetic tokenizer + structurally faithful
  template fixtures). Tests passed: 124. Failed: 0.
- Three failures during development were fixed: one real defect (raise classification) and two fixture
  artefacts (a 2,000-character single-token string; a fixture template that never rendered its tools).
- A separate live suite (`npm run verify:live`, deliberately **not** in CI) re-checks recorded counts
  against five real models: Qwen2.5 6,237 · Llama 3.1 6,119 · Phi-3.5 1,900 · SmolLM2 4,810 ·
  Granite 3.3 6,478. All pass, all tile exactly.

## Build Status
- npm install: pass
- npm test: pass (124)
- npm run verify:live: pass (5 models)
- npm run build: pass
- Local preview: pass
- Production workflow dry-run: pass

## Browser dry-run (production build, port 4196)

A stale service worker from a previous tool was found serving the shared preview port and was
unregistered before testing — the known trap.

- Sample loads through the real ingestion path: 20 messages, 6 tools parsed
- Qwen2.5 → 6,237 tokens, matching the live test exactly; all five segment colours render
- Budget view: 21 rows, nested disclosure "includes 1,242 tokens of tool definitions the template
  spliced in here", packer drops correctly and reports honestly when a budget is impossible
- Compare: Qwen 6,237 vs Phi-3.5 1,900 (−4,337) badged "template ignores tool definitions"
- Cache (Llama 3.2, real second render one day on): shared prefix 24 tokens, 0.4% reusable, 6,095 lost,
  diagnosed as a date change — the template breaking its own cache
- Training: 1,486 trained / 4,751 masked, no anomalies
- Mobile 375px: drawer hidden on load, closes via × and Escape, `elementFromPoint` returns real content
  (no [hidden] overlay trap), all three modals open/close, no horizontal overflow
- PWA: manifest complete with a maskable icon, apple-touch-icon 200, sample 200, notices 200
- No console errors

Screenshot capture in the preview pane returned blank after scrolling, so visual verification beyond
the first viewport was done by DOM assertion rather than by image.

## Deployment
- Repo created: yes
- GitHub Pages enabled: yes
- DNS (Cloudflare CNAME): created
- TLS certificate: `https_certificate.state` polled to **approved**
- Live check: `https://chatprint.benrichardson.dev/` → 200, correct title, sample and manifest 200
- Deploy workflow: completed success
- PR created: https://github.com/ben-gy/chatprint/pull/1

## Errors & Resolutions

| Error | Resolution |
|---|---|
| Design + 3 refutation agents failed: session limit | Synthesis and review done inline against the five completed dossiers; reported honestly above |
| Llama refused the sample ("only supports single tool-calls at once") | Sample restructured to sequential tool calls so the flagship demo works on every model |
| Double beginning-of-text token | `add_special_tokens: false` on every encode |
| Attribution collapsed on Llama | Prefixes rendered with tools + a degeneracy guard |
| Template raise reported as render error | All renders routed through one Environment |
| Both GPT encodings in one 1.46 MB chunk | Split per encoding; precache excludes them |
| `diff` dependency unused | Removed |
| Privacy modal title double-escaped | Passed plain text to the modal |

## Open items for review
- The tool is honest about Claude being an estimate, but a reviewer may reasonably prefer dropping the
  Claude row entirely, since Anthropic's free `count_tokens` is exact and this is a proxy.
- `ggufscope` shipped the same day and shares the jinja render (~1.1% of its code). The overlap is
  disclosed and quantified in `plan.md`, the README and the PR rather than glossed over.
