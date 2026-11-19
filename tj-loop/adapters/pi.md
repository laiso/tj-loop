# Adapter: Pi

## Arrangement

- Worker: a Pi process the supervisor itself spawns in RPC mode (`pi --mode rpc`), driven over stdin/stdout (LF-delimited JSON). RPC is point-to-point: a supervisor can only drive a Pi process it spawned.
- Supervisor: the process that hosts the worker, or an agent driving that host on a schedule.
- Terminal multiplexer: not required. (For an already-running interactive session, key input through a multiplexer is the fallback — see the last bullet.)

## Mechanics
- Read: the stdout event stream, plus the `get_state` command (`isStreaming`, `messageCount`, `pendingMessageCount`); the `agent_settled` event means the worker will not continue on its own.
- Write: the `prompt` command starts a run on an idle session; `steer` and `follow_up` deliver input while it is streaming.
- Heartbeat: the supervisor process itself (it owns the child), or cron around it.
- An already-running interactive Pi session is different: reading works (tail its session file under `~/.pi/agent/sessions/`), but there is no external write path — use terminal key input, or a helper extension loaded into the worker in advance that calls `pi.sendUserMessage` on a signal you control.
