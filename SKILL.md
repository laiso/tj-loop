---
name: session-supervisor
description: Run a long task as a pair of sessions — a worker and a supervisor. The supervisor wakes up periodically, reads the worker's state, judges progress, and sends a resume instruction when the worker stalls. Works with Claude Code natively, and with Codex, OpenCode, and Pi through the adapters below. Use when a long autonomous run needs to be kept going.
---

# Session Supervisor

## Roles

- Worker session: runs the task.
- Supervisor session: checks the worker's state periodically, judges progress, and intervenes only when needed.

Unlike a loop that repeats the same instruction without judging, here the supervisor reads progress and decides: let it continue, resume it, or hand it to the human.

## Common rules (all environments)

- State lives in files, not in the supervisor's memory. The worker appends progress to a named file (e.g. progress.md); the supervisor appends one line per judgment or intervention to its own log (e.g. supervisor-log.md). A supervisor run can start fresh and still pick up where the last one left off.
- On each wake-up, the supervisor reads the worker's state and the tail of the progress file, then judges:
  - Progressing (working, and the progress file is moving): do nothing.
  - Stalled (idle with no progress change, or blocked): send a resume instruction, attaching the last state from the progress file.
  - Done: summarize the result for the human and stop.
  - Needs judgment (work outside the worker's permissions, the worker declined an instruction, the same stall repeats): stop intervening and hand it to the human.
- One resume instruction per stall. Never resend for the same stall. If a resume does not move it, hand it to the human.
- The periodic mechanism is the supervisor's own business (a loop inside a session, or cron outside it). The worker's environment does not need a goal mode: continuation is supplied from outside.

Only the mechanics differ per environment: how the supervisor reads state, how it delivers an instruction, and where the heartbeat lives. Pick the adapter for the worker's environment.

## Adapter: Claude Code

- Launch the worker: `claude --bg --dangerously-skip-permissions "<task>"`. Note the printed ID and session name.
- Read: the in-session `ListAgents` tool, or `claude agents --json` via Bash (status busy/idle, state working/done). Works without a terminal.
- Write: the in-session `SendMessage` tool. There is no CLI command for sending into a running session, so this adapter runs inside the supervisor session.
- Heartbeat: `/loop <interval>` in the supervisor session, with a prompt that applies the common rules.
- Verified behavior (Claude Code 2.1.328, demo on 2026-11-19):
  - An instruction lands in the worker's inbox and is read at the worker's next tool round. During one long tool call it is not delivered until that call returns (a message sent during a 150-second single call was read right after the call ended, about 146 seconds later).
  - An instruction arrives as a request, not a command. The worker judges within its own permissions before complying. In the demo, the worker declined an out-of-scope instruction (delete its own files), passed it to the user, and continued its task. Do not force a declined instruction.
- Stop: stop the /loop; stop the worker with `claude stop <id>`.

## Adapter: Codex

Codex has its own /goal (internal continuation judged by Codex itself). This adapter puts the judgment outside, and can sit on top of a /goal worker.

### Arrangement A: app-server (native read/write)

- Run the worker's threads on `codex app-server` (default Unix socket; the WebSocket transport is documented as experimental). A human-facing TUI can attach to the same server with `codex --remote <addr>`, so the human and the supervisor share the worker.
- Read (JSON-RPC): `thread/read` and `thread/list` return a runtime status per thread (`notLoaded` / `idle` / `systemError` / `active`, with flags such as `waitingOnApproval`); `thread/status/changed` pushes changes; `thread/goal/get` reads the worker's goal state.
- Write: `turn/start` starts a new turn on an idle thread (this is the resume instruction); `turn/steer` adds input to a running turn; `turn/interrupt` stops a turn.
- Heartbeat: cron (or any scheduler) waking the supervisor, which connects to the app-server per wake-up.

### Arrangement B: plain TUI session in tmux or Herdr (terminal route)

Use when the worker is a standalone `codex` TUI session that no app-server hosts.

- Read: tail the session's rollout file (`$CODEX_HOME/sessions/YYYY/MM/DD/rollout-*.jsonl`) and capture the pane (`tmux capture-pane -p`, or `herdr pane read <pane>`).
- Write: key input — `tmux send-keys`, or `herdr agent prompt <agent> "<instruction>"`. Key input lands as typed text, not as a request the worker can decline. Check the pane's input line before and after sending: text already sitting in the input box can mix with what you send.
- Heartbeat: cron waking the supervisor.
- This is the arrangement codex-whip documents (see the repository README's references).

## Adapter: OpenCode

OpenCode is client/server: the TUI is a client of the server, so a session a human is driving in the TUI is reachable through the API.

- Run (or reuse) `opencode serve` (default port 4096).
- Read: `GET /api/session` (list), `GET /api/session/active`, `GET /api/session/{sessionID}/message` (timeline); or the SDK (`@opencode/client`: `client.session.*`, `client.event.subscribe()`). On 1.x: `GET /session/status` returns busy/idle per session.
- Write: `POST /api/session/{sessionID}/prompt` (2.x) — a prompt sent to an idle session restarts its agent loop, which is the resume action. On 1.x: `POST /session/:id/prompt_async`.
- Heartbeat: cron; the supervisor is just an HTTP client. (OpenCode 2 also has an internal /goal and /schedule; neither is required here.)

## Adapter: Pi

- Arrangement: the supervisor spawns the worker itself as `pi --mode rpc` and drives it over stdin/stdout (LF-delimited JSON). RPC is point-to-point: a supervisor can only drive a Pi process it spawned.
- Read: the stdout event stream, plus the `get_state` command (`isStreaming`, `messageCount`, `pendingMessageCount`); the `agent_settled` event means the worker will not continue on its own.
- Write: the `prompt` command starts a run on an idle session; `steer` and `follow_up` deliver input while it is streaming.
- Heartbeat: the supervisor process itself (it owns the child), or cron around it.
- An already-running interactive Pi session is different: reading works (tail its session file under `~/.pi/agent/sessions/`), but there is no external write path — use terminal key input, or a helper extension loaded into the worker in advance that calls `pi.sendUserMessage` on a signal you control.

## Stopping

- Stop the heartbeat first (the /loop, the cron job, or the supervisor process), then stop the worker if it is still running.
