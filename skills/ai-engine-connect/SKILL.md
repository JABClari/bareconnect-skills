---
name: ai-engine-connect
description: >-
  Connect a client-only storefront frontend to a Bareconnect store using a
  publishable key. Browse the live catalog, build a cart, and hand off to
  Bareconnect-hosted checkout. READ + CART ONLY — no admin, no order writing,
  no payment handling. Use when a user wants their own (or AI-built) frontend to
  sell from an existing Bareconnect store, managed in the Bareconnect admin.
version: 1.0.0
---

# Bareconnect — AI Engine Connect (client-only storefront)

You are building a **storefront frontend** that reads live data from an existing **Bareconnect**
store and sells from it. Bareconnect holds the products, the cart, the checkout, and the admin.
You build the frontend; you never build an admin.

## When to use this skill
- The user has (or is building) a frontend and wants it wired to their existing Bareconnect store.
- They gave you a **store reference** (`bcs_…`) and a **publishable key** (`bcpk_live_…`), generated
  from **Bareconnect admin → Settings → Connect**.

## When NOT to use this skill
| If the user wants… | Use instead |
|---|---|
| a server that reads orders, edits the catalog, fulfils, refunds | the **Bareconnect Headless API** (secret key, server-side) — not a skill |
| their store shoppable by autonomous AI agents | **UCP** (`ucp-connect`) |
| to manage orders / settings | tell them to use the **Bareconnect admin**; this skill never builds admin UI |

## The shared model (applies to everything below)

- **Auth = a publishable key + a store reference.** The publishable key is a **buyer-facing**
  credential. It is **not a secret** — putting it in frontend config and committing it is fine.
- **READ + CART ONLY.** With this key you may: read published products, read collections/categories,
  create a cart, add/update/remove cart items, read the cart, and request a hosted-checkout URL. You
  may **not** read orders, edit products, or do anything from the admin. There is no such endpoint
  for this key — do not try (you'll get `401`/`403`).
- **Never handle payment.** Checkout is a **redirect to a Bareconnect-hosted page**. Never build a
  card form, never collect payment details, never create the order yourself — the hosted checkout
  creates the (unpaid-until-payment) order.
- **Persist the cart.** Store the cart `token` in `localStorage` so the cart survives reloads.
- **Multi-currency.** Bareconnect stores sell in many currencies (GHS, NGN, KES, USD and more).
  Every price returns its own `currency` code — render with that, never a hardcoded symbol or an
  assumed GHS. Examples in the reference use GHS only as an example.
- **No mocking.** If the catalog is empty or a call fails, show a real empty state or surface the
  error — never fall back to fake/placeholder products.

## Auth
```
Base URL:  https://bareconnect.com/api/storefront/v1
Headers:   X-Bareconnect-Store: <STORE_REF>       # e.g. bcs_XXXXXXXX
           X-Bareconnect-Key:   <PUBLISHABLE_KEY>  # e.g. bcpk_live_XXXXXXXX
```
Put `<STORE_REF>` and `<PUBLISHABLE_KEY>` in the frontend config/env. They are safe to expose.
Products are keyed by a public **`handle`** (slug), never an internal id. Rate limit: **120 req/min**
per key. If the merchant set a domain allowlist, requests must come from an allowed `Origin`.

## The run
1. **Scaffold** the frontend (framework of the user's choice) with a small `bareconnect.js`
   transport that attaches the two headers to every request.
2. **Catalog** — `GET /products` → grid (each card: `handle`, `title`, `price.amount`,
   `price.compare_at_price`, `availability`, `image`, `url`). Filters: `search`, `collection`,
   `category`, `min_price`, `max_price`. Pagination: `?page=&limit=` (max 50).
3. **Product** — `GET /products/{handle}` → detail with `variants[]` (`id`, `options`, `price`,
   `available`), `images[]`, `description_html`; add-to-cart uses the variant's `id`.
4. **Cart** — `POST /carts` (create → `token`) · `POST /carts/{token}/items`
   (`{ handle, variant_id?, quantity }`) · `PATCH /carts/{token}/items/{lineId}` (`{ quantity }`,
   0 removes) · `DELETE /carts/{token}/items/{lineId}` · `GET /carts/{token}`. Persist the token.
   A cart is **single-currency** — the first item sets it; mixing currencies returns `409`.
5. **Checkout** — `POST /carts/{token}/checkout` returns a **Bareconnect-hosted checkout URL** (on
   the store's own domain). Redirect the customer there. The frontend's job ends at the redirect.
   > See `references/storefront/INSTRUCTIONS.md` for exact request/response shapes.

## Verify (before declaring done)
- The catalog renders at least one **real** product (or a genuine empty state — never mock data).
- An item adds to the cart and the subtotal is correct; the cart survives a page reload.
- Checkout redirects to the merchant's Bareconnect-hosted checkout URL.

## Final setup (always tell the user this)
> "Your storefront is connected. To take real payments and go live, open your Bareconnect admin,
> connect a payment method, and set the store to live. If you locked the key to specific domains
> under Settings → Connect, add your deployed domain there."

If a call fails because payment isn't connected or the domain isn't allow-listed, flag it and
continue; never fall back to mock data.
