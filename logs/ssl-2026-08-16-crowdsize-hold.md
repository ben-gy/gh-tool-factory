# Fleet SSL sweep — 2026-08-16
**Checked:** 200 (site 75 · game 62 · tool 63)
**Live:** 192 → 192 · **Still broken:** 8
**Cert budget:** ~36.6 of 50 per week. Property ceiling ~373.
## On hold — deliberately not cycled

- metascrub (tool) `authorization_created` on `metascrub.benrichardson.dev` — 6d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- au-insolvency (site) `authorization_created` on `au-insolvency-tracker.benrichardson.dev` — 6d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- au-approvals (site) `authorization_created` on `au-build-approvals.benrichardson.dev` — 6d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- noisewell (tool) `authorization_created` on `noisewell.benrichardson.dev` — 6d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- au-inflation (site) `new` on `au-cpi-explorer.benrichardson.dev` — 6d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- facet (game) `new` on `facet-dice.benrichardson.dev` — 6d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- castwell (tool) `new` on `castwell-cast.benrichardson.dev` — 6d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- crowdsize (tool) `new` on `crowdsize.benrichardson.dev` — 2d left, Fresh hostname minted 08-15T19:29Z, stalled at cert=new since creation; cycled alone 08-16 22:10Z. The 08-17 run fires at almost exactly the 24h cooldown boundary and would abandon that authorization (same trap as castwell on 08-15). Config verified identical to au-spectrum, which shipped 5h earlier the same night and issued. Lapses 08-18 so normal repair resumes on its own.

## Still broken

| Property | Cat | Class | cert_state | probe |
|---|---|---|---|---|
| au-inflation | site | A | `new` | tls |
| au-approvals | site | A | `authorization_created` | tls |
| au-insolvency | site | A | `authorization_created` | tls |
| facet | game | A | `new` | tls |
| crowdsize | tool | A | `new` | tls |
| castwell | tool | A | `new` | tls |
| noisewell | tool | A | `authorization_created` | tls |
| metascrub | tool | A | `authorization_created` | tls |
