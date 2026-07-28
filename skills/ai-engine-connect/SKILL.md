---
name: ai-engine-connect
description: >-
  Connect a client-only storefront frontend to a Bareconnect store using a
  publishable key. Browse the live catalog, build a cart, and hand off to
  Bareconnect-hosted checkout. READ + CART ONLY — no admin, no order writing,
  no payment handling. Use when a user wants their own (or AI-built) frontend to
  sell from an existing Bareconnect store, managed in the Bareconnect admin.
version: 1.1.0
---

# Bareconnect — AI Engine Connect (client-only storefront)

You are building a **storefront frontend** that reads live data from an existing **Bareconnect**
store and sells from it. Bareconnect holds the products, the cart, the checkout, and the admin.
You build the frontend; you never build an admin.

## When to use this skill
- The user has (or is building) a frontend and wants it wired to their existing Bareconnect store.
- They gave you — or can generate, from **Bareconnect admin → Settings → Connect** — a **store
  reference** (`bcs_…`) and a **publishable key** (`bcpk_live_…`). If they haven't yet, ask for them
  up front; see *Credentials*.

## When NOT to use this skill
| If the user wants… | Use instead |
|---|---|
| a server that reads orders, edits the catalog, fulfils, refunds | the **Bareconnect Headless API** (secret key, server-side) — not a skill |
| their store shoppable by autonomous AI agents | **UCP** (`ucp-connect`) |
| to manage orders / settings | tell them to use the **Bareconnect admin**; this skill never builds admin UI |

## The shared model (applies to everything below)

- **Auth = a publishable key + a store reference.** The publishable key is a **buyer-facing**
  credential — a leak is not a financial incident. That is *not* a licence to hardcode it. It goes
  in the platform's secrets/env store, never in source, never in a commit — see *Credentials* below.
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
- **Never guess a field name.** Use the exact shapes in `references/storefront/INSTRUCTIONS.md`.
  Do not port field names from Shopify/Medusa/Commerce.js habits — see *Shapes* below.

## Shapes — the #1 source of runtime crashes

Every response is documented. Read the shape **before** writing render code; do not infer it from
another platform's API. The most common wrong guesses:

| Wrong (do not use) | Right | Note |
|---|---|---|
| `cart.lines` | `cart.items` | Cart lines are `items[]` |
| `line.id` | `line.line_id` | The mutation path is `/items/{line_id}` |
| `line.price.amount` | `line.unit_price` (number) | On a **cart line**, prices are plain numbers |
| `cart.subtotal.amount` | `cart.subtotal` (number) | Cart totals are plain numbers |
| `line.currency` assumed absent | `cart.currency` (top-level) | Cart is single-currency |
| `product.price` as number | `product.price.amount` + `.currency` | On a **product**, price is an object |

Note the asymmetry that catches everyone: **products** carry `price: { amount, currency }`;
**cart lines** carry flat `unit_price` / `line_total` numbers plus a top-level `cart.currency`.

**Defensive rendering is mandatory.** An optional chain that stops at the parent still throws:
```js
cart?.items.reduce(...)      // ✗ throws if items is undefined (empty/erroring cart)
(cart?.items ?? []).reduce(...)  // ✓
```
Guard every array before `.map`/`.reduce`/`.length`, and every price before formatting. A cart that
has been created but never filled, or a failed fetch, both yield an object with no `items`.

**Prove the shape before building on it.** Fetch one real product and one real cart, log the raw
JSON, and confirm the field names against the reference table — *then* write the components. One
`console.log` of a live response costs seconds; guessing costs a debugging cycle per file.

**Provider/context wiring.** If cart state lives in a `CartProvider`, mount it above *everything*
that reads the cart — navbar/header cart badges included. A cart hook called outside its provider
returns `undefined`, which looks exactly like a shape bug but isn't.

## Auth
```
Base URL:  https://bareconnect.com/api/storefront/v1
Headers:   X-Bareconnect-Store: <STORE_REF>       # e.g. bcs_XXXXXXXX
           X-Bareconnect-Key:   <PUBLISHABLE_KEY>  # e.g. bcpk_live_XXXXXXXX
```
Products are keyed by a public **`handle`** (slug), never an internal id. Rate limit: **120 req/min**
per key. If the merchant set a domain allowlist, requests must come from an allowed `Origin`.

## Credentials — ask, then store properly

**Ask before you build.** Once you've read this skill and understand the job, stop and ask the user
for the two values — do not scaffold against placeholders and do not invent a `bcs_demo` /
`bcpk_test_…` to "unblock" yourself. A storefront with fake credentials is a storefront that has
never been proven to work.

> "To connect your storefront I need two values from your Bareconnect admin →
> **Settings → Connect**: your **store reference** (`bcs_…`) and your **publishable key**
> (`bcpk_live_…`). I'll put them in this project's secrets, not in the code."

**Store them in the platform's secret manager — always.** Publishable ≠ public-by-default. Standards
don't bend because a particular value happens to be low-risk: the same handling that protects a
secret key is what stops a *secret* key from ever being pasted into source by muscle memory.

| Platform | Where the keys go |
|---|---|
| Replit | **Secrets** pane (never `.replit`, never inline) |
| Next.js / Vite / CRA | `.env.local` (git-ignored), exposed via the framework's public prefix |
| Vercel / Netlify / Cloudflare | project **Environment Variables**, per environment |
| Anything else | that platform's env/secret store — never a committed file |

Rules that hold everywhere:
- **Never** hardcode either value in a component, a config object, or a fetch call.
- Read them once, in one place (the `bareconnect.js` transport). Nothing else touches them.
- Ensure `.env*` is git-ignored **before** writing the file; commit a `.env.example` with empty
  values instead. Never commit a real key, even a publishable one.
- Don't print keys to logs, error messages, or the chat transcript after collecting them.
- If a value is missing at runtime, fail loudly with "Bareconnect credentials not configured" —
  never silently fall back to a demo store or mock data.
- The user pastes credentials to *you*; treat that message as sensitive — put the values straight
  into the secret store and don't echo them back.

Client-side exposure of the publishable key in the shipped bundle is expected and fine — that is
what it's for. What is not fine is it living in the repo. The defence for a publishable key is the
**domain allowlist** (Settings → Connect), not obscurity — tell the user to set it before launch.

If the user asks you to use a **secret** key (`bcsk_…`) in the frontend, refuse: that key belongs to
the server-side Headless API and would expose their whole admin. Point them at the Headless docs.

## The run
0. **Collect credentials** (above) and put them in the platform's secret store before any code.
1. **Scaffold** the frontend (framework of the user's choice) with a small `bareconnect.js`
   transport that reads the two values from env and attaches them as headers on every request.
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
- **Empty-cart path renders without throwing** — load the app with no cart and with a cart created
  but empty. This is where `items`-is-undefined crashes surface.
- Field names in the code match `references/storefront/INSTRUCTIONS.md` exactly (`items`,
  `line_id`, `unit_price`, `line_total`, `subtotal`, `currency`).
- **No credential appears anywhere in the repo** — grep the tree for `bcs_` and `bcpk_`; the only
  hits should be `.env.example` placeholders and docs. `.env*` is git-ignored.

## Final setup (always tell the user this)
> "Your storefront is connected. To take real payments and go live, open your Bareconnect admin,
> connect a payment method, and set the store to live. If you locked the key to specific domains
> under Settings → Connect, add your deployed domain there."

If a call fails because payment isn't connected or the domain isn't allow-listed, flag it and
continue; never fall back to mock data.
