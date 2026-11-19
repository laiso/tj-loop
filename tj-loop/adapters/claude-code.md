# Adapter: Claude Code

## Arrangement

- Worker: a Claude Code session (background).
- Supervisor: another Claude Code session. The write path (SendMessage) exists only inside a session, so the supervisor cannot be a plain script.
- Terminal multiplexer: not required.

## Mechanics

- Launch the worker: `claude --bg --dangerously-skip-permissions "<task>"`. Note the printed ID and session name.
- Read: the in-session `ListAgents` tool, or `claude agents --json` via Bash (status busy/idle, state working/done). Works without a terminal.
- Write: the in-session `SendMessage` tool. There is no CLI command for sending into a running session, so this adapter runs inside the supervisor session.
- Heartbeat: `/loop <interval>` in the supervisor session, with a prompt that applies the common rules in SKILL.md.
- Verified behavior (Claude Code 2.1.328, demo on 2026-11-19):
  - An instruction lands in the worker's inbox and is read at the worker's next tool round. During one long tool call it is not delivered until that call returns (a message sent during a 150-second single call was read right after the call ended, about 146 seconds later).
  - An instruction arrives as a request, not a command. The worker judges within its own permissions before complying. In the demo, the worker declined an out-of-scope instruction (delete its own files), passed it to the user, and continued its task. Do not force a declined instruction.
- Stop: stop the /loop; stop the worker with `claude stop <id>`.
