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

Use the atomic `create_task_with_tags` RPC — single transaction, no window where the Task exists with no Primary Tag.

```bash
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/create_task_with_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "p_title": "Review the Q2 deck",
    "p_notes": "Focus on slides 3–7",
    "p_base_score": 80,
    "p_due_date": "2026-05-10T17:00:00Z",
    "p_tag_ids": ["<work-tag-uuid>", "<deck-review-tag-uuid>"]
  }'
# -> { "id": "<task-uuid>", ... }
```

If you don't yet know the Tag list at creation time, fall back to the two-step path: `POST /tasks` followed by `POST /rpc/set_task_tags`. The atomic RPC is preferred whenever the Tag list is known up front.

#### Choosing a Base Score

Base Score is a 0–100 number, and it is **relative** — it only means something next to the User's other Tasks. There is no fixed rubric, and you should not invent one. **On boot, read the User's existing Tasks (`list_tasks(status='active')`, with a glance at `deferred` and `completed`) and calibrate against them before scoring anything new.** Scoring is inherently relative, and the live list is the best reference you have for where a new Task belongs — anchor new Tasks against the scores already on the board rather than restarting the scale at 100 every time.

As a rough orientation until you've seen the User's data:

- **High (~75–100):** must happen soon and clearly matters — a near-term deadline or something the User flagged as important (e.g. "Review the Q2 deck" before the meeting, "Ship the website redesign").
- **Medium (~40–70):** real work with no hard deadline, or one piece of a larger effort (e.g. "Pick a new color palette" as a Child of a redesign).
- **Low (~0–35):** nice-to-have, someday, or low-stakes errands the User wouldn't mind slipping.

Don't inflate Base Score for time pressure: the **Urgency Boost** is added on top at read time from the `due_date` (reaching +10 at the due date, saturating at +20 once a week overdue). Score the underlying importance and let the due date carry the urgency — a Base-Score-60 Task due tomorrow already outranks a Base-Score-70 Task with no due date once Effective Score is computed.

**Reuse Tags before inventing them.** Always `GET /tags` first and match by name. If a Tag doesn't exist and you need it, create it via `POST /tags` with a hex color of your choice (the User can recolor it later via `update_tag`, or delete one via `delete_tag`). Don't spam new Tags for synonyms — the User has to live with the result on their dashboard.

### Mark a Task complete

```bash
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"completed"}'
```

The `completed_at` timestamp is set by a server-side trigger; you don't need to send it. The Task moves to the **Archive**.

### Defer a Task to **Later** / restore

```bash
# "Set this aside for now" — moves the Task to the Later tab (deferred_at set).
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"deferred"}'

# Restore from Later (or from the Archive) back onto the dashboard.
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"active"}'
```

Base Score is preserved on defer — restoring puts the Task back at its original priority. `deferred_at` is set/cleared by server triggers; the iOS Later tab sorts by `deferred_at desc`.

### Update a Task's Base Score

```bash
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"base_score":65}'
```

### Break a Task into Child Tasks (Task Family)

When the User says "break this down" or "add three subtasks under X", **prefer creating Child Tasks over Checklist Items** when each step deserves its own score, tags, due date, or attachments. Use Checklist Items only for "the literal steps I will perform" (call, email, file).

A Task with one or more Children is called a **Family**. The Parent and the Children are full Tasks — each has its own `id`, `base_score`, `tags`, `due_date`, `status`, attachments, and Comments. Depth is capped at **one level**: a Child cannot itself have Children (the server enforces this via the `tasks_validate_parent` trigger).

```bash
# 1. Create the Parent.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/create_task_with_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "p_title": "Ship the website redesign",
    "p_base_score": 80,
    "p_tag_ids": ["<work-tag-uuid>"]
  }'
# -> { "id": "<parent-uuid>", ... }

# 2. Create each Child by passing p_parent_task_id.
# Children do NOT inherit the Parent's Tags automatically — pass p_tag_ids
# explicitly if the Children should be tagged.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/create_task_with_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "p_title": "Pick a new color palette",
    "p_base_score": 60,
    "p_parent_task_id": "<parent-uuid>",
    "p_tag_ids": ["<work-tag-uuid>", "<design-tag-uuid>"]
  }'
```

**Family completion rule (server-enforced):** A Parent Task cannot be marked completed while any of its Children are non-completed. PATCHing `status=completed` on a Family with active Children returns `22023` with `"cannot complete a family while it has active children"`. Mark the Children done first, then the Parent.

```bash
# Listing a Family's Children — use list_children, or the equivalent filter.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?parent_task_id=eq.<parent-uuid>&select=*,task_tags(position,tag:tags(*))&order=base_score.desc" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY"

# Emulating the User's Dashboard view — top-level only, Child Tasks excluded.
# (Child Tasks render inside their Parent on the iPhone, not on the Dashboard.)
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?status=eq.active&parent_task_id=is.null&select=*,task_tags(position,tag:tags(*))&order=base_score.desc" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY"
#
# Caveat — orphan Children: the iOS Dashboard ALSO surfaces Children whose
# Parent is currently in Later (`status='deferred'`) or in the Archive
# (`status='completed'`) at the top level, until the Parent is restored.
# The query above returns a *subset* of what the User sees on the
# Dashboard in that case. For sub-second-accurate mirroring, follow up
# with a second pass: list the non-active Parents (status=deferred or
# completed) and then `list_children` for each — those Children belong
# on the rendered Dashboard too. Most agent flows can skip this; it
# matters only when the agent is replicating the User's view exactly.

# Move an existing Task under a Parent (or unparent it — pass null).
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/set_task_parent" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_task_id":"<child-uuid>","p_parent_task_id":"<new-parent-uuid>"}'

# Promote a Child to standalone.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/set_task_parent" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_task_id":"<child-uuid>","p_parent_task_id":null}'
```

**When the User asks the agent to "reorganize" or "group these tasks":** look at the active list (`list_top_level_tasks(status='active')`), propose a grouping, and use `create_task` + `set_task_parent` (or `create_task_with_tags` with `p_parent_task_id`) to build the Family. Narrate the moves in the response — "I moved 'Draft slides' under 'Q3 review'" — so the User can verify before the dust settles.

**Error semantics for parent ops:**
- `42501` — the Task you're trying to move isn't yours.
- `23503` — the Parent you specified isn't yours or doesn't exist (same error — the server does not distinguish, to close the cross-tenant UUID-existence oracle).
- `22023` — depth violation, self-parent, completed-parent, or promote-with-children. The message explains which.

### Break a Task into Checklist Items

```bash
# Each item carries a `position` (double-precision rank). Append with
# (max_position + 1); insert between two siblings with (above + below) / 2.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/checklist_items" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"task_id":"<task-uuid>","text":"draft the outline","position":1}'

# Toggle done.
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/checklist_items?id=eq.<item-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"is_done":true}'
```

Use Checklist Items to "break down" a Task instead of cramming a bullet list into `notes`. The iPhone dashboard renders them with checkboxes; `notes` does not.

### Log progress with Comments

```bash
# Add a Comment. Use Comments for progress / status updates instead of
# rewriting the Task's notes field.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/comments" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{"task_id":"<task-uuid>","body":"Called the contractor — left a voicemail."}'

# Edit within 5 minutes; older Comments return 403.
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/comments?id=eq.<comment-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"body":"Called the contractor — they will call back tomorrow."}'

# Read the progress log with any Comment-attached Attachments embedded.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/comments?task_id=eq.<task-uuid>&select=*,attachments(*)&order=created_at.asc" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY"
```

### Read an Attachment

`list_attachments` returns metadata only. To read the bytes, request a short-lived signed URL via `get_attachment_url`:

```bash
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/get_attachment_signed_url" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_attachment_id":"<attachment-uuid>"}'
# -> {"url":"https://<project>.supabase.co/storage/v1/object/sign/attachments/...","expires_in":60}

curl -sSL "$signed_url" -o /tmp/attachment.bin
```

The URL is valid for 60 seconds. Don't cache it — request a fresh one when you need to read.

## 3. Suggested toolset for your agent

Whatever framework you're using to build OpenClaw, expose roughly this toolset to the model. Fewer tools beats more tools, but the v1.2 surface has been designed so each tool maps to a distinct User intent on the iPhone.

**Tasks**

| Tool | What it does | Underlying call |
|---|---|---|
| `list_tasks(status?)` | Returns ALL Tasks with embedded Tags (standalone + Children mixed). `status` ∈ `{active, deferred, completed}` or omit for all. | `GET /tasks?status=eq.<...>&select=*,task_tags(...)` |
| `list_top_level_tasks(status?)` | Returns ONLY standalone Tasks and Parent Tasks — Child Tasks are excluded. Use this when emulating the User's Dashboard view. | `GET /tasks?parent_task_id=is.null&status=eq.<...>` |
| `list_children(parent_task_id, status?)` | Returns one Family's Children. | `GET /tasks?parent_task_id=eq.<parent>` |
| `create_task(title, base_score, notes?, due_date?, parent_task_id?)` | Creates a Task without Tags. Pass `parent_task_id` to create a Child under an existing Parent. | `POST /tasks` |
| `create_task_with_tags(p_title, p_base_score, p_notes?, p_due_date?, p_parent_task_id?, p_tag_ids?)` | **Preferred** create path. Atomic Task + ordered Tag attach. First Tag = Primary. Pass `p_parent_task_id` to create a Child Task. | `POST /rpc/create_task_with_tags` |
| `update_task(id, fields)` | Patch any subset of mutable fields. `parent_task_id` is patchable too, but prefer `set_task_parent`. | `PATCH /tasks?id=eq.<id>` |
| `complete_task(id)` | Sets status to `completed` → moves to Archive. **Rejected if the Task is a Parent with active Children.** | `PATCH /tasks?id=eq.<id>` with `{status:'completed'}` |
| `defer_task(id)` | Sets status to `deferred` → moves to Later. Base Score preserved. | `PATCH /tasks?id=eq.<id>` with `{status:'deferred'}` |
| `restore_task(id)` | Sets status to `active` → returns to dashboard from Later or Archive. | `PATCH /tasks?id=eq.<id>` with `{status:'active'}` |
| `set_task_tags(id, tag_ids)` | Replace the ordered Tag list on an existing Task. First ID becomes Primary. | `POST /rpc/set_task_tags` |
| `set_task_parent(p_task_id, p_parent_task_id)` | Move a Task between Families, or pass `null` to make it standalone. | `POST /rpc/set_task_parent` |

**Tags**

| Tool | What it does | Underlying call |
|---|---|---|
| `list_tags()` | Returns the User's Tags. Always call before creating a new Tag. | `GET /tags` |
| `create_tag(name, color)` | Creates a Tag. Color is `#rrggbb`. | `POST /tags` |
| `update_tag(id, fields)` | Rename or recolor an existing Tag. | `PATCH /tags?id=eq.<id>` |
| `delete_tag(id)` | Delete a Tag. Cascade-removes references on Tasks. | `DELETE /tags?id=eq.<id>` |

**Checklist Items** — flat boolean steps under one Task. Use these for "the literal steps I will perform"; use **Child Tasks** (above) for "independent units of work that deserve their own score, tags, and due date."

| Tool | What it does | Underlying call |
|---|---|---|
| `list_checklist_items(task_id)` | Returns items ordered by position. | `GET /checklist_items?task_id=eq.<id>&order=position.asc` |
| `add_checklist_item(task_id, text, position, is_done?)` | Append/insert. `position` is a fractional rank. | `POST /checklist_items` |
| `update_checklist_item(id, fields)` | Rename (`text`), toggle (`is_done`), reorder (`position`). | `PATCH /checklist_items?id=eq.<id>` |
| `delete_checklist_item(id)` | Delete. | `DELETE /checklist_items?id=eq.<id>` |

**Comments** — chronological progress log per Task.

| Tool | What it does | Underlying call |
|---|---|---|
| `list_comments(task_id)` | Returns Comments chronologically with Comment-attached Attachments embedded. | `GET /comments?task_id=eq.<id>&select=*,attachments(*)&order=created_at.asc` |
| `add_comment(task_id, body)` | Append. | `POST /comments` |
| `update_comment(id, body)` | Edit body. Server enforces a 5-minute window from `created_at`; older Comments 403. | `PATCH /comments?id=eq.<id>` |
| `delete_comment(id)` | Delete. No time restriction. Cascade-removes attachments on this Comment. | `DELETE /comments?id=eq.<id>` |

**Attachments** — agent reads, plus the three-call upload flow (see §5).

| Tool | What it does | Underlying call |
|---|---|---|
| `list_attachments(task_id)` | Returns Attachment metadata for a Task (excluding Comment-attached; those are embedded by `list_comments`). | `GET /attachments?task_id=eq.<id>` |
| `get_attachment_url(p_attachment_id)` | Returns `{url, expires_in:60}` — a short-lived signed URL the agent can GET to read the blob. Ownership-checked via RLS server-side. | `POST /rpc/get_attachment_signed_url` (virtual RPC handled by api-key-auth) |

### Tools you should **not** expose

- **`rescore_all`** — Rescore All is the User's call, made via the iPhone app. Your agent should not initiate it on its own. **However**, your agent should accept being asked to rescore as a foreground action: when the User says "rescore everything," walk the active Tasks list, compute new Base Scores per your logic, and `update_task` each one. That's just `update_task` in a loop — there's no special "rescore" endpoint.
- **`delete_task`** — permanent delete is an explicit User action from the Archive screen. Letting an agent permanently delete Tasks invites disasters. If the User asks the agent to "get rid of" a Task, the agent should **complete** it (which moves it to the Archive) or **defer** it (which moves it to Later) rather than DELETE it.
- **`generate_api_key` / `revoke_api_key`** — the API itself doesn't permit these via API-key auth (JWT only). Don't try.
- **`upload_attachment`** — agents can both upload and read Attachments. Uploads use the three-call `request_attachment_upload` → `PUT` → `confirm_attachment_upload` flow described below; reads use `get_attachment_url`. The iPhone is no longer the only uploader.

## 4. Rescore All notifications (v1.0 vs v1.x)

When the User taps **Rescore All** in the app, the backend records the request. **In v1.0, the backend does not push the request to OpenClaw.** Two options for handling this:

1. **Polling.** Have OpenClaw poll a future endpoint (e.g. `GET /rescore-requests?status=eq.queued`) every minute or so. When it sees a queued request, walk the active Tasks list, write new Base Scores via `update_task`, and (eventually) mark the request handled.
2. **Manual trigger.** The User runs OpenClaw with their own command surface (a Slack message, an iMessage, a CLI), and tells it directly: "Hey, rescore everything." The agent walks the list and writes new Base Scores. The Rescore All button in the app is then a UX hint to the User, not a wire-level trigger.

**v1.x will add a webhook.** The User will register a URL in their KlickBox settings; the Rescore All button will POST to that URL with the queued request_id; OpenClaw will respond by walking the list. This is the long-term answer. Don't build polling in a way that's hard to remove — the webhook will make it dead code.

## 5. Uploading attachments

Agents can attach images, PDFs, audio, or generic files to a Task. Three-call flow:

1. **`request_attachment_upload`** — `POST /rpc/request_attachment_upload`
   Body: `{ p_task_id, p_kind, p_filename, p_byte_size, p_mime_type }`
   Returns: `{ attachment_id, upload_url, expires_at }`

2. **`PUT <upload_url>`** with the raw bytes as the body. The URL is valid for 60 seconds. Use `Content-Type` matching `p_mime_type`. Returns 200 on success.

3. **`confirm_attachment_upload`** — `POST /rpc/confirm_attachment_upload`
   Body: `{ p_attachment_id }`
   Returns: `true` once the blob is in Storage; `false` if the blob is not yet visible (retry) or the row was already confirmed by someone else.

If step 3 returns `false` repeatedly, double-check the PUT response. The server only flips the row to visible when it can physically see the bytes in Storage — a lying agent cannot mark uploads as complete.

### Allowed MIME types (`p_kind` must match)

| `p_mime_type`              | `p_kind` |
|----------------------------|----------|
| `image/*`                  | `photo`  |
| `application/pdf`          | `pdf`    |
| `audio/*`                  | `audio`  |
| `text/*`, `application/zip`, `application/octet-stream` | `file` |

Anything else → 415 with `unsupported mime type`.

### Limits

| Limit                  | Value       | Trigger                                              |
|------------------------|-------------|------------------------------------------------------|
| Per-file size          | 25 MiB      | 413 `file too large`                                 |
| Daily upload count     | 5 / 24h     | 429 with `Retry-After` header                        |
| Daily upload bytes     | 50 MiB / 24h | 429 with `Retry-After` header                        |

Rate limits charge on **request** (not confirm). An agent that requests then never PUTs still counts against its daily allowance — design retries accordingly.

### Worked example (curl, generic 5-byte file)

```bash
# 1. Request — declared size MUST match the bytes you intend to PUT.
curl -X POST "$BASE/api-key-auth/rpc/request_attachment_upload" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_task_id":"<task-uuid>","p_kind":"file","p_filename":"notes.txt","p_byte_size":5,"p_mime_type":"text/plain"}'
# => {"attachment_id":"…","upload_url":"https://…?token=…","expires_at":"…"}

# 2. PUT the exact bytes. The server's confirm step re-checks that the
#    object exists in Storage; a mismatched byte length will land but
#    the agent's reported size will diverge from the actual blob.
curl -X PUT "<upload_url>" \
  -H "Content-Type: text/plain" \
  --data-binary "hello"

# 3. Confirm
curl -X POST "$BASE/api-key-auth/rpc/confirm_attachment_upload" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_attachment_id":"<from step 1>"}'
# => true
```

### Querying usage

`POST /rpc/get_storage_stats` (no body) returns the same counters the iPhone Settings screen shows: attachment count, bytes used, today's upload count + bytes, and the daily caps. Useful for an agent that wants to defer non-essential uploads when close to the cap.

## 6. Operational tips

- **Idempotency.** None of the write endpoints are idempotent on retry — `POST /tasks` will create a duplicate Task if you call it twice. If your agent retries on transport errors, dedupe at the agent layer (e.g. only retry on connect-time failures, not on 5xx after the request has been sent). v1.x will add idempotency keys.
- **Rate limits.** None enforced in v1.0. Be polite — the dashboard query is cheap, but writes are expensive in aggregate. A sensible upper bound for an agent doing maintenance is roughly one request per second per User.
- **Time zones.** Send `due_date` as UTC ISO 8601. The User's iPhone renders in the User's local time. Don't assume your server's time zone matches the User's.
- **Errors.** The Edge Function returns 401 if your key is missing, malformed, or revoked. PostgREST returns 4xx with a JSON body explaining what went wrong; respect those messages — the model usually does better with the original error than with a paraphrase.
- **Telemetry on the backend.** We log resolved `user_id` and the proxied path. We do not log your bearer token. We do not log request bodies in v1.0 — if you set notes to something sensitive, that's between you and your Postgres host.

## 7. A minimum viable OpenClaw

If you're starting from scratch, the smallest useful agent loop is:

1. On boot, `list_tags()` and `list_tasks(status='active')` so the model has the User's vocabulary and current state.
2. Listen on whatever inbound channel the User chose (iMessage, Slack, voice, …).
3. Convert the inbound message to a tool call. Sensible defaults:
   - *"remind me to / I need to / add a task"* → `create_task_with_tags`
   - *"the X task is done"* → `complete_task`
   - *"set X aside for now / snooze X"* → `defer_task`
   - *"break X into steps"* → repeated `add_checklist_item`
   - *"log that I called the contractor"* → `add_comment`
4. Respond to the User on the same channel with a one-line confirmation.

That's enough to be useful on day one. The fancier behavior — proactive triage, calendar integration, Rescore All on demand — can layer on top without changing the surface.

## 8. Reference SDKs and smoke test

If you don't want to write the HTTP calls by hand, the repo ships reference clients that wrap them:

- **TypeScript / Deno**: `backend/openclaw-reference/typescript/` — drop-in for any Deno or Node 18+ agent. See `typescript/README.md`.
- **Python**: `backend/openclaw-reference/python/` — `uv sync && uv run`. Async, httpx-based. See `python/README.md`.

Both clients consume the machine-readable contract at `./tools.json`. Adding an eighth tool means editing one JSON file and adding a method to each client.

To verify your deploy end-to-end after pasting a fresh API key:

```bash
KLICKBOX_API_KEY=klkb_live_… \
KLICKBOX_BASE_URL=https://oyarcsgekpltnxjmidqk.functions.supabase.co \
  ./backend/openclaw-reference/smoke-test.sh
# → 22 passed, 0 failed
```

The smoke test walks the v1.2 tool surface (create with tags, defer/restore, checklist, comments, the three-call attachment upload + listing + signed-URL fetch, storage stats, complete, tag cleanup, and a final delete_task cascade verification) and creates a probe Task / Tag prefixed `smoke-probe-`. A SQL clean-up one-liner is in `backend/openclaw-reference/README.md`.
