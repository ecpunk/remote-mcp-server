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
