# Clients — per-client identity and scope for gateway-hosted sessions

See `../GATEWAY.md` for why this exists and how it maps onto a gateway agent — this file just documents the convention for the folders below it. Don't restate gateway mechanics here, and don't restate an app's REST contract here either — see that app folder's own `README.md`/`CLAUDE.md`. App-folder names below (`iou/`, etc.) are illustrative, same as `GATEWAY.md`'s — swap in your own.

One subfolder per client. `client-slug/` is the template — copy it, don't edit it in place.

## What a client folder documents

- **Channel identity** — the exact WhatsApp number / Slack user ID (etc.) this client sends from. Get this directly from the client, never inferred or guessed.
- **Granted scope** — which app folder(s) (e.g. `../iou/`) this client can use, and which of *that app's own Allowlist* actions specifically. Must be a subset of the app's Allowlist — never wider than what the app folder itself permits, and never an action the app folder doesn't list at all.
- **Credentials** — which `.env` / credential store this client's gateway agent loads. If multiple clients use the same underlying app deployment, they still each get their own agent entry and credential reference in the gateway config — see `../GATEWAY.md`'s mapping table for why (client-to-client isolation is not automatic).
- **Confirmation** — fixed policy across all clients for now: client confirms directly, restated in full before any action the granted app folder classifies as irreversible-mutating fires. No per-client override exists yet; if one becomes necessary, add it here when it's actually needed.

## Adding a new client

1. Confirm the client's channel identity with them directly.
2. Copy `client-slug/README.md` to `<slug>/README.md`, fill in the fields above.
3. Register the identity → agent binding in the gateway's own config (DM pairing/allowlist), scoped to exactly this client's granted actions.
4. Dry-run the confirmation flow with the client before treating the channel as live for them.
