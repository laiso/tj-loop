# Adapter: OpenCode

Status: not yet run end-to-end; written from the official OpenCode documentation (server API).

## Arrangement

- Worker: an OpenCode session served by `opencode serve`. A session a human drives in the TUI is the same session, because the TUI is a client of the server.
- Supervisor: any agent that can make HTTP calls on a schedule.
- Terminal multiplexer: not required.

## Mechanics

- Run (or reuse) `opencode serve` (default port 4096).
- The API differs between OpenCode 1.x and 2.x. Check the version of the binary serving the worker (`opencode --version` on that host) before choosing endpoints.
- Read: `GET /api/session` (list), `GET /api/session/active`, `GET /api/session/{sessionID}/message` (timeline); or the SDK (`@opencode/client`: `client.session.*`, `client.event.subscribe()`). On 1.x: `GET /session/status` returns busy/idle per session.
- Write: `POST /api/session/{sessionID}/prompt` (2.x) — a prompt sent to an idle session restarts its agent loop, which is the resume action. On 1.x: `POST /session/:id/prompt_async`.
- If the server requires authentication, send the credentials it was started with on every request. A rejected request (401) means the state cannot be read; apply the common rule for that case.
- Heartbeat: cron; the supervisor is just an HTTP client. (OpenCode 2 also has an internal /goal and /schedule; neither is required here.)
- Stop: stop the heartbeat. Leave the worker session as is; stop the server only if the supervisor started it.
