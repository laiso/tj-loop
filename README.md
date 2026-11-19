# tj-loop — Tom and Jerry

![Jerry keeping Tom moving](assets/tom-and-jerry.gif)

A supervisor/worker pair pattern for long-running coding-agent sessions.

- **Tom** is the worker: a coding-agent session running a long task.
- **Jerry** is the supervisor: a second session that wakes up on a heartbeat, reads Tom's state, judges whether Tom is progressing or stalled, and sends a resume instruction to keep Tom going.

## How this differs from the Ralph loop

In the Ralph loop, the outer script repeats the same prompt to a fresh session and makes no judgment; the goal is fixed. Here, the judgment sits **outside the running session**: Jerry watches the *same* session, decides whether it is progressing, stalled, done, or in trouble, and acts on it — resume it, or hand it to the human. Tom keeps his context; Jerry needs no memory, because the state lives in files.

## Minimal requirements

1. A session is running a task on its own.
2. A supervisor runs periodically. The mechanism is free: `/loop` inside Claude Code, or cron outside it.
3. The supervisor can **see** the running session's state and output.
4. The supervisor can **act on** the running session — deliver an instruction it will pick up.

Everything else (goal files, reports, dashboards, scoring gates) is an implementation detail of a particular setup.

## What's here

`SKILL.md` — the **session-supervisor** skill for Claude Code, verified end to end:

1. Start the worker in the background: `claude --bg --dangerously-skip-permissions "<task>"`
2. In the supervisor session, run `/loop <interval>` with a prompt that reads the worker's state (`ListAgents` / `claude agents --json`), reads its progress file, and sends a resume instruction with `SendMessage` when the worker stalls.
3. The supervisor logs every judgment to a file and stops when the worker is done.

## Verified behavior (Claude Code 2.1.328, 2026-11-19)

- A stall at a chunk boundary was detected 26 seconds after the worker went idle (1-minute heartbeat) and the worker resumed on the supervisor's next instruction.
- A message sent during one long 150-second tool call stayed unread across two heartbeats and was read right after the call returned — delivery happens at the worker's next tool round.
- An instruction arrives as a request, not a command: the worker declined an out-of-scope instruction (delete its own files), passed it to the user, and continued its task. The supervisor did not force or resend it.

## Planned

- A **Codex adapter** behind the same interface (read state / send instruction / run periodically): pane capture via tmux or Herdr for reading, key-event input for writing, cron as the heartbeat.

## References

- [codex-whip](https://github.com/agticorp/codex-whip) by agticorp — the closest prior work we found in a survey on 2026-11-19: a manager LLM, woken periodically, reads a Codex session's state via tmux and injects continuations.
- [Ralph loop](https://ghuntley.com/ralph/) by Geoffrey Huntley — the starting point this pattern builds on.

## Status

Initial announcement, 2026-11-19. A longer write-up is in progress.
