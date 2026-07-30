# Build Log: Sketchwell
**Date:** 2026-07-31
**Status:** deployed

## Idea Source
IDEAS.md (first queued idea, now removed):
> **P2P encrypted collaborative whiteboard/doc** — real-time co-editing of a whiteboard (or a plain-text/markdown doc) with a CRDT synced over an encrypted DataChannel — no Google account, nothing stored anywhere. Genuinely useful "collaborative whiteboard no signup" / "shared scratchpad private" demand, and the CRDT + E2E combo is ambitious. Artefact: export the board as PNG/SVG or the doc as markdown, and a room link to invite others live.

Fleshed out as a **whiteboard** (not the doc variant): drawing strokes form a far cleaner,
honestly-CRDT data model than a text sequence (which needs RGA/Yjs-grade machinery), it is
more Googleable ("collaborative whiteboard no signup"), and it produces stronger artefacts
(PNG + scalable SVG). The last six tools were all P2P-communication (chat/transfer/call), so
this deliberately sits in a different domain — creative collaboration — while reusing the
proven trystero + crypto + eventlog scaffold from chatwell.

## Tool Details
- **Name:** Sketchwell
- **Repo:** ben-gy/sketchwell
- **Category:** p2p-collaboration (index: fun)
- **Audience:** anyone who says "let me draw it for you" on a call and won't make a Miro/FigJam account
- **Stack:** Vite 6 + vanilla TypeScript + Vitest
- **Browser APIs:** WebRTC data channels (full mesh via trystero/nostr), Web Crypto (PBKDF2 → AES-GCM-256, SHA-256 SAS), Canvas 2D + devicePixelRatio, Pointer Events (coalesced), File System Access, Web Share, Clipboard, qrcode, Service Worker (PWA)
- **Worker strategy:** main-thread — ops are tiny AES-GCM blobs and the board is bounded; transport is already off-thread inside trystero

## Privacy Model
- **Protected:** the drawing itself — every op and the full-board sync bundle — travels over direct DTLS data channels and is additionally sealed in AES-GCM-256 keyed from the room passphrase; no server is ever in the path; nothing stored server-side.
- **Not protected:** public nostr relays learn an ephemeral room id; IP visible to peers + STUN; live cursor positions + display names ride the DTLS channel as presence (not under the extra AES layer); no forward secrecy in v1; symmetric-NAT-with-no-relay pairs may fail to connect.
- **Trust surface:** static bundle + TLS chain; public relays + STUN (encrypted handshake only); cookie-less Cloudflare Web Analytics beacon; feedback widget (only if you press Send).

## Architecture Decisions
- **The CRDT** is a first-party op-based **LWW-element-set of shapes**: an add-wins register
  keyed by shape id (a live-streamed stroke re-sends the same id with more points + a higher
  Lamport stamp, so the finished stroke beats its partials), remove tombstones, and a
  monotonic clear barrier. A Lamport clock gives a total order for z-ordering + last-writer-
  wins. All ops are commutative + idempotent, so applying a peer's whole op-log in any order
  converges — which is exactly what lets a newcomer receive the board as one unordered,
  encrypted sync bundle. Chose this over a text-CRDT/Yjs because strokes are naturally a
  grow-only set and the whole engine is pure + unit-testable without a network.
- **Shared logical board (1600×1000), fitted per client** so everyone sees the identical
  composition regardless of screen size; no pan/zoom in v1 (kept scope honest).
- **Streaming vs commit:** in-progress strokes broadcast throttled `add` ops (live to peers,
  shown locally via a canvas overlay so no double-draw), then a final authoritative `add` on
  pointer-up committed to the local board.
- Reused chatwell's crypto/room/wordlist/eventlog/glossary/modal scaffold verbatim (salt +
  content swapped); wrote board/draw/canvas/session/ui/main/sample fresh.

## Test Results
- Tests written: 61 (across 7 files)
- Tests passed: 61
- Tests failed: 0
- Coverage: CRDT (convergence/commutativity, idempotent union, LWW-per-id, remove-before-add, clear barrier, z-order, snapshot-sync, Lamport clock), geometry + SVG export (fit inverse, hit-testing, per-kind serialisation, XML escaping), wire codec + framing, crypto (round-trip, fresh IV, wrong-key failure, empty payload, SAS determinism/symmetry, fingerprint), room/URL routing, format/colour.

## Build Status
- npm install: pass
- npm test: pass (61/61)
- npm run build: pass (tsc + vite; 140 KB / 51 KB gzip)
- Local preview: pass
- Production workflow dry-run: pass (CI "Deploy to GitHub Pages" completed successfully)

## Deployment
- Repo created: yes (ben-gy/sketchwell)
- GitHub Pages enabled: yes (workflow build type)
- DNS: Cloudflare CNAME sketchwell → ben-gy.github.io created
- TLS cert: still issuing at handover (https_enforced=false) — config is correct; per the
  pages-cert-issuance-stalls note this can lag on Let's Encrypt rate limits from bursty
  subdomain creation. ben-gy.github.io/sketchwell/ serves meanwhile; no over-toggling.
- PR created: https://github.com/ben-gy/sketchwell/pull/1
- Workflow triggered: yes (completed success)

## Dry-run verification (local preview, real relays)
- Two tabs joined the same room over the **real nostr relays**: `2 here`, SAS matched on both
  sides (emerald jersey blossom orbit), the newcomer received the full 12-shape board via the
  encrypted sync bundle, and live ops propagated **both directions** with both boards converging
  to the identical shape count (15) — real transport + crypto + CRDT end-to-end.
- Undo removes the last own shape; PNG offscreen render produces real pixels; SVG export emits a
  valid document; modals open + close (Escape); event-log drawer opens + closes via × and Escape.
- `[hidden]` guard verified: on load the drawer computes `display:none` and the viewport centre is
  the canvas, not an overlay (desktop + 375px mobile). No horizontal scroll on mobile; toolbar fits
  and wraps; PWA manifest has name/start_url/scope, a maskable icon, and an opaque apple-touch-icon
  (200). No console errors.
- **Not verifiable in this pipeline:** a genuine multi-device session on real separate machines/
  networks (both tabs shared one machine over the real relays, which proves the protocol but not
  cross-network NAT traversal). The canvas is a pointer-drawing surface; freehand was driven through
  the real commit path via a test hook rather than by a physical pen/touch drag.

## Errors & Resolutions
- **Canvas stuck at 1×1.** The canvas sizing relied solely on a ResizeObserver, which measured the
  container at 0 on the first headless frame and didn't re-fire on a later viewport change, leaving
  the canvas at its 1px fallback. Fixed by adding a `window` resize listener plus a short "settle"
  loop that re-measures until the container reports a real size. Re-verified: canvas fills at both
  desktop and 375px, dpr-2 backing store. (In a real browser the first layout is usually non-zero,
  but this makes it bulletproof.)
- No other errors.
