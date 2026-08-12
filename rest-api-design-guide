# REST API Design Guide

## URI Naming

- Use **nouns**, not verbs — HTTP method implies the action
  `GET /users` ✅ not `GET /getUsers` ❌
- Use **plural nouns** for collections: `/users`, `/users/{id}`
- **Lowercase** only, **hyphens** for multi-word paths (not underscores/camelCase)
  `/user-roles` ✅ not `/user_roles` or `/userRoles`
- **Forward slash** for hierarchy, no **trailing slash**
  `/users/{id}/orders` ✅ not `/users/{id}/orders/`
- No file extensions: `/items` not `/items.json` (use `Content-Type` header instead)
- Limit nesting to ~2 levels (`collection/item/collection`); flatten deeper relations
  `/orders/99/products` ✅ not `/customers/1/orders/99/products` ❌
- Avoid mirroring DB schema; avoid chatty APIs (too many small resources)
  - ❌ `/customer_addresses_tbl`, `/order_line_items_tbl` (raw table names)
  - ❌ separate calls `GET /users/1/name`, `GET /users/1/email`, `GET /users/1/address`
  - ✅ `GET /users/1` returns `{ "name", "email", "address" }` combined
- Non-resource actions (e.g. calculations): treat as pseudo-resource, use sparingly
  - `GET /calculator/add?operand1=99&operand2=1` → `{ "result": 100 }`
  - `POST /scripts/{id}/status` with `{ "action": "execute" }` instead of `POST /scripts/{id}/execute`

## HTTP Methods

| Verb | Collection (`/users`) | Item (`/users/1`) |
|---|---|---|
| GET | List all | Get one |
| POST | Create new | ❌ (405) |
| PUT | Bulk update | Replace (idempotent) |
| PATCH | — | Partial update |
| DELETE | Remove all | Remove one |

- **PUT** must be idempotent (same request → same result)
- **PATCH** uses JSON Merge Patch (`application/merge-patch+json`) or JSON Patch (`application/json-patch+json`)
- Server assigns URI on `POST`, never the client

**Response body by method:**

| Method | Body? |
|---|---|
| POST | Yes — return created resource (201) |
| PUT | Optional — 200 + resource, or 204 if omitted |
| PATCH | Yes — return updated resource (200) |
| DELETE | No — 204 No Content |

## Status Codes

| Code | Use |
|---|---|
| 200 OK | Success, has body |
| 201 Created | Resource created (POST/PUT) — include `Location` header |
| 202 Accepted | Async processing started |
| 204 No Content | Success, no body |
| 400 Bad Request | Invalid input |
| 404 Not Found | Resource doesn't exist |
| 405 Method Not Allowed | Verb not supported on this URI |
| 409 Conflict | State conflict |
| 415 Unsupported Media Type | Bad Content-Type |
| 500 | Server error |

## JSON Body Conventions

- Field naming: **camelCase** (or snake_case) — pick one, stay consistent
- Always `Content-Type: application/json`
- Error body shape:
```json
{
  "errorCode": "40001",
  "errorMessage": "Missing 'email' field",
  "errorContext": { "requestId": "abc123" },
  "timestamp": "2026-08-12T12:00:00Z"
}
```

## Query Parameters

```
GET /orders?page=5&size=10              # pagination
GET /orders?status=shipped&minCost=100  # filtering
GET /orders?sort=-createdAt             # sorting (- = desc)
GET /orders?fields=id,name              # field selection
```
- Set sane defaults (`limit=25`) and an upper cap to prevent abuse
- Validate `fields` server-side to avoid leaking restricted data

## Versioning

Path-based is most common and cache-friendly:
```
/api/v1/users
/api/v2/users
```
