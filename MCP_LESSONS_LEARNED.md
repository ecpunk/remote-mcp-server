# MCP Lessons Learned (Remote Connector)

This summarizes practical issues encountered while bringing up an authless remote MCP endpoint for Claude web.

## 1) Handshake Issues Were Mostly Payload Shape, Not Version
- The server accepted multiple protocol versions when initialize payload was complete.
- Missing `clientInfo` in `initialize.params` can trigger `Invalid request parameters`.
- Always test with a fully formed `initialize` object before debugging version support.

## 2) Header Requirements Matter
- MCP streamable HTTP expects:
  - `Content-Type: application/json`
  - `Accept: application/json, text/event-stream`
- Missing/incorrect `Accept` header produced immediate 406-style failures.

## 3) Endpoint Path Compatibility Helps Real Connectors
- Some clients probe `/` while others use `/mcp`.
- Mapping both paths at the proxy reduced connector startup failures.

## 4) Streaming Behavior Can Mislead CLI Tests
- Stream responses can remain open after first event.
- `curl --max-time` may exit with timeout even when initialize succeeded.
- Validate by checking first `event: message` / `data:` payload, not only curl exit code.

## 4a) Browser CORS Is a Real Connector Gate
- Browser-based connectors can fail even when direct server/curl checks pass.
- Ensure CORS allows Claude web origins:
  - `https://claude.ai`
  - `https://app.claude.ai`
- Missing CORS response headers may look like generic auth/connection failures in the UI.

## 5) DNS Can Be Correct Publicly and Wrong Locally
- Public resolvers returned correct answers while local resolver returned empty.
- Troubleshoot with both:
  - `dig @public-resolver host`
  - local resolver (`getent`, local stub query)
- If needed, temporarily force resolution for testing, then fix resolver config.

## 6) Keep Runtime Changes Minimal and Reversible
- Middleware/proxy experiments should be small and easy to roll back.
- Prefer fixing request shape and endpoint routing before deep protocol rewrites.
- Verify with a test matrix after each change, then restart service cleanly.

## 7) Generic Validation Flow
1. Confirm MCP service is active and listening.
2. Test local initialize with full payload.
3. Test public HTTPS initialize through proxy.
4. Test protocol versions (`2025-03-26`, `2024-11-05`, `2024-10-07`).
5. Confirm response includes `serverInfo` and `capabilities`.

## 8) Repo Hygiene Before Publishing
- Remove app/domain-specific details unless needed.
- Never commit real secrets, tokens, or credential files.
- Keep docs generic and reusable; keep local deployment specifics private.

## 9) Wildcard Certificate Ops Gotcha
- Wildcard certs (`*.example.com`) require DNS challenge for issuance and renewal.
- HTTP challenge is not sufficient for wildcard renewals.
- Confirm renewal automation (`certbot.timer`/cron) and periodically run `certbot renew --dry-run`.

## 10) AI Chat Endpoints Benefit From Live Data Context Injection
- For project copilots, generic prompts underperform without current project state.
- Inject a compact snapshot of live records (e.g., room/task/status/owner/priority) into the system prompt.
- Build this snapshot server-side at request time so responses reflect latest state.

## 11) Keep Short Per-Session Conversation Memory Server-Side
- Preserve a bounded window (e.g., last 10 messages) to support follow-up questions.
- Key history by authenticated user/session identifier and trim aggressively.
- This gives continuity without unbounded token growth or long-lived state risks.

## 12) Gate AI Endpoints With User JWT, Not Service Keys
- AI chat endpoints should require user authentication, not only backend service credentials.
- Keep MCP/service API-key access scoped to machine-to-machine routes where needed.
- This separation reduces accidental exposure of expensive model endpoints.

## 13) Keep Provider Secrets Server-Side Only
- Browser clients should never receive provider API keys.
- Frontend calls backend endpoints; backend injects secrets from environment/config.
- Validate logs and error payloads to ensure no secret/token leakage.
