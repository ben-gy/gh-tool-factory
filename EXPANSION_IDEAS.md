# Expansion Ideas

Ideas for improving existing tools with new features, supported formats, additional browser APIs, or polish. These are NOT new tool ideas — they're enhancements to products we've already built. The scheduled task logs these here instead of building duplicate tools.

Review periodically and fold the best ones into the existing repos.

<!-- Format:
- **repo-name**: Description of the enhancement, new format support, or additional capability. Include URLs for any libraries or specs referenced.
-->

- **pagesmith**: A parallel PDF page-organiser ("collate") was built to completion on 2026-07-12 before discovering that a concurrent factory run had shipped **pagesmith** (same tool: merge/split/reorder/rotate/delete PDF pages) live at pagesmith.benrichardson.dev. Collate was un-published (DNS removed) as the duplicate. If any of collate's touches are worth folding into pagesmith, they are: (1) **Background thumbnail rendering that survives a hidden tab** — pdf.js pauses `page.render` when the tab is hidden (it waits on `requestAnimationFrame`), which can freeze thumbnail generation; collate renders thumbnails off the `busy` lock, wraps each render in a cancel-on-timeout, and resumes via a `visibilitychange` listener so export is never blocked by a paused render. (2) **Extract-selected** — export only the currently-selected pages as a separate PDF (one-click split). (3) **File System Access "Save as…"** with a graceful download fallback. Reference impl: `tools/2026-07-12-collate`, kept locally — **that local directory is now the only copy of collate's code**, so do not clean it until these three touches are folded in (the descriptions above are detailed enough to reimplement from if it is lost).

- **DECIDED 2026-07-23 — the duplicate repos are dead.** `ben-gy/collate` and `ben-gy/pagewell` were both **archived** (read-only, private, out of the org's active list, and no longer generating dependabot PRs). The keep-or-delete question that sat open for eleven days is settled: **pagesmith is canonical**, both duplicates are dead. Deleting them outright was requested but the baked-in token carries only `gist, read:org, repo, workflow` — `gh repo delete` needs `delete_repo`, and `gh auth refresh` is interactive. To finish the job: `gh auth refresh -h github.com -s delete_repo` then `gh repo delete ben-gy/collate --yes && gh repo delete ben-gy/pagewell --yes`. Nothing of value is in either repo — their only PRs were dependabot bumps plus pagewell's original build PR. **Future runs: `2026-07-12-collate` and `2026-07-12-pagewell` are known-dead duplicates, not undeployed backlog items.**

- **scribewell**: Additions surfaced while a duplicate transcription tool (echoscribe) was mistakenly started on 2026-07-06 before catching that scribewell already covers this space. Fold these into scribewell rather than shipping a second transcriber: (1) **Microphone capture** via MediaRecorder + getUserMedia so users can record and transcribe in one step (scribewell's plan lists mic as a v1 non-goal). (2) **JSON export** with per-segment `{start,end,text}` + metadata, alongside the existing txt/srt/vtt. (3) **Editable transcript** — render each segment as a `contenteditable` row so users can fix words before export, reading edits back from the DOM keyed by segment index (timestamps stay fixed). (4) **Determinate transcription progress + streaming partial text** by manually windowing the PCM into 30 s slices and running the pipeline per-window (the high-level transformers.js ASR pipeline exposes no per-chunk callback), giving a real % bar and a live-preview pane instead of an indeterminate spinner. Reference impl was written at tools/2026-07-06-echoscribe (removed after the duplicate was caught).

- **ggufscope**: predict Ollama's chat-template auto-detection. Ollama's `server/model.go`
  `detectChatTemplate` pulls the Jinja `chat_template` out of the GGUF and Levenshtein-matches it
  against 37 indexed known templates; distance ≥ 100 means NO match and the model silently falls
  back to `{{ .Prompt }}` — raw passthrough that produces malformed chat with no error. Measured:
  TinyLlama and Llama-3.1-8B match at distance 0, but Llama-3.2-1B (1053), Qwen2.5-0.5B (1664) and
  Qwen3-32B (3418) all MISS. ggufscope already has the Jinja string, so it can run the identical
  algorithm in-browser against Ollama's `template/index.json` and tell the user "Ollama will not
  recognise this template — here is the Go template and the matching `PARAMETER stop` lines to paste
  in". Nobody else does this. Cost: vendoring Ollama's template index (MIT).
  https://github.com/ollama/ollama/blob/main/template/template.go

- **searchpack**: Retrieval quality is the one weak spot (measured 2/5 top-3 on conversational sample queries; static embeddings are ~35 MTEB Retrieval vs ~43 for all-MiniLM-L6-v2). Three cheap, independent levers, in order of expected payoff: (1) **query expansion from the corpus** — at build time, precompute for each shipped row its top-k nearest neighbouring rows and ship a small expansion table, then add weakly-weighted neighbours to the query vector; this is how you recover synonymy that mean-pooling cannot ("callback" → "webhook"). (2) **Index the heading trail as its own micro-chunk** with a boost, so "Verifying that an event really came from us" is directly matchable rather than diluted into 150 words of prose. (3) **Field-weighted BM25** (heading terms weighted above body terms, BM25F-style) — the postings already exist, only the scorer changes. Also worth testing: `minishlab/potion-retrieval-32M` (MTEB Ret 35.06 vs 34.95, model.safetensors 129,210,456 B, trivially parsed 8-byte-header safetensors) as an optional larger-download/better-quality model choice — the loader is the only thing that differs, since both are pure lookup tables.
- **backslide**: three enhancements cut from v1 on 2026-08-04 with the reason recorded, so nobody
  re-litigates them. (1) **Token-level diff drill-down** via `gpt-tokenizer` o200k_base +
  `diff.diffArrays` over token-id arrays. Cut because it costs **1,065,794 B gzipped** and, measured
  across 11 realistic cases, produced hunk counts *identical* to `diffWords` in 8, was clearer in 1
  (CJK only), and was actively **worse** on whitespace reflow — 7 noise hunks where `diffJson` on
  parsed objects correctly reports 0. Only worth adding behind a lazy import for a CJK-heavy user.
  (2) **A promptfoo `tests` CSV export.** backslide's `regressions.jsonl` is an *output* record and
  `promptfoo eval` cannot consume it; what it consumes is `tests: file://…csv` with the special
  columns `__expected`, `__description`, `__metadata:*`
  (https://www.promptfoo.dev/docs/configuration/test-cases/). Emitting one row per newly-broken case
  with `vars` as columns and `__expected` carrying the flipped assertion would make the set
  difference *executable* — which is the one thing `promptfoo eval --filter-failing <path>` cannot
  express, because it returns everything failing in B rather than "failed in B ∧ passed in A".
  (3) **Braintrust / LangSmith first-class adapters.** v1 reaches them through the generic column
  picker; Braintrust needs `is_root === true` span filtering plus a content-hash-of-`input` join key
  (its own OpenAPI prose says "if you run the same experiment twice, the `input` should be
  identical"), and LangSmith's bulk export is **zstd-compressed Parquet**, which is a large WASM
  dependency for one dialect.

- **searchpack**: Ship an optional prebuilt search UI (`search-ui.js`, a few KB: input, debounce, keyboard nav, highlighted snippets, ARIA combobox) alongside `search.js` in the zip. Right now the README hands users a raw `pack.search()` call and they write their own dropdown — the single most likely place for the integration to stall.

- **scrubjay**: **known-needle mode.** Let the user drop a `.env` (or paste a list of values) and search a dataset for *those exact strings* alongside the existing 29 recognisers. Surfaced while building `leakmap` on 2026-08-08, which is built entirely on that mechanic. It is genuinely additive rather than a duplicate: measured on a real Vite build, scrubjay's compiled recognisers covered 8–9 of 16 shipped values (50–56%) with two junk hits (`email-address` firing on a Sentry DSN, `credit-card` on a 17-digit run inside an Airtable token), where known-needle matching covers 100% of what is byte-identical. It does not hollow out leakmap either — scrubjay still takes one structured-data file (`index.html:133` accepts eight extensions, `main.ts:908` takes `files[0]`, `sniff.ts:53` refuses a ZIP by magic bytes), still cannot walk a directory, and has no sourcemap half. The one thing to port with it is leakmap's **searchability gate**: a user's `.env` contains `NODE_ENV=production` and `PORT=3000`, and searching a corpus for those produces tens of thousands of meaningless hits (measured: 76,738 match events from a 43-value `.env` over 5 MB of real bundles, of which `1` alone was 37,781). Reference impl: `tools/2026-08-08-leakmap/src/domain/gate.ts` and `variants.ts`.

- **tokenpeek**: **Supabase-shaped JWT recognition.** Surface `role: anon` vs `role: service_role` in the existing claims table with the consequence spelled out, since the entire discrimination is one claim — measured on Supabase's own published demo keys, the two tokens are structurally identical (segment lengths 36,95,43, unpadded) and differ only in that string. `service_role` is the Postgres role carrying `BYPASSRLS`, so it skips every row-level-security policy the user wrote. Also worth adding: the current `sb_publishable_…` / `sb_secret_…` formats, which have no decodable payload at all — a JWT-only implementation in 2026 misses `sb_secret_` entirely, and Supabase documents the legacy JWTs as deprecated by the end of 2026 with both working simultaneously today. Wording to reuse (deliberately neither "safe" nor "leaked") is in `tools/2026-08-08-leakmap/src/domain/jwt.ts`, `supabaseRoleAdvice`.
