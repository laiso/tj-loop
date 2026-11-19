# Adapter: OpenCode

OpenCode is client/server: the TUI is a client of the server, so a session a human is driving in the TUI is reachable through the API.

- Run (or reuse) `opencode serve` (default port 4096).
- Read: `GET /api/session` (list), `GET /api/session/active`, `GET /api/session/{sessionID}/message` (timeline); or the SDK (`@opencode/client`: `client.session.*`, `client.event.subscribe()`). On 1.x: `GET /session/status` returns busy/idle per session.
- Write: `POST /api/session/{sessionID}/prompt` (2.x) — a prompt sent to an idle session restarts its agent loop, which is the resume action. On 1.x: `POST /session/:id/prompt_async`.
- Heartbeat: cron; the supervisor is just an HTTP client. (OpenCode 2 also has an internal /goal and /schedule; neither is required here.)
