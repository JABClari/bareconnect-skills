# Bareconnect Headless API

The server-side API for building your own product on top of Bareconnect. You hold a **secret
key**, keep it on your server, and drive stores, catalogs, orders, and webhooks programmatically.
Bareconnect stays invisible behind your own app.

> **This is not the storefront API.** If you're building a browser frontend, use a **publishable**
> key and the [skills](../skills). The secret key here can read and change everything — never put
> it in a browser, a prompt, or an AI build tool.

## Authentication
Every request carries your secret key as a bearer token:
```
Authorization: Bearer <SECRET_KEY>
```
Keys are issued per integration with a fixed set of scopes (least privilege). Request a key at
**partners@bareconnect.com**.

## Scopes
| Scope | Grants |
|-------|--------|
| `stores:read` | list & inspect linked stores |
| `stores:write` | provision & update stores |
| `catalog:read` | browse products |
| `catalog:write` | create/update/delete products |
| `orders:read` | view orders, fulfilment status, per-order profit |
| `orders:write` | update fulfilment/delivery, import external orders |
| `orders:confirm` | confirm payment via your own gateway (BYOP) |
| `webhooks:manage` | register/list/remove webhooks |

## Endpoint groups
- **Stores** — `GET/POST /stores`, `GET/PATCH /stores/{id}`
- **Products** — `GET/POST /stores/{id}/products`, `PATCH/DELETE …/{productId}`
- **Orders** — `GET …/orders`, `GET …/orders/{id}` (includes a `profit` block), `PATCH …/fulfill`,
  `PATCH …/deliver`, `POST …/confirm-payment`, `POST …/orders/import` (import external orders,
  idempotent on `external_reference`)
- **Webhooks** — `GET/POST /webhooks`, `DELETE /webhooks/{id}`,
  `GET /webhooks/{id}/deliveries`. Events include `order.created`, `order.paid`,
  `order.fulfilled`, `order.delivered`, `order.imported`, `product.*`, `store.*`.

## Conventions
- Cursor pagination (`after` + `limit`, `has_more` + `next_cursor` in `meta`).
- Money is returned in the **store's currency** (`price_in_store_currency`).
- Per-key rate limiting; `X-RateLimit-*` headers on every response.
- Machine-readable errors: `error_code`, `request_id`, `X-Request-Id`.

## Machine-readable spec
[`../openapi/headless.v1.yaml`](../openapi/headless.v1.yaml) — OpenAPI 3.1 for this API.
(The Storefront API / AI Engine Connect has its own spec: `../openapi/storefront.v1.yaml`.)

## Full reference
> **TODO:** port the complete, sanitised endpoint reference here from the internal
> `WHITE_LABEL_HEADLESS_TECHNO.md` (request/response bodies, examples, webhook payloads). Strip all
> internal notes before publishing.
