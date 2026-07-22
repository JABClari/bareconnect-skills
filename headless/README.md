# Bareconnect Headless API

### Partner Management · Integration Guide · v1

The server-side API for building your own product on top of Bareconnect. You hold a **secret
key**, keep it on your server, and drive stores, catalogs, orders, and webhooks programmatically.
Bareconnect stays invisible behind your own app.

> **This is not the storefront API.** If you're building a browser frontend, use a **publishable**
> key and the storefront skill at [`skills/ai-engine-connect/`](../skills/ai-engine-connect). The
> secret key here can read and change everything — never put it in a browser, a prompt, or an AI
> build tool.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Base URL & Transport](#base-url--transport)
4. [Authentication](#authentication)
5. [Scopes](#scopes)
6. [Request Headers](#request-headers)
7. [Response Envelope](#response-envelope)
8. [Errors](#errors)
9. [Rate Limiting](#rate-limiting)
10. [Identity — `GET /me`](#identity)
11. [Stores](#stores)
12. [Products](#products)
13. [Orders](#orders)
14. [Webhooks](#webhooks)
15. [Webhook Event Reference](#webhook-event-reference)
16. [Troubleshooting](#troubleshooting)
17. [Security Best Practices](#security-best-practices)

---

## Overview

The Bareconnect Partner Management API gives your platform programmatic access to the merchants you have onboarded through Bareconnect. You can provision stores, read and write catalog data, manage order fulfilment, confirm payments via your own gateway (BYOP), and receive real-time events through webhooks — all without your merchants ever touching the Bareconnect dashboard.

**Who this is for:** developers building white-label commerce platforms, aggregator marketplaces, or headless storefronts on top of Bareconnect merchant infrastructure.

**Key design principles:**
- Every response includes a `request_id` for tracing — surfaced both in the JSON body and the `X-Request-Id` response header
- Every error uses a machine-readable `error_code` in SCREAMING_SNAKE_CASE, safe to `switch` on in your code
- Store isolation is enforced at the database query level — your key can never read or write a store that isn't linked to your partner account
- Rate limiting is per API key (not per IP), so separate keys for production and staging environments don't share budget

---

## Architecture

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │                      Your Partner Platform                           │
 │          (Aggregator · White-label · Headless Storefront)            │
 └──────────────────────────────┬───────────────────────────────────────┘
                                │  HTTPS only
                                │  Authorization: Bearer bc_mgmt_<key>
                                │  Content-Type: application/json
                                │  Accept: application/json
                                ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │            Bareconnect  /api/partner/v1                              │
 │                                                                      │
 │  ┌────────────────────────────────────────────────────────────────┐  │
 │  │              Auth & Rate-Limit Layer  (middleware)             │  │
 │  │                                                                │  │
 │  │  1  Extract Bearer token from Authorization header             │  │
 │  │  2  Look up the API key → verify not revoked                   │  │
 │  │  3  Load Partner → verify the account is active                │  │
 │  │  4  Rate limit: 60 req/min per key                              │  │
 │  │       key = partner_rate:{key.id}:{floor(time/60)}             │  │
 │  │       limit = 60 req/min  |  window = 60 s (calendar minute)   │  │
 │  │  5  Touch the key's last_used_at                              │  │
 │  │  6  Attach the partner key for downstream use                  │  │
 │  │  7  Log request (method, endpoint, status, ms, ip)             │  │
 │  │  8  Inject X-Request-Id + X-RateLimit-* on every response      │  │
 │  │  9  Normalise error shapes (inject error_code + request_id)    │  │
 │  └────────────────────────────────────────────────────────────────┘  │
 │                                                                      │
 │  ┌───────────────────────────┐  ┌──────────────────────────────┐    │
 │  │  Identity & Stores        │  │  Catalog & Orders            │    │
 │  │  GET  /me                 │  │  GET    /stores/{id}/products │    │
 │  │  GET  /stores             │  │  POST   /stores/{id}/products │    │
 │  │  GET  /stores/{id}        │  │  PATCH  /stores/{id}/         │    │
 │  │  POST /stores             │  │           products/{id}       │    │
 │  │  PATCH /stores/{id}       │  │  DELETE /stores/{id}/         │    │
 │  │  GET  /webhooks           │  │           products/{id}       │    │
 │  │  POST /webhooks           │  │  GET    /stores/{id}/orders   │    │
 │  │  DELETE /webhooks/{id}    │  │  GET    /stores/{id}/         │    │
 │  └───────────────────────────┘  │           orders/{id}         │    │
 │                                 │  PATCH  /stores/{id}/         │    │
 │                                 │           orders/{id}/fulfill  │    │
 │                                 │  PATCH  /stores/{id}/         │    │
 │                                 │           orders/{id}/deliver  │    │
 │                                 │  POST   /stores/{id}/         │    │
 │                                 │           orders/{id}/        │    │
 │                                 │           confirm-payment      │    │
 │                                 └──────────────────────────────┘    │
 └──────────────────────────────────────┬───────────────────────────────┘
                                        │
             ┌──────────────────────────┼──────────────────────────┐
             │                          │                          │
             ▼                          ▼                          ▼
    ┌────────────────┐       ┌────────────────┐       ┌────────────────┐
    │    stores      │       │   products     │       │    orders      │
    │                │       │                │       │                │
    │  ← scoped to   │       │                │       │                │
    │  your partner  │       └────────────────┘       └────────────────┘
    │  account       │
    └────────────────┘
             │
             │  Events fire → queued webhook delivery job
             │                        │
             │                        ▼
             │             ┌──────────────────────┐
             └─────────────│  Your Webhook        │
                           │  Endpoint (HTTPS)    │
                           └──────────────────────┘
```

### Store Isolation Model

Every partner API key belongs to a Partner. Every partner has linked stores. A store-access check runs on every store-scoped endpoint, confirming the requested store is linked to your partner account:

```
store is linked to partner_id = {your_partner_id} AND store_id = {requested_store_id}
```

If no such link exists, the endpoint returns `404 not_found` — regardless of whether the store exists. This is intentional: returning `403` would tell an attacker the store ID is valid. Returning `404` prevents UUID enumeration.

### Request Logging

Every request — regardless of success or failure — is recorded in the partner API request log:

```
partner_id | key_id | method | endpoint | status_code | duration_ms | ip | created_at
```

This powers per-partner API call charts, error rates, and top endpoints.

---

## Base URL & Transport

```
https://bareconnect.com/api/partner/v1
```

- All requests must be made over **HTTPS**. HTTP is not accepted.
- All request and response bodies are **JSON** (`Content-Type: application/json`, `Accept: application/json`).
- Responses always include `X-Request-Id` and `X-RateLimit-*` headers (see below).

---

## Authentication

Every request requires your API key in the `Authorization` header as a Bearer token:

```
Authorization: Bearer bc_mgmt_<your_key>
```

Your key is issued by the Bareconnect team. Contact **partners@bareconnect.com** to request a key, specifying the scopes your integration requires.

> **Keys are shown once.** Store your key immediately in an environment variable or a secrets manager. It cannot be retrieved after issuance — only revoked and replaced.

```bash
# Minimal example
curl https://bareconnect.com/api/partner/v1/me \
  -H "Authorization: Bearer bc_mgmt_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -H "Accept: application/json"
```

### Key prefix convention

All keys begin with `bc_mgmt_` followed by a random string. The prefix is stored for display purposes; the full key is hashed server-side and never stored in plaintext.

---

## Scopes

Each key is issued with a specific set of scopes. Calling an endpoint without the required scope returns `403 INSUFFICIENT_SCOPE`. Request only the scopes your integration actually needs — the principle of least privilege.

| Scope | What it grants |
|-------|----------------|
| `stores:read` | List and inspect merchant stores linked to your partner account |
| `stores:write` | Provision new stores and update store settings |
| `catalog:read` | Browse products across linked stores |
| `catalog:write` | Create, update, and delete products in linked stores |
| `orders:read` | View orders, fulfilment status, and transaction history (incl. per-order **profit**) |
| `orders:write` | Update fulfilment/delivery status **and import external orders** |
| `orders:confirm` | Confirm payment on an order via your own gateway (BYOP/Hybrid only) |
| `webhooks:manage` | Register, list, and remove webhook endpoints |
| `stitch:render` | Render Stitch server-side data-binding (issued for Stitch integrations; no endpoint in this API surface) |

> **Scope design recommendation:** issue separate keys per integration concern. A read-only analytics consumer should not hold `catalog:write`. A fulfilment worker does not need `stores:write`.

---

## Request Headers

| Header | Required | Value |
|--------|----------|-------|
| `Authorization` | ✅ | `Bearer bc_mgmt_<your_key>` |
| `Content-Type` | For POST/PATCH | `application/json` |
| `Accept` | Strongly recommended | `application/json` |

> **`Accept: application/json` is important.** Without it, Laravel's validation errors return an HTML redirect (302) rather than a JSON `422`. Always send this header.

---

## Response Envelope

### Success responses

```json
{
  "data": { ... }
}
```

List endpoints:

```json
{
  "data": [ ... ],
  "meta": {
    "count":       30,
    "has_more":    true,
    "next_cursor": "14d794d4-7281-465d-afe1-fc0ff5cdcf91"
  }
}
```

> Products and orders use **cursor pagination** (see [Cursor Pagination](#cursor-pagination)). The store list uses offset pagination (`total`, `per_page`, `current_page`, `last_page`).

### Error responses

```json
{
  "error":      "insufficient_scope",
  "error_code": "INSUFFICIENT_SCOPE",
  "message":    "This key does not have the required 'orders:write' scope.",
  "required":   "orders:write",
  "request_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
}
```

> `required` names the exact missing scope — useful for programmatic key-permission checks in your SDK.

The `X-Request-Id` response header carries the same UUID. Include it when contacting support.

---

## Errors

### Error code reference

| error_code | HTTP | When it fires |
|------------|------|---------------|
| `MISSING_CREDENTIALS` | 401 | No `Authorization` header, or header is not `Bearer ...` |
| `INVALID_KEY` | 401 | Key does not exist, has been revoked, or the hash does not match |
| `PARTNER_INACTIVE` | 403 | The partner account linked to this key has been deactivated |
| `INSUFFICIENT_SCOPE` | 403 | Key lacks the scope required by this endpoint |
| `NOT_FOUND` | 404 | Resource does not exist, or it exists but is not linked to your partner account |
| `ALREADY_PAID` | 409 | `confirm-payment` called on an order already marked as paid — response includes `order_id` |
| `ALREADY_FULFILLED` | 409 | `fulfill` called on an order already fulfilled — response includes `order_id` |
| `ALREADY_DELIVERED` | 409 | `deliver` called on an order already delivered — response includes `order_id` |
| `NOT_APPLICABLE` | 422 | `confirm-payment` called by a `bareconnect`-mode partner (BYOP does not apply) |
| `FREE_TIER_LIMIT_REACHED` | 422 | Product creation blocked — store is on the free tier and has hit the product cap |
| `UNSUPPORTED_COUNTRY` | 422 | `POST /stores` — Bareconnect could not resolve a supported currency for the given country |
| `VALIDATION_ERROR` | 422 | Request body failed field validation — see `errors` field for per-field detail |
| `RATE_LIMIT_EXCEEDED` | 429 | Key has made more than 60 requests in the current 60-second window |

### Validation error shape

```json
{
  "error":      "validation_error",
  "error_code": "VALIDATION_ERROR",
  "message":    "The given data was invalid.",
  "errors": {
    "price":  ["The price field is required."],
    "name":   ["The name may not be greater than 255 characters."]
  },
  "request_id": "uuid"
}
```

The `errors` object maps field names to arrays of failure messages. Fix all flagged fields before retrying.

---

## Rate Limiting

### Per-key, per-minute window

Rate limiting is enforced **per API key** — not per IP and not per partner account. This means:

- A partner with two keys (production + staging) gets **60 req/min each**, independently
- IP rotation does not help — the limit is on the key itself
- Two integrations sharing a key share a budget — issue separate keys per integration

**Window:** a 60-second calendar-minute window. The window resets at the next full minute boundary (not 60 seconds from your first request).

### Headers on every response

```
X-RateLimit-Limit:     60
X-RateLimit-Remaining: 54
X-RateLimit-Reset:     1778153520
```

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Maximum requests per minute for this key |
| `X-RateLimit-Remaining` | Requests remaining in the current window |
| `X-RateLimit-Reset` | Unix epoch timestamp when the window resets (next full minute) |

### 429 response

```json
{
  "error":      "rate_limit_exceeded",
  "error_code": "RATE_LIMIT_EXCEEDED",
  "message":    "Too Many Attempts.",
  "request_id": "uuid"
}
```

With headers:
```
X-RateLimit-Limit:     60
X-RateLimit-Remaining: 0
X-RateLimit-Reset:     1778153520
```

### Handling 429

Calculate when to retry from `X-RateLimit-Reset`:

```js
// Node.js example
if (response.status === 429) {
  const resetAt = parseInt(response.headers['x-ratelimit-reset'], 10);
  const waitMs  = (resetAt * 1000) - Date.now() + 100; // +100ms buffer
  await new Promise(resolve => setTimeout(resolve, waitMs));
  // retry
}
```

---

## Identity

### `GET /me`

Verify your key is active and inspect its metadata. No scope required.

```bash
curl https://bareconnect.com/api/partner/v1/me \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Accept: application/json"
```

**Response `200 OK`**

```json
{
  "partner": {
    "id":           "5fb814e2-a68d-4fde-a988-0cb820aef7aa",
    "name":         "Your Platform Name",
    "slug":         "your-platform",
    "payment_mode": "hybrid"
  },
  "key": {
    "name":         "Production Key",
    "prefix":       "bc_mgmt_...",
    "scopes":       ["stores:read", "catalog:read", "orders:read"],
    "last_used_at": "2026-05-07T14:59:09+00:00"
  }
}
```

`payment_mode` values:

| Value | Meaning |
|-------|---------|
| `bareconnect` | Bareconnect handles all payments. `orders:confirm` returns `422 NOT_APPLICABLE`. |
| `byop` | Your platform handles payment. Bareconnect checkout is hidden on your merchants' storefronts. |
| `hybrid` | Both rails active. Merchants can accept Bareconnect payments, or you can confirm externally via `orders:confirm`. |

---

## Stores

### `GET /stores`
**Scope:** `stores:read`

List all merchant stores linked to your partner account, newest first. Cursor-paginated.

**Query parameters**

| Param | Type | Description |
|-------|------|-------------|
| `limit` | integer | Results per page (default `20`, max `100`) |
| `after` | string | Cursor from `meta.next_cursor` for the next page |

**Response `200 OK`**

```json
{
  "data": [
    {
      "id":            "14d794d4-7281-465d-afe1-fc0ff5cdcf91",
      "name":          "Kofi's Kente",
      "domain":        "arc-kofis-kente",
      "domain_url":    "https://arc-kofis-kente.bareconnect.com",
      "country":       "Ghana",
      "commerce_mode": true,
      "created_at":    "2026-05-01T10:00:00+00:00"
    }
  ],
  "meta": {
    "count":       1,
    "has_more":    false,
    "next_cursor": null
  }
}
```

> **Pagination note:** Results are ordered oldest-first by `id` (ASC). Use `meta.next_cursor` as the `after` param to fetch the next page.

---

### `GET /stores/{store_id}`
**Scope:** `stores:read`

Get a single store by ID.

**Response `200 OK`** — same shape as a single `data` item from the list.

> **Isolation:** Requesting a store not linked to your account returns `404 NOT_FOUND`, indistinguishable from a non-existent store. This prevents UUID enumeration.

---

### `POST /stores`
**Scope:** `stores:write`

Provision a new merchant store under your partner account. Bareconnect runs the full provisioning sequence:

```
1. Validate request body
2. Derive domain: {3-char-partner-prefix}-{store-name-slug}
      prefix = first 3 chars of your partner slug (hyphens/underscores stripped)
      if domain is taken: append -1, -2, -3 ... until unique
3. Resolve currency from country name
4. Create the store record
      trial_ends_at = 2037-12-31  ← safe far-future within MySQL TIMESTAMP range
      show_branding = from your partner account setting
5. Create an internal system user
      Merchants never log into Bareconnect — it is headless from their perspective
6. Link the store to your partner account immediately
7. Create default storefront navigation
8. Initialise the store setup wizard
9. Dispatch Telegram notification (if configured)
10. Fire store.created webhook event
11. Return 201 with store details
```

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Merchant / store display name (max 255) |
| `country` | string | ✅ | Country name — determines currency (e.g. `"Ghana"` → GHS, `"Nigeria"` → NGN) |
| `category` | string | — | Store category (default: `"general"`) |
| `email` | email | — | Contact email. Auto-generated from the domain if omitted. |
| `phone` | string | — | Contact phone (max 30 chars) |

**Response `201 Created`**

```json
{
  "data": {
    "id":            "14d794d4-7281-465d-afe1-fc0ff5cdcf91",
    "name":          "Kofi's Kente",
    "domain":        "arc-kofis-kente",
    "domain_url":    "https://arc-kofis-kente.bareconnect.com",
    "country":       "Ghana",
    "currency":      "GHS",
    "commerce_mode": true,
    "created_at":    "2026-05-07T12:00:00+00:00"
  }
}
```

Store the returned `id` — this is the `store_id` for all subsequent catalog, orders, and fulfilment requests. **Always check the returned `currency`** — it's derived from `country` (see the table below).

> **`domain` vs `domain_url`:** `domain` is the bare slug (e.g. `arc-kofis-kente`) — use this as a stable identifier for lookups. `domain_url` is the ready-to-use HTTPS storefront URL. Both fields appear on every store object across all endpoints.

#### Country → currency

`currency` is derived from the `country` you send:

| Currency | Countries |
|----------|-----------|
| `GHS` | Ghana |
| `NGN` | Nigeria |
| `KES` | Kenya |
| `UGX` | Uganda |
| `ZAR` | South Africa, Namibia |
| `GBP` | United Kingdom, Gibraltar, Isle of Man, Jersey, Guernsey |
| `EUR` | Eurozone members (Germany, France, Italy, Spain, Netherlands, Ireland, …) |
| `USD` | **fallback** — any country not in the lists above |

**Unrecognised country → USD + a warning.** If your `country` isn't a supported market, the store is
**still created** (in USD) but the `201` response includes a `warnings` array so you can catch a typo
or a placeholder before going live:

```json
{
  "data": { "id": "…", "name": "Test Store", "country": "Atlantis", "currency": "USD", "…": "…" },
  "warnings": [
    {
      "code":    "country_defaulted_to_usd",
      "message": "The country \"Atlantis\" isn't one of Bareconnect's supported markets, so this store was created in USD. If that's not intended, delete it and recreate with a supported country."
    }
  ]
}
```
Send a country from the table for the right currency; use `DELETE /stores/{id}` to remove a mistaken one.

**Possible errors**

| error_code | When |
|------------|------|
| `INSUFFICIENT_SCOPE` | Key lacks `stores:write` |
| `UNSUPPORTED_COUNTRY` | The derived currency isn't configured on the platform (rare) |
| `VALIDATION_ERROR` | `name` or `country` missing, or `email` is not a valid email address |

---

### `PATCH /stores/{store_id}`
**Scope:** `stores:write`

Update settings on an existing merchant store. Send only the fields you want to change — all fields are optional.

**Request body**

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Store display name (max 255) |
| `general_email` | email | General contact email (max 255) |
| `support_email` | email | Support / customer-facing email (max 255) |
| `category` | string | Store category (max 100) |

**Response `200 OK`** — updated store object:

```json
{
  "data": {
    "id":            "14d794d4-7281-465d-afe1-fc0ff5cdcf91",
    "name":          "Kofi's Kente Updated",
    "domain":        "arc-kofis-kente",
    "domain_url":    "https://arc-kofis-kente.bareconnect.com",
    "country":       "Ghana",
    "currency":      "GHS",
    "commerce_mode": true,
    "created_at":    "2026-05-01T10:00:00+00:00"
  }
}
```

**Behaviour notes:**
- Sending an empty body (or a body with no recognised fields) returns `200` with the current store data — it is a no-op, not an error.
- Fires a `store.updated` webhook event after a successful update.

---

### `DELETE /stores/{store_id}`
**Scope:** `stores:write`

Delete a store you provisioned. This is a **soft delete** — the store is deactivated and hidden
(it drops out of `GET /stores`), but its data is retained on Bareconnect's side and can be restored
on request. It is never hard-deleted via the API.

**Safety rail:** if the store has **paid, unfulfilled orders**, the request is rejected with `409`
so a live merchant can't be wiped by accident. Fulfil or refund those orders first, or override with
`?force=true`.

**Query parameters**

| Param | Type | Description |
|-------|------|-------------|
| `force` | boolean | `true` deletes even when paid, unfulfilled orders exist. Default `false`. |

**Response `200 OK`**

```json
{
  "data": {
    "id":      "14d794d4-7281-465d-afe1-fc0ff5cdcf91",
    "deleted": true
  }
}
```

**Response `409 Conflict`** — blocked by the safety rail:

```json
{
  "error":       "store_has_open_orders",
  "error_code":  "STORE_HAS_OPEN_ORDERS",
  "message":     "This store has 3 paid, unfulfilled order(s). Fulfil or refund them first, or retry with ?force=true.",
  "open_orders": 3,
  "request_id":  "…"
}
```

**Behaviour notes:**
- Only your own stores are deletable; another partner's store returns `404`.
- Deleting an already-deleted (or unknown) store returns `404`.
- Fires a `store.deleted` webhook event after a successful delete.

**Example**

```bash
curl -X DELETE https://bareconnect.com/api/partner/v1/stores/$STORE_ID \
  -H "Authorization: Bearer $BC_KEY"

# override the safety rail:
curl -X DELETE "https://bareconnect.com/api/partner/v1/stores/$STORE_ID?force=true" \
  -H "Authorization: Bearer $BC_KEY"
```

---

## Products

### Cursor Pagination

Products use cursor-based pagination. Unlike offset pagination, cursor pagination is stable — items added or deleted while you are paging do not cause duplicates or skipped records.

**Query parameters**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | integer | 30 | Items per page (max: 100) |
| `after` | string | — | Cursor from the previous page's `meta.next_cursor` |

**First page:**
```
GET /stores/{id}/products?limit=50
```

**Subsequent pages:**
```
GET /stores/{id}/products?limit=50&after={meta.next_cursor}
```

**Response meta shape:**
```json
{
  "meta": {
    "count":       50,
    "has_more":    true,
    "next_cursor": "7a2799bb-0001-0002-0003-000000000001"
  }
}
```

When `has_more` is `false`, `next_cursor` is `null` and you have reached the last page. The ordering is stable (`id ASC` — UUID lexicographic order, consistent across requests).

---

### `GET /stores/{store_id}/products`
**Scope:** `catalog:read`

List products for a merchant store.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id":            "uuid",
      "store_id":      "uuid",
      "status":        "published",
      "url":           "ankara-tote-bag",
      "type":          "physical",
      "name":          "Ankara Tote Bag",
      "description":   "Handmade from premium ankara fabric.",
      "price":         45.00,
      "compare_price": 60.00,
      "sku":           "ATB-001",
      "quantity":      12,
      "created_at":    "2026-05-01T10:00:00+00:00",
      "updated_at":    "2026-05-01T10:00:00+00:00"
    }
  ],
  "meta": {
    "count":       1,
    "has_more":    false,
    "next_cursor": null
  }
}
```

`status` is `"published"` or `"draft"`.

---

### `POST /stores/{store_id}/products`
**Scope:** `catalog:write`

Create a new product. Currently supports physical products only.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Product name (max 255) |
| `price` | number | ✅ | Selling price in the store's default currency |
| `description` | string | — | Product description (max 5000) |
| `sku` | string | — | Stock keeping unit (max 100) |
| `quantity` | integer | — | Stock count (default: 0) |
| `compare_price` | number | — | Original / compare-at price |
| `track_inventory` | boolean | — | Decrement stock on orders (default: false) |
| `published` | boolean | — | Publish immediately (default: false — creates as draft) |

**Response `201 Created`** — the created product object (same shape as list items).

**UCP auto-sync note:** If your partner account has `auto_sync_ucp = true`, every product created here is automatically listed on UCP (`ucp_channel = true`) so AI agents / LLMs can discover it. It only becomes *visible* on UCP once it is also published (`published: true`) — the flag controls listing eligibility, publishing controls visibility (catalog cached ~5 min). When the flag is off, products are created un-listed. The flag is managed by the Bareconnect team on your partner record. Products can then be fetched directly via `GET /api/ucp/v1/products/{ucp_id}`, and prices are reported in each product's own currency.

**Free tier note:** If your partner account has `free_plan_enabled = true`, product creation is blocked once a store reaches the `free_tier_products` limit. The response is:

```json
{
  "error":      "free_tier_limit_reached",
  "error_code": "FREE_TIER_LIMIT_REACHED",
  "message":    "This store has reached the free plan product limit (5). Upgrade the store's plan to add more products.",
  "limit":      5,
  "current":    5
}
```

---

### `PATCH /stores/{store_id}/products/{product_id}`
**Scope:** `catalog:write`

Update an existing product. Send only the fields you want to change — all fields are optional.

**Request body** — same fields as create (all `sometimes` instead of `required`).

**Response `200 OK`** — the updated product object.

Fires a `product.updated` webhook event.

---

### `DELETE /stores/{store_id}/products/{product_id}`
**Scope:** `catalog:write`

Soft-delete a product. The product is removed from the storefront immediately but the record is preserved for order history. Returns `404 NOT_FOUND` if the product does not exist or belongs to an unlinked store.

**Response `200 OK`**

```json
{
  "success": true,
  "message": "Product deleted."
}
```

Fires a `product.deleted` webhook event.

---

## Orders

Orders also use cursor-based pagination — the same `after` + `limit` pattern as products. See [Cursor Pagination](#cursor-pagination).

### `GET /stores/{store_id}/orders`
**Scope:** `orders:read`

List orders for a merchant store.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id":                      "uuid",
      "store_id":                "uuid",
      "store_order_id":          "BC-0042",
      "order_id":                "uuid",
      "price":                   90.00,
      "price_in_store_currency": 90.00,
      "cart_currency":           "GHS",
      "paid":                    true,
      "fulfilled":               null,
      "delivered":               null,
      "is_manual_payment":       false,
      "notes":                   null,
      "created_at":              "2026-05-06T09:00:00+00:00",
      "updated_at":              "2026-05-06T09:00:00+00:00"
    }
  ],
  "meta": {
    "count":       30,
    "has_more":    true,
    "next_cursor": "uuid"
  }
}
```

> ⚠️ On the **list** endpoint, `fulfilled` and `delivered` are **ISO 8601 timestamp strings or `null`** (the time the state was set), **not booleans**. The single-order **detail** endpoint below returns them as booleans plus separate `_at` timestamps — the two shapes differ, so don't assume the list gives booleans.

---

### `GET /stores/{store_id}/orders/{order_id}`
**Scope:** `orders:read`

Get a single order by ID.

**Response `200 OK`**

```json
{
  "data": {
    "id":                      "uuid",
    "store_id":                "uuid",
    "store_order_id":          "BC-0042",
    "order_id":                "uuid",
    "price":                   90.00,
    "price_in_store_currency": 90.00,
    "cart_currency":           "GHS",
    "paid":                    true,
    "fulfilled":               true,
    "fulfilled_at":            "2026-05-06T12:00:00+00:00",
    "delivered":               false,
    "delivered_at":            null,
    "is_manual_payment":       false,
    "notes":                   null,
    "profit": {
      "tracked":            true,
      "amount":             38.50,
      "margin_percent":     42.8,
      "revenue":            90.00,
      "cost":               51.50,
      "discount":           0.00,
      "delivery":           15.00,
      "currency":           "GHS",
      "missing_cost_lines": 0
    },
    "created_at":              "2026-05-06T09:00:00+00:00",
    "updated_at":              "2026-05-06T12:00:00+00:00"
  }
}
```

`fulfilled` and `delivered` are booleans. The `_at` fields hold the ISO 8601 timestamp when the state was set, or `null` if not yet in that state.

**The `profit` block** is computed by the same engine every Bareconnect surface uses:

- `tracked` — `true` only when **every** sold line has a recorded cost (`cost_at_sale`, snapshotted from the product's *cost per item* at sale time).
- `amount` — profit = `revenue − cost − discount`. **`null` when `tracked` is `false`** (some line has no recorded cost). It is deliberately **never `0`** for the untracked case — a partner must be able to tell *"no margin"* apart from *"cost not recorded"*. Check `tracked` before trusting `amount`.
- `delivery` — the delivery amount the customer paid. **Shown for context, NOT subtracted from profit** (the merchant priced the zone; we don't model their courier cost). Reported, never invented.
- `missing_cost_lines` — how many lines lacked a cost. `> 0` explains why `tracked` is `false`.

---

### `PATCH /stores/{store_id}/orders/{order_id}/fulfill`
**Scope:** `orders:write`

Mark an order as fulfilled (items packed / handed to courier).

**Response `200 OK`**

```json
{
  "success":      true,
  "message":      "Order marked as fulfilled.",
  "order_id":     "uuid",
  "fulfilled_at": "2026-05-06T12:00:00+00:00"
}
```

Returns `409 ALREADY_FULFILLED` if already fulfilled — body includes `order_id` so you can correlate without re-parsing the URL. Fires `order.fulfilled` on success.

---

### `PATCH /stores/{store_id}/orders/{order_id}/deliver`
**Scope:** `orders:write`

Mark an order as delivered (customer has received the item).

**Response `200 OK`**

```json
{
  "success":      true,
  "message":      "Order marked as delivered.",
  "order_id":     "uuid",
  "delivered_at": "2026-05-06T14:00:00+00:00"
}
```

Returns `409 ALREADY_DELIVERED` if already delivered — body includes `order_id`. Fires `order.delivered` on success.

---

### `POST /stores/{store_id}/orders/import`
**Scope:** `orders:write`

Import an order that originated **outside Bareconnect** — e.g. a sale from the partner's own website, marketplace, or POS — so it lives alongside native orders. Idempotent: keyed on `(partner, external_reference)`.

**Request body**

| Field | Rules | Notes |
|-------|-------|-------|
| `external_reference` | **required**, string, ≤255 | Your ID for this order. The idempotency key. |
| `source` | string, ≤50 | Where it came from (e.g. `"shopify"`, `"pos"`). |
| `order_type` | `order` \| `appointment` | Defaults to `order`. |
| `total` | **required**, numeric ≥ 0 | |
| `currency` | **required**, string ≤10 | Stored upper-cased. |
| `status` | string, ≤50 | Your own status label. |
| `payment_status` | string, ≤50 | |
| `customer.name` | string, ≤255 | |
| `customer.email` | email, ≤255 | |
| `customer.phone` | string, ≤50 | |
| `line_items[].name` | required with `line_items`, ≤255 | |
| `line_items[].quantity` | numeric ≥ 0 | |
| `line_items[].unit_price` | numeric ≥ 0 | |
| `line_items[].total_price` | numeric ≥ 0 | |
| `metadata` | object | Free-form. |
| `occurred_at` | date | Defaults to now. |

**Response** — `201 Created` when new, **`200 OK` when an existing record with the same `external_reference` was updated.**

```json
{
  "data": {
    "id":                 "uuid",
    "store_id":           "uuid",
    "external_reference": "SHOP-1042",
    "source":             "shopify",
    "order_type":         "order",
    "total":              120.00,
    "currency":           "GHS",
    "status":             "processing",
    "payment_status":     "paid",
    "customer":           { "name": "Ama", "email": "ama@example.com", "phone": "+233..." },
    "line_items":         [ { "name": "Kente scarf", "quantity": 1, "unit_price": 120, "total_price": 120 } ],
    "metadata":           {},
    "occurred_at":        "2026-05-06T09:00:00+00:00",
    "created_at":         "2026-05-06T09:05:00+00:00"
  }
}
```

Fires the **`order.imported`** webhook (payload includes `external_reference`, `total`, `currency`, and `is_new`). Subscribe to it like any other event.

---

### `GET /stores/{store_id}/external-orders`
**Scope:** `orders:read`

Read back the orders you've imported into a store — the way to confirm what landed. Cursor-paginated
(`after` + `limit`, `limit` max 100).

**Response `200 OK`**

```json
{
  "data": [
    {
      "id":                 "uuid",
      "store_id":           "uuid",
      "external_reference": "SHOP-1042",
      "source":             "shopify",
      "order_type":         "order",
      "total":              120.00,
      "currency":           "GHS",
      "status":             "completed",
      "payment_status":     "paid",
      "customer":           { "name": "Ama", "email": "ama@example.com", "phone": "+233..." },
      "line_items":         [ { "name": "Kente scarf", "quantity": 1, "unit_price": 120, "total_price": 120 } ],
      "metadata":           {},
      "occurred_at":        "2026-05-06T09:00:00+00:00",
      "created_at":         "2026-05-06T09:05:00+00:00"
    }
  ],
  "meta": { "count": 1, "has_more": false, "next_cursor": null }
}
```

### `GET /stores/{store_id}/external-orders/{external_reference}`
**Scope:** `orders:read`

Fetch a single imported order by **your own** `external_reference` — the direct way to confirm a
specific import persisted (imports are idempotent per `external_reference`, so this is 1:1).

**Response `200 OK`** — the same object shape as above under `data`. Unknown reference → `404`.

```bash
curl https://bareconnect.com/api/partner/v1/stores/$STORE_ID/external-orders/SHOP-1042 \
  -H "Authorization: Bearer $BC_KEY"
```

---

### `POST /stores/{store_id}/orders/{order_id}/confirm-payment`
**Scope:** `orders:confirm` · **BYOP and Hybrid partners only**

Confirm that payment has been received through your own payment gateway. Marks the order as `paid` on Bareconnect.

> Returns `422 NOT_APPLICABLE` if your `payment_mode` is `bareconnect`. Check `GET /me` first if you are unsure of your mode.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reference` | string | ✅ | Your payment gateway's transaction reference (max 255) |
| `amount_confirmed` | number | ✅ | Amount your gateway received |
| `currency` | string | ✅ | ISO 4217 currency code (e.g. `GHS`, `NGN`, `USD`) |
| `note` | string | — | Optional internal note (max 500) |

**Response `200 OK`**

```json
{
  "success":  true,
  "message":  "Payment confirmed. Order marked as paid.",
  "order_id": "uuid",
  "paid_at":  "2026-05-06T12:30:00+00:00"
}
```

Returns `409 ALREADY_PAID` if the order is already paid — body includes `order_id`. Fires `order.paid` on success.

---

## Webhooks

Webhooks let Bareconnect notify your platform in real time when events occur. Instead of polling, you register an HTTPS endpoint and receive `POST` requests whenever subscribed events fire.

### `GET /webhooks`
**Scope:** `webhooks:manage`

List all registered webhook endpoints for your partner account.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id":         "uuid",
      "url":        "https://yourplatform.com/webhooks/bareconnect",
      "events":     ["order.paid", "order.fulfilled"],
      "active":     true,
      "created_at": "2026-05-06T10:00:00+00:00"
    }
  ]
}
```

---

### `POST /webhooks`
**Scope:** `webhooks:manage`

Register a new webhook endpoint.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | ✅ | Your HTTPS endpoint URL (max 500) |
| `events` | array | ✅ | One or more event names to subscribe to (see [Event Reference](#webhook-event-reference)) |

```json
{
  "url":    "https://yourplatform.com/hooks/bareconnect",
  "events": ["order.paid", "order.fulfilled", "order.delivered", "store.created"]
}
```

**Response `201 Created`**

```json
{
  "data": {
    "id":         "uuid",
    "url":        "https://yourplatform.com/hooks/bareconnect",
    "events":     ["order.paid", "order.fulfilled"],
    "active":     true,
    "secret":     "whsec_...",
    "created_at": "2026-05-06T10:00:00+00:00"
  },
  "notice": "Save the secret now — it will not be shown again. Use it to verify X-Bareconnect-Signature headers on incoming deliveries."
}
```

> **The `secret` is shown once.** Store it in your environment configuration immediately. You use it to verify the `X-Bareconnect-Signature` header on every incoming delivery.

---

### `DELETE /webhooks/{webhook_id}`
**Scope:** `webhooks:manage`

Remove a webhook endpoint. Deliveries already in flight will still complete.

**Response `200 OK`**

```json
{ "success": true, "message": "Webhook removed." }
```

---

## Receiving and Verifying Webhooks

### Delivery format

Bareconnect sends a `POST` request to your endpoint:

```
POST https://yourplatform.com/hooks/bareconnect
Content-Type:              application/json
X-Bareconnect-Event:       order.paid
X-Bareconnect-Delivery:    <unique-delivery-uuid>
X-Bareconnect-Signature:   sha256=<hmac-hex>
```

### Verifying signatures

Always verify the `X-Bareconnect-Signature` before processing. This confirms the payload came from Bareconnect and was not tampered with in transit.

**Signature spec (exact):**

| Property | Value |
|----------|-------|
| Algorithm | **HMAC-SHA256** |
| Key | your webhook `secret` (returned once from `POST /webhooks`) |
| Signed content | the **raw request body bytes** — the JSON exactly as received. **Nothing else is concatenated:** no timestamp, no delivery id, no event name. |
| Encoding | lowercase **hex** (not base64) |
| Header | `X-Bareconnect-Signature: sha256=<hex>` — note the literal `sha256=` prefix |
| Timestamp | **none** — there is no `t=` element and no timestamped signing scheme |

So: your assumption is correct — it's **HMAC-SHA256, hex, over the raw body**, just with a `sha256=` prefix on the header value.

```
expected = "sha256=" + HMAC-SHA256(raw_request_body, your_webhook_secret)   # hex digest
verify   = timing_safe_equals(expected, X-Bareconnect-Signature header)
```

**Node.js**
```js
const crypto = require('crypto');

function verifySignature(rawBody, signatureHeader, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(rawBody)       // raw bytes — not parsed JSON
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signatureHeader)
  );
}
```

**Python**
```python
import hmac, hashlib

def verify_signature(raw_body: bytes, signature_header: str, secret: str) -> bool:
    expected = 'sha256=' + hmac.new(
        secret.encode(), raw_body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

**PHP**
```php
function verifySignature(string $rawBody, string $signatureHeader, string $secret): bool {
    $expected = 'sha256=' . hash_hmac('sha256', $rawBody, $secret);
    return hash_equals($expected, $signatureHeader);
}
```

> Use the **raw request body bytes** — not the parsed/re-serialised JSON object. Re-serialisation can alter whitespace or key ordering, breaking the HMAC.

### Endpoint requirements

- Return any `2xx` within **10 seconds** or the delivery is considered failed
- Return `2xx` even if you are still processing asynchronously — acknowledge immediately, queue the work
- Handle deliveries **idempotently** — use `X-Bareconnect-Delivery` to deduplicate retries
- Do **not** rely on delivery order — events may arrive out of sequence under high load

### Retry policy

Failed deliveries (non-2xx, timeout, or connection refused) are retried with exponential backoff:

| Attempt | Delay |
|---------|-------|
| 1st retry | 5 minutes |
| 2nd retry | 30 minutes |
| 3rd retry | 2 hours |

After 3 failed attempts the delivery is permanently abandoned.

---

## Webhook Delivery Log

Every delivery attempt — successful or not — is recorded in Bareconnect's delivery log. This gives you full visibility into what was sent, when, how many attempts were made, and what your endpoint returned.

### `GET /webhooks/{webhook_id}/deliveries`
**Scope:** `webhooks:manage`

List past delivery attempts for a specific webhook endpoint, newest first.

**Query parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status: `delivered`, `failed`, or `pending`. Omit for all. |
| `limit` | integer | Results per page (default `20`, max `100`) |
| `after` | string | Cursor for the next page (from `meta.next_cursor`) |

**Response `200 OK`**

```json
{
  "data": [
    {
      "id":            "uuid",
      "delivery_id":   "abc123-...",
      "event":         "order.paid",
      "status":        "delivered",
      "http_status":   200,
      "attempts":      1,
      "response_body": "{\"ok\":true}",
      "delivered_at":  "2026-05-07T17:04:16+00:00",
      "failed_at":     null,
      "created_at":    "2026-05-07T17:04:10+00:00"
    },
    {
      "id":            "uuid",
      "delivery_id":   "def456-...",
      "event":         "store.created",
      "status":        "failed",
      "http_status":   503,
      "attempts":      3,
      "response_body": "Service Unavailable",
      "delivered_at":  null,
      "failed_at":     "2026-05-07T19:10:00+00:00",
      "created_at":    "2026-05-07T17:05:00+00:00"
    }
  ],
  "meta": {
    "has_more":    false,
    "next_cursor": null
  }
}
```

### Delivery status reference

| Status | Meaning |
|--------|---------|
| `pending` | Delivery is in flight or queued for a retry — not yet resolved |
| `delivered` | Your endpoint returned `2xx` within 10 seconds |
| `failed` | All retry attempts exhausted; delivery permanently abandoned |

### Fields explained

| Field | Description |
|-------|-------------|
| `delivery_id` | Stable UUID for this delivery — matches the `X-Bareconnect-Delivery` header sent to your endpoint. Use this to deduplicate |
| `event` | The event name, e.g. `order.paid` |
| `http_status` | The HTTP status code your endpoint returned (null if connection refused or timed out) |
| `attempts` | Number of delivery attempts made (1–3) |
| `response_body` | First 2 000 characters of your endpoint's response body, for debugging |
| `delivered_at` | ISO 8601 timestamp of the successful delivery |
| `failed_at` | ISO 8601 timestamp when the final attempt was permanently abandoned |

> **Tip:** Cross-reference `delivery_id` with the `X-Bareconnect-Delivery` header in your own request logs to correlate exactly which Bareconnect dispatch maps to which entry in your server log.

---

## Webhook Event Reference

### All valid events (11 total)

| Event | Fires when |
|-------|-----------|
| `store.created` | A new merchant store is provisioned via `POST /stores` |
| `store.updated` | A store's settings are updated via `PATCH /stores/{id}` |
| `store.deleted` | A store is soft-deleted via `DELETE /stores/{id}` |
| `order.created` | A new order is placed in a linked store |
| `order.paid` | An order is marked as paid (including via `confirm-payment`) |
| `order.fulfilled` | An order is marked as fulfilled via `PATCH .../fulfill` |
| `order.delivered` | An order is marked as delivered via `PATCH .../deliver` |
| `order.imported` | An external order is imported via `POST .../orders/import` |
| `product.created` | A product is created via `POST /stores/{id}/products` |
| `product.updated` | A product is updated via `PATCH /stores/{id}/products/{id}` |
| `product.deleted` | A product is soft-deleted via `DELETE /stores/{id}/products/{id}` |

### Payload shapes

**`store.created`**
```json
{
  "event":      "store.created",
  "store_id":   "uuid",
  "store_name": "Kofi's Kente",
  "domain":     "arc-kofis-kente",
  "country":    "Ghana",
  "created_at": "2026-05-07T12:00:00+00:00"
}
```

**`store.deleted`**
```json
{
  "event":      "store.deleted",
  "store_id":   "uuid",
  "store_name": "Kofi's Kente",
  "domain":     "arc-kofis-kente",
  "deleted_at": "2026-05-07T12:00:00+00:00"
}
```

**`store.updated`**
```json
{
  "event":      "store.updated",
  "store_id":   "uuid",
  "store_name": "Kofi's Kente",
  "updated_at": "2026-05-07T14:30:00+00:00"
}
```

**`order.created`**
```json
{
  "event":          "order.created",
  "order_id":       "uuid",
  "store_id":       "uuid",
  "store_order_id": "BC-0042",
  "price":          90.00,
  "currency":       "GHS",
  "paid":           true,
  "created_at":     "2026-05-07T12:00:00+00:00"
}
```

> Fires from the native Bareconnect checkout flow for all stores owned by a partner. For BYOP orders, use the `order.paid` event — partners already control when the order is created in their own system.

**`order.paid`**
```json
{
  "event":          "order.paid",
  "order_id":       "uuid",
  "store_id":       "uuid",
  "store_order_id": "BC-0042",
  "reference":      "TXN-20260507-9913",
  "amount":         90.00,
  "currency":       "GHS",
  "method":         "native",
  "paid_at":        "2026-05-07T12:30:00+00:00"
}
```

> `method` is `"native"` for Bareconnect-native checkout payments, `"byop"` for partner-confirmed BYOP payments (via `POST .../confirm-payment`).

**`order.fulfilled`**
```json
{
  "event":          "order.fulfilled",
  "order_id":       "uuid",
  "store_id":       "uuid",
  "store_order_id": "BC-0042",
  "fulfilled_at":   "2026-05-07T12:00:00+00:00"
}
```

**`order.delivered`**
```json
{
  "event":          "order.delivered",
  "order_id":       "uuid",
  "store_id":       "uuid",
  "store_order_id": "BC-0042",
  "delivered_at":   "2026-05-07T14:00:00+00:00"
}
```

**`product.created` / `product.updated`**
```json
{
  "event":      "product.created",
  "store_id":   "uuid",
  "product_id": "uuid",
  "name":       "Ankara Tote Bag",
  "created_at": "2026-05-07T10:00:00+00:00"
}
```

**`product.deleted`**
```json
{
  "event":      "product.deleted",
  "store_id":   "uuid",
  "product_id": "uuid",
  "deleted_at": "2026-05-07T11:00:00+00:00"
}
```

---

## Troubleshooting

### Common errors and what to do

---

**"I get HTML instead of a JSON error"**

You are missing `Accept: application/json`. Laravel returns an HTML redirect (302) for validation errors when this header is absent.

```bash
# Wrong
curl -X POST .../stores -d '{}'

# Correct
curl -X POST .../stores \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

**"401 MISSING_CREDENTIALS — I sent the Authorization header"**

The header must be exactly:
```
Authorization: Bearer bc_mgmt_<key>
```
Note the space after `Bearer`. A common mistake is `Bearer:` or using a different casing. Some HTTP clients silently strip the `Authorization` header on redirects — ensure you are hitting the HTTPS endpoint directly.

---

**"401 INVALID_KEY — my key was working yesterday"**

Keys are revoked immediately when a partner account is deleted or when the Bareconnect team rotates credentials. Contact partners@bareconnect.com. Do not hard-code keys in source code — always read from environment variables so rotation requires only a config change.

---

**"404 NOT_FOUND — the store definitely exists"**

The store exists but is not linked to your partner account. The store-access check looks at whether the store is linked to your partner account — if the link is missing or points to a different partner, you see 404. This is intentional and identical to "does not exist" to prevent UUID enumeration. Contact the Bareconnect team to confirm the store link.

---

**"403 INSUFFICIENT_SCOPE — I thought I had that scope"**

Check `GET /me` to inspect the actual scopes on your key. The key you are using may not be the one you think. Scope sets are fixed at key issuance — to add a scope you must request a new key.

---

**"422 VALIDATION_ERROR — but my JSON looks right"**

Possible causes:
- Wrong `Content-Type` header (`text/plain` instead of `application/json`)
- Sending a number as a string (`"price": "45"` instead of `"price": 45`)
- Sending a field not in the accepted list (unknown fields are silently ignored, but the validation failure is on a required field you may have omitted)
- Email field contains spaces or is not a valid email format

Check the `errors` object in the response — it maps field names to arrays of failure messages.

---

**"422 NOT_APPLICABLE on confirm-payment"**

Your partner account is configured as `payment_mode: bareconnect`. The BYOP flow does not apply. Call `GET /me` to check your mode. If you need BYOP access, contact Bareconnect to change your payment mode.

---

**"422 FREE_TIER_LIMIT_REACHED"**

The store has hit the product cap set by your partner free tier configuration. Options:
1. Activate a subscription plan for the store via the Bareconnect super admin (partner-managed billing)
2. Ask Bareconnect to raise your `free_tier_products` limit

---

**"422 UNSUPPORTED_COUNTRY on POST /stores"**

Bareconnect could not resolve a supported currency for the country string you sent. Use the full English country name (e.g. `"Ghana"`, `"Nigeria"`, `"Uganda"`, `"Kenya"`). Country codes (`GH`, `NG`) are not accepted.

---

**"409 ALREADY_FULFILLED / ALREADY_PAID / ALREADY_DELIVERED"**

These are idempotency guards. Fetch the current order state with `GET /stores/{id}/orders/{id}` and check `fulfilled`, `paid`, `delivered` before writing. If you need to guarantee idempotent writes regardless, catch the 409 and treat it as a success (the state you wanted is already set).

---

**"429 RATE_LIMIT_EXCEEDED — I am under 60 req/min"**

Rate limiting is per-key, per-minute calendar window. If two processes or threads share the same key, their counts are combined. Solutions:
- Issue separate keys per process
- Add request queuing / throttling on your side to stay under 60
- Read `X-RateLimit-Reset` from any response and wait until that epoch before retrying when you get a 429

---

**"Webhooks are not arriving"**

1. Check the registered URL with `GET /webhooks` — confirm `active: true` and the URL is correct
2. Verify your endpoint returns `2xx` within 10 seconds
3. Check that your endpoint is publicly reachable (not localhost or behind a VPN)
4. Confirm the event you expect is in your webhook's `events` array
5. If in development, use a tunnel (e.g. ngrok) and check its request log

---

**"Webhook signature verification fails"**

Most common causes:
- Using the parsed JSON string instead of the **raw request body bytes** for HMAC input
- Using the wrong secret (confirm it matches what was returned at webhook creation time)
- Adding/removing whitespace when logging or forwarding the body before verification
- The secret was not saved at creation — it cannot be retrieved. Delete and re-create the webhook.

---

## Security Best Practices

### Key management
- **Never commit keys to source control.** Use environment variables or a secrets manager (AWS Secrets Manager, HashiCorp Vault, Doppler).
- **Rotate keys periodically.** Contact partners@bareconnect.com to issue a replacement key before revoking the old one.
- **Scope down.** Issue separate keys per integration. A read-only analytics consumer must not hold `catalog:write`.
- **Log key usage.** Monitor `last_used_at` from `GET /me` — an unusually high last-used timestamp can indicate key reuse by an unintended process.

### Webhook security
- **Always verify signatures** before processing any delivery. Reject anything with a non-matching or missing `X-Bareconnect-Signature`.
- **Use `hash_equals` / `crypto.timingSafeEqual` / `hmac.compare_digest`** for signature comparison. Standard string equality (`===`) is vulnerable to timing attacks.
- **Verify using the raw body**, not parsed JSON. Parsing and re-serialising changes whitespace, breaking the HMAC.
- **Deduplicate with `X-Bareconnect-Delivery`.** Under retries, the same delivery UUID is reused. Store processed delivery IDs and skip duplicates to prevent double-processing orders.
- **Respond fast** (`2xx` within 10 s). Do heavy work in a background job.

### General
- **All traffic over HTTPS.** HTTP requests are rejected at the server.
- **Store IDs are UUIDs.** Do not attempt to guess or enumerate them — the API returns identical `404 NOT_FOUND` for "does not exist" and "exists but not yours."
- **Rate limit headers are your dashboard.** Read `X-RateLimit-Remaining` on every response. Build backpressure into your client before hitting 0 — not after.

---

## Quick-Start: Full Integration Flow

```bash
# 1. Verify your key
curl https://bareconnect.com/api/partner/v1/me \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Accept: application/json"

# 2. Provision a merchant store
curl -X POST https://bareconnect.com/api/partner/v1/stores \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"name": "Kofi'\''s Kente", "country": "Ghana", "category": "fashion", "email": "kofi@example.com"}'
# → save the returned store "id" as $STORE_ID

# 3. Update store settings
curl -X PATCH https://bareconnect.com/api/partner/v1/stores/$STORE_ID \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Content-Type: application/json" \
  -d '{"support_email": "help@kofis-kente.com"}'

# 4. Add a product
curl -X POST https://bareconnect.com/api/partner/v1/stores/$STORE_ID/products \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ankara Tote", "price": 45.00, "quantity": 20, "published": true}'

# 5. List products (cursor pagination)
curl "https://bareconnect.com/api/partner/v1/stores/$STORE_ID/products?limit=50" \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Accept: application/json"
# → if meta.has_more=true, fetch next page: ?limit=50&after={meta.next_cursor}

# 6. List orders
curl "https://bareconnect.com/api/partner/v1/stores/$STORE_ID/orders?limit=30" \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Accept: application/json"

# 7. Fulfill an order
curl -X PATCH "https://bareconnect.com/api/partner/v1/stores/$STORE_ID/orders/$ORDER_ID/fulfill" \
  -H "Authorization: Bearer bc_mgmt_xxxx"

# 8. Confirm BYOP payment (hybrid/byop partners only)
curl -X POST "https://bareconnect.com/api/partner/v1/stores/$STORE_ID/orders/$ORDER_ID/confirm-payment" \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Content-Type: application/json" \
  -d '{"reference": "TXN-2026-0507", "amount_confirmed": 90.00, "currency": "GHS"}'

# 9. Register a webhook
curl -X POST https://bareconnect.com/api/partner/v1/webhooks \
  -H "Authorization: Bearer bc_mgmt_xxxx" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://yourplatform.com/hooks/bc", "events": ["order.paid", "order.fulfilled", "store.created"]}'
# → save the returned "secret" immediately
```

---

## Machine-readable spec

[`../openapi/headless.v1.yaml`](../openapi/headless.v1.yaml) — OpenAPI 3.1 for this API. The Storefront API / AI Engine Connect has its own publishable-key spec: `../openapi/storefront.v1.yaml`.

---

*For access, key rotation, or billing enquiries: **partners@bareconnect.com***
