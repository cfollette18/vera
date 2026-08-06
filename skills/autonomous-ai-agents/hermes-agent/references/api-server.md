# Hermes API Server (OpenAI-Compatible HTTP Gateway)

Exposes Hermes Agent as an OpenAI-compatible HTTP API with `/v1/chat/completions`,
`/v1/responses`, `/v1/models`, and more. Used by Open WebUI, LobeChat, and any
OpenAI-compatible frontend.

**Source:** `gateway/platforms/api_server.py`

## Quick Setup

### 1. Append to `~/.hermes/.env`

```env
API_SERVER_ENABLED=true
API_SERVER_KEY=<secret-key>   # min 8 chars; generate with: python3 -c "import secrets; print(secrets.token_hex(16))"
```

### 2. Enable in `~/.hermes/config.yaml`

```yaml
platforms:
  api_server:
    enabled: true
```

Note: `platforms` must be a nested key under the top-level config (not a top-level key).
The env var `API_SERVER_ENABLED=true` also auto-enables it without the yaml entry.

### 3. Start the gateway

```bash
hermes gateway run
```

### 4. Verify

```bash
curl -s -H "Authorization: Bearer <your-key>" http://127.0.0.1:8642/health
# → {"status": "ok", "platform": "hermes-agent"}

curl -s -H "Authorization: Bearer <your-key>" http://127.0.0.1:8642/v1/models
# → {"object":"list","data":[{"id":"hermes-agent",...}]}
```

## Connection Details

| Item | Value |
|------|-------|
| Base URL | `http://127.0.0.1:8642/v1` |
| Default port | `8642` |
| Auth | Bearer token — header: `Authorization: Bearer <API_SERVER_KEY>` |
| Key env var | `API_SERVER_KEY` |
| Enable env var | `API_SERVER_ENABLED=true` |

## Config Options (env vars)

| Env var | Default | Description |
|---------|---------|-------------|
| `API_SERVER_ENABLED` | `false` | Set to `true` to enable |
| `API_SERVER_KEY` | `""` | Secret Bearer token (required for non-localhost bind) |
| `API_SERVER_PORT` | `8642` | Port to listen on |
| `API_SERVER_HOST` | `127.0.0.1` | Bind address. Use `0.0.0.0` for LAN access but ONLY with a real `API_SERVER_KEY` |
| `API_SERVER_CORS_ORIGINS` | `""` | Comma-separated allowed origins, e.g. `https://my-app.com` |
| `API_SERVER_MODEL_NAME` | `"hermes-agent"` | Model name reported to clients |

## Security Behavior

- **Localhost (127.0.0.1):** API key optional — gateway allows all requests without auth
- **Non-localhost bind (0.0.0.0):** Gateway **refuses to start** without a real `API_SERVER_KEY` (min 8 chars)
- **Placeholder keys rejected:** A key shorter than 8 chars or clearly placeholder (e.g. `"your-key-here"`) also causes startup refusal

For LAN/external exposure: run behind a reverse proxy (nginx, Caddy) with TLS termination, or use an SSH tunnel.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/chat/completions` | OpenAI Chat Completions format |
| `POST` | `/v1/responses` | OpenAI Responses API format (stateful) |
| `GET` | `/v1/responses/{id}` | Retrieve stored response |
| `DELETE` | `/v1/responses/{id}` | Delete stored response |
| `GET` | `/v1/models` | List available models |
| `GET` | `/v1/capabilities` | API capabilities for external UIs |
| `POST` | `/v1/runs` | Start agent run (returns run_id immediately, 202) |
| `GET` | `/v1/runs/{run_id}` | Get run status |
| `GET` | `/v1/runs/{run_id}/events` | SSE stream of run lifecycle events |
| `POST` | `/v1/runs/{run_id}/stop` | Interrupt a running agent |
| `GET` | `/health` | Health check |
| `GET` | `/health/detailed` | Rich status for dashboard probing |

## Session Continuity

- Chat Completions is stateless by default. Opt-in continuity via header:
  `X-Hermes-Session-Id: <session-id>`
- Responses API is stateful via `previous_response_id`

## Troubleshooting

**`curl` returns "Invalid API key":**
- Verify the Bearer token matches exactly what's in `API_SERVER_KEY` in `.env`
- Check for trailing whitespace in `.env`

**Gateway not responding:**
- Confirm gateway is running: `hermes gateway status`
- Check `~/.hermes/logs/gateway.log` for errors

**Connection refused on LAN:**
- Ensure `API_SERVER_HOST=0.0.0.0` is set AND `API_SERVER_KEY` is a real key (≥8 chars)
- Check firewall allows port 8642

**Config nesting issue:**
`platforms` must be indented under the top-level yaml structure. Correct:
```yaml
platforms:
  api_server:
    enabled: true
```
Incorrect (common mistake):
```yaml
platforms:
api_server:    # ← wrong: no indentation
  enabled: true
```
