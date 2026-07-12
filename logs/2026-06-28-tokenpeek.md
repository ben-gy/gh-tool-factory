# Build Log: tokenpeek
**Date:** 2026-06-28 (finished & deployed 2026-07-01)
**Status:** deployed

## Idea Source
IDEAS.md was empty (no `-` lines), so this was a researched idea. A prior
scheduled run had scaffolded a `tools/2026-06-28-tokenpeek/` directory and
registered a placeholder entry in registry.json marked `built_not_deployed`
("prior run stopped before deploy; needs finishing + deploy"). This run picked
that up, completed the implementation (main.ts UI + full CSS + tests), verified
it end-to-end in a real browser, and deployed it.

**Chosen concept:** the offline alternative to online "JWT decoder" websites.
The Googleable problem: *"offline jwt decoder"*, *"is it safe to paste a jwt
online"*, *"jwt verify online"*, *"jwt decode without uploading"*. Pasting a
production access token into a random web decoder is a genuine risk — the token
can carry live credentials and PII. tokenpeek gives the same workflow with a
hard guarantee that nothing leaves the tab.

## Tool Details
- **Name:** tokenpeek
- **Repo:** ben-gy/tokenpeek
- **Category:** encryption-privacy (developer/security utility)
- **Audience:** developers, API engineers, security folks working with JWTs
- **Stack:** Vite 6 + vanilla TypeScript + Vitest (no framework)
- **Browser APIs:** Web Crypto (`subtle.verify` / `sign` / `importKey` /
  `exportKey` / `generateKey`) across HMAC, RSASSA-PKCS1-v1_5, RSA-PSS, ECDSA and
  Ed25519; Web Workers; Clipboard; Drag & Drop / File; Service Worker (PWA)
- **Worker strategy:** single dedicated Web Worker for the HMAC weak-secret
  crack, streaming determinate progress + throughput back to the main thread

## Privacy Model
- **Protected:** the token (decoded in-tab, never uploaded/stored/URL'd); any
  secret, PEM or JWK pasted to verify/crack/sign; generated private keys
- **Not protected:** first page load hits GitHub Pages' CDN (sees IP, not the
  token; PWA runs offline after); a JWT is not encrypted so its claims are
  readable by any holder; the crack wordlist only catches *weak* secrets so
  "not found" ≠ strong
- **Trust surface:** the hash-pinned static bundle + TLS chain to GitHub Pages.
  Zero third-party scripts/fonts/analytics; strict CSP restricts network to
  `'self'`; no JWKS is fetched (key is pasted, so no third party learns which
  key is being checked)

## Architecture Decisions
- **Vanilla TS over React:** the app is a single decode→inspect→act workflow
  with a tabbed tool panel — no deep interacting component tree, so vanilla keeps
  the bundle tiny (42.5 KB JS / 14.7 KB gzipped).
- **Pure core (`jwt.ts`):** base64url + decode + claim analysis are
  dependency-free pure functions, so they're exhaustively unit-testable in jsdom
  without any DOM/network.
- **All crypto through `crypto.subtle` (`crypto.ts`):** one alg-spec table maps
  JWS alg names → Web Crypto params for import/verify/sign/generate; symmetric
  vs asymmetric key material (secret / PEM SPKI+PKCS8 / JWK) is auto-detected.
- **Crack in a Worker:** keeps the main thread painting a live progress bar; the
  built-in wordlist (~144 known-weak secrets) plus optional user candidates.
- **Zero runtime deps:** the only non-dev dependency is `vite-plugin-pwa` (build
  time). Nothing ships to the client but the app itself.

## Test Results
- Tests written: 42 (across 2 files)
- Tests passed: 42
- Tests failed: 0
- `tests/jwt.test.ts` (26): base64url round-trips (bytes, utf-8/emoji, padded
  base64, invalid-length + illegal-char rejection), decodeJwt happy path +
  errors (2-seg, 5-seg JWE detection, non-JSON header, non-object payload,
  unsecured `none`), alg classifiers, claim analysis (expired/valid/nbf/none,
  standard-claim ordering, nested-object rendering), formatRelative, prettyJson
- `tests/crypto.test.ts` (16): spec helpers; HMAC verify against the canonical
  jwt.io token (right/wrong secret); HS256/384/512 sign→decode→verify
  round-trips; `none` token; ES256 + RS256 keygen→sign(private PEM)→verify(public
  PEM & JWK) round-trips; rejects HMAC keygen, public-key signing, non-string
  HMAC secret, and garbage key material

## Build Status
- npm install: pass
- npm test: pass (42/42)
- npm run build: pass (after one fix — see below)
- Local preview: pass
- Production workflow dry-run: pass (real-browser verification via preview MCP —
  decode → verify(valid) → crack(found "your-256-bit-secret" via worker) →
  forge(HS256) → keygen(ES256, 4 key blocks); error paths (malformed + JWE) and
  Threat Model modal confirmed; desktop + mobile screenshots; zero console errors)

## Deployment
- Repo created: yes (ben-gy/tokenpeek, private)
- GitHub Pages enabled: yes (build_type=workflow)
- DNS: Cloudflare CNAME `tokenpeek` → `ben-gy.github.io` created
- Custom domain: https://tokenpeek.benrichardson.dev — live, HTTPS 200,
  https_enforced set
- PR created: https://github.com/ben-gy/tokenpeek/pull/1
- Workflow triggered: yes — "Deploy to GitHub Pages" completed successfully

## Errors & Resolutions
- **TS build error (jwt.ts:125):** TypeScript 5.7's stricter typed-array
  generics rejected assigning a `Uint8Array<ArrayBufferLike>` (from
  `base64UrlToBytes`) to a `let signatureBytes` inferred as
  `Uint8Array<ArrayBuffer>`. Fixed by annotating `let signatureBytes:
  Uint8Array = new Uint8Array(0)`. Rebuilt clean.
- **Preview server path:** the preview MCP resolves `.claude/launch.json`
  relative to the worktree root, which doesn't contain `tools/`. Fixed by using
  an absolute `--prefix` path to the tool directory in the launch config.
- **`https_enforced` PUT:** initial attempt with `-f` (string) 422'd
  ("not of type boolean"); resolved with `-F https_enforced=true` (typed field).
- **Registry race:** the shared registry.json advanced under concurrent runs
  (unheic, scribewell) and already held a `built_not_deployed` tokenpeek
  placeholder. Updated that entry in place to `deployed` rather than duplicating.
