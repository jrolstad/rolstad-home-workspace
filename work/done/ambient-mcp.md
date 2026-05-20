# Ambient Weather MCP Server

Create an MCP server that can run locally or be hosted as a Lambda function, Cloudflare Worker, or other lightweight hosting tool.
This MCP server needs to wrap the Ambient Weather API.  The api is documented at https://ambientweather.docs.apiary.io/#

A golang SDK for this API is in the ambientweather-api-helper repository.
A popular python SDK that wraps the API is at https://github.com/bachya/aioambient
An existing MCP server is at https://github.com/NanaGyamfiPrempeh30/ambient-weather-mcp

---

## Plan

Create a new Python/FastMCP server (`jrolstad/ambient-mcp`) wrapping the Ambient Weather REST API. Matches the pattern of becu-mcp, usps-mcp, heritagebanknw-mcp.

### New Repo Structure

Create `jrolstad/ambient-mcp` on GitHub, add as submodule at `repos/ambient-mcp`:

```
repos/ambient-mcp/
├── server.py           # FastMCP tool definitions + in-process cache
├── ambient_client.py   # Ambient Weather REST API calls
├── requirements.txt    # mcp>=1.3.0, httpx, python-dotenv
├── Dockerfile          # Container image (python:3.11-slim)
├── .env.example        # AMBIENT_APP_KEY, AMBIENT_API_KEY
└── README.md
```

### Ambient Weather API

Base URL: `https://rt.ambientweather.net/v1`

- `GET /devices?applicationKey={app}&apiKey={api}` — list all devices + latest observation in `lastData`
- `GET /devices/{mac}?applicationKey={app}&apiKey={api}&limit={n}&endDate={epoch_ms}` — historical observations (max 288/call)

**Rate limit**: 1 req/second per apiKey — caching is essential.
**Auth**: `AMBIENT_APP_KEY` + `AMBIENT_API_KEY` from https://ambientweather.net/account

### Tools

| Tool | Description |
|------|-------------|
| `get_devices()` | List all weather stations with metadata (name, location, MAC) |
| `get_latest_observation(mac: str)` | Most recent reading — pulled from `lastData` in devices response, no extra API call |
| `get_historical_data(mac: str, hours: int = 24, limit: int = 288)` | Recent observations up to `limit` records |
| `reset_cache()` | Clear in-process cache |

### Key Implementation Details

**`ambient_client.py`**
- `AmbientClient` class with `_app_key` and `_api_key` from env vars
- `async def list_devices()` → `GET /devices`
- `async def get_device_data(mac, limit=288, end_date_ms=None)` → `GET /devices/{mac}`
- Use `httpx.AsyncClient`, timeout=15s

**`server.py`**
- Mirror `becu-mcp/server.py` exactly: `FastMCP("ambient")`, `@mcp.tool()` decorators, in-process dict cache
- Cache TTL: **10 minutes** (stations report every 5 min; 10 min balances freshness vs. rate limits)
- `get_latest_observation` reads from `get_devices()` cache — no extra API call
- `get_historical_data` passes `endDate=int(time.time() * 1000)` (epoch milliseconds, not seconds)
- HTTP transport toggle via `MCP_TRANSPORT` env var (stdio locally, streamable-http when hosted)

**`Dockerfile`**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py ambient_client.py ./
ENV MCP_TRANSPORT=streamable-http
ENV PORT=8000
EXPOSE 8000
CMD ["python", "server.py"]
```

### Hosting Options (identical Dockerfile for all)

**Option A: Azure Container Apps (~$0/month)**
- Scales to zero; free tier easily covers low-frequency weather queries
- `az containerapp env create` → `az containerapp create --image <acr>.azurecr.io/ambient-mcp`
- Set env vars: `AMBIENT_APP_KEY`, `AMBIENT_API_KEY`, `MCP_AUTH_TOKEN`, `PORT=8000`

**Option B: AWS App Runner (~$5–11/month)**
- No cold starts, always-on; push to ECR → App Runner → set env vars

**Option C: Railway (~$0 free / ~$5/month paid)**
- Simplest setup; GitHub-connected deploy

### Auth
1. `AMBIENT_APP_KEY` + `AMBIENT_API_KEY` → query params on every API call
2. `MCP_AUTH_TOKEN` → set on hosting platform; Claude.ai sends as `Authorization: Bearer` natively

### Integration Updates After Deployment

**`~/.claude.json`** — add new entry:
```json
"ambient": {
  "type": "http",
  "url": "https://<hosted-url>/mcp",
  "headers": { "Authorization": "Bearer <MCP_AUTH_TOKEN>" }
}
```

**Claude.ai Routines** — add remote MCP connector:
- Name: `ambient`, URL: `https://<hosted-url>/mcp`, Auth: Bearer `<MCP_AUTH_TOKEN>`

### Implementation Sequence

1. Create `jrolstad/ambient-mcp` GitHub repo
2. Implement `ambient_client.py` (2 methods: `list_devices`, `get_device_data`)
3. Implement `server.py` (4 tools + cache + transport toggle) — use `repos/becu-mcp/server.py` as template
4. Add `Dockerfile`, `requirements.txt`, `.env.example`
5. Test locally in stdio mode: wire up as local MCP, call `get_devices()` and verify real station data returns
6. Choose hosting option and deploy; verify HTTPS endpoint
7. Update `~/.claude.json`; add connector in Claude.ai
8. Add `repos/ambient-mcp` as submodule in workspace
9. Move this todo to `work/done/`

### Pitfalls

- **Rate limit**: 1 req/second per apiKey — cache TTL of 10 min protects this
- **`lastData` shortcut**: `GET /devices` already returns latest observation — use it, don't make a second call
- **Historical data ordering**: `/devices/{mac}` returns records newest-first
- **`endDate` is epoch milliseconds**: not seconds — pass `int(time.time() * 1000)`
- **FastMCP version**: Pin `mcp>=1.3.0` for Streamable HTTP transport

### Critical Reference Files

- `repos/becu-mcp/server.py` — canonical template for server.py + cache pattern
- `repos/becu-mcp/becu_client.py` — canonical template for client module structure
- `repos/ambientweather-api-helper/pkg/ambient/ambient.go` — reference for all data fields the API returns
