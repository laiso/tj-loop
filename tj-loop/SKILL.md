---
name: tj-loop
description: Run a long task as a pair of sessions — a worker (Tom) and a supervisor (Jerry). The supervisor wakes up periodically, reads the worker's state, judges progress, and sends a resume instruction when the worker stalls. Works with Claude Code natively, and with Codex, OpenCode, Pi, and Antigravity through the adapter files in this directory. Use when a long autonomous run needs to be kept going.
---

# tj-loop — Session Supervisor

## Roles

- Worker session (Tom): runs the task.
- Supervisor session (Jerry): checks the worker's state on a fixed interval, judges progress, and intervenes only when needed.

## Common rules (all environments)

- State lives in files, not in the supervisor's memory. The worker appends progress to a named file (e.g. progress.md); the supervisor appends one line per judgment or intervention to its own log (e.g. supervisor-log.md). A supervisor run can start fresh and still pick up where the last one left off.
- When launching the worker, put the progress file in the task text: its path, and the instruction to append progress as the work goes. The judgments below depend on that file moving.
- The supervisor wakes up on a fixed interval — its heartbeat. The heartbeat is the supervisor's own business: a loop inside the supervisor's session, or a scheduler such as cron outside it. The worker's environment does not need its own continuation feature (some environments have one, such as /goal): continuation is supplied from outside.
- On each wake-up, the supervisor reads the worker's state, the tail of the progress file, and the tail of its own log (its last judgment, and whether it already intervened in this stall), then judges:
  - Progressing (working, and the progress file is moving): do nothing.
  - Stalled (no progress change across two consecutive wake-ups, or blocked — waiting for approval, waiting for input, or stopped on an error): send a resume instruction, attaching the last state from the progress file.
  - Done: summarize the result for the human and stop.
  - Needs judgment (work outside the worker's permissions; the worker will not proceed, including declining an instruction in environments where instructions arrive as requests; the same stall repeats after a resume): stop intervening and hand it to the human.
  - State cannot be read (the worker session is gone or unreachable): do not treat it as Done. Hand it to the human.
- One resume instruction per stall. Never resend for the same stall. If a resume does not move it, hand it to the human.
- Handing to the human means: report in the supervisor session's own output, and record the handoff in the supervisor log.

## Adapters

Only the mechanics differ per environment: how the supervisor reads state, how it delivers an instruction, and where the heartbeat lives. Before starting, read the adapter for the worker's environment:

- Claude Code — `adapters/claude-code.md` (native session messaging)
- Codex — `adapters/codex.md` (app-server, or a terminal route: tmux, Herdr, Orca, etc.)
- OpenCode — `adapters/opencode.md` (server API)
- Pi — `adapters/pi.md` (RPC)
- Antigravity (agy) — `adapters/antigravity.md` (ACP, or a terminal route: tmux, Herdr, Orca, etc.)

## Stopping

- Stop the heartbeat first, then stop the worker if it is still running. Each adapter gives the environment's specifics.
