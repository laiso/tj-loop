---
name: session-supervisor
description: Run a long task as a pair of Claude Code sessions — a worker and a supervisor. The worker runs in the background; this session checks its state on a /loop, judges progress, and sends a resume instruction when it stalls. Use when a long autonomous run needs to be kept going.
---

# Session Supervisor

## Roles

- Worker session: runs the task. Started in the background.
- Supervisor session (this session): checks the worker's state periodically with /loop, judges progress, and intervenes only when needed.

Unlike a loop that repeats the same instruction without judging, here the supervisor reads progress and decides: let it continue, resume it, or hand it to the human.

## Procedure

### 1. Start the worker

```bash
claude --bg --dangerously-skip-permissions "<task>"
```

- Note the printed ID and session name.
- Include in the task instruction: append progress to a named file (e.g. progress.md). State lives in files; do not rely on the supervisor's memory.
- Decide where the supervisor log (e.g. supervisor-log.md) goes.

### 2. Start the supervisor loop

In this session:

```
/loop 5m <supervisor prompt>
```

The supervisor prompt must include:

- Read the worker's status and state with ListAgents (or `claude agents --json` via Bash).
- Read the progress file's last update and tail.
- Judge:
  - Progressing (busy / working, and the progress file is moving): do nothing.
  - Stalled (idle with no progress change, or blocked): send a resume instruction with SendMessage, attaching the last state from the progress file.
  - Done (state is done): summarize the result for the human and stop the loop.
  - Needs judgment (work outside the worker's permissions, the worker declined an instruction, the same stall repeats): stop intervening and hand it to the human.
- Append one line per judgment or intervention to the supervisor log.

### 3. Intervention limits

- Never resend for the same stall. One resume instruction per stall.
- If a resume does not move it, hand it to the human.

## Verified behavior (experiments on Claude Code 2.1.328, 2026-11-19)

- State reading works via `claude agents --json` and the in-session ListAgents tool.
- Instructions are sent with the in-session SendMessage tool. A shell script cannot send one (there is no CLI command for sending a message to a running session), so this skill assumes it runs inside the supervisor session.
- An instruction lands in the worker's inbox and is read at the worker's next tool round. During one long tool call it is not delivered until that call returns (verified in a demo on 2026-11-19: a message sent during a 150-second single call was read right after the call ended, about 146 seconds later).
- An instruction arrives as a request, not a command. The worker judges within its own permissions before complying. If it declines, do not force it; hand it to the human (verified in the same demo: the worker declined an out-of-scope instruction to delete its files, passed it to the user, and continued its task).

## Stopping

- The supervisor loop ends when the worker completes.
- To stop early, stop the /loop. To stop the worker: `claude stop <id>`.
