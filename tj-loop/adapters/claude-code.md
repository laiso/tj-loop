# Adapter: Claude Code

Status: verified end-to-end on Claude Code 2.1.328 (2026-11-19).

## Arrangement

- Worker: a Claude Code session (background).
- Supervisor: another Claude Code session. The write path (SendMessage) exists only inside a session, so the supervisor cannot be a plain script.
- Terminal multiplexer: not required.

## Mechanics

- Launch the worker: `claude --bg --dangerously-skip-permissions "<task>"`. Note the printed ID and session name.
- Read: the in-session `ListAgents` tool, or `claude agents --json` via Bash (status busy/idle, state working/done). Works without a terminal.
- Write: the in-session `SendMessage` tool. There is no CLI command for sending into a running session, so this adapter runs inside the supervisor session.
- Heartbeat: `/loop <interval>` in the supervisor session, with a prompt that applies the common rules in SKILL.md.
- Delivery is not immediate: an instruction is read when the worker's current tool call returns, so a long call delays delivery by that call's length. After sending an instruction, wait at least one heartbeat interval before judging whether it landed.
- An instruction is a request, not a command: the worker checks it against its own permissions and may decline it (for example, work outside the task's scope). A decline is final. Do not resend or force it; hand it to the human.
- Stop: stop the /loop; stop the worker with `claude stop <id>`.
