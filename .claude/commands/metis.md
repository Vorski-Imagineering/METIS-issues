# METIS Read API — connection skill

You are about to query the METIS Read API (`/api/v1/`). This is the AI-client endpoint.
It uses a two-step Bearer token auth flow. Mostly read-only; write endpoints are
`POST /api/v1/relationships/{relationship_id}/update` and
`POST /api/v1/memberships/{membership_id}/update`.

---

## Required environment

```
METIS_URL=https://app.the-gathering.earth
```

`METIS_API_KEY` is the shared login gate secret. Read it from the project `.env` file:

```bash
METIS_API_KEY=$(grep '^API_LOGIN_SECRET=' .env | cut -d= -f2-)
```

You also need `METIS_EMAIL` and `METIS_PASSWORD`:
- Try to read them from `.env` first (`METIS_EMAIL=` and `METIS_PASSWORD=` lines)
- If missing, ask the user
- After a **successful** login, write any credentials that were missing back to `.env` (append or update the relevant lines) so they are available next time

---

## Step 1 — Load credentials and log in

```bash
METIS_URL="https://app.the-gathering.earth"
METIS_API_KEY=$(grep '^API_LOGIN_SECRET=' .env | cut -d= -f2-)
METIS_EMAIL=$(grep '^METIS_EMAIL=' .env | cut -d= -f2-)
METIS_PASSWORD=$(grep '^METIS_PASSWORD=' .env | cut -d= -f2-)
```

If either `METIS_EMAIL` or `METIS_PASSWORD` is empty after the above, ask the user for the missing value(s) before proceeding.

After a successful login, persist any credential that was not already in `.env`:
```bash
# Only run for values that were missing — append to .env
echo "METIS_EMAIL=the-email" >> .env
echo "METIS_PASSWORD=the-password" >> .env
```

```bash
curl -s -X POST "${METIS_URL}/api/v1/auth/login" \
  -H "X-Metis-Api-Key: ${METIS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"email": "'"${METIS_EMAIL}"'", "password": "'"${METIS_PASSWORD}"'"}'
```

**Success (200):**
```json
{
  "token": "metis_agentic_<id>_<secret>",
  "token_type": "Bearer",
  "expires_at": "...",
  "expires_in_seconds": 86400,
  "person": {"id": 42, "name": "Alice", "description": "...", "photo_url": null, "actor_kind": "person", "contact": {}}
}
```

Store the `token` value. It is valid for 24 hours.

**Error codes:**
- `401` — wrong password or missing/invalid `X-Metis-Api-Key`
- `403` — account exists but has no linked METIS Person

---

## Step 2 — Make authenticated calls

All subsequent requests use:
```
Authorization: Bearer <token>
```

Helper (set once per session):
```bash
METIS_TOKEN="metis_agentic_..."
```

---

## Available endpoints

The authoritative, always-current endpoint list, parameter schemas, and response shapes
are served live by the app. Fetch them after logging in:

```bash
curl -s "${METIS_URL}/api/v1/openapi.json" \
  -H "Authorization: Bearer ${METIS_TOKEN}"
```

The Swagger UI is at `${METIS_URL}/api/v1/docs`.

---

### `POST /api/v1/auth/logout` — revoke token

```bash
curl -s -X POST "${METIS_URL}/api/v1/auth/logout" \
  -H "Authorization: Bearer ${METIS_TOKEN}"
```
**Response 200:** `{"revoked": true}`

---

### `GET /api/v1/auth/whoami` — current person

**Response 200:** `{"authenticated": true, "person": {"id": 42, "name": "Alice", ...}}`

---

### `GET /api/v1/search` — search people + holons together

| Param | Required | Default | Description |
|---|---|---|---|
| `q` | yes | — | Min 2 chars |
| `types` | no | `person,holon` | Comma-separated subset |
| `limit_per_type` | no | 20 | Max 50 per type |

Returns ranked results (name/channel hits ranked above description-only hits).

---

### `GET /api/v1/people` — search people by name

| Param | Required | Default | Description |
|---|---|---|---|
| `q` | yes | — | Case-insensitive name substring (min 1 char) |
| `limit` | no | 100 | Max 100 |
| `offset` | no | 0 | Increment by `limit` until `has_more` is `false` |

**Response 200:** `{query, limit, offset, count, has_more, items: [PersonPublic]}`

---

### `GET /api/v1/people/{person_id}` — get person by PK

Returns 404 if not found.

---

### `GET /api/v1/people/{person_id}/memberships` — list memberships for a person

| Param | Required | Default | Description |
|---|---|---|---|
| `limit` | no | 50 | Max 200 |

**Response 200:** `{count, items: [{membership_id, holon, journey_name, step_title, follow_up_after, responsible_person}]}`

`responsible_person` is a full PersonPublic object or `null`.

---

### `GET /api/v1/people/{person_id}/notes` — list notes on a person's memberships

| Param | Required | Default | Description |
|---|---|---|---|
| `limit` | no | 50 | Max 200 |

**Response 200:** `{count, limit, items: [{id, body, note_type, created_at, author_person}]}`

---

### `GET /api/v1/holons` — search holons

At least one of `q` or `type` must be provided.

| Param | Required | Description |
|---|---|---|
| `q` | no | Case-insensitive name substring |
| `type` | no | `organisation`, `domain`, `event`, or `camp` |
| `parent` | no | Filter by parent Holon PK |
| `limit` | no | Default 100, max 100 |
| `offset` | no | Increment by `limit` until `has_more` is `false` |

**Response 200:** `{query, type, parent, limit, offset, count, has_more, items: [HolonPublic]}`

---

### `GET /api/v1/holons/{holon_id}` — get holon by PK

Returns 404 if not found.

---

### `GET /api/v1/holons/by-slug/{slug}` — get holon by slug

Returns 404 if not found.

---

### `GET /api/v1/holons/{holon_id}/memberships` — list memberships in a holon

| Param | Required | Default | Description |
|---|---|---|---|
| `limit` | no | 50 | Max 200 |

**Response 200:** `{count, items: [{membership_id, person, journey_name, step_title, follow_up_after, responsible_person}]}`

---

### `GET /api/v1/holons/{holon_id}/relationships` — list holon relationships

Lists relationships where the holon appears as `from_holon` or `to_holon`.

| Param | Required | Default | Description |
|---|---|---|---|
| `limit` | no | 50 | Max 200 |

**Response 200:** `{count, items: [{relationship_id, from_holon, to_holon, journey_name, step_title, follow_up_after, responsible_person}]}`

---

### `GET /api/v1/holons/{holon_id}/notes` — list notes referencing a holon

Notes on the holon directly, via its memberships, or via its relationships; newest first.

| Param | Required | Default | Description |
|---|---|---|---|
| `limit` | no | 50 | Max 200 |

**Response 200:** `{count, limit, items: [{id, body, note_type, created_at, author_person}]}`

---

### `GET /api/v1/responsible` — unified follow-up worklist

Membership (person-side) and HolonRelationship (holon-side) follow-up assignments, ordered
by `follow_up_after` ascending (items with no date sort last).

`kind="person"` items carry `person`+`holon`; `kind="holon"` items carry `from_holon`+`to_holon`.
The `id` field means different things by kind: for `kind=person` it is the `membership_id`; for `kind=holon` it is the `relationship_id`.

| Param | Required | Default | Description |
|---|---|---|---|
| `responsible` | no | — | Filter by responsible Person PK |
| `type` | no | all | Comma-separated: `person`, `organisation`, `domain`, `event`, `camp`. `person` selects Membership items; holon types select HolonRelationship items. |
| `when` | no | none | Comma-separated: `overdue`, `today`, `future`. Omitted = no date filter (includes undated). When set, undated items are excluded. |
| `limit` | no | 100 | Max 100 |
| `offset` | no | 0 | Page offset |

**Response 200:**
```json
{
  "count": 2, "limit": 100, "offset": 0, "has_more": false,
  "items": [
    {
      "kind": "person", "id": 34, "follow_up_after": "2025-03-24",
      "journey_name": "Contact", "step_title": "Invited",
      "responsible_person": {"id": 7, "name": "Victor", ...},
      "person": {"id": 32, "name": "Alice", ...},
      "holon": {"id": 1, "name": "Global", ...}
    },
    {
      "kind": "holon", "id": 9, "follow_up_after": "2025-03-26",
      "journey_name": "Partnership", "step_title": "Negotiating",
      "responsible_person": null,
      "from_holon": {"id": 4, "name": "Summit Camp", ...},
      "to_holon": {"id": 11, "name": "Beta Inc", ...}
    }
  ]
}
```

`journey_name`, `step_title`, `follow_up_after`, and `responsible_person` are `null` when not set.

**Errors:** `400` if `type` or `when` contains an unrecognised value.

---

### `POST /api/v1/relationships/{relationship_id}/update` — update a holon relationship

Record a note + optional follow-up date / step change on a holon-to-holon relationship.
Only works for `kind=holon` items. Pass the `relationship_id` from `/responsible`. Do NOT
use with a `kind=person` `id` — it will return `404`.

**Request body:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `note` | string | yes | Stored verbatim (trimmed). Must be non-empty. |
| `follow_up_after` | date \| null | no | ISO `YYYY-MM-DD` to set, `null` to clear. Omit to leave unchanged. |
| `step_slug` | string | no | Active step slug on the relationship's journey. |
| `advance_step` | boolean | no | Move to the next active step. |

`step_slug` and `advance_step: true` are mutually exclusive.

**Response 200:** `{relationship, note, changes}` — `changes` reports `current_step` and/or `follow_up_after` as `{old, new}`. Empty when nothing changed.

**Errors:** `400` (empty note, malformed date, invalid step_slug, both step controls supplied, no next step), `403` permission denied, `404` not found.

---

### `POST /api/v1/memberships/{membership_id}/update` — update a person membership

Record a note + optional follow-up date / step change on a person membership.
Use the `membership_id` from `/responsible` (`kind=person` items) or `/people/{id}/memberships`.

**Request body:** same fields as the relationship update endpoint above.

**Additional behavior:** the note is attached to both the person's and the holon's note feeds.

**Response 200:** `{membership, note, changes}`

**Permissions:** a caller who can edit the membership's holon may update it.

**Errors:** `400` (empty note, malformed date, invalid step_slug, both step controls supplied, no next step), `403` permission denied, `404` not found.

---

## Error shape

All errors from `/api/v1/` return:
```json
{"code": "unauthenticated", "message": "Authentication required.", "retryable": false}
```

| code | HTTP | retryable |
|---|---|---|
| `unauthenticated` | 401 | false — re-login required |
| `permission_denied` | 403 | false |
| `not_found` | 404 | false |
| `validation_error` | 400 | false |
| `server_error` | 500 | **true** — safe to retry |

Framework validation errors (missing required params) return HTTP `422` with a different shape:
```json
{"detail": [{"type": "...", "loc": ["query", "q"], "msg": "..."}]}
```
Treat `400` and `422` both as non-retryable bad input.

---

## Access model

- **Reads:** any valid token can read every Person, Holon, Membership, relationship, and note. No per-record visibility check. Private fields are excluded at the projection level but note bodies are fully readable by any authenticated client.
- **Writes:** object-scoped. For relationships, the caller must be able to edit at least one holon side. For memberships, the caller must be able to edit the membership's holon.

---

## Public field projections

**PersonPublic:** `id`, `name`, `description`, `photo_url`, `actor_kind`, `contact`

**HolonPublic:** `id`, `name`, `slug`, `type`, `description`, `parent_id`, `logo_url`, `links`

`logo_url` is `null` when no logo is set. `contact` and `links` are returned to authenticated token holders.

---

## Scope notes

- No CORS — designed for server-side AI clients, not browser JS.
- List endpoints return `[]` when nothing matches (never 404).
- `GET` endpoints do not create or modify records.
