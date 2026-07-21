# Build Log: Reactwell
**Date:** 2026-07-22
**Status:** deployed

## Idea Source
IDEAS.md (first queued idea). Original:

> **JSX scratchpad / instant React preview** — drop a `.jsx`/`.tsx` file (or paste code) and it transpiles and live-renders the React component in the browser with zero setup: no npm install, no Vite, no dev server. Transpile 100% in-browser with **esbuild-wasm**, mount the result into a sandboxed `<iframe>` so a component crash or infinite loop can't take down the app, and surface build + runtime errors inline with the offending line. Bundle React/ReactDOM locally so it works fully offline; optionally resolve extra bare imports from esm.sh with a clear notice. Add an in-page CodeMirror editor with hot re-render on keystroke (debounced), and use the File System Access API to re-open and live-watch the dropped file.

## Tool Details
- **Name:** Reactwell
- **Repo:** ben-gy/reactwell
- **Category:** developer-utilities (indexed under `files`)
- **Audience:** Developers, PR reviewers, React learners — anyone who wants a quick look at a component without scaffolding a project. Dark, dense, terminal-professional aesthetic (React cyan accent).
- **Stack:** Vite 6 + vanilla TypeScript + Vitest
- **Browser APIs:** WebAssembly (esbuild-wasm), Web Workers, import maps + `data:` URL ES modules, sandboxed `<iframe srcdoc>` (allow-scripts, opaque origin), React 18 UMD (vendored), postMessage, File System Access API, CodeMirror 6, Clipboard + Web Share, Service Worker (PWA)
- **Worker strategy:** Dedicated esbuild worker (`esbuild.initialize({ worker: false })` inside our own worker so the compile runs off the main thread with no nested worker)

## Privacy Model
- **Protected:** Source transpiled entirely on-device; React bundled into the page (offline); component runs in an isolated opaque-origin iframe; no accounts/cookies/fingerprinting.
- **Not protected:** With "Fetch extra libraries" on, package *names* go to esm.sh (not the code); Share encodes (not encrypts) the snippet; infinite loops can freeze the preview frame until reset.
- **Trust surface:** GitHub Pages static bundle over TLS; esm.sh (opt-in); Cloudflare Web Analytics beacon (anonymous page views); feedback.benrichardson.dev (only on Send).

## Architecture Decisions
- **Vanilla TS, not React:** the app *shell* has no complex multi-pane state; React only lives inside the preview iframe. Kept the bundle small.
- **How React reaches the sandbox:** the preview iframe is `sandbox="allow-scripts"` with NO `allow-same-origin` → opaque origin → it cannot read parent Blob URLs. So everything is inlined as text and bare specifiers resolve through an **import map whose targets are `data:` URLs** (origin-independent). `react` maps to a shim that re-exports `window.React`, which the vendored React 18 UMD assigns. The automatic JSX runtime is synthesised from `React.createElement`.
- **React 18, not 19:** React 19 dropped UMD builds; the UMD global-assignment script is what makes the offline shim approach work, so pinned to 18.3.1 and vendored `umd/*.production.min.js` into `src/vendor/` (React's `exports` map hides the UMD path from Vite's resolver, so a package import fails — hence vendoring + `?raw`).
- **Externalise imports via a plugin, not `packages:'external'`:** in the no-filesystem wasm build, `packages:'external'` fails to catch the auto-injected `react/jsx-runtime`, so a custom `onResolve` plugin marks every non-relative import external — esbuild then never touches a filesystem that doesn't exist.
- **PWA:** app shell precached; the 12 MB esbuild wasm is runtime-cached (CacheFirst) rather than precached, keeping installs fast (per the jsquash "don't precache wasm" lesson).

## Test Results
- Tests written: 30 (transform: 20, router: 6, util: 4)
- Tests passed: 30
- Tests failed: 0
- Coverage: base64 (unicode + chunk-boundary), react-shim/import-map construction, CDN on/off + blocked-import handling, srcdoc assembly, esbuild error-line parsing, share-hash round-trip (URL-safe), debounce.

## Build Status
- npm install: pass
- npm test: pass (30/30)
- npm run build: pass
- Local preview: pass
- Production workflow dry-run: pass (compile → render, error overlay with line number, Privacy modal, mobile event-drawer × close at 375px)

## Deployment
- Repo created: yes (ben-gy/reactwell)
- GitHub Pages enabled: yes (custom domain reactwell.benrichardson.dev, DNS + TLS provisioned, HTTPS 200 live)
- PR created: https://github.com/ben-gy/reactwell/pull/1
- Workflow triggered: yes (initial deploy + one redeploy for the preview-colour fix — both succeeded)
- Registry + index updated; IndexNow pinged; hub sitemap refresh triggered.

## Errors & Resolutions
1. **Vite couldn't import `react/umd/react.production.min.js`** — React 18's `exports` map doesn't expose the UMD path. Fixed by vendoring both UMD files into `src/vendor/` and importing them with `?raw`.
2. **First compile failed: "Could not resolve react/jsx-runtime"** — `packages:'external'` didn't externalise the auto-injected JSX-runtime import in the wasm build. Fixed with a custom `onResolve` plugin that externalises every non-relative import.
3. **Drop overlay permanently visible** — the classic `[hidden]` trap: `.drop-overlay{display:flex}` overrode the low-specificity UA `[hidden]{display:none}`. Fixed with an explicit `.drop-overlay[hidden]{display:none}`.
4. **Preview text invisible on dark-mode machines** (caught during the production dry-run) — the preview iframe used `color-scheme: light dark`, so default text went white on the white preview canvas (buttons stayed visible via their own UA background, masking it). Fixed by pinning the preview to `color-scheme: light` with explicit `#fff`/`#111`. Required a cache-busting reload to verify past a stale GitHub Pages HTML cache.
