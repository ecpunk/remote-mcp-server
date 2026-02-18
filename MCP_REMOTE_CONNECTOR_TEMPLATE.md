# MCP Remote Connector Template (Authless)

Use this as a reusable template for exposing an MCP server to Claude web as a remote custom connector.

## 1) Goal
- Expose a stable MCP endpoint over HTTPS.
- Keep transport authless unless your connector explicitly requires OAuth.
- Ensure `initialize` handshake succeeds and returns `serverInfo` + `capabilities`.

## 2) Reference Architecture
- MCP server process (streamable HTTP) on local port (example: `8091`)
- Reverse proxy (Nginx/Caddy/Traefik) terminating TLS
- Public URL (example: `https://mcp.example.com`)

## 3) Required MCP Behavior
- Endpoint accepts `POST` for initialize on your MCP route (commonly `/mcp`).
- Client request should include:
  - `Content-Type: application/json`
  - `Accept: application/json, text/event-stream`
  - JSON-RPC `initialize` with:
    - `protocolVersion`
    - `capabilities`
    - `clientInfo` (recommended for compatibility)

Minimal working initialize payload:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {},
    "clientInfo": {
      "name": "connector-test",
      "version": "1.0"
    }
  }
}
```

Expected first response event includes:
- `result.protocolVersion`
- `result.capabilities`
- `result.serverInfo`

## 4) Reverse Proxy Template (Nginx)

```nginx
server {
    server_name mcp.example.com;

    location = / {
        proxy_pass http://127.0.0.1:8091/mcp;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }

    location /mcp {
        proxy_pass http://127.0.0.1:8091/mcp;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
}
```

Notes:
- Mapping both `/` and `/mcp` helps with connector clients that probe root first.
- Keep streaming/buffering settings compatible with SSE/streamable responses.

## 4a) Browser CORS Requirements (Claude Web)
- If your MCP endpoint is called from a browser-based client, CORS must explicitly allow Claude origins.
- At minimum, allow:
  - `https://claude.ai`
  - `https://app.claude.ai`
- Missing CORS headers can make a connector fail in-browser even when `curl` tests pass.
- Also allow methods/headers used by MCP requests (`POST`, `OPTIONS`, `Content-Type`, `Accept`).

Example allow-list (app/server config):

```text
ALLOW_ORIGINS=https://claude.ai,https://app.claude.ai
```

## 5) Test Matrix (Copy/Paste)

Local test:

```bash
curl -i -X POST http://127.0.0.1:8091/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"curl","version":"0.0"}}}'
```

Public HTTPS test:

```bash
curl -i -X POST https://mcp.example.com/ \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"curl","version":"0.0"}}}'
```

Version compatibility tests:
- `2025-03-26`
- `2024-11-05`
- `2024-10-07`

## 6) Common Failure Patterns
- `406 Not Acceptable`: missing `Accept: application/json, text/event-stream`
- `Invalid request parameters`: often missing `clientInfo` or malformed `params`
- `404` on root: connector may be calling `/` while server only serves `/mcp`
- Curl timeout with exit code `28`: response stream kept open; inspect first event line
- DNS resolves publicly but not locally: local resolver cache/upstream mismatch

## 7) Production Hardening Checklist
- [ ] TLS enabled with valid cert chain
- [ ] Service managed by systemd (auto-restart)
- [ ] Proxy buffering disabled for stream endpoint
- [ ] Secrets excluded from git (`.secrets/`, API keys, tokens)
- [ ] Health/logging in place for quick connector triage

## 8) Wildcard SSL + Renewal Note
- If you use a wildcard certificate (`*.example.com`), do not use HTTP challenge for issuance/renewal.
- Wildcard certs require DNS challenge.
- Example pattern:

```bash
certbot certonly --authenticator dns-<provider> ... -d example.com -d "*.example.com"
```

- Ensure auto-renewal is configured (`certbot.timer` via systemd or a cron job) and test with dry-run.
