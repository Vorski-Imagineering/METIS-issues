# METIS Read API — connection skill

You are about to query the METIS Read API (`/api/v1/`). This is the AI-client endpoint.
It uses a two-step Bearer token auth flow. The surface is mostly read-only; the one write
endpoint is `POST /api/v1/relationships/{relationship_id}/update` (add a note + optional
follow-up/step change on an existing holon relationship).

---

## Required environment

```
METIS_URL=https://app.the-gathering.earth
```

`METIS_API_KEY` is the shared login gate secret. Read it from the project `.env` file:

```bash
METIS_API_KEY=$(grep '^API_LOGIN_SECRET=' .env | cut -d= -f2-)
```

You also need `METIS_EMAIL` and `METIS_PASSWORD` — ask the user if not provided. Do **not** guess or invent them.

---

## Step 1 — Load credentials and log in

```bash
METIS_URL="https://app.the-gathering.earth"
METIS_API_KEY=$(grep '^API_LOGIN_SECRET=' .env | cut -d= -f2-)

curl -s -X POST "${METIS_URL}/api/v1/auth/login" \
  -H "X-Metis-Api-Key: ${METIS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"email": "'"${METIS_EMAIL}"'", "password": "'"${METIS_PASSWORD}"'"}'
```

**Success (200):**
```json
{
  "token": "metis_read_<id>_<secret>",
  "token_type": "Bearer",
  "expires_at": "...",
  "expires_in_seconds": 86400,
  "person": { "id": 42, "name": "Alice", ... }
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
METIS_TOKEN="metis_read_..."
AUTH="-H 'Authorization: Bearer ${METIS_TOKEN}'"
```

---

## Available endpoints

The authoritative, always-current endpoint list, parameter schemas, and response shapes
are served live by the app. Fetch them after logging in:

```bash
curl -s "${METIS_URL}/api/v1/openapi.json" \
  -H "Authorization: Bearer ${METIS_TOKEN}"
```

Parse `paths` for endpoint definitions and `components.schemas` for response shapes.
The Swagger UI is at `${METIS_URL}/api/v1/docs`.

**Illustrative calls:**

```bash
# Search people
curl -s "${METIS_URL}/api/v1/people?q=alice" \
  -H "Authorization: Bearer ${METIS_TOKEN}"

# Get a holon by slug
curl -s "${METIS_URL}/api/v1/holons/by-slug/borderland-2026" \
  -H "Authorization: Bearer ${METIS_TOKEN}"

# Responsible worklist (overdue items for person 42)
curl -s "${METIS_URL}/api/v1/responsible?responsible=42&when=overdue" \
  -H "Authorization: Bearer ${METIS_TOKEN}"
```

Key behavioural notes not obvious from the schema:
- `/holons` requires at least one of `q` or `type`
- `/search` requires `q` ≥ 2 chars; `/people` requires `q` ≥ 1 char
- `/responsible` `type` param uses `person` for Membership items and holon-type names for HolonRelationship items
- `/responsible` `when` omitted = no date filter (includes undated); when set, undated items are excluded
- `kind=person` items in `/responsible` carry `person`+`holon`; `kind=holon` items carry `from_holon`+`to_holon`
- `POST /relationships/{id}/update` — the one write endpoint: requires `note` (non-empty); `step_slug` and `advance_step` are mutually exclusive

---

## Error shape

All errors from `/api/v1/` return:
```json
{ "code": "unauthenticated", "message": "Authentication required.", "retryable": false }
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
{ "detail": [{ "type": "...", "loc": ["query", "q"], "msg": "..." }] }
```
Treat `400` and `422` both as non-retryable bad input.
