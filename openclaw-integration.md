# Integrating OpenClaw with KlickBox

This guide is for whoever is wiring an OpenClaw up to a KlickBox account: the User who deploys it, or the agent that reads this and writes the calls. Sections 1 and 2 are setup and speak to the User; from section 3 on it addresses the agent directly.

KlickBox does not host OpenClaw — you run it on hardware you control (a home server, a VPS, a Mac mini). KlickBox's only job is to authenticate your OpenClaw's API key and serve your Tasks through the same REST API the iPhone app uses.

Terms in **bold** are defined in the domain glossary (`./context.md`). You do not need to read it first; look terms up there when one is unclear.

**If you are here for the Idea Bank, section 6 is the part you cannot skip.** Its failure modes are all silent: the call succeeds, the app shows the Idea handled, and the User's captured words are gone or permanently stuck.

**Contents**

1. [Get an API Key](#1-get-an-api-key) — generate, rotate, revoke
2. [Endpoints OpenClaw should know about](#2-endpoints-openclaw-should-know-about) — the two base-URL forms, core Task/Tag calls, Effective Score, Families, Checklist Items, Comments
3. [Suggested toolset for your agent](#3-suggested-toolset-for-your-agent) — the tool tables, the capability boundary, tools you should not expose
4. [Handling "rescore everything"](#4-handling-rescore-everything)
5. [Uploading attachments](#5-uploading-attachments) — the three-call flow, MIME allowlist, limits
6. [Processing the Idea Bank](#6-processing-the-idea-bank) — **read before writing the loop**; opens with the rules checklist
7. [Operational tips](#7-operational-tips) — idempotency, rate limits, the error table
8. [A minimum viable OpenClaw](#8-a-minimum-viable-openclaw)
9. [Reference SDKs and smoke test](#9-reference-sdks-and-smoke-test)

## 1. Get an API Key

In KlickBox on your iPhone:

1. Settings → **API Key** section → **Generate API Key**.
2. The reveal sheet appears with the plaintext key. Use **Copy** (clipboard) or **Share** (AirDrop, Notes, Messages) to get the key off your phone.
3. The format is `klkb_live_<43 chars>`. Only a SHA-256 hash of it reaches our servers, so we can never recover or re-send your plaintext. The plaintext itself stays in your iPhone's Keychain, which is why Settings → **Manage API Key** can re-reveal and re-copy it later without rotating.
4. Paste it into your OpenClaw config (typical env var name: `KLICKBOX_API_KEY`).

### If a key leaks, revoke it from your iPhone

All of it happens in the app, in seconds. You never need database access.

- **Rotate API Key** (Settings → **Manage API Key**) mints a replacement and revokes the previous key on the server as part of the same transaction. The instant the new key exists the old one is dead, and the next request carrying it gets `401 invalid_credentials`. Paste the new key into your OpenClaw config.
- **Delete API Key** revokes the key on the server and scrubs it from the device, leaving you with no key at all. Use this when you want your agent to stop now and are not ready to issue a replacement. Your Tasks are untouched, and you can generate a new key whenever you want the agent back.
- **Copy key** re-copies the existing plaintext from your Keychain without changing anything, so an agent already using the key keeps working.

Two properties worth knowing:

- **Revocation is one-way.** A revoked key can never be un-revoked, by you or by us (the database rejects the attempt). The row survives so `created_at`, `last_used_at`, and `revoked_at` stay auditable. Settings shows when the key was last used, so you can tell whether a leaked key was actually exercised before you killed it.
- **A User has at most one live key.** Minting a key automatically revokes every prior live key for that User, so "rotate" really means "replace." If you run more than one OpenClaw, they share one key.

## 2. Endpoints OpenClaw should know about

OpenClaw talks to one base URL — the KlickBox-hosted Supabase project that backs every KlickBox account:

```
https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth
```

**Two forms of this URL exist and they are not interchangeable.** Get this wrong and *every* call fails with `403 {"error":"path_not_allowed"}`, which looks exactly like a broken deployment or a missing table — it is neither.

| You are writing | Use | Why |
|---|---|---|
| Raw HTTP (`curl`, `fetch`, any hand-rolled client) | `https://<ref>.functions.supabase.co/api-key-auth` | the full endpoint; you append `/tasks`, `/rpc/…` yourself |
| A client built from `tools.json` (including the reference SDKs and anything you generate) | `https://<ref>.functions.supabase.co` — **no `/api-key-auth`** | the contract carries `base_path: "/api-key-auth"` and the client prepends it for you |

`KLICKBOX_BASE_URL` is the second form. Passing the first form there produces `…/api-key-auth/api-key-auth/tasks`, which the proxy rejects before it reaches the database. The tell is that the same table returns 200 to a direct `curl` seconds later.

The host is the same for every User; per-User isolation is enforced by your API Key (server-side hash → User), not by the URL. Every path under that URL mirrors PostgREST: the Edge Function authenticates your API key and proxies the request to the database with your User scope, so PostgREST's filtering, ordering, embedding and pagination syntax all work as documented upstream.

The proxy forwards an **allowlist** of tables and RPCs, not the whole database. That allowlist covers everything in this guide plus a few join and child tables the iPhone app writes through (see "Tools you should not expose" in §3); treat `./tools.json` as the surface you should actually use. A path outside the allowlist is rejected before it reaches Postgres. Two things are deliberately unreachable with an API Key: API-key management (minting and revoking require a signed-in session on the iPhone) and account deletion. The most relevant endpoints for an agent are below.

### Read the active Dashboard

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

**Reuse Tags before inventing them.** Always `GET /tags` first and match by name. If a Tag doesn't exist and you need it, create it via `POST /tags` with a hex color of your choice (the User can recolor it later via `update_tag`, or delete one via `delete_tag`). Don't spam new Tags for synonyms — the User has to live with the result on their Dashboard.

#### Computing Effective Score

`base_score` is stored; **Effective Score** is not, and is what the Dashboard sorts by. Compute it yourself:

```python
# Rounding: the iOS app rounds half away from zero (Swift .rounded());
# Python's round() is banker's — the two differ only at exact halves.
def urgency_boost(due_date, now):        # -> 0..20
    if due_date is None:
        return 0
    days = (due_date - now).total_seconds() / 86400
    if days <= 0:                        # overdue
        late = -days
        return 20 if late >= 7 else round(10 + (late / 7) * 10)
    if days >= 30:                       # too far out to matter
        return 0
    return round(((30 - days) / 30) * 10)

def effective_score(task, now):
    return max(0, min(100, task["base_score"] + urgency_boost(task["due_date"], now)))
```

Sort descending. **Tie-break** (common at 100 and 0): the more recently modified Task first — `updated_at` desc. A Task whose `defer_until` is still in the future is hidden from the Dashboard even though it is `status='active'`; exclude it the same way you exclude a Deferred Task. One more wrinkle when mirroring the User's screen exactly: under the default Family Priority Mode (`urgent_child`), a Parent's Dashboard position is lifted to `max(parent's Effective Score, most urgent active Child's Effective Score)` — see **Task Family** in the glossary.

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
# "Set this aside for now" — moves the Task to the Later view (deferred_at set).
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"deferred"}'

# Restore from Later (or from the Archive) back onto the Dashboard.
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status":"active"}'
```

Base Score is preserved on defer — restoring puts the Task back at its original priority. `deferred_at` is set/cleared by server triggers; the iOS Later view sorts by `deferred_at desc`.

### Hide a Task until a date (`defer_until`), and recurrence

`defer_until` is **not** the same as deferring. `status='deferred'` moves the Task to the **Later** view and it stays there until someone restores it. `defer_until` leaves the Task `status='active'` and simply hides it from the Dashboard until the instant passes, after which it reappears on its own. "Snooze until Monday" is `defer_until`; "set this aside, I'll deal with it eventually" is `defer_task`.

```bash
curl -sS -X PATCH \
  "$BASE/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"defer_until":"2026-08-03T08:00:00Z"}'      # null un-hides immediately
```

Treat a Task whose `defer_until` is in the future like a Deferred Task when answering "what should I work on next".

`recurrence_rule` is a client-interpreted JSON blob; completing a Task that carries one spawns the next instance **client-side**, so nothing recurs while the phone is closed and the server never creates rows for you.

```json
{"frequency": {"v": 1, "kind": "weekly"}, "anchor": "2026-07-28T09:00:00Z"}
```

`kind` is `daily` | `weekly` | `monthly` | `everyN` (`n` is the day interval, read only for `everyN` — e.g. `{"v":1,"kind":"everyN","n":3}`). Read it freely; only write it if you emit exactly that shape — clients treat a malformed rule as not-recurring and silently stop recurring the Task. Pass `null` to remove recurrence. A recurring Parent does not clone its Children on spawn, and when a recurring Child spawns, the spawned next instance starts standalone — the completed instance stays in the Archive as a Child of its original Parent.

### Update a Task's Base Score

```bash
curl -sS -X PATCH \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"base_score":65}'
```

### Set or change the **Primary Tag** on an existing Task

The **Primary Tag** is whichever Tag sits at **position 0** in the Task's ordered `task_tags` list — it's the Tag whose color the Dashboard uses, and the one the iPhone marks with a ⭐️. There is **no dedicated "primary" column and no `make_primary` endpoint**: you set the Primary by re-sending the *whole* ordered Tag list with the Tag you want first. Use the `set_task_tags` RPC, which **replaces** the list atomically.

```bash
# Make <consulting-tag-uuid> the Primary on an existing Task, keeping
# <work-tag-uuid> as a secondary. set_task_tags is REPLACE, not merge —
# you MUST include every Tag the Task should keep, in the order you want.
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/set_task_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "p_task_id": "<task-uuid>",
    "p_tag_ids": ["<consulting-tag-uuid>", "<work-tag-uuid>"]
  }'
```

**The two mistakes agents make here:**

1. **Trying to PATCH a `primary_tag` / `category` field on `/tasks`.** That field does not exist — `PATCH /tasks` cannot change Tags at all. Tag membership and order only change through `set_task_tags` (or `create_task_with_tags` at creation).
2. **Sending only the one Tag you want to promote.** Because `set_task_tags` *replaces* the list, `p_tag_ids: ["<consulting>"]` would **drop every other Tag** on the Task. First read the current list, then resubmit it reordered:

```bash
# 1. Read the Task's current ordered Tags (position 0 = current Primary).
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/tasks?id=eq.<task-uuid>&select=id,task_tags(position,tag:tags(*))" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY"
# -> tags currently [work(pos 0), consulting(pos 1)]

# 2. Resubmit the SAME set, reordered so the new Primary is first.
#    (consulting promoted to position 0, work kept at position 1.)
curl -sS \
  "https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth/rpc/set_task_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_task_id":"<task-uuid>","p_tag_ids":["<consulting-tag-uuid>","<work-tag-uuid>"]}'
```

Passing an empty `p_tag_ids: []` clears all Tags (the Task then renders in a neutral color until re-tagged). The KlickBox iPhone app reads `position 0` as the Primary Tag, so a reorder you make here surfaces as the ⭐️ in the User's app on its next sync.

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

Use Checklist Items to "break down" a Task instead of cramming a bullet list into `notes`. The iPhone Dashboard renders them with checkboxes; `notes` does not.

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

Whatever framework you're using to build OpenClaw, expose roughly this toolset to the model. Fewer tools beats more tools, but this surface has been designed so each tool maps to a distinct User intent on the iPhone. The authoritative list, with its version, is `tools.json`; the tables below mirror it.

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
| `restore_task(id)` | Sets status to `active` → returns to Dashboard from Later or Archive. | `PATCH /tasks?id=eq.<id>` with `{status:'active'}` |
| `set_task_tags(id, tag_ids)` | Replace the ordered Tag list on an existing Task — also the way to set/change the **Primary Tag** (first ID = Primary = position 0). REPLACE, not merge: include every Tag to keep. See "Set or change the Primary Tag" above. | `POST /rpc/set_task_tags` |
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
| `get_attachment_url(p_attachment_id)` | Returns `{url, expires_in:60}` — a short-lived signed URL the agent can GET to read the blob. Ownership-checked via RLS server-side. Also works for Idea Entry attachments. | `POST /rpc/get_attachment_signed_url` (virtual RPC handled by api-key-auth) |

**Idea Bank**: the User's capture surface for ongoing Projects, processed by your agent on a schedule the User controls. See "Processing the Idea Bank" below for the loop and the per-media rules.

| Tool | What it does | Underlying call |
|---|---|---|
| `list_projects()` | Returns the User's Idea Bank Projects (archived included). Always call before creating one; reuse existing Projects. | `GET /projects?select=*&order=name.asc` |
| `create_project(name, color)` | Creates a Project. Name unique per User, case-insensitive (409 on duplicate); color is `#rrggbb`. | `POST /projects` |
| `update_project(id, fields)` | Rename, recolor, archive (set `archived_at`), or unarchive (`archived_at: null`). | `PATCH /projects?id=eq.<id>` |
| `list_ideas(needs_processing?, id?)` | Returns Ideas with the embedded ordered Project list (position 0 = Primary Project; empty = Inbox). `needs_processing=true` is the processing loop's entry point. | `GET /ideas?select=*,idea_projects(position,project:projects(*))&order=content_updated_at.desc` |
| `get_idea_thread(idea_id)` | One Idea's full thread, oldest first, with each Entry's Attachments embedded. | `GET /idea_entries?idea_id=eq.<id>&select=*,attachments(*)&order=created_at.asc` |
| `create_idea(...)` | Atomic Idea + first Entry + ordered Project links, for capturing a thought the User sends you. Every field optional; first Project UUID becomes Primary. | `POST /rpc/create_idea_with_projects` |
| `add_idea_entry(idea_id, ...)` | Append an amendment to an Idea's thread. Bumps `content_updated_at` server-side, which reopens a Processed Idea. | `POST /idea_entries` |
| `set_idea_projects(p_idea_id, p_project_ids)` | Replace the ordered Project list. REPLACE semantics like `set_task_tags`; position 0 = Primary; empty array = Inbox. Refiling never reopens an Idea. | `POST /rpc/set_idea_projects` |
| `mark_idea_processed(p_idea_id, p_processed_through, p_summary?)` | The receipt for one processed Idea. `p_processed_through` MUST be the `content_updated_at` you fetched, never now(). | `POST /rpc/mark_idea_processed` |
| `reopen_idea(id)` | Clears the processed marker for a "redo this one" request. | `PATCH /ideas?id=eq.<id>` with `{processed_at: null}` |
| `update_idea(id, fields)` | Patch the title. You may set a short neutral title on untitled captures; the User sees it in the app. | `PATCH /ideas?id=eq.<id>` |
| `delete_idea(id)` | Permanent delete of the Idea, its thread, and their Attachments. Only on explicit User instruction. | `DELETE /ideas?id=eq.<id>` |

#### Capability boundary: who may call what

Two different actors read this contract, and they do **not** get the same toolset.

- **Your OpenClaw** (this guide) gets everything listed above.
- **The In-App Session** — the short Claude turns that run inside the KlickBox app — gets only the read/capture subset of the Idea Bank: `list_projects`, `list_ideas`, `get_idea_thread`, `create_idea`, `add_idea_entry`, `set_idea_projects`.

**`mark_idea_processed` must never appear in an In-App Session toolset.** Processing means "I read this and filed it into the User's workspace", and the In-App Session has no workspace to file into. If it can call the RPC, it can close an Idea that nobody processed, and the Idea will never come back to you: the marker is the only record that the work happened. The same reasoning keeps `reopen_idea`, `update_idea`, `create_project`, `update_project` and `delete_idea` out of that toolset, and `delete_idea` additionally requires an explicit User instruction even in yours.

This boundary is machine-readable, so you do not have to enforce it from prose. In `tools.json`, every restricted tool carries a `restricted_to` array:

```json
{ "name": "mark_idea_processed", "restricted_to": ["openclaw"], "...": "..." }
```

A tool without the field carries no restriction. If you generate a toolset from `tools.json` for anything other than the User's own OpenClaw, filter on `restricted_to` rather than copying the whole list; the top-level `actors` block documents the vocabulary.

### Tools you should **not** expose

- **`rescore_all`** — Rescore All is the User's call, made via the iPhone app. Your agent should not initiate it on its own. **However**, your agent should accept being asked to rescore as a foreground action: when the User says "rescore everything," walk the active Tasks list, compute new Base Scores per your logic, and `update_task` each one. That's just `update_task` in a loop — there's no special "rescore" endpoint.
- **`delete_task`** — permanent delete is an explicit User action from the Archive screen. Letting an agent permanently delete Tasks invites disasters. If the User asks the agent to "get rid of" a Task, the agent should **complete** it (which moves it to the Archive) or **defer** it (which moves it to Later) rather than DELETE it.
- **`generate_api_key` / `revoke_api_key`** — the API itself doesn't permit these via API-key auth (JWT only). Don't try.
- **`PATCH` / `DELETE` on `/idea_entries`** — the proxy allows the table because the iPhone app writes through it, but no tool in the contract exposes it and you should not add one. Those are the User's own words, and deleting a single Entry is permanent with no undo. Amend a thread with `add_idea_entry`; never rewrite or remove what the User wrote. (Apart from the processing marker itself — `mark_idea_processed`, and `reopen_idea` when the User asks — `title` via `update_idea` is the only Idea field you should write.)
- **Direct writes to `/task_tags` and `/idea_projects`** — reachable, and wrong. Both are ordered join tables with a uniqueness rule on `position`; hand-writing rows corrupts the ordering that defines the **Primary Tag** and **Primary Project**. Use `set_task_tags` / `set_idea_projects`, which replace the list atomically.
- **Direct `POST` to `/attachments`** — creates a metadata row with no blob behind it, which every reader (including the iPhone) will then fail to open. The only supported upload path is the three-call flow in §5.
(**`upload_attachment` is not on this list.** Agents can both upload and read Attachments: uploads use the three-call `request_attachment_upload` → `PUT` → `confirm_attachment_upload` flow in §5, reads use `get_attachment_url`. The iPhone is not the only uploader.)

## 4. Handling "rescore everything"

There is **no rescore endpoint and no rescore notification**. Nothing in the API tells your agent that the User wants a rescore, and there is nothing to poll for one.

The way this works today is a **manual trigger**: the User asks your agent directly on whatever channel they already use with it (a Slack message, an iMessage, a CLI) — "rescore everything." Your agent then lists the active Tasks, computes new Base Scores with its own judgment, and writes each one with `update_task`. That is all a **Rescore All** is: `update_task` in a loop. Do not look for a bulk endpoint.

Your agent should accept that instruction when the User gives it, and should not initiate a rescore on its own — reshuffling every priority unprompted is not a thing an agent should decide to do.

## 5. Uploading attachments

Agents can attach images, PDFs, audio, or generic files to a Task. Three-call flow:

1. **`request_attachment_upload`** — `POST /rpc/request_attachment_upload`
   Body: `{ p_task_id, p_kind, p_filename, p_byte_size, p_mime_type }`
   Returns: `{ attachment_id, upload_url, expires_at }`

2. **`PUT <upload_url>`** with the raw bytes as the body. The URL is valid for 60 seconds. Use `Content-Type` matching `p_mime_type`. Returns 200 on success.

3. **`confirm_attachment_upload`** — `POST /rpc/confirm_attachment_upload`
   Body: `{ p_attachment_id }`
   Returns: `true` once the blob is in Storage; `false` if the blob is not yet visible (retry) or the row was already confirmed by someone else.

If step 3 returns `false` repeatedly, double-check the PUT response. The server only flips the row to visible when it can physically see the bytes in Storage, so a confirm can never run ahead of the actual upload.

**Uploads attach to Tasks only.** `request_attachment_upload` takes a `p_task_id` and there is no Comment or Idea-Entry equivalent — the phone is the only uploader for those. So the artifacts you produce while processing the Idea Bank live in *your* workspace, not back on the Idea: report where they went in `p_summary`, and if the User wants something actionable out of it, create a Task (see "Turning an Idea into a Task") and attach the file there.

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
curl -X POST "$BASE/rpc/request_attachment_upload" \
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
curl -X POST "$BASE/rpc/confirm_attachment_upload" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_attachment_id":"<from step 1>"}'
# => true
```

### Querying usage

`POST /rpc/get_storage_stats` (no body) returns the same counters the iPhone Settings screen shows: attachment count, bytes used, today's upload count + bytes, and the daily caps. Useful for an agent that wants to defer non-essential uploads when close to the cap.

## 6. Processing the Idea Bank

**Idea Bank: the rules, in order.** Each links to the prose that explains it; none is optional.

1. [Fetch the marker before you read, never after](#fetch-the-marker-before-you-read-never-after) — keep each Idea's `content_updated_at` as the exact string you received.
2. [Page every list, and never walk `list_ideas` with a growing offset](#do-not-walk-list_ideas-with-a-growing-offset) — `206` is success, not an error.
3. [Detect a Reopen before you diff](#the-user-can-close-an-idea-without-you) — `processed_at` null while you already hold an artifact means redo from scratch.
4. [Reconcile all three dimensions](#reconciling-a-reopened-idea) — the title, each Entry's `(id, updated_at)`, each Attachment's `(id, status)`. If the diff finds no difference at all, stop and do not mark.
5. [Filter Attachments to `synced`, skip `pending_upload` for this pass, and never close over an unfetchable blob](#entry-attachments).
6. [Apply the per-media rules](#per-media-rules) — the User's original words stay verbatim in the artifact.
7. [Write and flush your artifact before you mark](#the-loop) — the mark is the only record the work happened.
8. [Retitle before you mark, then re-fetch the marker](#manners) — your own title write moved the content clock.
9. [Echo the marker as the exact string](#echoing-the-marker) — `mark_idea_processed` with the string from step 1, never `now()`; `p_summary` is a hard 500-character cap.
10. [`22023` means your marker is newer than the content — branch on the message, and re-process before re-marking](#when-mark_idea_processed-returns-22023).

The Idea Bank is the User's capture surface for ongoing Projects (a book, a talk). Your job, on whatever schedule the User configures on your side, is to pull Open Ideas, transform each one into useful project material in your own workspace, and mark it processed. KlickBox shows the User which Ideas you have handled; anything they amend afterward comes back to you automatically.

An Idea is Open when `needs_processing` is true. That state is derived server-side from two timestamps (`processed_at is null or content_updated_at > processed_at`), so there is no status flag for you to maintain. What you do have to get right is the marker you hand back and how you work out what changed, which is what the rest of this section is about.

### The two rules that keep content from being lost

These two are the ones almost everyone gets wrong. Three more further down are just as load-bearing and are flagged where they appear: **never walk `list_ideas` with a growing offset**, **fetch the marker before you read, never after**, and **detect a Reopen before you diff**. Nothing in this section is decoration.

**Rule 1 — reconcile everything the Idea is made of, never just the new tail.** An Idea reopens for **five** different reasons, and only one of them appends to the end of the thread:

| What changed | What moves on the wire |
|---|---|
| An Entry was **added** | a new Entry `id` appears |
| An Entry was **edited** | that Entry's `updated_at` moves; `created_at` does not |
| An Entry was **deleted** | an Entry `id` disappears |
| The **title** was edited | `ideas.title` changes; no Entry moves at all |
| An **Attachment** arrived, finished uploading, or was removed | that Attachment's `id` or `status` changes; no Entry moves at all |

If you diff by "what is newer than last time" you catch only the first. If you diff by Entry alone you still miss the last two, and those two are silent: the Idea comes back to you with `needs_processing` true and every Entry looking untouched. Snapshot **all three** of title, Entries, and Attachments. See "Reconciling a reopened Idea" below.

**Rule 2 — echo `content_updated_at` back as the exact string you received.** Copy the JSON string verbatim into `p_processed_through`. Do not parse it into a date object and re-serialize it, and do not re-format it. Details and the failure it prevents are under "Echoing the marker" below.

### The loop

1. `list_ideas` with `needs_processing=eq.true`. Keep each Idea's **`content_updated_at` string** exactly as received: that is your receipt for step 7, and the one value you must not transform. Also keep its `processed_at`. `null` means either you have never seen this Idea **or** the User cleared the marker to ask for a redo — check whether you already hold an artifact for it to tell those apart, and see "The User can close an Idea without you" for the redo branch. It is not what you reconcile against; see step 2. Page through the results (see "Pagination" below), or a first run against a backlog will stop after one page.
2. `get_idea_thread` for the full thread, paging the same way. Reconcile against your artifact on **all three** dimensions — the Idea's `title`, each Entry's `(id, updated_at)`, and each Attachment's `(id, status)`. An Entry-only diff misses the two reopens that move no Entry. See "Reconciling a reopened Idea".
3. Ensure a folder per Project in your workspace, named exactly after the Project (`Book/`, `Keynote/`). Ideas with an empty `idea_projects` list go in `Inbox/`. An Idea with several Projects is filed under its Primary Project (position 0) with cross-references from the others.
4. Process each added or edited Entry per the media rules below. Maintain **one artifact per Idea** (for example `Book/ideas/<short-slug>.md`) and update it in place; do not create a second file for a reopened Idea.
5. Never destroy the User's original words. Every artifact keeps a verbatim "Original" section and a separate "Processed" section (summary, extraction, connections to other Ideas in the same Project).
6. **Write and flush your artifact first.** The mark is the only record that the work happened, so marking before your files are durable means a crash leaves the Idea showing Processed in the User's app, with a summary describing work that no longer exists anywhere, and it never comes back to you.
7. `mark_idea_processed` with `p_processed_through` set to the `content_updated_at` **string** from step 1 (**never `now()`**, never a reformatted timestamp) and `p_summary` set to one human sentence the User will read in the app, naming what you did and where it went ("Summarized + OCR'd 2 screenshots into Keynote/opening-story.md"). `p_summary` is capped at **500 characters** and `title` at **200** — these are database constraints, not truncation. Over-run and the entire call fails with a `23514` check violation, which means the mark never lands and the Idea stays Open with the work already done. Budget the summary before you send it.

Marking is deliberately forgiving in one direction and strict in the other. The marker is **monotonic**: the server stores `greatest(existing, sent)`, so re-sending an older value never moves `processed_at` backwards and cannot un-process an Idea you already closed. Retrying a cached marker is therefore safe. A marker **older** than the current content is accepted and simply leaves the Idea Open: that is the amend-while-you-worked case, and it is not an error, so do not treat the successful response as proof the Idea is now closed. Read `needs_processing` on the returned row if you want to know. A marker **newer** than the current content is rejected — see below.

### Reconciling a reopened Idea

Store a **snapshot** in your artifact alongside whatever you produced. Three parts, and you need all three:

```json
{
  "title": "Opening story: the lighthouse keeper",
  "entries": { "<entry-uuid>": "<that entry's updated_at string>" },
  "attachments": { "<attachment-uuid>": "synced" }
}
```

On the next pass, fetch the thread and compare:

| Situation | What it means | What to do |
|---|---|---|
| Entry `id` fetched, not in your snapshot | Entry **added** | Process it, add it to the artifact. |
| Entry `id` in both, `updated_at` **differs** | Entry **edited** | Re-process that Entry **in place**: replace its Original text, redo its Processed section. Do not append a second copy. |
| Entry `id` in both, `updated_at` identical | Unchanged | Leave it. Read it for context, don't file it again. |
| Entry `id` in your snapshot, **not** in the fetched thread | Entry **deleted** | See "Handling a deletion" below. Do **not** apply this branch unless you fetched the thread to completion. |
| `title` differs from your snapshot | Title **edited** | Update the artifact's heading. No Entry will have moved. |
| Attachment `id` fetched, not in your snapshot | Attachment **added** | Handle per its `status` (see "Entry Attachments"). |
| Attachment `id` in both, `status` changed to `synced` | The blob **finished uploading** | This is the one you were waiting for. Download and process it now. No Entry will have moved. |
| Attachment `id` in your snapshot, not fetched | Attachment **removed** | Drop it from the artifact, same care as a deleted Entry. |

Four things about this are deliberate.

**Compare `updated_at`, not `created_at`.** An edit bumps `updated_at` and leaves `created_at` untouched, so a `created_at` filter reports zero new material on an Idea that reopened precisely because the User rewrote something. You would then re-mark it processed and the edit would be gone.

**Compare against your own recorded value, not against `processed_at`.** Testing `updated_at > processed_at` looks equivalent and is not: the two timestamps come from different clocks (`updated_at` is stamped at transaction start, the Idea's content clock at write time), so an edit written inside a long transaction can carry an `updated_at` that sits *before* a marker committed earlier. Comparing a fetched value against the value you previously recorded is a plain equality check on your own observation, and no server-side clock can skew it. If you have no recorded value — a first pass, or an artifact predating this format — treat everything as added and process it.

**Deletions are only visible as an absence.** No timestamp filter can return a row that no longer exists, so the set comparison is the only mechanism that detects a removal at all.

**Title and Attachment changes move nothing in the thread.** A retitle and an upload completing both reopen the Idea while every Entry's `updated_at` stays exactly where it was. If your snapshot holds only Entries, those two reopens look like "nothing changed" and you will re-mark the Idea and never read the photo. This is why `title` and attachment `status` are in the snapshot.

**If the comparison reports no difference at all, stop and do not mark.** Every reopen has a cause. Finding none means you are not looking at one of the five dimensions — most often you snapshotted Entries only, or your thread fetch was truncated by paging. Re-read the Idea row and the full thread before doing anything else. Re-marking on "no difference" is how content goes missing without an error anywhere.

#### Handling a deletion

An absent `id` means one of two things, and they need opposite responses:

- The User deleted it.
- **Your fetch was incomplete** — a page you never requested, a request that failed, a loop that exited early.

They look identical. So apply the deletion branch **only when you know you fetched the whole thread**: the paging loop ran to a short page, every request returned 2xx, and nothing threw. If any of that is uncertain, treat the Idea as not-yet-processed, leave it Open, and try again next pass. An Idea processed twice costs nothing; an artifact pruned against a truncated fetch destroys material the server no longer has.

When the deletion is real, **prefer moving the material to a "Removed by the User" section over deleting it outright**. Deleting a single Entry in the app is permanent and offers no undo, so once the User does it your artifact may be the only copy of those words left anywhere. Keep it, marked clearly as removed and dated, unless the User has told you they want removals purged. Just never leave removed material presented as if it were current.

An Idea always has at least one Entry — the original capture, created with the Idea itself. **A thread that comes back empty is a failed fetch, not an empty Idea.** Never write an empty artifact and never mark such an Idea processed.

### Echoing the marker

`content_updated_at` is a Postgres `timestamptz` and reaches you as a JSON string with **microsecond** precision and a numeric UTC offset:

```
"2026-07-24T16:40:19.881123+00:00"
```

Trailing zeros in the fractional part are trimmed, so the fraction is 0 to 6 digits and its length varies row to row (`...:19.88112+00:00`, `...:19.8+00:00`, `...:19+00:00` are all values you will see). Take the string out of the JSON response and put that same string into `p_processed_through`:

```python
idea = ideas[0]
marker = idea["content_updated_at"]        # keep the raw string
...                                         # do the work
mark_idea_processed(p_idea_id=idea["id"], p_processed_through=marker, p_summary=...)
```

The failure this avoids is not exotic. It is what the most idiomatic line in several languages does:

```javascript
// WRONG in JavaScript / TypeScript, and silently so.
const marker = new Date(idea.content_updated_at).toISOString();
// "2026-07-24T16:40:19.881123+00:00" -> "2026-07-24T16:40:19.881Z"
```

`Date` has millisecond resolution, so the microseconds are simply gone. Go does the same: `time.RFC3339` drops the fractional part entirely, costing up to a full second. Python's `datetime` happens to round-trip microseconds, so `fromisoformat(...).isoformat()` survives — but only by luck, and only on 3.11+, which is not a property worth betting the User's content on.

A marker truncated even a microsecond short is a **legal, older marker**. The call succeeds, no error is raised, `needs_processing` stays true, and the Idea is reprocessed on every pass forever while the User watches it never close. Treating the value as an opaque string sidesteps the entire class, in every language.

**What is compared is the instant, not the text.** The server parses your string into a `timestamptz`, so a genuinely equivalent spelling of the same instant is fine and there is no textual-identity requirement. The reason to echo the string anyway is that "equivalent spelling" is exactly what date libraries get wrong: they normalize the offset *and* truncate the fraction in the same call, and only the second one hurts. Keeping the string removes the judgment call.

One consequence worth planning for: if your JSON layer coerces timestamp-shaped strings into date objects for you (many typed clients and ORMs do), the damage is already done before your code runs. Take `content_updated_at` from the raw response body, or configure that layer to leave it alone.

### When `mark_idea_processed` returns `22023`

`22023` on this RPC means your marker is **newer than the Idea's current content** — you are claiming to have processed content that does not exist. It is not a staleness signal: holding stale data produces the no-error path above, where the mark lands and the Idea simply stays Open.

In practice a future marker means one of these bugs on your side:

- You passed `now()` or your own wall clock instead of the fetched value.
- Your clock is ahead of the server's.
- You cached a marker from a previous pass and echoed it after the Idea was rebuilt, or you re-sent a marker you had already adjusted.

The correct response is to **re-fetch, process the new content, and then mark with the value you just fetched**:

1. `list_ideas` with `id=eq.<idea-uuid>` and read the current `content_updated_at`.
2. `get_idea_thread` and reconcile as above — actually read the content.
3. `mark_idea_processed` with the string from step 1.

Never re-fetch a marker and immediately re-send it without re-processing. That closes the Idea against content you have not read, which is exactly the loss this error exists to prevent. If you find yourself retrying `22023` in a loop, the fix is in your timestamp handling, not in the retry.

**`22023` is a generic "invalid argument" code, so branch on the message, not the code alone.** The same code comes back from this RPC when you send `p_processed_through: null`, and from `create_idea` / `set_idea_projects` for a bad `kind`/`link_url` pairing or a repeated project id. Only one of those is the future-marker case, and only that one requires re-processing:

| Message contains | Meaning | What to do |
|---|---|---|
| `newer than the idea content` | Future marker | Re-fetch, **re-process**, then mark with the value just fetched. |
| `p_processed_through is required` | You sent null | Send the marker you already hold. Do **not** re-process; you have read this content. |
| anything else | Not from this call's marker logic | Handle per the tool that raised it. |

A handler that applies the re-fetch-and-reprocess recipe to every `22023` will loop forever on the null case; one that applies the null recipe to every `22023` will re-mark over unread content. Read the message.

`P0002` `idea not found` means the Idea was deleted, never existed, or is not yours — deliberately indistinguishable. Drop it from your queue rather than retrying.

#### Fetch the marker before you read, never after

The safety of this whole loop rests on one ordering, so it is worth making explicit: **read `content_updated_at` first, then read the content, then mark.** That is the order in "The loop" above and in the recovery recipe, and it is not incidental.

A marker taken *before* you read can only be older than or equal to what you read, and an older marker is the safe direction: it leaves the Idea Open and you see the amendment next pass.

The tempting "improvement" is to refresh the marker just before marking, so it is nice and current — especially after a long pass on a big PDF. **Do not.** Any Entry the User added while you worked is then covered by a marker you fetched after it landed, so the Idea closes over content you never read. No error is raised, `needs_processing` goes false, and it never comes back to you. The stale-looking marker is the feature.

### Pagination

PostgREST caps rows per response. A request that returns the cap is a signal that there is more, not a complete answer — so a first run against a backlog of Ideas will otherwise process one page and report itself finished. Page explicitly on `list_ideas` and on `get_idea_thread`, both of which can exceed a page (a long-running Idea accumulates Entries).

Use either form:

```bash
# Range header. Ask for the total with Prefer: count=exact; the response's
# Content-Range is "0-99/247" — offset-end/total.
curl -sS \
  "$BASE/ideas?needs_processing=eq.true&select=*,idea_projects(position,project:projects(*))&order=content_updated_at.desc" \
  -H "Authorization: Bearer $KEY" \
  -H "Range: 0-99" \
  -H "Prefer: count=exact" -D -

# Or limit/offset as query params.
curl -sS \
  "$BASE/ideas?needs_processing=eq.true&order=content_updated_at.desc&limit=100&offset=100" \
  -H "Authorization: Bearer $KEY"
```

A ranged request that returns only part of the set comes back **`206 Partial Content`**, not `200`. That is success, not an error: treat `200` and `206` the same. An agent that accepts only `200` throws away every page after the first.

The cap is a server setting, not a constant you should hard-code — always send an explicit `limit` or `Range` rather than relying on whatever the server's default happens to be. Do not infer "there is no more" from a response smaller than *your* limit unless you set one — read `Content-Range`, whose format is `first-last/total` (`0-99/247`), and page until `last + 1 >= total`.

#### Do not walk `list_ideas` with a growing offset

This is the one that quietly halves a backlog, so it is worth stating plainly. You are paging a filtered set (`needs_processing=eq.true`) **while removing rows from it**: every `mark_idea_processed` drops that Idea out of the filter. Offsets computed against the old set address the wrong rows in the new one.

Concretely, with 300 Ideas and `limit=100`: `offset=0` gives you Ideas 1-100, you process and mark them, and the set is now 200 rows. `offset=100` addresses rows 101-200 of *that* set, which are the original Ideas 201-300. Ideas 101-200 are never fetched. `offset=200` returns nothing, your loop stops, and your agent reports a clean sweep of a backlog it two-thirds processed. Nothing errors. `Range` headers have exactly the same problem.

Two correct shapes:

**Re-request the first page each time** (simplest, and right because the set shrinks):

```python
LIMIT = 100
seen = set()
while True:
    page = list_ideas(needs_processing=True, limit=LIMIT)   # always offset 0
    fresh = [i for i in page if i["id"] not in seen]
    for idea in fresh:
        seen.add(idea["id"])
        process_and_mark(idea)     # marking removes it from the filter
    # Stop only when this page held nothing new AND was not full. A full page
    # of Ideas you deliberately left Open (pending uploads, stalls) means
    # there is more behind them — breaking on `not fresh` alone would report
    # a clean sweep of a backlog you never reached.
    if not fresh and len(page) < LIMIT:
        break
    if not fresh:
        # Every Idea at the head of the queue is one you already tried this
        # run. Page past them rather than re-reading the same page forever.
        page = list_ideas(needs_processing=True, limit=LIMIT, offset=len(seen))
        if not page:
            break
```

The `seen` set stops an Idea you failed to process, or deliberately left Open, from spinning the loop. Add it to `seen` *before* processing, as above, so an Idea that throws does not retry endlessly inside one run — it stays Open and comes back on the next.

**Or keyset-paginate** on the sort key, which is stable under deletion from the set:

```
&needs_processing=eq.true&order=content_updated_at.desc,id.desc&limit=100
# next page: &content_updated_at=lt.<last row's content_updated_at>
```

Either way, **mark each Idea as you finish it** rather than batching the marks to the end — an interrupted run then resumes instead of redoing the backlog. That is the same advice as before; it is only offset arithmetic that it conflicts with.

`get_idea_thread` is different and safe to offset-page: Entries are not removed from the set as you read them. Order by `created_at.asc,id.asc` and walk it normally. Always send an explicit `order` on any paged request, or page boundaries are not stable between calls.

### Entry Attachments

Each Entry in a thread carries an `attachments` array (empty for text-only Entries). An Attachment row looks like this:

```json
{
  "id": "e04b7c11-93a2-4d6f-8b05-1f7c62d94a38",
  "user_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "task_id": null,
  "comment_id": null,
  "idea_entry_id": "7b3e5d90-4a1c-4f28-b6e7-9c0d2f8a1e53",
  "kind": "photo",
  "status": "synced",
  "mime_type": "image/jpeg",
  "byte_size": 184320,
  "remote_path": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d/e04b7c11-93a2-4d6f-8b05-1f7c62d94a38.jpg",
  "created_at": "2026-07-24T16:40:19.881123+00:00",
  "updated_at": "2026-07-24T16:40:21.004518+00:00"
}
```

That is the whole row: there are no other fields. In particular **there is no original filename.** The name the User picked on their phone is not stored server-side — the upload call takes one, but uses it only to derive a sanitized extension for `remote_path`, which is `<user_id>/<attachment_uuid>.<ext>`. So the extension is all that survives, and no endpoint will give you `Q3-budget.csv` back.

**Name files yourself, from fields that do exist.** A rule that works:

```
<project>/ideas/<idea-slug>/<kind>-<n>.<ext>
                                             # ext = the part of remote_path
                                             # after the last dot
Book/ideas/lighthouse-keeper/photo-1.jpg
Book/ideas/lighthouse-keeper/file-2.csv
```

Take `<ext>` from `remote_path`, `<kind>` from the Attachment's `kind`, and `<n>` from its position among that Entry's Attachments in `created_at` order. Fall back to a `mime_type`-derived extension if `remote_path` somehow has none. Record the Attachment `id` next to whatever you name it, so the name stays stable across passes even if ordering changes — the `id` is the only durable handle you have.

If the User refers to a file by a name you never received, say so plainly rather than guessing which Attachment they mean.

`byte_size` is what the uploader *declared* and may be null on older rows, so treat it as a hint, not a checksum.

Two fields decide what you do with it.

**`status` — read this before you touch the blob.** It is `pending_upload` or `synced`, and which one you see depends on who created the row. Rows created through the agent upload flow (§5's `request_attachment_upload` three-call sequence) start at `pending_upload` and flip to `synced` only once the bytes are confirmed in Storage — but that flow is **Task-only**, so in an Idea thread you should never actually meet one: every Attachment on an Entry is phone-created, and phone rows never pass through `pending_upload`. The app publishes the metadata row as `synced` immediately and ships the bytes separately, often over a slow or dropped connection — so a thread fetched a second after capture legitimately contains a `synced` row whose blob is not in Storage yet. Such a row is not waiting on any status flip; if you cannot fetch its blob, that is the **unfetchable** case described further down, not a pending one.

`get_attachment_url` **fails on a `pending_upload` row**: there is no object to sign, so it comes back `502 storage_sign_failed`. That status looks like a transient server fault and is not one — retrying it will fail identically until the agent upload flow that created the row calls `confirm_attachment_upload`. Check `status` first rather than reading the 502 as something to back off and retry.

So: **keep only the Attachments whose `status` is `synced`, and skip the rest for this pass.** In an Idea thread the filter is defensive — no supported writer produces a `pending_upload` row there, so meeting one means someone hand-wrote a `POST /attachments` outside the supported flows — but it costs nothing and keeps you honest. Fetch the thread as usual and filter in your own code — the Attachments arrive embedded, so this costs no extra request:

```bash
curl -sS \
  "$BASE/idea_entries?idea_id=eq.<idea-uuid>&select=*,attachments(*)&order=created_at.asc" \
  -H "Authorization: Bearer $KEY"
```

```python
for entry in thread:
    ready   = [a for a in entry["attachments"] if a["status"] == "synced"]
    pending = [a for a in entry["attachments"] if a["status"] != "synced"]
```

Filter client-side rather than with a PostgREST filter on the embedded table: an embedded filter changes which *Entries* come back as well, and you want every Entry in the thread regardless of whether its files have landed. The id-set reconciliation above depends on seeing the whole thread.

Do not treat a skipped `pending_upload` row as a failure, and do not retry it in a tight loop. Handle it like this:

- Process everything else in the Idea normally.
- **Still call `mark_idea_processed`** with the marker you fetched, and say so in the summary ("filed 2 of 3 attachments; one still uploading").
- Note the skipped Attachment `id` **in your artifact**, not only in the summary. `p_summary` is overwritten by the next successful mark, so a breadcrumb left only there disappears the next time that Idea reopens for any unrelated reason.
- Revisit on a later pass. The `pending_upload` → `synced` lifecycle belongs to **Task** attachments (§5's agent upload flow); in an Idea thread every supported-writer row is phone-created and `synced` from the start (its INSERT is what reopened the Idea), so no flip is coming and no reopen will announce anything: a phone blob that has not landed yet surfaces as an unfetchable `synced` row — handle it via the unfetchable rule below, not by waiting for a flip. Keep the Attachment `status` in your snapshot anyway — the schema does bump `content_updated_at` if a stray `pending_upload` row ever flips, and the reconciliation costs nothing. Either way you still need an age check for uploads that never land — see below.

**Some uploads never land.** An upload flow that requested a slot and never confirmed the bytes leaves a row at `pending_upload` forever, and a phone that is wiped, reset, or simply never reopens the app leaves a `synced` row whose blob never arrives. There is no server-side timeout and no terminal state in either case, so the Idea never reopens and the notification never comes. Do not wait silently: if an Attachment is still `pending_upload` — or still unfetchable — after a few passes, or more than a day or so after its `created_at`, **tell the User** on whatever channel you use with them, naming the Idea. They are the only one who can resolve it, and from their side the app simply shows the Idea as handled.

**`kind` — which media rule applies.** It is `photo`, `pdf`, `audio`, or `file`. `mime_type` is the precise type within that bucket (`image/jpeg`, `application/pdf`, `audio/m4a`, `text/markdown`) and is what you should switch on when the `kind` bucket is too coarse — `file` in particular covers everything from a Markdown export to a zip.

`remote_path` is the object's path inside the private `attachments` Storage bucket (`<user_id>/<attachment_uuid>.<ext>`). It is informational: you cannot GET it directly, and the bucket is not public. Always go through `get_attachment_url`, which returns a signed URL valid for **60 seconds** — request it immediately before you fetch, and never cache or store it.

### Per-media rules

Two independent fields carry the word "kind", and the rules below are indexed by both, so keep them straight:

- **`idea_entries.kind`** — `note`, `quote`, or `link`. Describes the Entry's **text**. It defaults to `note`, so a great many Entries are `note`.
- **`attachments.kind`** — `photo`, `pdf`, `audio`, or `file`. Describes an attached **file**.

They are orthogonal. An Entry always has a text kind and may also carry any number of Attachments — a `note` Entry with two screenshots is completely ordinary, and it is the common shape for "the User photographed something and typed a line about it". So for each Entry: **apply the one text rule that matches `idea_entries.kind`, then apply the file rule for each `synced` Attachment**, and write them into a single artifact section for that Entry. An Entry whose `body` is empty and whose content is entirely in its Attachments is also ordinary; there is no text rule to apply in that case.

**Text: note** (`idea_entries.kind=note`): copy the text verbatim into the artifact's Original section; add a Processed section only when it earns its keep: a distillation for long notes, connections to related Ideas, or nothing at all for a one-liner (don't pad). If `body` is empty, the Attachments are the content — say nothing about the missing text.

**Text: quote** (`idea_entries.kind=quote`): preserve the quote **character-for-character**, with no cleanup and no paraphrase, alongside its `source` attribution and capture date. In the Processed section add why it likely matters to this Project (one or two sentences), and flag missing attribution ("source not recorded") rather than guessing one.

**Text: link** (`idea_entries.kind=link`): fetch `link_url`. Store the URL, the page title, the access date, a summary proportional to the page (a paragraph for an article, a line for a tool's homepage), and the passages most relevant to the Project quoted verbatim. If `body` carries a User note, treat it as the angle they care about and summarize for that angle. If the fetch fails, record the URL with "unreachable on <date>" and still mark the Idea processed; do not retry forever.

**File: image / screenshot** (`attachments.kind=photo`): download and keep the original. If it contains text (a screenshot of a tweet, an article, a slide, a whiteboard), transcribe the text **verbatim**; transcription is the point of most screenshots. Then describe what it shows and why it plausibly relates to the Project. For pure photos (a scene, a diagram sketch), a faithful description plus any legible labels.

**File: PDF** (`attachments.kind=pdf`): download and keep the original file in the Project folder. Extract the text; produce a structured summary (thesis, key points, anything directly relevant to the Project); quote crucial passages verbatim with page numbers.

**File: audio memo** (`attachments.kind=audio`): download, transcribe verbatim (keep the transcript in the artifact), then summarize the substance. The User speaks these on the move; expect fragments, keep their phrasing in the transcript, and clean things up only in the summary.

**File: generic file** (`attachments.kind=file`): everything the other three buckets do not cover — `text/*`, `application/zip`, `application/octet-stream`. Always keep the original in the Project folder; that alone is the minimum acceptable handling, because this bucket is where a User parks something they want kept, not necessarily something they want read. Give it a name of your own (the server keeps no original filename — see above) and keep the extension from `remote_path`. Then branch on `mime_type`:

- **Text-ish** (`text/plain`, `text/markdown`, `text/csv`, anything under `text/`): read it and treat it exactly like a long `note` Entry — verbatim in Original, a distillation in Processed. For a CSV, describe the columns and what the rows appear to be rather than transcribing the table.
- **Archives** (`application/zip`): do not unpack it. Record its size and the User's `body` text if any. Unpacking an archive from an artifact folder is not your call.
- **Unknown** (`application/octet-stream`, anything you cannot parse): record the size and mime type, and note that the contents were not readable. Do not guess at what is inside.

Never fail an Idea because one file was **unreadable**: keep the original, say what you could not do, and mark the Idea processed.

**Unfetchable is a different case, and you must not close over it.** If you could not *download* a `synced` blob at all — the 60-second URL expired mid-transfer, the connection dropped, Storage returned 5xx — you do not have the bytes, and nothing will reopen the Idea for you: the row is already `synced`, so no status flip is coming. Retry with a fresh signed URL. If it still fails, **leave the Idea Open** (skip the mark entirely) and pick it up next pass. "Unreadable" means you have the file and cannot parse it; "unfetchable" means you never got it.

**Amended Ideas** (`processed_at` non-null but newer content): update the existing artifact in place, integrating rather than appending, and note the update date. Which Entries to touch, and what to do about Entries that were edited or deleted rather than added, is the id-set reconciliation described under "Reconciling a reopened Idea" above — that table is the algorithm, and a `created_at`-based shortcut will lose the User's edits.

### Turning an Idea into a Task

Ideas never become Tasks automatically, and **there is no link between the two records** — no field on either side points at the other. Creating a Task from an Idea is an ordinary `create_task_with_tags`; if you want the connection to survive, you have to carry it yourself (put the Idea id in your artifact, and reference the Idea in the Task's `notes`).

```bash
curl -sS "$BASE/rpc/create_task_with_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" -H "Content-Type: application/json" \
  -d '{"p_title":"Draft the lighthouse-keeper opening",
       "p_notes":"From Idea c9d1f3a7… (Book). See Book/ideas/lighthouse-keeper.md",
       "p_base_score":55,
       "p_tag_ids":["<writing-tag-uuid>"]}'
```

Two rules:

- **Tags and Projects do not cross over.** An Idea's Projects are not Tags. Pick or create a Tag by name; never pass a Project UUID as a Tag id.
- **Creating the Task is not processing the Idea.** You still owe the Idea a `mark_idea_processed` with the marker you fetched, and the summary should say where the Task went ("Filed to Book/ and opened Task 'Draft the lighthouse-keeper opening'"). Do not delete or otherwise close the Idea because a Task now exists.

### The User can close an Idea without you

Two things the User does in the app change your queue underneath you. Neither is an error, and neither needs a response from you.

- **Mark Captured.** The User swipes an Open Idea and says, in effect, "my agent doesn't need this one." The app sets `processed_at` to that Idea's own `content_updated_at`, so it becomes Processed and drops out of your `needs_processing=eq.true` results. If an Idea disappears from your queue between passes, this is usually why. Do not reopen it, and do not treat a shrinking queue as a lost fetch.
- **Reopen.** The User clears the marker on an already-Processed Idea (`processed_at` back to null) to ask for a redo.

  **Detecting it is a rule you must implement, not an observation.** The signal is: **you already have an artifact for this Idea, and `processed_at` came back `null`.** That is the one case where the snapshot comparison is the wrong tool — the content did not change, so it reports no differences, and if that is your only branch you will re-mark the Idea and silently no-op the User's explicit request for a redo. Check `processed_at is null` *before* you diff:

  ```python
  if idea["processed_at"] is None and have_artifact(idea["id"]):
      redo_from_scratch(idea)      # discard the old answer, process the whole thread
  else:
      reconcile_snapshot(idea)     # the normal path
  ```

  Rewrite the artifact from the whole thread rather than diffing. A Reopen means the previous answer was not good enough, so producing the same answer again is a failure even though nothing errors.

Both are ordinary PATCHes on `/ideas`, and you have the same two tools (`reopen_idea`, and the marker semantics of `mark_idea_processed`) if the User asks you directly.

### Manners

- Reuse existing Projects; never invent one the User didn't create (`list_projects` first, the same rule as Tags).
- **Project changes do not reopen an Idea, so nothing will tell you an Idea was refiled.** If the User moves an Idea from the Inbox to `Book/` after you processed it, `content_updated_at` does not move, the Idea stays Processed, and your artifact sits in the wrong folder indefinitely. Record each Idea's Primary Project in your snapshot and re-check it whenever you touch the Idea for any reason; if you want folders to track the User promptly, run a cheap `list_ideas` sweep (no thread fetch) on your own schedule and move any artifact whose Primary Project has changed. Refiling is not reprocessing — move the artifact, do not redo the work.
- You may set a short neutral `title` on untitled Ideas via `update_idea`. Retitle **before** you `mark_idea_processed`, never after: a title is content, so writing it bumps `content_updated_at` and reopens the Idea you just closed.

  **Then re-fetch the marker before you mark.** Your own write moved the content clock past the marker you took in step 1, so marking with the old one leaves the Idea Open — and on the next pass your snapshot already contains the title *you* wrote, the diff finds nothing, and the "no difference means stop" rule strands the Idea Open forever. Re-reading `content_updated_at` after a write of your own is the one safe exception to "fetch the marker before you read": you know exactly what changed, because you changed it. Re-fetch, update your snapshot's `title`, then mark.

  Keep the title faithful to their words; the User sees it in the app.
- Never `delete_idea` during processing; deletion is the User's call.
- **Do not PATCH `processed_at` or `content_updated_at` directly to shortcut the loop — and do not expect an error if you do.** Neither write is rejected. The content clock is *upward-only*: the server keeps `greatest(what you sent, what is already stored)`, so a backwards write is silently discarded (2xx, no error, no effect — 204 on a bare PATCH) while a **forwards** write really does move the clock and reopens the Idea for every actor, including you. A `processed_at` you PATCH is separately clamped down to the final `content_updated_at`, so a marker you meant to set into the future silently becomes a present one. Both clamps err toward leaving the Idea Open, which is why they never fail loudly. `mark_idea_processed` is the only path that tells you when your marker is wrong; use it.
- **Nothing on the server checks that you actually read an Idea.** `mark_idea_processed` rejects a marker newer than the content, and the database guarantees the marker can never sit in the future — that is all. A marker equal to the current `content_updated_at` is always accepted, so a loop that marks every Open Idea without reading one of them succeeds, closes all of them, and they never come back to you. The marker is the only record that the work happened. Never write a bulk "mark everything processed" cleanup, and never mark an Idea whose thread you did not successfully fetch.
- Batch politely: the processing loop is a read-heavy scan plus one `mark_idea_processed` per Idea. There is no need for per-Entry writes.
- Mark each Idea as you finish it, rather than batching the marks to the end of the run. An interrupted run then resumes instead of redoing the whole backlog.

## 7. Operational tips

- **Idempotency.** Assume write endpoints are not idempotent on retry — `POST /tasks` will create a duplicate Task if you call it twice. The one deliberate exception is `mark_idea_processed`, whose marker is monotonic: re-sending the same or an older value is safe and cannot un-process an Idea. If your agent retries on transport errors, dedupe at the agent layer (e.g. only retry on connect-time failures, not on 5xx after the request has been sent). v1.x will add idempotency keys.
- **Rate limits.** Your key has a ceiling of **600 requests per minute**. Exceed it and you get `429` with a `Retry-After` header giving the seconds until the window rolls; back off for that long and retry. The ceiling exists so a leaked key cannot be driven hard, not to pace you: a sensible working rate for an agent doing maintenance is roughly one request per second, which is an order of magnitude under the cap. There is also a **per-IP** limiter — 1200 requests per minute and **20 auth failures per minute**, returning `429 too_many_requests` — so an agent hard-retrying a bad key will IP-block itself within seconds and then misread the 429 as a quota problem; honour `Retry-After` and fix the key instead. Attachment uploads have their own separate daily caps (see §5).
- **Time zones.** Send `due_date` as UTC ISO 8601. The User's iPhone renders in the User's local time. Don't assume your server's time zone matches the User's.
- **Errors — there are two different body shapes.** Anything the Edge Function rejects itself returns `{"error": "<code>"}`; anything the database rejects is PostgREST's shape, forwarded unchanged:

  ```json
  { "error": "path_not_allowed" }                                        // Edge Function
  { "code": "22023", "message": "…", "details": null, "hint": null }     // PostgREST
  ```

  When this guide says "branch on the message", it means `body.message` in the second shape. Respect those messages — the model usually does better with the original error than with a paraphrase.

  | Status | Where from | Body | Meaning / what to do |
  |---|---|---|---|
  | 401 | Edge Fn | `missing_credentials` | no `Authorization` header |
  | 401 | Edge Fn | `malformed_credentials` | header is not exactly `Bearer <key>` |
  | 401 | Edge Fn | `invalid_credentials` | key unknown, revoked, or expired — the three are deliberately indistinguishable. Stop; do not retry |
  | 403 | Edge Fn | `path_not_allowed` | the path is outside the proxy's allowlist — usually a doubled base URL (see §2), not a broken deploy |
  | 429 | Edge Fn | `key_rate_limited` | over your key's 600/min ceiling. Honour `Retry-After` |
  | 429 | Edge Fn | `too_many_requests` | per-IP limiter (see Rate limits above) |
  | 502 | Edge Fn | `storage_sign_failed` | no blob to sign — check the Attachment's `status` first; this is **not** transient |
  | 503 | Edge Fn | `backend_unavailable` | transient; retry with backoff |
  | 400 | PostgREST | `22023` | invalid argument. **Read the message** — see §6 for the three distinct `22023` cases, or a Family guard (see §2) |
  | 400 | PostgREST | `23514` | a length CHECK failed (Idea `title` > 200, `p_summary` > 500) |
  | 403 | PostgREST | `42501` | not yours, or an ownership guard tripped |
  | 404 | PostgREST | `P0002` | not found / not yours / never existed — indistinguishable. Drop it from your queue |
  | 409 | PostgREST | `23503`, `23505` | referenced row not yours or nonexistent (`23503`); duplicate name, e.g. a Project (`23505`) |
- **Telemetry on the backend.** We log the resolved `user_id` and the proxied path. We do not log your bearer token, and we do not log request or response bodies, so the contents of Tasks, Comments and Ideas do not appear in our logs. They are of course stored in the KlickBox database, which is where the iPhone app reads them from; see the privacy policy for how that data is handled and how to delete it.

## 8. A minimum viable OpenClaw

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

## 9. Reference SDKs and smoke test

If you don't want to write the HTTP calls by hand, generate a client from the machine-readable contract at `./tools.json`, which is the file published alongside this guide — it carries every tool's method, path, body schema, and doc string. KlickBox maintains reference clients in TypeScript/Deno and Python as thin wrappers over the same contract, but they are **not publicly distributed**; the contract file is the thing to generate from.

One caveat if you use a typed client or generate one: `p_processed_through` must reach the server as the string you received. A deserializer that maps timestamp-shaped strings onto a native date type truncates the value before your code ever sees it, which is the silent stuck-Open failure described in section 6. The contract types that field as an opaque string for exactly this reason.

To sanity-check a fresh API key before you build against it, the cheapest end-to-end probe is a create-then-delete round trip:

```bash
BASE=https://oyarcsgekpltnxjmidqk.functions.supabase.co/api-key-auth

# Create a throwaway Task.
curl -sS "$BASE/rpc/create_task_with_tags" \
  -H "Authorization: Bearer $KLICKBOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"p_title":"probe: delete me","p_base_score":1}'
# -> {"id":"<uuid>", ...}

# Read it back, then remove it.
curl -sS "$BASE/tasks?id=eq.<uuid>" -H "Authorization: Bearer $KLICKBOX_API_KEY"
curl -sS -X DELETE "$BASE/tasks?id=eq.<uuid>" -H "Authorization: Bearer $KLICKBOX_API_KEY"
```

A `401` comes back as `missing_credentials`, `malformed_credentials`, or `invalid_credentials`; a revoked, an expired, and a never-existent key all produce the third, deliberately indistinguishably. Anything else reaching Postgres means auth is working.

For the Idea Bank specifically, the round trip worth rehearsing before you trust your loop is the **amend-while-processing race**: create an Idea, note its `content_updated_at`, append an Entry, then call `mark_idea_processed` with the *original* marker. The Idea must come back `needs_processing: true`. If it comes back `false`, your marker handling is wrong and you will lose amendments in production.

Prefix any probe rows with something recognizable so you can find and delete them afterwards.
