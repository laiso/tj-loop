---
name: tj-loop
description: Run a long task as a pair of sessions — a worker (Tom) and a supervisor (Jerry). The supervisor wakes up periodically, reads the worker's state, judges progress, and sends a resume instruction when the worker stalls. Works with Claude Code natively, and with Codex, OpenCode, and Pi through the adapter files in this directory. Use when a long autonomous run needs to be kept going.
---

# tj-loop — Session Supervisor

## Roles

- Worker session (Tom): runs the task.
- Supervisor session (Jerry): checks the worker's state periodically, judges progress, and intervenes only when needed.

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

## Adapters

Only the mechanics differ per environment: how the supervisor reads state, how it delivers an instruction, and where the heartbeat lives. Before starting, read the adapter for the worker's environment:

- Claude Code — `adapters/claude-code.md` (native session messaging)
- Codex — `adapters/codex.md` (app-server, or the tmux / Herdr terminal route)
- OpenCode — `adapters/opencode.md` (server API)
- Pi — `adapters/pi.md` (RPC)

## Stopping

- Stop the heartbeat first (the /loop, the cron job, or the supervisor process), then stop the worker if it is still running.
