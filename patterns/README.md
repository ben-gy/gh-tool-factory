# patterns/ — shared building blocks

Proven, production-tested files. **Copy these into a new tool and adapt —
re-rolling one from scratch is a defect.**

| File | What it gives you | How to use |
|---|---|---|
| `feedback.ts` | Self-contained feedback dialog: footer trigger, bug/idea toggle, optional email, focus trap, honeypot + dwell-time anti-spam, scoped `fbw-` class names, light/dark aware. Zero dependencies. | Copy to `src/feedback.ts`, call `mountFeedback()` once after the footer exists in the DOM. **MANDATORY on every tool** (UX surface #13). |

## feedback.ts

GENERATED from `gh-feedback/widget/feedback.ts`. Never edit the copy — edit the
canonical file and re-run `gh-feedback/scripts/distribute.mjs`.

```ts
import { mountFeedback } from './feedback';
mountFeedback();
```

Submissions POST to `https://feedback.benrichardson.dev`, which resolves the
target repo from the request Origin and files a labelled issue on the tool's own
repo. The `feedback-triage` routine picks those up daily.

**CSP — this is the one that bites tools.** Most tools ship a restrictive
`connect-src`. It MUST include `https://feedback.benrichardson.dev`:

```
connect-src 'self' blob: data: https://cloudflareinsights.com https://feedback.benrichardson.dev
```

Without it every submission fails silently — the user sees a generic error and
the only trace is a console CSP violation.

**Threat Model wording.** The Trust Surface section must disclose it:

> Feedback you choose to send (and an email address, only if you supply one) is
> sent to feedback.benrichardson.dev. Nothing is sent unless you open the
> feedback form and press Send; your files and data never are.
