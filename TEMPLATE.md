# <App Name> — Tool Schema

This file is the tool schema. Runtime (Claude Code session, this folder) reads it, builds requests, fires them. Not code — prose a runtime parses.

**Scope:** this folder = this app only. No bleed to other apps/folders.

## Auth

- Header: `<e.g. X-API-Key>`
- Value source: `<env var / file / where to find it>`
- Applies to: all endpoints below unless stated otherwise

## Canonical sources

If any value below (base URL, auth, shared enum) is also defined elsewhere in this folder, name the owning file here. Everything else points to it, doesn't repeat it.

- `<field>` → owned by `<file>`

If this folder wraps a large third-party API, the upstream docs are the canonical source for the full surface — this file does not restate them. Link once:

- Full API reference: `<upstream docs URL>`
- This file defines only the subset in **Scope / Allowlist** below.

## Scope / Allowlist

Only endpoints listed here are callable by the runtime. Anything else in the upstream docs is out of scope — do not call it even if technically reachable, even if the user's phrasing implies it. Default is deny, not "docs allow it."

| endpoint | upstream doc link | why this folder needs it |
|---|---|---|
| | | |

To permit a new endpoint: add a row here, then add its full contract in **Endpoints** below. Do not call an endpoint that has a contract but no allowlist row, or vice versa.

## Guardrails

General limits that apply across all endpoints, on top of the per-call audit below.

- **Rate limit:** max `<N>` calls per `<time window>` from this folder. If a single user request would require more, stop and ask rather than looping past it.
- **Blast radius cap:** max `<amount / quantity / resource count>` touched by any one call. A request that would exceed this is treated as irreversible-tier (see Mutating Call Safety) regardless of the endpoint's usual tier.
- **Scope cap:** never call an endpoint outside the Allowlist above, even to satisfy an otherwise-reasonable user request — surface the gap to the user instead.

## Endpoints

Repeat this block per endpoint.

### `<endpoint_name>`

**Inputs**
| field | type | required | notes |
|---|---|---|---|
| | | | |

**Process**
- Method + path: `<VERB> /path`
- What it does server-side (1-2 lines, only if non-obvious from name)
- Risk tier: `read-only` / `reversible-mutating` / `irreversible-mutating` — determines which gate applies, see Mutating Call Safety below

**Outputs**
- Success shape: `<JSON example>`
- Error shape: `<JSON example>`
- What changes in app state (mutating endpoints only)

**Example**
```bash
curl <example call with real shape>
```

## Mutating call safety

Which gate applies is set by the endpoint's Risk tier above. Both tiers share the same checklist; they differ in what happens when it passes.

**Shared checklist** — runtime checks this against the constructed request, pass/fail, not vibes:

- [ ] Target `<resource, e.g. tab/account/ID/server>` matches what user named, not a guess
- [ ] Amount/sign/direction/effect matches user intent (e.g. debt vs payment, create vs delete)
- [ ] JSON shape matches the Inputs table exactly — no missing/extra required fields
- [ ] Value is inside sane bounds (no unit mismatch, no obvious typo-magnitude) and inside the Guardrails blast-radius cap
- [ ] If any field was inferred (not stated explicitly by user), flag it back to user before sending

### Tier: reversible-mutating

Effect can be corrected or undone by another call (edit a note, add then remove a record). Checklist fail → ask user, don't send. Checklist pass → fire directly, no confirmation needed.

### Tier: irreversible-mutating

Effect cannot be undone by another call (delete, deprovision, transfer, anything with no corresponding undo endpoint) — also anything a Guardrails cap escalated to this tier. Checklist passing is necessary but not sufficient:

- [ ] Checklist above passes
- [ ] Runtime states back to the user, in plain terms, exactly what will happen and that it cannot be undone
- [ ] User gives explicit go-ahead in that turn

Never auto-fire this tier regardless of checklist result — hard stop for human confirmation every time, not just on failure.

## Non-mutating calls

Read-only endpoints skip the checklist entirely — fire directly, no confirm needed. Still subject to the Scope/Allowlist and rate-limit guardrails above.
