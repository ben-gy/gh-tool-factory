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

## Cause 7 — the sweep read registries it never refreshed (FIXED 2026-08-09)

`fleet-ssl.mjs` read the three `registry.json` files straight off disk. Nothing pulled them,
and the routine runs unattended at 08:10, four hours after the factories ship and push. On
2026-08-09 the local clones were two commits behind: `haggle` (game) and `runledger` (tool)
had shipped, been pushed, and were invisible. The run reported `catalog 181` against a fleet
of 183 and both properties went unprobed.

Two unswept properties is the smaller half. **`--prune-dns` builds its "claimed by the fleet"
set from these same files**, so a property that shipped hours ago is indistinguishable from
an orphan — and both of the day's new properties were duly nominated for deletion. The only
thing that stopped it was the 48h grace window on the record's `created_on` from Cause 6b,
which meant the backstop was doing the work of the input. A property that shipped 49 hours
before a stale sweep has no such protection.

This is the same shape as Cause 4: **state that lives outside the script's knowledge, silently
disagreeing with it.** There the ledger did not know a hostname had moved; here the catalog did
not know the fleet had grown. Both fail in the direction of acting on a stale picture.

Fixed by fast-forwarding the three registry repos before the catalog loads:

- fast-forward **only** — a registry with local commits or an uncommitted `registry.json` is
  left exactly as it is, because rewinding or merging an operator's in-progress edit to get a
  cleaner sweep is not a trade this script may make;
- any registry that could **not** be brought current **disables `--prune-dns` for the whole
  run**, rather than letting the grace window absorb an incomplete claim set. Deleting a live
  property's record is the worst thing this script can do and it is not reversible from here;
- what was refreshed, skipped, or left stale is printed and written to the run log, so a stale
  input is never silent;
- `--no-pull` opts out.

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

## Non-cause — two properties that are not on a `benrichardson.dev` Pages cert (checked, benign)

A trailing-7-day budget measurement (the method in the 08-06 amendment) reports every
property with no `https_certificate`. Two entries show up there permanently and are **not**
breakage — recorded here so each morning's measurement does not re-investigate them:

- **`au-worksafe`** — `cname: null`, no certificate, `https_enforced: true`, and healthy. It
  is deliberately served from **`ben-gy.github.io/au-worksafe/`** rather than a custom
  subdomain, because on 2026-08-04 the Cloudflare zone was at its 200-record cap. It rides
  `github.io`'s own certificate, so it costs nothing from the Let's Encrypt bucket and
  `https_enforced` on it is harmless. Note this is the first property in the fleet to use
  **path-based hosting** — one of the four architectural options listed under Cause 2's
  deadline — arrived at by accident rather than decision.
- **`provenova`** — the Pages API 404s because the repo has no Pages site at all. Its
  registry `url` is `https://provenova.net`, an external apex outside the zone. Out of this
  routine's reach by the standing guardrail; reported, never fixed.

Neither is takeover-shaped. The Cause 5 signal is registry host ≠ repo Pages `cname` *for a
property that should have one*; a property with no custom domain by design does not qualify.

### Zone capacity note (2026-08-08)

The 08-07 prune took the zone to 192/200; it now sits at **195/200**. The 200-record cap that
forced `au-worksafe` onto a path is therefore no longer binding, but it is close enough that
it will bind again within days at three properties/day. The cap is a second, nearer ceiling
than the Let's Encrypt one at ~373 properties, and it arrives first.

### Zone capacity 2026-08-09 — 198/200, THE BINDING CONSTRAINT

**Two records of headroom: 185 CNAME + 5 MX + 5 AAAA + 3 TXT = 198 of 200.** At three new
properties per day the zone fills **tomorrow, 2026-08-10**, and the fourth property to ship
after that gets no DNS record at all.

This has overtaken the Let's Encrypt ceiling by a wide margin and is the number that now
matters. The certificate ceiling at ~373 properties is roughly **mid-October**; the zone cap
is **days away** at 183 properties. Note also how the two interact: a property that cannot get
a record cannot get a certificate either, so the zone cap will present as a TLS failure and
this routine will diagnose it as one.

Pruning cannot buy meaningful room. The 08-09 sweep found only five orphan-shaped candidates
and GitHub reports every one of them claimed — including `metascrub-app`, which is Cause 5's
hijack and not ours to delete. **There is nothing left to reclaim.** The 08-07 prune of five
records was a one-off recovery of the 08-04 renames, not a repeatable source of headroom.

The options are the same four architectural ones under Cause 2's deadline, and `au-worksafe`
already demonstrates the cheapest of them working in production: path-based hosting under
`ben-gy.github.io` costs zero DNS records and zero certificate budget. It arrived by accident
on 08-04 when the zone last hit its cap. **This is an operator decision and this routine will
not attempt any of them.** What it will do, absent a decision, is report the same failure every
morning as new properties fail to get records.

### Budget measured a third day (2026-08-08): 38 / 50

14 · 3 · 13 · 3 · 3 · 1 · 1 across 08-02…08-08. Twelve spare. `leakmap` and `sparklight` both
shipped and issued this morning while all seven held properties stayed frozen. Three
consecutive measurements now agree: **issuance works and the backlog is hostname-scoped.**

### Budget measured a fourth day (2026-08-09): 29 / 50

3 · 13 · 3 · 3 · 2 · 3 · 2 across 08-03…08-09. **Twenty-one spare** — the 08-04 spike of 13
has begun rolling out of the trailing window, so the measured burn is falling even as the
fleet grows. Two properties issued this morning; all seven held stayed frozen. Four
consecutive measurements: the budget is not the constraint and has not been for a week.

Worth stating plainly, because the script's headline invites the opposite conclusion: the
modelled `burn ~35.2/50` it prints is now **six certificates above** the measured figure. The
model assumes 21 new certificates a week from three factories shipping daily, but a property
that reuses an existing hostname does not spend a token, and renewals are lumpier than
`N × 7/90`. Treat the printed line as a fleet-size proxy, never as evidence about headroom —
and note that it will keep pointing at Let's Encrypt while the real wall is the DNS zone.

### Zone capacity 2026-08-10 — 200/200, FULL. The prediction landed.

The 08-09 entry said the zone would fill today. It did: **200 of 200 records, zero headroom.**

`chunkforge` shipped this morning only because the 04:10 factory run hit error 81045, ran the
08-05 reclamation procedure, and found the `metascrub-app` hijack from Cause 5 — deleting it
freed exactly the one slot it needed. That is not a repeatable source of headroom; it was the
last orphan and it is now spent.

**The next property to ship gets no DNS record.** Tomorrow's 04:10 run is the first one with
nothing left to reclaim, and per the 08-09 note it will present as a TLS failure that this
routine will dutifully misdiagnose. The 08:10 sweep confirmed it independently: `--prune-dns`
ran clean against all four remaining orphan candidates and GitHub reports every one of them
claimed. There is nothing to delete.

The decision is unchanged and still the operator's: split across registered domains,
path-based hosting under `ben-gy.github.io` (already proven by `au-worksafe`), move the zone
to a plan with a higher record cap, or slow the factories. This routine will not attempt any
of them.

## Cause 7 — an untracked file collision silently disabled the prune (FIXED 2026-08-10)

The 08:10 run reported `REGISTRY tool: local commits — not fast-forwardable, left alone`. The
`tool` registry had **no local commits**: `HEAD` was a strict ancestor of `origin/main`, two
behind. The real error was a different one git returns from the same non-zero exit:

```
error: The following untracked working tree files would be overwritten by merge:
	logs/2026-08-10-chunkforge.md
```

The factory writes its build log into the working copy *and* ships the same path through a
PR, so the incoming commit collided with the untracked local copy and git refused the entire
fast-forward. `refreshRegistries()` attributed every `merge --ff-only` failure to local
commits, so the message sent an operator hunting for a divergent branch that did not exist.

**The reporting error was the smaller half.** A stale registry sets the flag that disables
`--prune-dns` wholesale — so on the one morning the zone hit 200/200, the sweep skipped the
only mechanism that reclaims records, and did so while printing a reason that was false. The
catalog was also short by one: 185 properties swept instead of 186, `chunkforge` invisible to
the very sweep meant to verify it.

Fixed in `fleet-ssl.mjs`: the failure is now reported with git's actual stderr, and a
collision of this shape is resolved by **moving the untracked file aside** to `*.superseded`
and retrying the fast-forward once. Moved, not deleted — git is about to write the
authoritative version, and a log the factory has already pushed is not the sweep's to
destroy. A park that ends in a successful fast-forward is flagged `advanced`, i.e. reported
but *not* treated as stale, because the registry is then current — which is exactly the
condition the prune requires.

Today's parked copy was the pre-amendment draft: 183 lines against the 207 on `origin/main`,
which had the Cause 5 takeover write-up appended by PR #7. Nothing was lost.

## Cause 8 — another routine reclaims broken properties' DNS records (FOUND 2026-08-11, ONGOING)

The zone hitting 200/200 did not stop the factories. It made them start taking records from
properties that already exist.

On 2026-08-10 at 22:20 AEST, `gh-site-factory` needed a slot for `au-public-service`, found
the zone full, audited it, and identified five records as "completely dead — no TLS
certificate was ever issued, so HTTPS fails outright (`http=000`) … all created in one bad
batch on 2026-08-04":

`au-cpi-explorer`, `au-build-approvals`, `au-insolvency-tracker`, `castwell-cast`, `facet-dice`

It then **renamed** `au-cpi-explorer` → `au-public-service`. The proof is the record's own
metadata: `au-public-service.benrichardson.dev` carries `created_on 2026-08-04T13:44:23Z`,
which is `au-cpi-explorer`'s creation stamp. The record was repurposed, not created.

**Those five records are not scrap. They are the five held properties**, placed on hold on
08-07 under Cause 6 precisely *because* they have no certificate — the 08-06 finding is that
these hostnames carry a validation taint and must be left untouched until **2026-08-14** to
learn whether it ages out. `http=000` is the symptom the hold exists to wait out. The site
factory read that symptom as abandonment.

The consequences compound:

- **`au-inflation` went from recoverable to dark.** Through 08-10 it probed `class=A / tls`
  — DNS resolved, cert stalled. On 08-11 it probes `class=B / dns`: NXDOMAIN. A stalled
  certificate can still issue; a hostname with no record cannot. The 08-14 re-probe for this
  property is now unanswerable — a frozen state and a deleted record look identical.
- **Four more are queued for the same treatment.** The factory's log states it plainly:
  *"Four dead records remain, so the next four runs are covered — then this blocks again."*
  Those four are `au-approvals`, `au-insolvency`, `castwell` and `facet`. At three factory
  runs a day the next is hours away, and the hold cannot stop it — **a hold prevents
  cycling, it does not own a DNS record.**
- **This routine misreports it.** Class B reads as "DNS missing" with no hint that another
  routine took the record. The 08-09 note predicted the zone cap would present as a TLS
  failure this sweep would misdiagnose; this is that prediction landing, in a worse form
  than expected — not a property that never got a record, but one that had it taken away.

The general shape is Cause 4 and Cause 7 again, escalated: **state outside the script's
knowledge, silently disagreeing with it.** There the ledger did not know a hostname had
moved and the catalog did not know the fleet had grown. Here a second autonomous routine is
editing the same zone against a different model of what a record is for, and neither routine
can see the other's reasoning. Both audits were competent in isolation; the collision is that
"no certificate" means *wait* to one and *free slot* to the other.

Note the site factory's own guardrails held on everything it could see — it verified the cap
with a throwaway record, refused to touch the 13 live aliases, renamed rather than deleted,
and logged the exact one-call restoration. It was not reckless. It simply had no way to know
these five hostnames were the subject of a standing decision, because that decision lives in
this routine's `state.json` and in this file.

### What this routine did about it (2026-08-11)

**Nothing to the zone, deliberately.** Restoring `au-cpi-explorer` requires a free record and
there is none; the only source is a slot now serving `au-public-service`, live and returning
200. Taking a working property offline to un-dark a broken one is not a trade this routine may
make, and the standing guardrails forbid the surgery in both directions. Reported and escalated.

### What would actually fix it

The record cap is the root cause and remains the operator's decision (Cause 2's four options;
`au-worksafe` already proves path-based hosting works). But this failure mode needs one thing
the architectural fix does not supply: **the factories and this routine need a shared, machine-
readable statement of which hostnames are spoken for.** The hold list is exactly that statement
and it is currently unreadable to the only processes that act on it. Until then, every morning
that starts with a full zone spends one held property.

### Cause 8 RESOLVED at the root — 2026-08-12, the zone went Pro

The operator upgraded `benrichardson.dev` to a **Pro** plan. The record cap is now **3,500**,
not 200 ([Cloudflare DNS plan limits](https://developers.cloudflare.com/dns/reference/all-features/):
Free 200 for zones created on/after 2024-09-01, Pro/Business 3,500). The zone sits at **203/3500**.
The constraint that drove Causes 6b, 7, 8 and the `au-worksafe` path-hosting workaround is gone.

**The entitlement lagged the plan by hours, and the lag is the trap worth recording.** The zone
reported `plan: Pro Website`, `is_subscribed: true`, `plan_pending: null` while `POST dns_records`
still returned `81045 Record quota exceeded`. A throwaway-record probe failed; the same probe ~10
minutes later succeeded. **The plan field is not the entitlement — only a write proves the quota.**
The site factory's instinct on 08-10 (create a throwaway record rather than trust the count) was
right for the opposite reason too: it is the only way to tell a lagging upgrade from a real cap.

Note the account-level DNS quota shipped 2026-06-10 is **Enterprise-only**; non-Enterprise per-zone
quotas behave exactly as before, so it is not a factor here.

#### What the damage was, and what was restored

The cap consumed **two** held properties before it was lifted — the second one landing after the
08-11 sweep had already reported and predicted it:

| Property | Hostname taken | When | Renamed to |
|---|---|---|---|
| `au-inflation` | `au-cpi-explorer` | 08-10T13:46Z | `au-public-service` |
| `au-approvals` | `au-build-approvals` | 08-11T13:54Z | `au-fuel-prices` |

Both were restored on 08-12 by re-creating the CNAME, which was safe and unambiguous because
**the repo's Pages `cname` and the registry `url` still agreed on the original hostname** — the
factories only ever moved the DNS record, never the property's own configuration. Both are back to
`class=A / tls` (DNS healthy, certificate stalled) and back on the hold list, i.e. returned to
exactly the state they were in before the cannibalisation, not merely un-dark.

The three remaining targets — `au-insolvency-tracker`, `castwell-cast`, `facet-dice` — were never
reached and are intact.

`ghostdep`, which shipped 08-11 on `ben-gy.github.io/ghostdep/` because no record was available,
was moved to `ghostdep.benrichardson.dev`: record created, Pages custom domain set, **certificate
approved within minutes**, `https_enforced` on, IndexNow submitted (202). No source change was
needed — the factory had already shipped `public/CNAME`, `sitemap.xml` and `robots.txt` pointing at
the custom domain, and the relative Vite `base` it fell back to resolves correctly at both URLs.

That immediate issuance is also the **fifth** consecutive datapoint that the Let's Encrypt budget is
not the constraint: a brand-new hostname issued on demand on the same morning seven tainted ones
stayed frozen.

#### What is NOT fixed

- **The seven held properties are unchanged.** The 08-06 hostname taint is a Let's Encrypt-side
  failed-validation block and has nothing to do with DNS capacity. The 08-14 re-probe still stands —
  though `au-inflation`'s answer is now weaker evidence than the others', because its record spent
  ~40h absent in the middle of the quiet week.
- **The coordination gap from Cause 8 remains open.** The factories and this routine still have no
  shared, machine-readable statement of which hostnames are spoken for. It stopped firing because
  the factories no longer have a reason to reclaim, not because anything reconciled them. If the
  zone is ever pressured again — or if a factory adopts a "tidy up dead records" habit on its own —
  it recurs exactly as before.
- **Two dormant repos still claim hostnames with no DNS record and no registry entry:**
  `collate` (archived, Pages never built, `status: null`) and `au-road-deaths` (Pages never built).
  Neither was restored: publishing a dormant property is outside this routine's remit, and with no
  DNS record neither is takeover-shaped. They are noise in any zone-vs-repo audit — the ghostdep
  08-11 audit flagged them — and clearing the stale Pages custom-domain claim on each would remove
  that noise. Operator's call.
- **`au-worksafe` remains on path-based hosting.** It works and costs nothing, but it is now there
  by inertia rather than necessity.

#### The ceiling that is now binding again

With the zone cap gone, the **Let's Encrypt 50/registered-domain/week limit returns as the only
structural wall**, at roughly **N ≈ 373 properties**, i.e. around mid-October 2026 at three per day
from 190. The four architectural options under Cause 2 are unchanged, and one of them — path-based
hosting — is now a deliberate choice rather than a forced one, with `au-worksafe` and the 08-11
`ghostdep` fallback both proving it works.

### Budget measured a fifth day (2026-08-12 08:10): 22 / 50

3 · 3 · 2 · 3 · 3 · 4 · 3 · 1 across 08-05…08-12. **Twenty-eight spare**, the lowest burn measured
yet and still falling as the 08-04 spike rolls out of the window. The 08-04 batch that started this
whole investigation is now entirely outside the trailing seven days.

The modelled line the script printed this morning was `~35.8/50` — **fourteen certificates above the
measured figure**, the widest the gap has been. Every measurement since 08-06 has said the same
thing and the divergence is growing monotonically with fleet size, exactly as the 08-09 note
predicted it would: the model is a fleet-size proxy, not a headroom measurement. Six consecutive
days now agree that **issuance works and the seven-property backlog is hostname-scoped**.

### `huntress` — a second path-hosted property, and this one cannot be moved (2026-08-12)

The 08-11 game factory run shipped `huntress` to `ben-gy.github.io/huntress/` for the same reason
`ghostdep` went there: the zone was full. It shows up in any budget measurement as a property with
no certificate. **It is not breakage** — it is the third entry in the "no `benrichardson.dev` Pages
cert" list above, alongside `au-worksafe` and `provenova`, and the sweep correctly scores it live.

It is worth distinguishing from `ghostdep`, because the two look identical in a registry listing and
only one of them was safely movable:

| | `ghostdep` | `huntress` |
|---|---|---|
| registry `url` | `ghostdep.benrichardson.dev` | `ben-gy.github.io/huntress` |
| `public/CNAME` in repo | present, custom host | **absent** |
| repo Pages `cname` | set after the move | `null` |

`ghostdep` could be moved on 08-12 without touching a line of its source because **the property
itself already declared the hostname it wanted** in three places, and the move only had to make DNS
and Pages agree with that declaration. `huntress` declares the opposite: every artefact it owns says
it lives at the path. Moving it would mean inventing a hostname for it and editing its source to
match — both standing guardrails of this routine, and the reason it was reported rather than fixed.

**This is a factory-side decision, not a sweep-side one.** The game factory's fallback fired on
08-11 when the cap was real; the cap is gone as of 08-12, so the next run should not need it. If a
factory ships another path-hosted property *after* today, the fallback has become unconditional and
should be looked at — the zone has 3,297 free records and no reason to reach for it.

### Same-date reruns overwrite the run log (known, unfixed)

`fleet-ssl.mjs` writes `ssl-<date>.md`, so a second run on the same day replaces the first run's
file rather than appending. The 08-12 00:27 session enabled `https_enforced` on `ghostdep` and its
log said so; this morning's 08:10 run overwrote that file and the line is gone from the record
(the change itself is intact and committed in `43e093a`).

Harmless on a day with one scheduled run, but this routine's logs *are* its durable memory — Cause 6
is the whole lesson about conclusions that do not survive into the next unattended run — so a log
that can silently lose a completed action is the same shape of problem, one size smaller. Worth
`--log=PATH` on any ad-hoc rerun until the script appends or suffixes instead.
