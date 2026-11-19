# Adapter: Antigravity (agy)

Status: Arrangement A verified in part on Antigravity CLI 1.5.12 (2026-11-19): initialize, session/new, session/prompt, and the streamed session/update notifications worked end to end for a single task; session list/load were not confirmed. Arrangement B is the form running in the WSH 2026 mock campaign (Neon pass, from 2026-11-19).

## Arrangement

### A: ACP (the supervisor hosts the worker)

- Worker: an Antigravity session inside an `agy acp` process the supervisor itself spawns. Headless: no terminal, no screen.
- Supervisor: any agent or program that can spawn a process and speak ACP (JSON-RPC over stdio) to it.
- Terminal multiplexer: not required.
- Sign-in: the `agy acp` process must run where the user's Antigravity sign-in is reachable — inside the user's desktop session. From a bare SSH shell, session/new fails with "Authentication required"; run `agy acp --login` once in that context first, or start the supervisor inside the desktop session.

### B: an already-running interactive session

- Worker: an interactive `agy` session started by the human in a terminal pane (a multiplexer such as tmux, Herdr, or Orca).
- Supervisor: any agent that can read the pane and send text to it.
- Terminal multiplexer: required (e.g. tmux, Herdr, Orca) — the pane is the read/write channel. ACP cannot attach to an interactive session, and a session started by hand cannot be taken over by arrangement A.

## Mechanics

### A: ACP

- Spawn `agy acp`. Add `--dangerously-skip-permissions` only when no permission prompts should reach the supervisor. The supervisor sends LF-delimited JSON-RPC on the process's stdin and reads messages on its stdout.
- Connect: `initialize` returns the server's capabilities (session list, load, resume, close). `session/new`, given the working directory, returns the session id and the models available to the account. The session id is the binding between supervisor and worker.
- Write: `session/prompt` sends an instruction (text; images and file references can be attached) and returns when the turn ends. `session/cancel` interrupts a running turn.
- Read: `session/update` notifications stream the worker's reply text, each tool call and its result, token usage, and the session title. A `session/request_permission` request means the worker is waiting for the supervisor's answer; an unanswered request shows up to the heartbeat as no progress.
- Heartbeat: the supervisor's own loop or an outside scheduler. Either keep the `agy acp` process alive between wake-ups, or start a new one per wake-up and re-attach with `session/load` (re-attach not yet verified).
- Stop: stop the heartbeat, then `session/close` or end the `agy acp` process; the worker session ends with it.

### B: interactive session

- Read: read the pane's visible output (e.g. `herdr pane read <pane>`).
- Write: send text to the pane, then an Enter key (e.g. `herdr pane send-text` + `herdr pane send-keys enter`). Before sending, read the pane and confirm the input field is empty — leftover text mixes with what you send.
- Stop: stop the heartbeat. Leave the worker session to the human, or stop it in the pane.
