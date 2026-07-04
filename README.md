# Remote MCP Server — Template & Field Notes (Authless)

A field-tested template for exposing a **Model Context Protocol (MCP)** server over HTTPS so a hosted AI client (e.g. Claude web) can reach it as a **remote custom connector** — plus the debugging notes from actually getting one to connect, which is where most of the time goes.

This is not a framework. It's the reference architecture, the exact handshake / proxy / CORS details that make a remote connector work, and a catalog of failure modes that present as one problem but are really another.

## What's here
- **[MCP_REMOTE_CONNECTOR_TEMPLATE.md](MCP_REMOTE_CONNECTOR_TEMPLATE.md)** — the reference architecture: an MCP process (streamable HTTP) behind a TLS-terminating reverse proxy, the required `initialize` handshake shape, a working Nginx config (both `/` and `/mcp` mapped), and a step-by-step validation flow.
- **[MCP_LESSONS_LEARNED.md](MCP_LESSONS_LEARNED.md)** — 11 concrete gotchas from bringing one up live, including AI-chat-endpoint notes (live-data context injection, bounded server-side session memory).

## Who it's for
Anyone standing up a remote MCP endpoint that a hosted AI client has to reach across a network boundary — where the hard parts are the handshake payload shape, transport headers, CORS, TLS, and DNS, not the tool code itself.

## The lessons that save the most time
- **Handshake failures are usually payload shape, not protocol version.** A complete `initialize` (including `clientInfo`) succeeds across versions; a missing field reads as "invalid request parameters."
- **Transport headers are load-bearing.** `Accept: application/json, text/event-stream` is required — the wrong `Accept` is an immediate 406.
- **Browser CORS is a real connector gate.** A connector can fail in the UI while direct curl/server checks pass; allow the Claude web origins (`https://claude.ai`, `https://app.claude.ai`) explicitly.
- **Map both `/` and `/mcp` at the proxy** — different clients probe different paths.
- **Streaming misleads CLI tests** — `curl --max-time` can report a timeout on a *successful* initialize; validate on the first `event: message` / `data:` payload, not the curl exit code.
- **DNS can be right publicly and wrong locally** — check both a public resolver and the local stub before assuming a server fault.

## Security posture
- Keep the transport **authless only when appropriate**; require OAuth when the connector or the exposed tools warrant it.
- **Never commit secrets** — handle keys server-side, keep credential files and deployment-specific details out of the repo.
- Treat the MCP endpoint as an exposed tool surface: scope tools deliberately and keep runtime changes small and reversible.

## Using it
1. Run your MCP server (streamable HTTP) on a local port.
2. Put it behind a TLS-terminating reverse proxy; map `/` and `/mcp` (see the Nginx template).
3. Add the Claude web origins to CORS.
4. Validate a full `initialize` locally, then through the public URL, confirming the response carries `serverInfo` + `capabilities` (see the validation flow).

---
*Sanitized, reusable template — deployment-specific details intentionally omitted.*
