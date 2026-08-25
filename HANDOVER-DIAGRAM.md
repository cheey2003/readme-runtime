# Handover: readme-runtime workflow diagrams

Paste this into a new session (Claude Design) to continue. Task: draw two diagrams of the readme-runtime pattern's workflow within the folder structure.

## Context (read these first — canonical, don't re-derive)

- `README.md` — pattern overview, folder-based workflow, onboarding steps
- `OPERATIONS.md` — persistent/cloud deployment shape (always-on instance, tmux per app, RC)
- `TEMPLATE.md` — per-app schema structure (Inputs/Process/Outputs, audit checklist)

One-line summary if you don't read further: folder = scope boundary, that folder's README.md = tool schema, a Claude Code session in that folder reads the README and acts as the REST client for natural-language commands.

## Diagram 1 — core runtime loop (single folder)

Show one app folder's operating cycle:

1. **Folder boundary** — draw as a literal box. Nothing crosses it except the NL command coming in and the result going out. This is the scope-enforcement mechanism — make it visually unmissable, it's the core idea of the whole pattern.
2. Inside the box: `README.md` sits as the schema — auth, endpoints, Inputs/Process/Outputs per endpoint.
3. NL command arrives → runtime (the CC session) parses it against the schema → constructs a request.
4. Branch here — three distinct paths, drawn as visibly different, by the endpoint's risk tier:
   - **Read-only** → fires directly, no gate.
   - **Reversible-mutating** → passes through a **checklist gate** (target matches, sign/amount/effect matches intent, JSON shape matches, no unflagged inferred fields) → pass fires the request, fail returns to user instead of firing.
   - **Irreversible-mutating** → passes through the same checklist gate, but a pass does *not* fire — it leads to a second, distinct **confirmation gate** where the runtime states plainly what will happen and cannot be undone, and waits for explicit user go-ahead. This path never auto-fires, checklist pass or fail.
5. Request → app's REST API (outside the folder box) → response → reported back to user.

Draw the two gates as visually distinct checkpoints (e.g. diamond shapes), not plain arrows — the checklist gate is automated (pass/fail), the confirmation gate is a hard stop for a human. That two-gate distinction on the irreversible path is the core safety idea worth making legible at a glance.

## Diagram 2 — persistent / cloud deployment shape

Show the operational layer from `OPERATIONS.md`:

1. Outer container: **always-on cloud instance**.
2. Inside it: an optional parent folder grouping related apps (e.g. `dispatch/`), containing one folder per app.
3. Each app folder has its own **tmux session** running as a parallel lane inside the instance — one lane per app, no shared state between lanes.
4. Each tmux session runs one Claude Code process, scoped to that one folder/README (same core loop as Diagram 1, just kept alive instead of started per command).
5. **RC (Remote Control)** layer reaching in from outside the instance — both desktop and mobile connect to it, able to reach any tmux lane without SSH.
6. Make clear: killing/restarting one lane doesn't touch another. The instance is the only shared resource; isolation is per-folder.

## Output format

Whatever Claude Design produces natively — this handover doesn't prescribe a file format. If it produces exportable assets meant to live in this repo (e.g. `diagram-core-loop.svg`, `diagram-deployment.svg`), drop them at repo root and add a link to each from the relevant section of `README.md` / `OPERATIONS.md` rather than duplicating the image elsewhere.
