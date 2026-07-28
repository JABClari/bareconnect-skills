# Storefront API — reference playbook

Publishable-key storefront endpoints. Every request sends:
```
X-Bareconnect-Store: <STORE_REF>       # bcs_…
X-Bareconnect-Key:   <PUBLISHABLE_KEY>  # bcpk_live_…
```
Base: `https://bareconnect.com/api/storefront/v1`. Products are addressed by public **`handle`**
(slug). Rate limit **120 req/min** per key (`429` when exceeded; `X-RateLimit-Reset` header).
Full contract in `../../../openapi/storefront.v1.yaml`.

> **Credentials:** ask the user for both values, then keep them in the platform's secrets/env store
> and read them in one place (the transport). Publishable ≠ commit-safe — never hardcode either value
> and never commit a `.env`. A `bcsk_…` **secret** key must never reach the frontend. See *Credentials*
> in `SKILL.md`.

> **Currency:** Bareconnect is multi-currency — stores sell in GHS, NGN, KES, USD and more. Every
> price carries its own `currency` field; **read it, never assume**. The examples below happen to
> use `GHS`, but yours will be whatever your store is set to. Render prices using the `currency`
> value returned, not a hardcoded symbol.

## Products
`GET /products` — live catalog. Query: `search`, `collection` (handle), `category` (handle),
`min_price`, `max_price`, `page`, `limit` (max 50).
```jsonc
{
  "data": [
    { "handle": "casual-mens-headgear", "title": "Casual men's headgear",
      "price": { "amount": 90, "compare_at_price": null, "currency": "GHS" },
      "availability": "in_stock", "collection": null, "image": "https://…", "url": "https://…" }
  ],
  "meta": { "total": 12, "per_page": 20, "current_page": 1, "total_pages": 1, "currency": "GHS" }
}
```
`GET /products/{handle}` — one product (full shape):
```jsonc
{ "data": {
  "handle": "…", "title": "…", "description": "…", "description_html": "<p>…</p>",
  "price": { "amount": 90, "compare_at_price": null, "currency": "GHS" },
  "availability": "in_stock", "collection": null, "category": "Store Front",
  "images": ["https://…"],
  "variants": [ { "id": "<variant-id>", "options": { "size": "M", "color": "Black" },
                 "price": 90, "compare_at_price": null, "available": true } ],
  "url": "https://<store-domain>/product/…"
} }
```
Unknown handle → `404 { "error": "product_not_found" }`.

## Collections
`GET /collections` → `{ "data": [ { "handle", "title", "description" } ] }`
`GET /collections/{handle}/products` → `{ "collection": {…}, "data": [ …product cards… ], "meta": {…} }`

## Cart
Prices are always recomputed server-side — you never send a price. To add a specific variant, pass
its `variant_id` (from the product's `variants[].id`).

| Method | Path | Body | Result |
|---|---|---|---|
| POST | `/carts` | optional `{ "items": [ { "handle", "variant_id?", "quantity" } ] }` | `201` → cart |
| GET | `/carts/{token}` | — | cart |
| POST | `/carts/{token}/items` | `{ "handle", "variant_id?", "quantity" }` | cart (merges same line) |
| PATCH | `/carts/{token}/items/{lineId}` | `{ "quantity" }` (≤0 removes) | cart |
| DELETE | `/carts/{token}/items/{lineId}` | — | cart |

Cart shape:
```jsonc
{ "data": {
  "token": "bccart_…", "currency": "GHS", "item_count": 2, "subtotal": 180,
  "items": [ { "line_id": "<uuid>", "handle": "…", "title": "…", "variant": { "size": "M" },
              "quantity": 2, "unit_price": 90, "line_total": 180, "currency": "GHS",
              "available": true, "image": "https://…" } ]
} }
```
Persist `token` in `localStorage`. A cart is **single-currency**: the first item sets the currency;
adding a product priced in another currency → `409 { "error": "currency_mismatch" }`. Out-of-stock
add → `409 { "error": "out_of_stock" }`. Unknown variant → `422 { "error": "invalid_variant" }`.

### Cart field names — exact, do not substitute
The cart is the most-mistyped shape in this API. Copy these names verbatim:

| Field | Type | Not |
|---|---|---|
| `data.items` | array of lines | ~~`lines`~~, ~~`line_items`~~, ~~`cart_items`~~ |
| `data.subtotal` | **number** | ~~`{ amount, currency }`~~ |
| `data.currency` | string, top-level | (lines do repeat it, but read the cart's) |
| `data.item_count` | number | ~~`items.length`~~ (this is summed quantity) |
| `items[].line_id` | string (uuid) | ~~`id`~~ — this is the id in `/items/{lineId}` |
| `items[].unit_price` | **number** | ~~`price.amount`~~ |
| `items[].line_total` | **number** | ~~`total.amount`~~ |
| `items[].variant` | object of options, or `null` | ~~`variant_id`~~ (that's the request field) |

Products vs cart lines differ on purpose:
`product.price` is an **object** (`{ amount, compare_at_price, currency }`);
`variants[].price` and every cart-line price are **plain numbers**.

`items` may be **absent or empty** on a freshly created cart, and the whole object may be `undefined`
while a request is in flight or after a failure. `cart?.items.reduce(...)` still throws — the
optional chain short-circuits on `cart`, not on `items`. Always `(cart?.items ?? [])`.

Before rendering, confirm against a live response:
```js
const r = await fetch(`${BASE}/carts/${token}`, { headers });
console.log(JSON.stringify((await r.json()).data, null, 2)); // read the real field names
```

## Checkout (hand-off only)
`POST /carts/{token}/checkout` — no body required.
```jsonc
→ { "data": { "checkout_url": "https://<store-domain>/storefront-checkout/bccart_…", "expires": null } }
```
Redirect the customer to `checkout_url`. Bareconnect rebuilds the cart on its own checkout, collects
payment + delivery, and creates the order (unpaid until payment clears), tagged `source=storefront_api`.
**The frontend never creates the order and never sees payment details.** Empty cart → `422`; a cart
with no buyable items → `409 no_available_items`.

## What this key CANNOT do (do not attempt)
Reading orders, listing customers, editing/creating/deleting products, fulfilment, refunds,
settings, cost/profit data. Those require the admin or the (separate) Headless secret-key API.

## Errors
Machine-readable JSON: `{ "error", "error_code", "message", "request_id" }`.
- `401 invalid_key` / `missing_credentials` — bad or missing store ref / key.
- `403 feature_unavailable` — AI Engine Connect isn't enabled for this store (beta-gated).
- `403 domain_not_allowed` — the request `Origin` isn't in the key's allowlist.
- `429 rate_limited` — over 120 req/min.
On an empty catalog, show a real empty state — never mock.
