# Fleet TLS — root cause analysis (2026-08-02)

Durable reference. Per-run logs are `ssl-<date>.md`, written by
`~/Code/lab/scripts/ssl/fleet-ssl.mjs`. This file explains *why* properties break.

## Symptom

A property resolves to GitHub's Pages IPs (`185.199.108-111.153`) and looks correctly
configured, but every browser hard-fails:

```
curl: (60) SSL: no alternative certificate subject name matches target host name
```

GitHub has the custom domain set but has not issued a certificate covering it, so it
serves one that doesn't match. Nothing in the repo, build or deploy log looks wrong.

## Cause 1 — the factories polled the wrong field (FIXED 2026-08-02)

All three factories confirmed TLS issuance by polling `.https_enforced`. That field is
`false` even on perfectly healthy properties — it is the "Enforce HTTPS" *setting*, not
the certificate state. The poll therefore never succeeded, each factory timed out after
5 minutes, shipped anyway, and the property went live with a stalled certificate.

**The correct signal is `.https_certificate.state === "approved"`**, confirmed with a real
request returning 200. All three factory SKILL.md files were corrected.

Note also: `-f https_enforced=true` fails with HTTP 422 (`not of type boolean`). The field
is typed — use `-F`.

## Cause 2 — Let's Encrypt rate limit (STRUCTURAL, NOT FIXED)

**Let's Encrypt issues 50 certificates per registered domain per 7 days.** Every property
is a subdomain of the same registered domain, `benrichardson.dev`, so all of them share
**one** budget. It is a token bucket refilling at roughly 7/day, not a weekly reset.

| Source | Certs/week |
|---|---|
| 3 factories × 1 new property/day | 21 |
| Renewals of the existing fleet (90-day certs, N × 7/90) | ~12 at N=155 |
| **Steady state before any repair** | **~33 / 50** |

This is the real reason the backlog exists and why it resists repair. On 2026-08-02,
cycling 20 stalled domains at once recovered exactly **5** and left 15 frozen with zero
state movement across five minutes of polling — the bucket had about five tokens left.

### AMENDMENT 2026-08-04 — this does not explain the current 15

Cause 2 is real and the ceiling maths below still stands, but it is **not** why the present
backlog is stuck, and the `RATE LIMITED` verdict that pointed here was a script artefact
(fixed; see `ssl-2026-08-04.md`). Two observations kill the rate-limit reading:

- New properties keep getting certificates — `au-bushfires`, `scrubjay`, `loraprep` are all
  `approved`. An exhausted per-registered-domain bucket would stall them too.
- Nine properties held an **undisturbed** authorization for 42h with zero state movement.
  A throttled queue waiting on refill does not behave that way.

DNS, Pages config, HTTP-01 reachability and CAA were all compared against the healthy
controls and are indistinguishable. Every remaining domain-scoped explanation predicts new
properties would fail too, so the cause is almost certainly **hostname-scoped** — most
likely Let's Encrypt's failed-validation limit, accumulated on exactly these hostnames
during the pre-08-02 era when authorizations never completed. If so, **cycling extends the
block rather than clearing it**, and the right move is to leave the nine untouched for a
week. Unconfirmed — treat as the leading hypothesis, not established fact.

### Consequences for repair

- A **frozen** `cert_state` (no movement across polls) means throttled, not broken. Stop.
- Re-cycling an in-flight authorization abandons it and spends another token for nothing.
  `fleet-ssl.mjs` enforces a 24h per-property cooldown for exactly this reason.
- The backlog drains across days. That is the correct behaviour, not a failure.

### The deadline

Solving `50 = 21 + N × (7/90)` gives **N ≈ 373 properties**. Beyond that, renewals plus
new builds permanently exceed the limit and *nothing* — new or renewing — can obtain a
certificate. At three new properties per day from 155, that is roughly **mid-October 2026**.

Certificate cycling cannot fix this. The options are architectural:

- spread properties across multiple registered domains (`ben.gy` has its own 50/week)
- serve properties as paths under one host (`lab.benrichardson.dev/{slug}`) — one cert
- move hosting to Cloudflare (Universal SSL covers `*.benrichardson.dev` with one cert)
- slow the factories

## Cause 3 — registry drift

Three separate ways the registry lied about reality, all of which hide breakage:

1. **URL ≠ deployed hostname.** `veilpix` was registered at `veilpix-app.benrichardson.dev`
   while the repo served `veilpix.benrichardson.dev`; same for `bgwipe`, `au-ndis-watch`,
   `au-visas` and `artemis-tracker`. The stale hostnames were orphan DNS records resolving
   to GitHub Pages with nothing claiming them — permanent 404s. All corrected; the four
   orphan records were deleted after confirming GitHub reported them unclaimed.
2. **`status: deployed` with no `url`.** Nine entries had null URLs, so nothing ever probed
   them. They turned out to be hand-built sites, not factory output, and were removed from
   the registry on 2026-08-02 (the GitHub repos were kept).
3. **Malformed JSON — duplicate keys.** `gh-site-factory/registry.json` had two entries
   merged into one object (a missing `},{` between `au-warming` and `quakewatch`). JSON
   parsers keep the last occurrence, so **`au-warming` was invisible to every tool reading
   the registry** and was never monitored. Repaired; the file now round-trips byte-identically,
   which is worth re-checking after any hand edit:

   ```bash
   python3 -c "import json;s=open('registry.json').read();print(json.dumps(json.loads(s),indent=2,ensure_ascii=False)+'\n'==s)"
   ```

## Cause 4 — the cooldown ledger did not know about hostnames (FIXED 2026-08-05)

The 08-04 evening session moved 14 properties to fresh hostnames **by hand**, so
`state.json` — which `fleet-ssl.mjs` keys by slug — still held timestamps belonging to the
*old* hostnames. The next morning's run read those 66h-old entries, judged the properties
out of cooldown, and re-cycled five authorizations that were about **nine hours old**,
abandoning them. It then printed `NOT ISSUING — …(66h)` against them, which under the
amendment above reads as "this hostname is blocked, move it again" — and a second rename
would have spent four more records from a DNS zone already at its 200/200 cap.

**The trap is general: any out-of-band change to a property's hostname invalidates the
ledger, and the ledger had no way to tell.** Fixed by keying it on `{ts, host}`:

- a hostname the script did not cycle itself is **adopted, not cycled** — if a flow is
  already open, its start time is unknowable, so first-observation is stamped and the host
  serves a full untouched cooldown;
- only a host with **no flow at all** (`certState` null) is cycled on sight, which is the
  documented case where cycling is the required trigger;
- `NOT ISSUING` is reported only when the recorded host is still the host in play.

**If you move a property to a new hostname by hand, you no longer need to touch
`state.json`** — the next run adopts it. Do not "help" by deleting its entry: that makes the
script cycle a hostname whose authorization may already be in flight.

## Non-cause — the CNAME file (checked, ruled out)

GitHub Pages reads the custom domain from a `CNAME` file in whatever it publishes; if that
file is missing, a deploy can clear the domain and destroy the certificate. This was
audited across the fleet and **no property is affected**.

The location depends on the build, and getting it wrong produces false positives — an audit
that only checked `public/CNAME` wrongly flagged seven healthy root-served sites, including
`benrichardson.dev` itself:

- **Vite / Actions build** (`build_type: workflow`, has `public/`): `public/` is copied into
  `dist/`, which is the uploaded artifact → the file belongs at **`public/CNAME`**.
- **Root-served** (`build_type: legacy`, no `public/`): the branch root is published → the
  file belongs at **`CNAME`** in the repo root.

Known-good control set for validating any future check — if these are flagged, the check is
wrong: `artemis-tracker`, `benrichardson.dev`, `leader-skills`, `pipboy`, `pullshark-board`,
`voynich-investigation`, `worldcupinvitations`.
