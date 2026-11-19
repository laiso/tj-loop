# Adapter: Pi

Status: not yet run end-to-end; written from the official Pi documentation (RPC mode).

## Arrangement

### A: RPC (the supervisor hosts the worker)

- Worker: a Pi session in a process the supervisor itself spawns in RPC mode (`pi --mode rpc`). RPC is point-to-point: a supervisor can only drive a Pi process it spawned.
- Supervisor: the process that hosts the worker, or an agent driving that host on a schedule.
- Terminal multiplexer: not required.

### B: an already-running interactive session

- Worker: an interactive Pi session started by the human.
- Supervisor: any agent that can read files and, for writing, drive a terminal or a helper extension (see Mechanics B).
- Terminal multiplexer: required for the key-input route; not required with the helper extension.

## Mechanics

### A: RPC

- The worker reads LF-delimited JSON commands on stdin and writes events on stdout. A command names its type: `{"type":"prompt","message":"<instruction>"}` starts a run on an idle session; `{"type":"get_state"}` answers with the session state.
- Read: the stdout event stream, plus `get_state` (`isStreaming`, `messageCount`, `pendingMessageCount`); the `agent_settled` event means the worker will not continue on its own.
- Write: `prompt` starts a run on an idle session; `steer` and `follow_up` deliver input while it is streaming.
- Heartbeat: the supervisor process itself (it owns the child), or cron around it.
- Stop: stop the heartbeat, then end the child process; the worker session ends with it.

### B: interactive session

- Read: tail the session's file under `~/.pi/agent/sessions/`.
- Write: there is no external write path into a running interactive session. Use terminal key input, or a helper extension loaded into the worker in advance that calls `pi.sendUserMessage` on a signal you control.
