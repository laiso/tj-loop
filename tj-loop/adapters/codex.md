# Adapter: Codex

Status: not yet run end-to-end; written from the official Codex documentation (app-server protocol and CLI).

## Arrangement

- Worker: a Codex session (thread).
- Supervisor: any agent that can run on a schedule (cron) and either speak JSON-RPC to the app-server (Arrangement A) or drive a terminal (Arrangement B).
- Terminal multiplexer: not required for A; required for B (a multiplexer such as tmux, Herdr, or Orca hosts the worker's TUI).
- Note: this adapter also works on top of a worker that runs Codex's own /goal; the judgment stays outside either way.

## Mechanics

### A: app-server (native read/write)

- Run the worker's threads on `codex app-server` (default Unix socket; the WebSocket transport is documented as experimental). A human-facing TUI can attach to the same server with `codex --remote <addr>`, so the human and the supervisor share the worker.
- Connect per wake-up and speak JSON-RPC 2.0 over the socket. A read is one round trip, e.g. `{"jsonrpc":"2.0","id":1,"method":"thread/list","params":{}}`.
- Read: `thread/read` and `thread/list` return a runtime status per thread (`notLoaded` / `idle` / `systemError` / `active`); the `waitingOnApproval` flag on an active thread means the worker is blocked. `thread/status/changed` pushes changes; `thread/goal/get` reads the worker's goal state.
- Write: `turn/start` starts a new turn on an idle thread (this is the resume instruction); `turn/steer` adds input to a running turn; `turn/interrupt` stops a turn.
- Heartbeat: cron (or any scheduler) waking the supervisor.

### B: plain TUI session in a terminal multiplexer (terminal route)

Use when the worker is a standalone `codex` TUI session that no app-server hosts.

- Read: tail the worker's rollout file under `$CODEX_HOME/sessions/YYYY/MM/DD/` (pick the file for the worker's session; if the worker is the only session, the most recently modified one) and capture the pane (`tmux capture-pane -p`, or `herdr pane read <pane>`).
- Write: key input — `tmux send-keys` (send the text, then Enter as a separate key to submit), or `herdr agent prompt <agent> "<instruction>"`. Key input lands as typed text, not as a request the worker can decline. Check the pane's input line before and after sending: text already sitting in the input box can mix with what you send.
- Heartbeat: cron waking the supervisor.

## Stopping

- Stop the heartbeat first. For a worker the supervisor launched, interrupt its current turn (A: `turn/interrupt`; B: Ctrl-C in the pane) and leave the session in place.
