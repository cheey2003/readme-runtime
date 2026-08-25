# Operations — persistent / cloud setup

How to run readme-runtime sessions as always-on, remotely-manageable instances rather than one-off local sessions. This is the deployment pattern Dispatch Room uses.

## Overview

```
cloud instance (always-on)
└── dispatch/                    ← parent folder, one per "group" of apps (optional)
    ├── dispatch-room/
    │   └── README.md            ← schema for this app
    │   └── (tmux session "dispatch-room", CC running)
    └── <other-app>/
        └── README.md
        └── (tmux session "<other-app>", CC running)
```

One tmux session per app folder. Each session's Claude Code instance only ever reads its own folder's README — same scope-boundary rule as local use, just kept alive instead of started per command.

## Setup steps

1. **Provision the cloud instance.** Any always-on host that can run Claude Code (persistent VM, container, etc). Out of scope here — depends on your provider.
2. **Create the folder structure.** One top-level folder per app (or a parent folder grouping related apps, e.g. `dispatch/dispatch-room/`). Each app folder gets its own `README.md` per `TEMPLATE.md`.
3. **Start a tmux session per app folder.**
   ```bash
   tmux new -s <app-name>
   cd <path-to-app-folder>
   ```
4. **Start Claude Code inside that tmux session**, in that folder. Point it at the README once, same as local use ("read README.md, use it to call the API").
5. **Turn on Remote Control (RC)** so the session is reachable from Claude desktop/mobile without shelling in. Enable RC for the session — check your Claude Code version's docs/settings for the exact activation step, since this is a newer feature and the flag/command may differ by version.
6. **Detach, don't kill.** `tmux detach` leaves the session (and the Claude Code process inside it) running. Reattach with `tmux attach -t <app-name>` from the instance directly, or manage via RC from desktop/mobile without attaching at all.

## Day-to-day

- Manage running sessions from Claude desktop or mobile via RC — send commands, check status, without SSH.
- To add a new app to an already-running instance: repeat steps 2-5 for the new folder — existing sessions are untouched, scope boundary holds per-folder as always.
- To check what's alive: `tmux ls` on the instance lists all app sessions.

## Notes

- No shared state between app sessions — killing/restarting one tmux session doesn't affect another.
- If the instance itself restarts, tmux sessions don't survive a reboot by default — either re-run steps 3-5 per app, or wrap in a supervisor (systemd, etc.) if that matters for your uptime needs. Not covered here since it's infra-specific.
