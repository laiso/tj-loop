# Adapter: Pi

- Arrangement: the supervisor spawns the worker itself as `pi --mode rpc` and drives it over stdin/stdout (LF-delimited JSON). RPC is point-to-point: a supervisor can only drive a Pi process it spawned.
- Read: the stdout event stream, plus the `get_state` command (`isStreaming`, `messageCount`, `pendingMessageCount`); the `agent_settled` event means the worker will not continue on its own.
- Write: the `prompt` command starts a run on an idle session; `steer` and `follow_up` deliver input while it is streaming.
- Heartbeat: the supervisor process itself (it owns the child), or cron around it.
- An already-running interactive Pi session is different: reading works (tail its session file under `~/.pi/agent/sessions/`), but there is no external write path — use terminal key input, or a helper extension loaded into the worker in advance that calls `pi.sendUserMessage` on a signal you control.
