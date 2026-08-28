# Gateway — extending the runtime to non-technical clients

Spec only — nothing here is implemented yet, written before any service code. Grounded in one real instance's folder names (`iou/`, `flow/<site>/`, `clients/<slug>/`) for concreteness — swap in your own app folders when applying this elsewhere, same as `OPERATIONS.md`'s `dispatch-room/` example.

## Problem

The base pattern (folder = scope boundary, that folder's `README.md` = tool schema, a Claude Code session started inside it acts as the REST client) is inherently single-operator: a human has to sit in the terminal to type requests and to answer the confirmation checklists (see this pattern's own Guardrails section — a mutating-call checklist, an irreversible-tier confirmation gate). Extending this to clients who aren't running Claude Code means the agent has to live somewhere they can reach without a terminal — a chat channel (WhatsApp, Slack, etc.) — which is a different runtime, not a repo tweak.

## What stays the same

App folders (`iou/`, `flow/<site>/`) keep owning the REST contract exactly as they do today: auth method, Allowlist, Guardrails, Mutating call safety checklist. None of that content changes — it gets a second consumer, not a rewrite.

## Decided: don't use OpenClaw's generic WordPress skill

2026-08-29, prompted by [wpmet.com's OpenClaw↔WordPress writeup](https://wpmet.com/connect-openclaw-to-wordpress/). That community skill's Application Password route grants the agent the same authorization level as a full WP admin hitting the REST API — not scoped to specific actions. The article's own advice — "treat downloaded OpenClaw skills like untrusted code... read the SKILL.md file... to ensure the agent isn't requesting dangerous permissions" — is the tell that it's broad by default, which is exactly the "full WP access" this project doesn't want for clients.

**Decision: never install it.** For any client granted `flow/<site>/`, that folder's own Flowkit REST contract (specific registered routes, gated by the `flowkit_execute_actions` capability, not generic post/page/user CRUD) is the skill/tool schema fed to their gateway agent — same as planned for `iou/`. The scope boundary comes from the Flowkit contract, never from Application Passwords or a generic community skill.

**Follow-up, Flowkit-side, not gateway-side:** flows have three trigger paths — cron (scheduled), an agentic route (REST, meant for programmatic/agent callers), and a normal manual REST call. Cron-triggered flows run fixed logic on a timer and don't need parameters. The agentic-route path is the one a client's chat request would actually hit, and today those routes are zero-parameter, one-hardcoded-action-per-flow (`send-weekly-events` takes no body). If a client needs to influence a flow's behavior (not just "run the fixed thing now"), that route needs typed parameters — same shape as `iou/`'s `remote_add` fields — still gated by the same capability check, still a named registered route. This is a change to the Flowkit plugin's own action definitions, unrelated to which gateway is used.

## What's new: the Gateway

A continuously-running, hosted process that:
- connects to one or more chat channels (WhatsApp, Slack, etc.)
- routes each incoming message to the right per-client agent, by sender identity
- runs that agent against a scoped skill (an app folder's README) and a scoped, registered set of tools — not an open "run any curl command" capability

**Candidate: [OpenClaw](https://github.com/openclaw/openclaw)** (`openclaw/openclaw`) — evaluated 2026-08-28. Real, active project (~388k★, 81k forks, org-owned, pushed same day checked), self-hosted, multi-channel (WhatsApp/Slack/Telegram/Discord/etc.), built around exactly this shape: a "Gateway" as the single control plane for sessions/routing/channels, per-agent config entries (`agents.entries.<id>`) with independent tool restrictions and separate credential storage under `~/.openclaw/agents/<agentId>/`, and "skills" that are literally `SKILL.md` files loaded into the agent's prompt — close to this repo's README-as-schema convention already.

Two things to verify before depending on it, not yet confirmed:
- **License** — marketing pages describe it as MIT; GitHub's own repo metadata reports the license as `"Other"` (custom/non-standard). Read the actual `LICENSE` file before using it for client-facing/commercial work.
- **Trust model** — its own docs state "Gateway and node as one operator trust domain" and that per-sender tool restrictions (`tools.toolsBySender`) "do not authenticate or sanitize other content in that model prompt." It supplies isolation *primitives* (per-agent credentials, per-agent tool deny-lists); it does not make client isolation automatic. That's this spec's job (below).

**Evaluated and dropped: [NanoClaw](https://github.com/nanocoai/nanoclaw)** — 2026-08-29. A lighter, container-per-agent alternative (real project, ~30.6k★, MIT-licensed, built directly on Anthropic's Agent SDK); its per-container isolation is a genuinely stronger answer to the trust-model caveat above than OpenClaw's config-level isolation. Dropped anyway: no native web-embeddable chat widget, no Chatwoot/Typebot support, and no generic webhook-channel mechanism — only WhatsApp/Telegram/Discord/Slack/Teams/iMessage/Matrix/Google Chat/Webex/Linear/GitHub/WeChat/email, added as copied adapter modules per channel, not configured. Since the frontend (WordPress-embedded chat) is a priority requirement here and OpenClaw's native `WebChat` script-tag widget directly covers it, OpenClaw stays the candidate.

This spec doesn't hard-depend on OpenClaw specifically — anything with per-sender routing, isolated per-agent config, and a tool-registration API would map the same way. If OpenClaw doesn't pan out, re-evaluate against this same mapping.

## Mapping the existing pattern onto a gateway agent

Per client, one gateway agent:

| existing concept | gateway equivalent |
|---|---|
| app folder's `README.md` | agent's skill (prompt content) — unchanged content, new consumer |
| app folder's `.env` | agent's own credential store — never shared across two clients' agents, even if both use the same app |
| app folder's Allowlist table | the exact set of tools registered for that agent — nothing outside it is reachable, regardless of what the model decides |
| app folder's Guardrails (blast-radius cap, scope cap) | checked in the tool's code before the HTTP call fires, not just written for the model to follow |
| app folder's Mutating call safety checklist | the tool implementation holds an irreversible-mutating call pending, restates exactly what will be posted, and fires only on an explicit confirming reply from that same sender — **client confirms directly**, decided 2026-08-28 |

The shift this forces: today the checklist is prose a trusted operator (you) is asked to follow inside an interactive session. Once the sender is a client instead of you, that checklist has to become enforced code — a registered tool, not a documented convention — because there's no guarantee a chat message is as careful as a terminal operator.

**Folders are local source-of-truth, not a network hop.** The Gateway doesn't "connect to" `iou/` or `flow/<site>/` the way it connects to WhatsApp or the IOU tracker itself — it just reads that folder's `README.md`/`.env` off local disk at agent-load time (skill content + credential), then makes the real outbound REST call straight to the actual target system (`iou.subsite.dev`, the WordPress site), same destination an interactive Claude Code session in that folder hits today. So the app-folder tree has to physically live on (or be synced to) whatever host runs the Gateway — a laptop-only copy can't back a hosted, client-facing agent.

This co-location requirement isn't new: `OPERATIONS.md` (this pattern's own always-on deployment doc) already establishes it for the single-operator case — one tmux session per app folder on an always-on cloud instance, each Claude Code instance reading only its own folder's README, reachable via Remote Control from desktop/mobile. That precedent is exactly why the folder-as-local-config assumption holds here too. What the Gateway changes is *who* the session serves: RC gives *you* remote access to your own kept-alive session — it's your agent, reachable by you, from another device, not a channel anyone else can message. There's no per-sender identity or allowlist in that model; reaching the session at all means having your own RC access to it. The Gateway replaces "one tmux+CC session only you can reach" with "one registered agent+tool-set per client, reachable over a chat channel by that client" — same co-located-folder requirement, a genuinely different consumer, not just a remote version of the same thing.

This is a structural limitation of the base pattern itself — see the Limitations section in `README.md` for the general statement of it.

## Client identity binding

New: `clients/<slug>/` — one folder per client, template at `clients/client-slug/`. Maps a channel identity (WhatsApp number, Slack user ID) to a client, and that client to a scoped subset of one or more app folders' Allowlists. See `clients/README.md`.

Unrecognized sender → no agent match → nothing runs. Identity binding is an allowlist, not open access.

## What doesn't transfer automatically

- Client-to-client isolation — achieved here by giving each client its own agent entry, its own credential store, and a tool set limited to its granted Allowlist subset. The gateway gives the mechanism; the `clients/` layer is what actually applies it per client.
- Confirmation flow — business logic per app, written once per app's tool set, not a gateway setting.

## Migration path

1. **Pilot one app, one client** — a Dispatch Room-style app with a written Allowlist, Guardrails, and Mutating call safety checklist already in place is the best first candidate. Wire identity binding → skill → registered tools (including the pending-confirmation state for any irreversible-mutating action) → audit logging, end to end, for a single client.
2. Only after the pilot's scope enforcement and confirmation flow are proven against real traffic, add more clients and/or more apps.

## Resolved: model auth

Decided 2026-08-29. OpenClaw supports three Anthropic auth paths — standalone API key, Claude CLI reuse, or a `claude setup-token` — the latter two borrowing an existing Claude Code login/Pro subscription instead of separate billing. Per OpenClaw's own docs: subscription-backed usage (CLI reuse or setup-token) draws from the same usage limits as interactive Claude Code sessions, and OpenClaw doesn't persist/refresh that login itself — it depends on Claude Code staying logged in on that host. OpenClaw's own docs recommend a standalone API key for production/always-on gateway hosts for exactly this reason.

- **Default: a dedicated Anthropic API key**, not CLI-reuse of the Pro subscription — this is a continuously-running multi-client service (per the migration path above), not local/desktop use, so it shouldn't share a usage pool with, or depend on the login state of, personal interactive Claude Code sessions.
- **Exception: test/alpha phase (pilot, one client) may stay on Claude CLI reuse / Pro subscription** — acceptable while traffic is a single client's worth and the gateway host is one you're actively watching, so a broken login or shared usage limit is a quick fix, not a client-facing outage. Switch to the dedicated API key before onboarding beyond the pilot.
- **Fallback if cost becomes an issue: a DeepSeek API key.** OpenClaw supports multiple model providers, so this is a config swap, not an architecture change. But it isn't a drop-in swap for the guardrail design in this spec: the mutating-call confirmation flow and scope enforcement are written as real tool code (not prompt-trusted), which limits the blast radius of a weaker model — but tool-*selection* and prompt-injection resistance still vary by model tier, and that's exactly the surface a multi-client chat gateway exposes to untrusted-ish input. Re-validate the confirmation flow and injection resistance against DeepSeek specifically before using it in production, don't assume parity with Claude.

## Flagged risk: OpenClaw's WhatsApp channel is unofficial

Checked 2026-08-29. The QR-pairing setup (`openclaw channels login --channel whatsapp`) is not a Meta developer/business account — it's [Baileys](https://github.com/openclaw/openclaw/issues/23093), a reverse-engineered WhatsApp Web protocol. Linking via QR is the same mechanism as WhatsApp Web in a browser; no Meta app registration or business verification involved.

This carries real, documented ban risk, not a theoretical one: OpenClaw's own repo has an unresolved feature request (#23093, auto-closed stale, never addressed by a maintainer) citing recurring forced logouts and account bans, including a report of a bot "mass-messag[ing] contacts, likely triggering bans." Enforcement is reported as escalating through 2026, specifically targeting automation signatures — same server IP across accounts, 24/7 activity, non-human message frequency — which is what a multi-client gateway looks like from WhatsApp's side, independent of whether the traffic itself is abusive.

**Going live on WhatsApp for real should mean the official WhatsApp Business Platform (Cloud API)**: a Meta Business/Developer account, an app with the WhatsApp product added, business verification, a permanent access token, and approved message templates for business-initiated messages — a REST+webhooks integration, unrelated to QR pairing. OpenClaw core has no native support for that path (the feature request asking for it went unaddressed); the only route found is a third-party plugin (`openclaw-kapso-whatsapp`, bridging via a service called Kapso) outside the `openclaw` org — unverified maturity, needs its own vetting before it touches client message data.

- **Test/alpha phase:** Baileys/QR pairing is acceptable, same carve-out logic as the Pro-subscription exception above — low volume, one client, actively watched.
- **Before onboarding beyond the pilot on WhatsApp specifically:** either move to the official Cloud API (via a vetted bridge, or once/if OpenClaw adds native support) or default new clients to Slack instead, which doesn't carry this unofficial-protocol risk.

## Open questions, to resolve at pilot time

- Which channel first — depends on which the actual pilot client already uses (WhatsApp vs. Slack). If WhatsApp, factor in the ban-risk note above.
- Where the gateway process runs — needs a persistent host, not a laptop, and the app-folder tree needs to be co-located there (see the folders-are-local-source-of-truth note above).
- Whether multiple clients share one app deployment/credential (scoped down per client) or each gets their own deployment — affects how `clients/<slug>/README.md`'s credentials field is filled in.

## Changelog

- **2026-08-29** — Moved here from a private dispatcher instance's repo, generalized cross-references (`OPERATIONS.md`/`README.md` are now same-repo siblings rather than cross-repo mentions). Content otherwise unchanged.
- **2026-08-29** — Added the RC-vs-Gateway distinction (RC = you reaching your own session, not a client-facing channel) and linked it to the Limitations section in `README.md`, stating the single-operator property as a structural limitation of the base pattern.
- **2026-08-29** — Clarified that app folders are local source-of-truth (skill + credential read off disk), not a network hop — the real REST call still goes straight to the target system, same as today's interactive-session flow. Noted the app-folder tree must be co-located with the Gateway host, and tied it to the existing precedent in `OPERATIONS.md` (tmux + Remote Control, single-operator case).
- **2026-08-29** — Decided against OpenClaw's generic community WordPress skill (Application Password route = full-admin-equivalent REST access, not scoped). `flow/<site>/`'s own Flowkit contract stays the skill source for WordPress clients. Noted follow-up: agentic-route flows are currently zero-parameter and would need typed params (Flowkit-side change) to do more than re-run a fixed action; cron-triggered flows don't need this.
- **2026-08-29** — Evaluated NanoClaw as an OpenClaw alternative, dropped it: no native web chat widget/Chatwoot/Typebot support, and the WordPress-embedded frontend is a priority requirement. OpenClaw's native `WebChat` widget covers that; staying with OpenClaw.
- **2026-08-29** — Flagged WhatsApp channel risk: OpenClaw's core WhatsApp support is Baileys (unofficial, reverse-engineered), with documented ban risk in OpenClaw's own unresolved GitHub issues, escalating in 2026. Going live on WhatsApp for real should use the official Business Platform (Cloud API) via a Meta developer/business account; no native OpenClaw support exists yet (only an unvetted third-party bridge plugin). QR/Baileys treated as test/alpha-only, same tier as the Pro-subscription exception.
- **2026-08-29** — Resolved model auth: default to a dedicated Anthropic API key (not Claude Code/Pro subscription reuse) for the gateway; DeepSeek API key kept as a cost fallback, flagged as needing its own injection-resistance validation, not an assumed drop-in. Pro-subscription/CLI-reuse route kept as an explicit exception for the test/alpha (single-client pilot) phase only.
- **2026-08-28** — Initial spec, written before any implementation. OpenClaw evaluated as gateway candidate (license and trust-model caveats above unresolved). Confirmation policy decided: client confirms directly.
