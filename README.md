# readme-runtime

A pattern for binding an LLM runtime to a single REST API — no MCP, no skills.

- **README = tool schema.** One markdown file documents an app's REST surface: endpoint, auth header, JSON shape, curl example. It's prose a runtime parses, not code.
- **Claude Code session = runtime.** The session reads the README, builds requests from natural-language commands, fires them.
- **Folder = scope boundary.** The folder a session runs in determines which README loads and which app it targets. No bleed into other folders/apps.

Not MCP (no protocol, no discovery, no typed registry). Not a skill (not portable across folders — the README only means something inside its own folder, by design).

## Why this approach

- **Zero infra.** No gateway service, no registry server, nothing to run, secure, or patch. The runtime is a Claude Code session; the schema is a file.
- **Trivially auditable.** Scope boundary is one folder + one markdown file — readable top to bottom, diffable, lives in git. No hidden config, no plugin loader.
- **Smaller attack surface.** No plugin/channel surface to harden — the only inputs are the folder's README and the user's own commands, no third-party integrations running code in-process.

Tradeoff: this buys tool-binding simplicity, not reach. It has no multi-channel access story of its own (today: tmux + Claude's own Remote Control) — a gateway-style tool (e.g. OpenClaw) solves that separately and can sit in front of a readme-runtime folder rather than replace it.

## How it works (folder-based workflow)

```
readme-runtime/
├── TEMPLATE.md          ← schema template, copy this per app
├── dispatch-room/
│   └── README.md        ← filled-out schema for Dispatch Room
└── <your-app>/
    └── README.md        ← filled-out schema for <your-app>
```

![Core runtime loop: user NL command in one folder is parsed against that folder's README schema, matched to an endpoint, read-only calls fire directly while mutating calls pass through an audit gate, then the app's REST API responds and the result is reported back to the user](images/task-flow-diagram.png)

1. Each app gets its own folder.
2. That folder's `README.md` is the only context the runtime needs for that app — endpoints, auth, request/response shapes, a mutating-call checklist.
3. You `cd` into the folder and start a Claude Code session there (or open a persistent one, e.g. via tmux).
4. The session reads the README, then acts as the client: parses your NL command, constructs the matching HTTP request, fires it, reports the result.
5. Scope is enforced by convention, not code — a session in `dispatch-room/` has no reason to read another app's README, so it doesn't act outside that API.

There is no cross-folder registry and no shared runtime process. Each folder is independent; the pattern is the convention of *how the README is written*, not shared infrastructure.

## Install / setup

No install step beyond having Claude Code itself. Per app:

1. Copy `TEMPLATE.md` into a new folder as `README.md`.
2. Fill in auth, endpoints, and the Inputs/Process/Outputs contract for each one (see template for structure).
3. Put any secrets (API keys, tokens) in that folder's local env — never hardcode them into the README itself, since the README is meant to be read (and potentially shared/committed).
4. Start a Claude Code session in that folder. Point it at the README once ("read README.md, use it to call the API") — after that, natural-language commands in that session are interpreted against the schema.

For a persistent/interactive setup (always-on cloud instance, tmux per app folder, managed remotely from desktop/mobile) see [`OPERATIONS.md`](OPERATIONS.md) — this is how Dispatch Rooms run today.

## Onboarding a new app

1. `mkdir <app-name>` at repo root.
2. Copy `TEMPLATE.md` → `<app-name>/README.md`.
3. Fill in every section — auth, canonical sources, scope/allowlist, guardrails, per-endpoint Inputs/Process/Outputs, and mutating-call safety.
4. Mark each endpoint's risk tier: read-only, reversible-mutating, or irreversible-mutating. Read-only fires directly. Reversible-mutating must pass the shared checklist, then fires automatically. Irreversible-mutating must pass the checklist *and* get explicit user confirmation every time — it never auto-fires, pass or fail.
5. Test with a real session: `cd <app-name>`, start Claude Code, ask it to read the README, then try one read-only call and one mutating call (with a value you can verify) before trusting it for real use.

## Repo structure

Single repo: one `TEMPLATE.md` at root, one subfolder per app, each with its own filled-out `README.md`. Apps don't share a runtime or registry — the folder boundary *is* the isolation, so keeping them in one repo is just for convenience of tracking the pattern's evolution, not because the apps are coupled.

## First instance

**Dispatch Room** — a self-hosted PHP IOU tracker (iou.subsite.dev) with REST endpoints (`remote_add`, `remote_balances`, `remote_transactions`, `remote_endpoints`). See `dispatch-room/README.md` once retrofitted against the template.

## Prior art

[Interpretable Context Methodology (ICM)](https://github.com/RinDig/Interpretable-Context-Methodology) — folder structure as agent architecture. Borrowed: stage-contract format (Inputs/Process/Outputs), canonical sources (one file owns a fact, others point to it), lightweight pre-mutation audits. Not borrowed: numbered stage folders, layered context loading, human checkpoints between stages — those solve a multi-step pipeline problem this pattern doesn't have (single-shot, one folder, one schema, one call per command).
