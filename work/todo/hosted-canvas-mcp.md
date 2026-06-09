# Host Canvas MCP

I need to host a Canvas MCP server so I can use it in Claude Routines.
The local one I am using is https://github.com/mtgibbs/canvas-lms-mcp .

I need to create a way to host it or make available as a lightweight mcp server that can be used by Claude Routines.

---

## Plan

Create a new Python/FastMCP server (`jrolstad/canvas-mcp`) that wraps the Canvas LMS REST API and deploy it to Railway as a persistent HTTPS service. Matches the pattern of becu-mcp, usps-mcp, heritagebanknw-mcp.

### New Repo Structure

Create `jrolstad/canvas-mcp` on GitHub, add as submodule at `repos/canvas-mcp`:

```
repos/canvas-mcp/
├── server.py           # FastMCP tool definitions + in-process cache
├── canvas_client.py    # Canvas REST API calls + pagination
├── requirements.txt    # mcp>=1.3.0, httpx, python-dotenv
├── Dockerfile          # Railway container image (python:3.11-slim)
├── .env.example        # CANVAS_API_TOKEN, CANVAS_BASE_URL, MCP_AUTH_TOKEN
└── README.md
```

### Phase 1 Tools (minimum for canvas-inquisitor)

| Tool | Canvas Endpoints |
|------|-----------------|
| `get_students()` | `GET /api/v1/users/self/observees` |
| `get_all_students_status(days_grades=14, low_grade_threshold=70)` | observees + courses + submissions + announcements |
| `get_recent_grades(student_id, days=14)` | courses + submissions per course |
| `get_courses(student_id)` | `GET /api/v1/courses?enrollment_type=observer&observed_user_id={id}&include[]=total_scores` |
| `list_assignments(course_id, student_id)` | `GET /api/v1/courses/{id}/assignments?student_ids[]={sid}&include[]=submission` |
| `get_missing_assignments(student_id)` | `GET /api/v1/users/{sid}/missing_submissions` |
| `reset_cache()` | (clears in-process cache) |

### Phase 2 Tools (full parity with mtgibbs server)

`get_announcements`, `get_calendar_events`, `get_comprehensive_status`, `get_discussions`, `get_due_this_week`, `get_feedback`, `get_inbox`, `get_people`, `get_stats`, `get_teacher_communications`, `get_todo`, `get_unsubmitted_past_due`, `get_upcoming_assignments`

### Key Implementation Details

**`canvas_client.py`**
- Single `CanvasClient` class with `base_url` and `Authorization: Bearer` header
- `async def get(path, params)` that follows Canvas pagination (`Link: <url>; rel="next"` headers) — Canvas defaults to 10 results/page, always set `per_page=50`
- Observer-scoped calls require `observed_user_id={student_id}` param
- `get_all_students_status` uses `asyncio.gather()` for per-student parallel fetches

**`server.py`**
- Mirror `becu-mcp/server.py`: `FastMCP("canvas")`, `@mcp.tool()` decorators, in-process dict cache
- Cache TTLs: students=1h, courses/grades=30min, assignments=30min, missing=15min
- HTTP transport toggle via `MCP_TRANSPORT` env var (stdio locally, streamable-http on hosted)

**`Dockerfile`**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py canvas_client.py ./
ENV MCP_TRANSPORT=streamable-http
ENV PORT=8000
EXPOSE 8000
CMD ["python", "server.py"]
```

### Auth
1. `CANVAS_API_TOKEN` env var → `Authorization: Bearer` on all Canvas calls
2. `MCP_AUTH_TOKEN` env var → set on hosting platform; Claude.ai MCP connector sends it as `Authorization: Bearer` natively

### Hosting Options (choose one — Dockerfile is identical for all)

**Option A: Azure Container Apps (~$0/month)**
- Scales to zero; estimated cost at 2x/day = **$0/month** (~0.5% of free tier)
- Cold start ~5–15s (acceptable for daily routine)
- Setup:
  1. `az containerapp env create` → creates environment
  2. `az containerapp create --image <acr>.azurecr.io/canvas-mcp`
  3. Set env vars: `CANVAS_API_TOKEN`, `CANVAS_BASE_URL`, `MCP_AUTH_TOKEN`, `PORT=8000`
  4. Azure assigns `https://canvas-mcp.<region>.azurecontainerapps.io`

**Option B: AWS App Runner (~$5–11/month)**
- Does not scale to zero — always-on baseline cost
- No cold starts
- Setup: push to ECR → App Runner → Create service → set env vars

**Option C: Railway (~$0 free tier / ~$5/month paid)**
- Simplest setup (GitHub-connected), free tier may sleep after inactivity
- Setup: Railway → New Project → Deploy from GitHub → set env vars

### Integration Updates After Deployment

**`~/.claude.json`** — replace `canvas-mtgibbs` entry:
```json
"canvas": {
  "type": "http",
  "url": "https://<hosted-url>/mcp",
  "headers": { "Authorization": "Bearer <MCP_AUTH_TOKEN>" }
}
```

**`repos/rolstad-home-skills/.claude/skills/canvas-inquisitor/SKILL.md`**
Replace all `mcp__canvas-mtgibbs__` prefixes with `mcp__canvas__` (7 occurrences).

**Claude.ai Routines** — add remote MCP connector:
- Name: `canvas`, URL: `https://<hosted-url>/mcp`, Auth: Bearer `<MCP_AUTH_TOKEN>`

### Implementation Sequence

1. Create `jrolstad/canvas-mcp` GitHub repo
2. Implement `canvas_client.py` (Phase 1 methods + pagination)
3. Implement `server.py` (7 tools + cache + transport toggle)
4. Add `Dockerfile`, `requirements.txt`, `.env.example`
5. Test locally in stdio mode
6. Choose hosting option and deploy, verify HTTPS endpoint
7. Update `~/.claude.json`, run `/canvas-inquisitor` against hosted URL — confirm parity
8. Add `canvas` connector in Claude.ai; update skill prefix; move this todo to done
9. (Optional) Add Phase 2 tools

### Pitfalls

- **Pagination**: Always follow `Link: rel="next"`; set `per_page=50` on all requests
- **Observer scope**: Use `observed_user_id={student_id}` param on course endpoints
- **FastMCP version**: Pin `mcp>=1.3.0` for Streamable HTTP transport support
- **Cold starts**: Azure Container Apps and Railway free tier may have ~5–15s cold starts after inactivity — acceptable for a daily routine

### Critical Reference Files

- `repos/becu-mcp/server.py` — canonical template for server.py + cache pattern
- `repos/becu-mcp/becu_client.py` — canonical template for client module
- `repos/rolstad-home-skills/.claude/skills/canvas-inquisitor/SKILL.md` — exact tool names to match
- `~/.claude.json` — existing Canvas token + URL, MCP server registry to update
