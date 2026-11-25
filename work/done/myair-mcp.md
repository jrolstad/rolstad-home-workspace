# ResMed myAir MCP Server

Create a read-only MCP server that exposes the CPAP therapy data shown in ResMed myAir (nightly usage, AHI, mask seal/leak, myAir score, device info). It runs **locally over stdio only**. Because this is health data, there is no hosted or HTTP mode.

ResMed has no public API. The canonical reverse-engineered client is the Home Assistant integration https://github.com/prestomation/resmed_myair_sensors (MIT, actively maintained, last commit 2026-11-21). Its `custom_components/resmed_myair/client/` folder is the reference for auth and queries.

---

## Plan

Create a new Python MCP server (mcp 2.x `MCPServer`) (`jrolstad/myair-mcp`) that follows the `repos/mysa-mcp` pattern: reverse-engineered consumer API, credential login, in-memory token cache, read-only.

### New Repo Structure

```
repos/myair-mcp/
├── server.py            # MCPServer tools + in-process cache (stdio)
├── myair_client.py      # Okta login (authn -> PKCE authorize -> token) + GraphQL calls
├── requirements.txt     # mcp>=2, httpx, python-dotenv, pyjwt
├── requirements-dev.txt # pytest, pytest-asyncio, respx
├── pytest.ini
├── tests/
│   ├── test_myair_client.py
│   └── test_server.py
├── .env.example         # MYAIR_USERNAME, MYAIR_PASSWORD, MYAIR_REGION=NA
├── LICENSE              # MIT, credit prestomation/resmed_myair_sensors
└── README.md
```

### Spike Results (2026-11-24)

Ran the login and both queries against the real account (NA, AirSense 11 AutoSet, 1 mask):

- **No MFA:** `authn` returned `SUCCESS` right away. The authorize step returned 302 with the code in the fragment. The token step returned `access_token` and `id_token`, and the access token lifetime is **3600s**. `myAirCountryId` = `US`.
- **`startMonth`/`endMonth` work at month level:** asking for 30 days back from 2026-11-24 returned everything from **2026-10-01**. The client must filter records down to the exact date window itself.
- **The full history comes back in one call:** records start on 2026-03-03 (when the account began). 365, 730, and 1825-day requests all returned the same 266 records, so there is no paging and no practical cap on `days`.
- **Unused nights show up as records** with `totalUsage: 0` (266 records, 253 used). They are not missing rows.
- **`sleepScore` is the total myAir score** (0–100). It is the sum of `usageScore` (max 70), `ahiScore` (5), `maskScore` (5) and `leakScore` (20). Example: `usage 425 min, ahi 0.0, leakPercentile 13.2, maskPairCount 1, score 100`.
- **Types:** `startDate` is an ISO date string, `ahi` and `leakPercentile` are floats, and the rest are ints. `lastSleepDataReportTime` is an ISO timestamp with an offset.
- **Latest record** was for 2026-11-23, uploaded at 13:55 UTC on 2026-11-24.

### myAir API (from the reference integration)

**Region config (NA):**

| Item | Value |
|------|-------|
| Okta host | `resmed-ext-1.okta.com` |
| Auth server ID | `aus4ccsxvnidQgLmA297` |
| OAuth client ID | `0oa4ccq1v413ypROi297` |
| Redirect URI | `https://myair.resmed.com` |
| GraphQL (AWS AppSync) | `https://graphql.myair-prd.dht.live/graphql` |
| AppSync `x-api-key` | `da2-cenztfjrezhwphdqtwtbpqvzui` (public web-app key) |

EU uses `id.resmed.eu` and `graphql.hyperdrive.resmed.eu` and requires email MFA. Support NA only for v1 and keep the region values in one dict so EU can be added later.

**Auth flow (NA, no MFA):**

1. `POST https://{okta}/api/v1/authn` with `{username, password}`. The response has `status: SUCCESS` and a `sessionToken`. `MFA_REQUIRED` means the server stops with a clear error.
2. `GET /oauth2/{authServer}/v1/authorize` with `client_id`, `response_type=code`, `response_mode=fragment`, `scope=openid profile email`, `redirect_uri`, a PKCE S256 `code_challenge`, `state`, `nonce`, and `sessionToken`. Do not follow the redirect. Read `code` from the `Location` header fragment.
3. `POST /oauth2/{authServer}/v1/token` with `grant_type=authorization_code`, `code`, `code_verifier`, `client_id`, and `redirect_uri`. This returns `access_token` and `id_token`.
4. Decode `id_token` without signature verification to get the `myAirCountryId` claim. It is required for the `rmdcountry` header.

There is no refresh token. When the token expires or a call returns 401, run the full login again.

**GraphQL headers:** `x-api-key`, `Authorization: Bearer <access_token>`, `rmdcountry`, `rmdproduct: myAir`, `rmdhandsetplatform: Web`, `rmdappversion`, `rmdlanguage: en`, `rmdhandsetid`, `rmdhandsetmodel`, `rmdhandsetosversion`. Copy the set in `client/graphql.py`, because AppSync rejects requests that are missing them.

**Queries:**

- `GetPatientSleepRecords`: `getPatientWrapper { patient { firstName } sleepRecords(startMonth: "YYYY-MM-DD", endMonth: "YYYY-MM-DD") { items { startDate totalUsage sleepScore usageScore ahiScore maskScore leakScore ahi maskPairCount leakPercentile sleepRecordPatientId } } }`
- `getPatientWrapper` (device): `masks { maskCode } fgDevices { serialNumber localizedName deviceSeries deviceFamily lastSleepDataReportTime fgDeviceManufacturerName fgDevicePatientId }`

### Tools

| Tool | Description |
|------|-------------|
| `get_devices()` | CPAP device(s) and mask: name, series, serial, last data upload time |
| `get_sleep_records(days: int = 30)` | Nightly records, newest first: date, usage minutes, AHI, leak percentile, mask on/off count, the four sub-scores and the total myAir score |
| `get_latest_sleep()` | Most recent night, served from the cached records with no extra call |
| `reset_cache()` | Clear the in-process cache (forces re-login on the next call) |

All tools are read-only. Leave out summary or aggregation tools, because Claude can compute averages from `get_sleep_records`.

### Key Implementation Details

**`myair_client.py`**
- `MyAirClient` class built from `MYAIR_USERNAME`, `MYAIR_PASSWORD`, and `MYAIR_REGION` env vars, using `httpx.AsyncClient(timeout=15, follow_redirects=False)`.
- `async def _login()` runs authn, then authorize, then token, and stores `access_token`, `id_token`, `country`, and the expiry from the JWT `exp`.
- `async def _query(op, query)` logs in lazily, retries once with a fresh login on 401 or `Unauthorized` GraphQL errors, and raises on `errors[]`.
- `async def get_sleep_records(start: date, end: date)` sends the month range that covers the window, then filters to `start <= startDate <= end` and sorts newest first. `async def get_devices()` normalizes the field names to snake_case dicts.
- Rename `sleepScore` to `myair_score` in the output so it's clear that it is the total score.
- Never log the password, tokens, or the raw response bodies.

**`server.py`**
- Follow the `repos/mysa-mcp/server.py` structure, on the mcp 2.x API: `MCPServer("myair")`, `@mcp.tool(annotations=ToolAnnotations(read_only_hint=True))`, and a dict cache. Call `mcp.run()` with stdio only and no `MCP_TRANSPORT` toggle, so it can't be exposed over HTTP by accident.
- Cache the full-history response (one call covers every `days` value) and slice it per request. Cache TTL is **30 minutes**. Data only changes once a day when the machine uploads after a session, and a long TTL keeps Okta logins rare.
- `get_latest_sleep` reads from the `get_sleep_records` cache.

**Tests:** mock HTTP with `respx`. Cover the login happy path, `MFA_REQUIRED`, re-login after a 401, the GraphQL `errors[]` response, empty sleep records, and cache hit/miss. Same structure as `repos/mysa-mcp/tests/`.

### Auth and Privacy

- `MYAIR_USERNAME` and `MYAIR_PASSWORD` come from env vars only. Commit `.env.example` and never `.env`.
- Local stdio only: no Dockerfile, no HTTP transport, no Claude.ai connector. Credentials live in the `env` block of the `~/.claude.json` entry.
- Responses are returned to the local Claude Code session only. Nothing is written to disk.
- Optional `MYAIR_DEVICE_TOKEN` (Okta `DT` cookie) is only needed if NA ever turns on MFA. Leave it out of v1 unless step 1 below shows MFA.

### Implementation Sequence

1. ~~**Spike (go/no-go)**~~: done 2026-11-24. Go. See Spike Results.
2. Create the `jrolstad/myair-mcp` GitHub repo.
3. Implement `myair_client.py` (login + 2 queries).
4. Implement `server.py` (4 tools + cache, stdio only) using `repos/mysa-mcp/server.py` as the template.
5. Add tests, `requirements*.txt`, `.env.example`, README (with the "unofficial API" disclaimer and credit), and LICENSE.
6. Run `pytest`, then wire up locally over stdio in `~/.claude.json` and call each tool against real data.
7. Add `repos/myair-mcp` as a submodule in the workspace.
8. Create the `/sleep-check` skill in `repos/rolstad-home-skills/.claude/skills/sleep-check/` (see below), add the junction in `.claude/skills/`, and add it to the skill list in the workspace `CLAUDE.md`.
9. Move this file to `work/done/`.

### `/sleep-check` Skill (after the server is done)

- Lives in `repos/rolstad-home-skills/.claude/skills/sleep-check/SKILL.md` and follows the format of the existing skills there (for example `home-weather`).
- Calls `mcp__myair__get_devices` and `mcp__myair__get_sleep_records` (default 7 nights; accepts an argument for N nights).
- Output: last night's snapshot (usage, AHI, leak, mask on/off, myAir score), then a table of the last N nights, then averages and a one-line trend. Flag nights under 4 hours or with AHI over 5. If last night hasn't uploaded yet, say so based on `last_sleep_data_report_time`.
- Not added to `/morning-report`: that report can be shared and health data shouldn't show up there by default.
- Gives no medical interpretation beyond the plain thresholds.

### Pitfalls

- **Unofficial API:** endpoints, client IDs, and the AppSync key can change without notice. Keep them as constants in one place, and watch the reference repo for breaking-change commits.
- **The authorize step must not follow redirects:** the code is in the `Location` fragment (`#code=...`), not in a query string or the response body.
- **`rmdcountry` is required:** it comes from the `myAirCountryId` claim in `id_token`. Missing it causes opaque AppSync errors.
- **No refresh token:** every expiry means a full username/password login. Repeated failed logins can lock the Okta account, so after an auth failure don't retry in a loop. Fail once and surface the error.
- **Date semantics:** `startDate` is the therapy-night date (the evening the session started). `totalUsage` is in minutes. Unused nights come back as records with `totalUsage: 0`.
- **Upload lag:** the machine uploads over cellular after a session ends, so last night may not be present early in the morning. `get_devices().last_sleep_data_report_time` shows this.
- **Medical disclaimer:** the README should state that the data is informational and not for medical decisions.

### Critical Reference Files

- `repos/mysa-mcp/server.py` and `repos/mysa-mcp/mysa_client.py`: template for structure, cache, and token handling
- `repos/mysa-mcp/tests/`: test layout
- `resmed_myair_sensors/custom_components/resmed_myair/client/auth.py`: Okta authn, PKCE, and MFA flow
- `resmed_myair_sensors/custom_components/resmed_myair/client/graphql.py`: required AppSync headers
- `resmed_myair_sensors/custom_components/resmed_myair/client/rest_client.py`: GraphQL query text
- `resmed_myair_sensors/custom_components/resmed_myair/client/regions.py`: NA and EU endpoint constants
