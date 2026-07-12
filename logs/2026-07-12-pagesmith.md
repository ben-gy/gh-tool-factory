# Build Log: Pagesmith
**Date:** 2026-07-12
**Status:** deployed

## Idea Source
Researched. IDEAS.md was empty (no `-` lines). Reviewed the registry and the on-disk `tools/` directory: found that audio transcription (scribewell ×3, echoscribe) and HEIC conversion (unheic) had already been attempted/shipped by other runs, so those domains were off the table. Picked an unserved, high-search-volume, privacy-sharp problem that builds and deploys reliably on pure-JS libraries: an in-browser **PDF page organizer** (merge / split / reorder / rotate / delete). Complements the existing `pdf-crush` (compression) as a sibling in a PDF-tools family. Verified `pagesmith` was free in both `registry.json` and `gh repo list ben-gy`. (Noted a concurrent run building a separate tool named `collate`; `pagesmith` does not collide.)

## Tool Details
- **Name:** Pagesmith
- **Repo:** ben-gy/pagesmith
- **Category:** document-tools
- **Audience:** Anyone uneasy about uploading sensitive PDFs (contracts, statements, medical letters, IDs) to a random free-PDF-tool site.
- **Stack:** Vanilla TypeScript + Vite 6 + Vitest
- **Browser APIs:** pdf.js (pdfjs-dist) + its Web Worker, Canvas 2D, IntersectionObserver, pdf-lib (dedicated Web Worker), Transferable ArrayBuffers, HTML5 Drag & Drop, File/Blob, Web Share, Clipboard, Service Worker (PWA)
- **Worker strategy:** Two workers — pdf.js's parsing/render worker for thumbnails, plus a dedicated `pdf-worker.ts` running pdf-lib for the final assembly so large exports never freeze the UI. A fresh worker per export allows hard-terminate on cancel.

## Privacy Model
- **Protected:** PDFs are read and rewritten entirely on-device; no page/thumbnail/byte is uploaded. No analytics/cookies/third-party fonts. Strict CSP (`default-src 'self'`, `connect-src 'self'`) forbids all network egress. Offline-capable via service worker.
- **Not protected:** Page-level only (deleting a page doesn't scrub content hidden elsewhere or deep metadata); password-encrypted PDFs can't be opened; GitHub Pages' CDN sees the initial page-load request.
- **Trust surface:** The static bundle + the TLS chain to GitHub Pages. No third-party runtime services.

## Architecture Decisions
- **Vanilla TS over React:** the UI is a single page grid with drag-reorder — no deep component tree, so React's overhead wasn't justified.
- **pdf-lib for assembly:** pure JS, no WASM, builds cleanly and is trivially unit-testable in Node — critical for a reliable autonomous deploy (avoids the heavy-ML build fragility that stalled earlier scribewell attempts).
- **Assembly core extracted** into `assemble-core.ts` (worker-free) so real pdf-lib round-trips can be integration-tested; the worker is a thin wrapper.
- **Source bytes cloned, not transferred,** to the assembly worker so originals remain usable for repeated edits/exports. pdf.js is also handed a *copy* of the bytes (it detaches the buffer it's given).
- **Vitest config split** into `vitest.config.ts` (the `test` key can't live in `vite.config.ts` alongside VitePWA due to a nested-vite type clash under vitest 2 + vite 6).

## Test Results
- Tests written: 44 (3 files) — `format.test.ts` (15), `pages.test.ts` (21), `assemble-core.test.ts` (8)
- Tests passed: 44
- Tests failed: 0
- Coverage: byte formatting, rotation normalisation, filename derivation, multi-select move/reorder, range-select, rotate/delete, assembly-manifest building, and real pdf-lib round-trips (reorder, merge, rotation, empty/out-of-range/missing-doc error paths, valid re-openable output).

## Build Status
- npm install: pass
- npm test: pass (44/44)
- npm run build: pass (clean tsc + Vite; fixed 3 initial TS errors — implicit-any annotation, ArrayBuffer typing in a test, and the vite/vitest config split)
- Local preview: pass
- Production workflow dry-run: pass — drove the built preview: dropped two generated PDFs → 3 page tiles; full merge export produced a valid `%PDF`…`%%EOF` doc; subset export produced a valid smaller PDF; rotate/delete/threat-model modal/glossary/Escape all worked; zero console or CSP errors. Desktop (light) + mobile 375px (dark) screenshots confirmed layout, thumbnail rasterisation, and footer attribution.

## Deployment
- Repo created: yes (ben-gy/pagesmith, private)
- GitHub Pages enabled: yes (Actions build type)
- DNS: Cloudflare CNAME pagesmith → ben-gy.github.io created (grey cloud)
- CNAME set + cycled to trigger TLS cert issuance (cert still provisioning at log time — normal, completes within a few minutes)
- PR created: https://github.com/ben-gy/pagesmith/pull/1
- Workflow triggered: yes — "Deploy to GitHub Pages" completed successfully on `main`

## Errors & Resolutions
- **TS7022 implicit-any** on `expectedNext` in `ui.ts` → added explicit `Element | null` annotation.
- **ArrayBuffer vs ArrayBufferLike** in `assemble-core.test.ts` helper → built a fresh `ArrayBuffer` and copied bytes in.
- **`test` not allowed in vite.config.ts** (VitePWA plugin type clash when importing `defineConfig` from `vitest/config`) → moved the Vitest `test` block into a standalone `vitest.config.ts`.
- **Preview environment quirk:** the headless preview runs at a 0×0 hidden viewport, so IntersectionObserver never fired and thumbnails stayed blank until `preview_resize` gave it a real viewport — an environment artifact, not a bug (pdf.js parsing/rendering both confirmed working).
