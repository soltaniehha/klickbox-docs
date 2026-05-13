# Integrating OpenClaw with KlickBox

This is a guide for **you, the User**, deploying your own OpenClaw against your KlickBox account. KlickBox does not host OpenClaw — you run it on hardware you control (a home server, a VPS, a Mac mini, etc.). KlickBox's only job is to authenticate your OpenClaw's API key and serve your Tasks through the same REST API the iPhone app uses.

If you haven't read `./context.md`, read it first. The terms in **bold** below are defined there.

## 1. Get an API Key

In KlickBox on your iPhone:

1. Settings → **API Key** section → **Generate API Key**.
2. The one-time-reveal sheet appears with the plaintext key. Use **Copy** (clipboard) or **Share** (AirDrop, Notes, Messages) to get the key off your phone before dismissing.
3. The format is `klkb_live_<43 chars>`. After you dismiss the sheet, only a SHA-256 hash is on our servers — we cannot recover the plaintext. The Settings list will show only a `klkb_•••••XXXX` masked preview from then on.
4. Paste it into your OpenClaw config (typical env var name: `KLICKBOX_API_KEY`).

If you lose a key, tap **Rotate API Key** to generate a fresh one. The old key is replaced locally; revocation on the server side is a separate operational step (use the Supabase Studio for now — automated revoke ships in v1.x).

## 2. Endpoints OpenClaw should know about

OpenClaw talks to one base URL — the KlickBox-hosted Supabase project that backs every KlickBox account:

```
https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth
```

The host is the same for every User; per-User isolation is enforced by your API Key (server-side hash → User), not by the URL. Every path under that URL mirrors PostgREST. The Edge Function authenticates your API key and proxies the request to the database with your User scope. The most relevant endpoints for an agent are below.

### Read the active dashboard

```bash
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?status=eq.active&select=*,task_tags(position,tag:tags(*))&order=base_score.desc" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY"
```

This returns every active Task with its ordered Tag list embedded. Position 0 is the **Primary Tag**. The agent computes **Effective Score** locally if it needs to reason about urgency.

### Create a Task with Tags and a Base Score

The Tag attach happens in a second call. The first call creates the Task (with the agent's chosen **Base Score** — Scorer = Creator), the second sets the ordered Tag list (first ID becomes the **Primary Tag**).

```bash
# 1. Create the Task.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{
    "title": "Review the Q2 deck",
    "notes": "Focus on slides 3–7",
    "base_score": 80,
    "due_date": "2026-05-10T17:00:00Z",
    "status": "active"
  }'
# -> { "id": "<task-uuid>", ... }

# 2. Attach Tags in order. First in the list = Primary Tag.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/set_task_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "p_task_id": "<task-uuid>",
    "p_tag_ids": ["<work-tag-uuid>", "<deck-review-tag-uuid>"]
  }'
```

**Reuse Tags before inventing them.** Always `GET /tags` first and match by name. If a Tag doesn't exist and you need it, create it via `POST /tags` with a hex color of your choice (the User can recolor it later). Don't spam new Tags for synonyms — the User has to live with the result on their dashboard.

### Mark a Task complete

```bash
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"completed"}'
```

The `completed_at` timestamp is set by a server-side trigger; you don't need to send it. The Task moves to the **Archive**.

### Update a Task's Base Score

```bash
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"base_score":65}'
```

## 3. Suggested toolset for your agent

Whatever framework you're using to build OpenClaw, expose roughly this toolset to the model. Fewer tools beats more tools.

| Tool | What it does | Underlying call |
|---|---|---|
| `list_tasks(status?)` | Returns active or completed Tasks with embedded Tags. | `GET /tasks?status=eq.<...>&select=*,task_tags(...)` |
| `create_task(title, base_score, notes?, due_date?, tag_ids?)` | Creates a Task and (if tag_ids provided) attaches them in order — first becomes Primary. | `POST /tasks` then `POST /rpc/set_task_tags` |
| `update_task(id, fields)` | Patch any subset of mutable fields. | `PATCH /tasks?id=eq.<id>` |
| `complete_task(id)` | Sets status to `completed`. | `PATCH /tasks?id=eq.<id>` with `{status:'completed'}` |
| `set_task_tags(id, tag_ids)` | Replace the ordered Tag list on a Task. First ID becomes Primary. | `POST /rpc/set_task_tags` |
| `list_tags()` | Returns the User's Tags. Always call this before creating a new Tag. | `GET /tags` |
| `create_tag(name, color)` | Creates a Tag. Color is `#rrggbb`. | `POST /tags` |

### Tools you should **not** expose

- **`rescore_all`** — Rescore All is the User's call, made via the iPhone app. Your agent should not initiate it on its own. **However**, your agent should accept being asked to rescore as a foreground action: when the User says "rescore everything," walk the active Tasks list, compute new Base Scores per your logic, and `update_task` each one. That's just `update_task` in a loop — there's no special "rescore" endpoint.
- **`delete_task`** — permanent delete is an explicit User action from the Archive screen. Letting an agent permanently delete Tasks invites disasters. If the User asks the agent to "get rid of" a Task, the agent should **complete** it (which moves it to the Archive) rather than DELETE it.
- **`generate_api_key` / `revoke_api_key`** — the API itself doesn't permit these via API-key auth (JWT only). Don't try.

## 4. Rescore All notifications (v1.0 vs v1.x)

When the User taps **Rescore All** in the app, the backend records the request. **In v1.0, the backend does not push the request to OpenClaw.** Two options for handling this:

1. **Polling.** Have OpenClaw poll a future endpoint (e.g. `GET /rescore-requests?status=eq.queued`) every minute or so. When it sees a queued request, walk the active Tasks list, write new Base Scores via `update_task`, and (eventually) mark the request handled.
2. **Manual trigger.** The User runs OpenClaw with their own command surface (a Slack message, an iMessage, a CLI), and tells it directly: "Hey, rescore everything." The agent walks the list and writes new Base Scores. The Rescore All button in the app is then a UX hint to the User, not a wire-level trigger.

**v1.x will add a webhook.** The User will register a URL in their KlickBox settings; the Rescore All button will POST to that URL with the queued request_id; OpenClaw will respond by walking the list. This is the long-term answer. Don't build polling in a way that's hard to remove — the webhook will make it dead code.

## 5. Operational tips

- **Idempotency.** None of the write endpoints are idempotent on retry — `POST /tasks` will create a duplicate Task if you call it twice. If your agent retries on transport errors, dedupe at the agent layer (e.g. only retry on connect-time failures, not on 5xx after the request has been sent). v1.x will add idempotency keys.
- **Rate limits.** None enforced in v1.0. Be polite — the dashboard query is cheap, but writes are expensive in aggregate. A sensible upper bound for an agent doing maintenance is roughly one request per second per User.
- **Time zones.** Send `due_date` as UTC ISO 8601. The User's iPhone renders in the User's local time. Don't assume your server's time zone matches the User's.
- **Errors.** The Edge Function returns 401 if your key is missing, malformed, or revoked. PostgREST returns 4xx with a JSON body explaining what went wrong; respect those messages — the model usually does better with the original error than with a paraphrase.
- **Telemetry on the backend.** We log resolved `user_id` and the proxied path. We do not log your bearer token. We do not log request bodies in v1.0 — if you set notes to something sensitive, that's between you and your Postgres host.

## 6. A minimum viable OpenClaw

If you're starting from scratch, the smallest useful agent loop is:

1. On boot, `list_tags()` and `list_tasks(status='active')` so the model has the User's vocabulary and current state.
2. Listen on whatever inbound channel the User chose (iMessage, Slack, voice, …).
3. Convert the inbound message to one of the seven tools above. Default to `create_task` for "remind me to / I need to / add a task" intents and `update_task` / `complete_task` for "the X task is done / move X to tomorrow."
4. Respond to the User on the same channel with a one-line confirmation.

That's enough to be useful on day one. The fancier behavior — proactive triage, calendar integration, Rescore All on demand — can layer on top without changing the surface.

## 7. Reference SDKs and smoke test

If you don't want to write the seven HTTP calls by hand, the repo ships reference clients that wrap them:

- **TypeScript / Deno**: `backend/openclaw-reference/typescript/` — drop-in for any Deno or Node 18+ agent. See `typescript/README.md`.
- **Python**: `backend/openclaw-reference/python/` — `uv sync && uv run`. Async, httpx-based. See `python/README.md`.

Both clients consume the machine-readable contract at `./tools.json`. Adding an eighth tool means editing one JSON file and adding a method to each client.

To verify your deploy end-to-end after pasting a fresh API key:

```bash
KLICKBOX_API_KEY=klkb_live_… \
KLICKBOX_BASE_URL=https://oyarcsgekpltnxjmidqk.functions.supabase.co \
  ./backend/openclaw-reference/smoke-test.sh
# → 8 passed, 0 failed
```

The smoke test creates a probe Task and Tag both prefixed `smoke-probe-`. A SQL clean-up one-liner is in `backend/openclaw-reference/README.md`.
