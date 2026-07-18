# Build Log: Scribewell (concurrent-run instance)
**Date:** 2026-07-12
**Status:** deployed (by concurrent run) — this instance contributed a bug fix, later superseded

> This log records a **second, concurrent instance** of the gh-tool-factory
> scheduled task that ran at the same time as the primary run. Both instances
> independently picked the same idea (in-browser Whisper transcription) because
> `IDEAS.md` was empty and `registry.json` did not yet contain the entry when
> each instance read it. The primary run's log is `2026-07-12-scribewell.md`.

## Idea Source
Researched. `IDEAS.md` empty → chose in-browser audio/video transcription with
OpenAI Whisper via `@huggingface/transformers` (WebGPU/WASM). Verified against
`registry.json` (no transcription tool) and `gh repo list ben-gy` (no scribewell)
**at read time** — the name was created moments later by the concurrent run.

## Tool Details
- **Name:** Scribewell
- **Repo:** ben-gy/scribewell (already created by concurrent run)
- **Category:** media-tools
- **Audience:** journalists, students, researchers, podcasters, captioners
- **Stack:** vanilla TypeScript + Vite
- **Browser APIs:** @huggingface/transformers (Whisper), WebGPU, WebAssembly
  (onnxruntime-web), Web Audio API (decodeAudioData), Web Workers, Transferable
  ArrayBuffer, Cache API, Clipboard, Web Share, Service Worker (PWA)
- **Worker strategy:** single dedicated worker hosting the Whisper pipeline;
  audio decoded on the main thread, PCM transferred to the worker.

## What this instance built
A complete, independent implementation at
`tools/2026-07-02-scribewell/` — nearly identical in structure to the primary
run's (same module split: audio, segments, formats, models, transcriber,
worker, eventlog, glossary, ui, main). 42 unit tests, production build green.

## Collision detection & resolution
1. `gh repo create ben-gy/scribewell` failed: "Name already exists" — created
   by the concurrent run at 05:58 UTC (mid-session).
2. Inspected the existing repo: `main` + `review` branches, Pages configured,
   registry/index/log already committed to `ben-gy/gh-tool-factory`.
3. **Did not duplicate.** Did not force-push over the concurrent run's work.

## Bug found & fixed (later superseded)
During live in-browser dry-run testing (WebGPU), this instance discovered a
**critical bug** in its own build AND in the concurrent run's then-deployed
version: `src/worker.ts` passed `task: 'transcribe'` and `language`
unconditionally. The default model **Tiny (English)** (`whisper-tiny.en`) is an
English-only checkpoint; transformers.js throws *"Cannot specify `task` or
`language` for an English-only model"* — so **every transcription with the
default selection failed**.

- Fixed locally: only set `task`/`language` for multilingual models.
- Pushed the fix to `ben-gy/scribewell` `main` (clone-based, minimal diff).
  `npm test` 42/42, `npm run build` green.
- Left a comment on review PR #2 flagging the bug + fix.

**Superseded:** the concurrent run then **force-pushed** a final single-commit
version (`b8200ca`) that already guards the case correctly
(`const isMultilingual = !model.endsWith('.en'); if (isMultilingual) { … }`) and
adds language selection. That force-push overwrote this instance's fix commit.
Posted a correction/all-clear comment on PR #2. The deployed version is correct
and does **not** have the bug.

## Verification
- Local: `npm test` 42/42 pass; `npm run build` succeeds; live WebGPU dry-run
  (file → decode 16 kHz mono → model load → inference → render) completed with
  no error; graceful "No speech detected" path confirmed for a non-speech tone.
- Production: `https://scribewell.benrichardson.dev/` → HTTP 200;
  `https://ben-gy.github.io/scribewell/` → 301 redirect to custom domain;
  Pages deploy for `b8200ca` completed successfully; current `main/worker.ts`
  confirmed to guard English-only models correctly.

## Registry / index / factory log
**Not modified by this instance** — already committed to `ben-gy/gh-tool-factory`
`main` by the concurrent run (registry entry status "deployed", index updated,
`logs/2026-07-12-scribewell.md` present). Avoided duplicate/conflicting writes to
the shared factory repo.

## Errors & Resolutions
- Name conflict on `gh repo create` → detected concurrent run, switched from
  "build & deploy" to "verify & fix existing".
- `.en` task/language bug → fixed; later found the concurrent run's final
  version already fixed it; posted correction.
- Dependabot workflow "failure" on the repo is unrelated noise (open PR for a
  vite bump already exists); the Pages **Deploy** workflow succeeded.

## Net outcome
Scribewell is **live, complete, and correct** at
https://scribewell.benrichardson.dev (delivered by the concurrent run). This
instance's durable contribution: independent verification and a bug-fix attempt
for the English-only-model failure, recorded on PR #2.
