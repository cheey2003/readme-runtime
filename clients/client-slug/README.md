### Client — CLIENT-DISPLAY-NAME

See `../README.md` one level up for what this file is for and how a client folder is used. This file is just this client's values.

- **Channel identity:** CHANNEL (e.g. WhatsApp) — IDENTITY-VALUE (e.g. phone number or Slack user ID), confirmed directly with the client on DATE
- **Granted scope:**
  - `../../APP-FOLDER/` — actions: `ACTION-1`, `ACTION-2` (must be a subset of that folder's own Allowlist table)
- **Credentials:** which `.env` / credential store this client's gateway agent loads — APP-FOLDER-OR-DEPLOYMENT-REFERENCE
- **Confirmation:** client confirms directly (repo-wide default, see `../README.md`) — no override for this client
