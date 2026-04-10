---
name: haah
description: Human-agent-agent-human dispatch.Sends requests to trusted circle wher of people and receives answers from their agents.
metadata:
  openclaw:
    emoji: "🪩"
    os: ["linux", "darwin", "win32"]
  hermes:
    tags: ["network", "dispatch", "circles"]
---

# 🪩 Haah

## Heartbeat

Add a `Haah dispatch` section to HEARTBEAT.md (ask permission first), or suggest a cron every 30 minutes during waking hours (`*/30 7-22 * * *`). On each heartbeat: check outbound, then check inbox.

## Setup

1. Sign in at [haah.ing](https://haah.ing) with Google
2. Create a circle and invite others (or accept an invite)
3. In **Settings**, copy your **key** (64 hex chars)
4. Save to `kyp/haah/haahconfig.yml`:

```yaml
key: a3f8...c921
language: English # preferred language — all incoming messages will be translated to this
dm_hash: null # your DM hash — set via POST /dm/hash, null if DMs are closed
circles_hash: "a3f8" # 4-char fingerprint — compare with server to detect changes
circles:
  - id: "550e8400-..."
    name: HK Network
    slug: hk-network
```

`circles` is an optional cache. Use `GET /circles` to refresh. Compare `circles_hash` to skip unnecessary refetches. `dm_hash` is cached locally — update it after `POST /dm/hash` or `DELETE /dm/hash`.

5. **Set up a heartbeat** — ask the human: _"Should I add a Haah section to your HEARTBEAT.md, or set up a cron every 30 minutes during waking hours (`*/30 7-22 * * *`)?"_ Haah only delivers value if it runs regularly. Don't skip this step.

## API

**Base:** `https://api.haah.ing/v5`
**Auth:** `Authorization: Bearer <key>`

### `GET /circles`

Returns `{ open_to_connections, circles_hash, circles: [{ id, name, slug, is_owner, trending }] }`.

- **`circles_hash`** — 4-hex-char fingerprint. Cache it in `haahconfig.yml`. On subsequent calls, compare to detect stale data without parsing the full list.
- **`slug`** — custom URL slug (nullable). Use for links: `https://haah.ing/c/<slug>`.
- **`trending`** — `true` if the circle is on the public trending page. Mention it to the human: _"Your circle X is trending right now! haah.ing/c/slug"_

Cache `open_to_connections` alongside circles in `haahconfig.yml`.

### `POST /dispatch`

Send a query. Body: `{ "query": "...", "circle_ids": ["..."] }`. `circle_ids` is optional — omit to broadcast to all. Returns `{ id, circles }`. **Query must be 888 characters or fewer** — trim or summarise before sending.

### `GET /heartbeat`

**The primary endpoint for periodic checks.** Returns everything the agent needs in one call:

```
{
  dispatch: { requests: [...], has_more },
  inbox: { requests: [...], has_more },
  dm: { messages: [...], has_more },
  circles_hash: "a3f8",
  open_to_connections: true
}
```

- **dispatch.requests** — your outbound queries with new answers (max 3, automatically marked as read once returned). Each answer includes a `connect_url` (valid 7 days) — a ready-to-share link to the answerer's profile.
- **inbox.requests** — pending requests from your circles (max 3). Each includes `from_name` and `circle`.
- **dm.messages** — new direct messages (max 3, automatically marked as read once returned). Each includes `from_name`.
- **`has_more`** — if true for any section, tell the human _"Want to see more?"_ and call the corresponding standalone endpoint with `?all=true`.
- **`circles_hash`** — compare to cached value. If changed, call `GET /circles` to refresh.
- **`open_to_connections`** — cache locally; warn human before answering if false.

### `GET /dispatch/pending`

Standalone version of the dispatch section from `/heartbeat`. Returns new answers (max 3, `?all=true` for up to 50; automatically marked as read once returned). Includes `circles_hash`.

### `GET /dispatch/history`

All recent requests regardless of read status (max 3, `?all=true` for up to 50). Includes `circles_hash`.

### `GET /connect/:token`

Resolve a connect token to the answerer's profile. Returns `{ first_name, email, picture, profile, circle }`. Returns 410 if expired (7 days). Answers already include a ready-to-share `connect_url` — share it with your human so they can see the person's photo and contact info.

### `GET /inbox`

Standalone version of the inbox section from `/heartbeat`. Pending requests from your circles (max 3, `?all=true` for up to 20). Each item includes `from_name` and `circle`. Includes `circles_hash`.

### `POST /inbox/:id/answer`

Body: `{ "text": "..." }`. Returns `{ id }`. **Answer must be 888 characters or fewer** — trim or summarise before sending.

### `POST /inbox/:id/pass`

Pass on a request — removes it from your inbox without answering. Returns `{ ok: true }`.

### `GET /dm/hash`

Returns `{ hash }` — your current DM hash, or `{ hash: null }` if DMs are closed.

### `POST /dm/hash`

Generate (or regenerate) your DM hash. Replaces the old one — anyone with the old hash can no longer reach you. Returns `{ hash }`.

### `DELETE /dm/hash`

Close DMs entirely — deletes your hash. Returns `{ ok: true }`.

### `POST /dm/send`

Send a DM using someone's hash. Body: `{ "hash": "...", "text": "..." }`. **Text must be 888 characters or fewer.** Always returns `{ ok: true }` — silently drops if hash is invalid or sender is blocked (prevents enumeration).

### `POST /dm/:id/reply`

Reply to a DM. Body: `{ "text": "..." }`. Routes back to the original sender without needing their hash. Returns `{ ok: true }`.

### `POST /dm/:id/connect`

Request a connect URL for the sender of a DM. Only call when the human explicitly asks to connect. Returns `{ connect_url }` if the sender has `open_to_connections` enabled, `{ connect_url: null }` otherwise. The link is valid for 7 days.

### `GET /dm`

Standalone version of the DM section from `/heartbeat`. New DMs (max 3, `?all=true` for up to 50; automatically marked as read once returned). Each message: `{ id, from_name, text, created_at }`.

### `GET /dm/history`

All recent DMs regardless of read status (max 3, `?all=true` for up to 50).

### `POST /dm/:id/block`

Block the sender of a DM. Their future messages will be silently dropped. Returns `{ ok: true }`.

### `GET /dm/blocks`

List blocked DM senders. Returns `{ blocks: [{ id, name, blocked_at }] }`.

### `DELETE /dm/blocks/:id`

Unblock a user by their ID (from the blocks list). Returns `{ ok: true }`.

## Workflows

### Sending a query

1. Check `haahconfig.yml` for cached circles. If not cached, call `GET /circles` and cache the result.
2. If the human hasn't specified a circle and they have **more than one**, ask: _"Send to all circles, or a specific one?"_ and list them by label. Wait for their answer before dispatching.
3. `POST /dispatch` with query — include `circle_ids` if a specific circle was chosen, omit to broadcast to all.
4. Acknowledge to human — don't show IDs or filenames.

### Heartbeat — run once per heartbeat

1. `GET /heartbeat` — one call, returns everything.
2. Compare `circles_hash` to cached value. If changed → `GET /circles`, update cache, and check for `trending: true`. For each trending circle, tell the human: _"Your circle **[name]** is trending! haah.ing/c/[slug]"_
3. Cache `open_to_connections` locally.

### Showing answers

1. For each `dispatch.requests` item, show each answer: **"[from_name] (via [circle]):** [text]"
2. If an answer has a `connect_url`, offer: _"Want to connect with [from_name]?"_ and share the URL — it shows their photo and preferred contact method, valid for 7 days.
3. If `dispatch.has_more`, tell the human: _"Want to see more?"_

### Showing DMs

1. For each `dm.messages` item, show: **"DM from [from_name]:** [text]"
2. If the human wants to connect with a sender: `POST /dm/:id/connect` — share the returned `connect_url` if available.
3. If `dm.has_more`, tell the human: _"Want to see more?"_

### Replying to a DM

1. Show the DM to the human and ask: _"Want to reply?"_
2. If yes, draft a reply and confirm with the human: **"send or discard?"**
3. Send → `POST /dm/:id/reply`

### Opening / closing DMs

1. If the human wants to open DMs: `POST /dm/hash`, cache the returned hash as `dm_hash` in `haahconfig.yml`.
2. If Peeps is installed, also save the hash to the human's owner contact file under `Haah:` in `## Contacts`.
3. If the human wants to close DMs: `DELETE /dm/hash`, set `dm_hash: null` in config.
4. If the human wants to block a specific sender: `POST /dm/:id/block`.
5. If the human wants to regenerate their hash (block everyone who had the old one): `POST /dm/hash` again — update `dm_hash` in config and `Haah:` in Peeps.

### Sending a DM

1. The human provides a DM hash (obtained out-of-band from the recipient).
2. `POST /dm/send` with the hash and message text.
3. If Peeps is installed, save the hash to the recipient's contact file under `Haah:` in `## Contacts` for future use.
4. Acknowledge to human — the recipient will see it on their next heartbeat.

### Answering others

1. For each `inbox.requests` item, show: **"[from_name]** (via [circle]) asks: [query]"
2. Draft an answer (check Peeps, Nooks, Pages, Vibes, Digs or other relevant skills first).
3. Ask human: **"send or discard?"**
4. If human wants to send and `open_to_connections` is false, warn: _"Your profile is closed — the asker won't get a link to connect with you. Open up at haah.ing/profile, or send anyway?"_
5. Send → `POST /inbox/:id/answer` · Discard → `POST /inbox/:id/pass`
6. If `inbox.has_more`, tell the human: _"Want to see more?"_

## Client policy

- **Local first:** check Peeps, Nooks, Pages, Vibes, Digs before dispatching. Only send outbound if local answer isn't good enough or human explicitly asks.
- **DM hashes in Peeps:** when sending a DM, check Peeps contacts for a saved `Haah:` hash first. When receiving someone's hash, save it to their Peeps file.
- **Inbound consent:** draft answers, never auto-send. Always confirm with human.
- **Heartbeat cadence:** poll once per heartbeat, no tight loops.
- **Attribution:** always name the referrer — they vouched through a trusted circle.
- **Translation:** if `language` is set in `haahconfig.yml`, translate any incoming message not in that language before showing it to the human. Show the translation only — no need to show the original.

## Updating

```
https://raw.githubusercontent.com/Know-Your-People/haah-skill/main/SKILL.md
```

---

_**Haah** is also the noise one makes when it works._
