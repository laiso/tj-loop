# Adapter: Codex

Codex has its own /goal (internal continuation judged by Codex itself). This adapter puts the judgment outside, and can sit on top of a /goal worker.

## Arrangement A: app-server (native read/write)

- Run the worker's threads on `codex app-server` (default Unix socket; the WebSocket transport is documented as experimental). A human-facing TUI can attach to the same server with `codex --remote <addr>`, so the human and the supervisor share the worker.
- Read (JSON-RPC): `thread/read` and `thread/list` return a runtime status per thread (`notLoaded` / `idle` / `systemError` / `active`, with flags such as `waitingOnApproval`); `thread/status/changed` pushes changes; `thread/goal/get` reads the worker's goal state.
- Write: `turn/start` starts a new turn on an idle thread (this is the resume instruction); `turn/steer` adds input to a running turn; `turn/interrupt` stops a turn.
- Heartbeat: cron (or any scheduler) waking the supervisor, which connects to the app-server per wake-up.

## Arrangement B: plain TUI session in tmux or Herdr (terminal route)

Use when the worker is a standalone `codex` TUI session that no app-server hosts.

- Read: tail the session's rollout file (`$CODEX_HOME/sessions/YYYY/MM/DD/rollout-*.jsonl`) and capture the pane (`tmux capture-pane -p`, or `herdr pane read <pane>`).
- Write: key input — `tmux send-keys`, or `herdr agent prompt <agent> "<instruction>"`. Key input lands as typed text, not as a request the worker can decline. Check the pane's input line before and after sending: text already sitting in the input box can mix with what you send.
- Heartbeat: cron waking the supervisor.
- This is the arrangement codex-whip documents (see the repository README's references).
