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

### AMENDMENT 2026-08-06 — the budget reading is now measured, not inferred

The trailing-7-day burn can be measured directly rather than modelled, because GitHub
reports `expires_at` and Let's Encrypt certificates run 90 days:

```bash
# issuance date ≈ expires_at - 90d, counted across every property in the three registries
gh api repos/ben-gy/<name>/pages --jq '.https_certificate.expires_at'
```

On 2026-08-06 that gave **37 / 50** — thirteen tokens spare, three of them spent that
morning on properties that issued within minutes. So when the backlog is frozen and new
properties are still issuing, `RATE LIMITED` is not merely unproven, it is measurably false.
**Measure before accepting a budget explanation.** The modelled `burn ~34.6/50` line the
script prints is an estimate from fleet size and is not evidence either way.

Note also that Cause 5 below feeds this number: certificates on hijacked subdomains are
issued against `benrichardson.dev` and count against the same bucket.

**Re-measured 2026-08-07: 39 / 50** (1 · 14 · 3 · 13 · 3 · 2 · 1 across 08-01…08-07), so
eleven tokens spare. `au-water-market` shipped and issued that morning while all seven
backlog properties stayed frozen. Two consecutive days of measurement now say the same
thing: **issuance works, the budget is not the constraint, and the backlog is hostname-
scoped.** The modelled `burn ~34.7/50` the script prints is still only a model.

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

## Cause 5 — dangling CNAME → subdomain takeover (FOUND 2026-08-06)

The most serious failure mode found so far, and the only one that is a *security* problem
rather than an availability one.

When a property moves hostname, the old record keeps pointing at `ben-gy.github.io` and
nothing of ours claims it. GitHub Pages routes by `Host`, so **anyone can claim that exact
hostname on their own Pages repo**, and GitHub will then issue them a Let's Encrypt
certificate for a `benrichardson.dev` subdomain. This happened to
`metascrub-app.benrichardson.dev` on 2026-07-15 and was not noticed for three weeks; it
served Indonesian gambling spam. See `ssl-2026-08-06.md` for the evidence and the fix.

Consequences worth holding onto:

- **A hijacked hostname scores as healthy.** It returns `200` over a valid certificate, so
  every probe-based check passes. `--fix-registry` only inspects properties whose probe
  fails, so it will never look at one.
- **The reliable signal is structural, not behavioural:** registry host ≠ the repo's Pages
  `cname`. Equivalently, a zone record `CNAME`ing to `ben-gy.github.io` that no repo in the
  fleet claims. Run both against **freshly pulled** registries — a property shipped an hour
  ago is unclaimed-looking for exactly the same reason a hijacked one is, and the check will
  happily nominate the day's newest property for deletion.
- **The attacker's certificate spends our budget.** It is issued against the registered
  domain `benrichardson.dev`, so it and its renewals come out of the same 50/week bucket as
  Cause 2. A hostile holder of a dangling subdomain can drain that bucket deliberately.
- **Every superseded hostname is an open invitation until the record is deleted.** The 08-04
  renames left 14; five still exist and GitHub reports all five unclaimed. Deleting a
  superseded record is not just quota hygiene, it closes the hole.

### Prevention

Renaming a property is a two-step operation and the second step is the one that gets
forgotten: set the new hostname **and delete the old record**. If the old record must stay
(links in the wild), keep the hostname claimed by leaving it as an additional custom domain
on the property's own repo — an unclaimed record is the vulnerable state, not the record
itself.

## Cause 6 — the routine could not carry a decision forward (FIXED 2026-08-07)

The 08-06 session concluded that seven hostnames will not validate and that cycling them is
the harmful move, and wrote the remedy in its log:

> **Tomorrow's 08:10 run will cycle them unless something stops it** … Either pass
> `--max-cycles=0` again, or bump `--cooldown-hours` past the intended quiet period.

Nothing stopped it. The scheduled task's command line is fixed, the operator is asleep at
08:10, and the log is not an input to anything. The 08-07 run cycled six of the seven — the
second time these authorizations have been abandoned, after the 08-05 run did the same.

**The general lesson is not "remember to pass the flag."** It is that a conclusion which
only exists in prose cannot survive into an unattended run, and the cooldown could not
express it: a cooldown answers "is this authorization in flight?", a question about hours,
whereas the decision here was "leave this hostname alone for a week." There was no way to
say that, so it was said in English and lost.

Fixed by adding **durable holds** to `fleet-ssl.mjs`, stored beside the cooldown ledger in
`~/.claude/scheduled-tasks/fleet-ssl-repair/state.json`:

```bash
node ~/Code/lab/scripts/ssl/fleet-ssl.mjs --hold=slug-a,slug-b --hold-days=7 \
  --hold-reason="why"                      # place; --release=slug (or =all) lifts early
```

A held property is still probed, still diagnosed and still counted broken — it is only never
cycled. Holds lapse by themselves at `until`, so a hold can never silently become a permanent
exclusion, and both the console output and the run log list what is held and for how long.

The seven were placed on hold until **2026-08-14** on 08-07. Re-probe then: a state that has
moved means the taint ages out, a state still frozen after a week untouched means these
hostnames are permanently dead and the properties need fresh ones.

### Cause 6b — `--prune-dns` could not see its own primary target (FIXED 2026-08-07)

The same run showed the prune path had never been able to delete the records it exists to
delete. Two independent bugs:

1. **It probed `https://`.** A superseded hostname has no certificate — that is what makes
   it superseded — so the probe died at the TLS handshake and the GitHub 404 was never
   read. The unclaimed page is served fine over plain **HTTP**. This is why the five records
   from the 08-04 renames survived a run that was explicitly passed `--prune-dns`.
2. **Candidates came only from the current run's registry corrections.** A record orphaned
   by an *earlier* run was never revisited, so those five were unreachable by any future
   sweep no matter how often it ran. Candidates now come from a sweep of the zone itself:
   every `CNAME → ben-gy.github.io` that no property claims by either name (registry url or
   the repo's Pages `cname`).

The safety bar is unchanged and still absolute — GitHub itself must answer *"There isn't a
GitHub Pages site here"* — plus a **48h grace window on the record's `created_on`**, which is
what stops the sweep nominating the day's newest property while its registry entry is still
unpushed. On 08-07 that guard did its job: it held back `kiln.benrichardson.dev`, created
~24h earlier. Records skipped for either reason are now listed in the log rather than passed
over silently.

All five 08-04 orphans were deleted on 08-07. The zone went 197 → **192 / 200**, and five
takeover-shaped holes closed. `metascrub-app` was correctly left alone: it answers 200, so it
is not unclaimed, and it is not this script's to delete.

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
