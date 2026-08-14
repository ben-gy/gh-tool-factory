# Fleet SSL sweep — 2026-08-15
**Checked:** 197 (site 74 · game 61 · tool 62)
**Live:** 190 → 190 · **Still broken:** 7
**Cert budget:** ~36.3 of 50 per week. Property ceiling ~373.
## On hold — deliberately not cycled

- metascrub (tool) `authorization_created` on `metascrub.benrichardson.dev` — 7d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- au-insolvency (site) `authorization_created` on `au-insolvency-tracker.benrichardson.dev` — 7d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- au-approvals (site) `authorization_created` on `au-build-approvals.benrichardson.dev` — 7d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- noisewell (tool) `authorization_created` on `noisewell.benrichardson.dev` — 7d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- au-inflation (site) `new` on `au-cpi-explorer.benrichardson.dev` — 7d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- facet (game) `new` on `facet-dice.benrichardson.dev` — 7d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.
- castwell (tool) `new` on `castwell-cast.benrichardson.dev` — 7d left, 08-15 canary result: castwell was cycled alone and did not move off 'new' in 36min (healthy hostnames issue in minutes). Cycling these hostnames looks futile; do NOT mass-cycle. Awaiting operator decision on fresh hostnames.

## Still broken

| Property | Cat | Class | cert_state | probe |
|---|---|---|---|---|
| au-inflation | site | A | `new` | tls |
| au-approvals | site | A | `authorization_created` | tls |
| au-insolvency | site | A | `authorization_created` | tls |
| facet | game | A | `new` | tls |
| castwell | tool | A | `new` | tls |
| noisewell | tool | A | `authorization_created` | tls |
| metascrub | tool | A | `authorization_created` | tls |
