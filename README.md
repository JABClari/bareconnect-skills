# Bareconnect for Developers & AI Builders

Build the frontend anywhere. Bareconnect is the commerce backend — products, carts, orders,
checkout, payments, and an admin — that your storefront talks to.

This repository is the public developer surface for Bareconnect. It has three parts:

| Directory | What it is | Who it's for |
|-----------|-----------|--------------|
| [`skills/`](./skills) | **AI-builder skills** — instruction files an AI coding agent (Lovable, Cursor, Bolt, Manus, Claude, …) fetches and follows to wire a storefront to Bareconnect | anyone building a frontend with an AI tool |
| [`headless/`](./headless) | **Headless API reference** — the server-side API for building your own stack on top of Bareconnect | developers & partners building their own app/admin |
| [`ucp/`](./ucp) | **UCP** — the protocol that lets AI agents discover and buy from Bareconnect stores | agent builders, marketplaces, AI commerce |

---

## Two ways to connect — pick by where your code runs

Bareconnect issues two kinds of credentials, and the difference is simply **where the key can
safely live**:

### 1. Publishable key → for the browser / AI build tools
- Safe to expose. Goes in your storefront's frontend config.
- Can **shop**: read published products, build a cart, place an order, start checkout.
- Cannot do anything from the admin (no reading other orders, no editing the catalog, no refunds).
- You manage the store in the **Bareconnect admin**; the storefront just sells.
- This is what the [`skills/`](./skills) use.

### 2. Secret key → for your server only
- Never goes in a browser, a prompt, or an AI tool. Server-to-server only.
- Can do **everything**: read orders, manage the catalog, fulfil, refund, settle.
- You build your own admin/stack; Bareconnect is the engine behind it.
- This is the [`headless/`](./headless) API.

> **Rule of thumb:** if your code runs in a browser, use a **publishable** key. If it runs on a
> server you control, you may use a **secret** key. Never put a secret key anywhere a user could
> see it.

---

## How skills work

A **skill** is a plain-Markdown instruction file that an AI coding agent reads mid-build. You give
your AI tool a short prompt with your Store ID, your publishable key, and the URL of a skill; the
agent fetches the skill and follows it to build a working storefront.

Each skill is a folder with a `SKILL.md` (frontmatter + guide) and optional `references/` deep
playbooks — the Agent Skills format. Machine-readable API specs live in [`openapi/`](./openapi).
Per-agent packaging is described in [`PLATFORMS.md`](./PLATFORMS.md).

Start here: **[skills/ai-engine-connect](./skills/ai-engine-connect)**.

---

## Getting your keys

1. Create a store on Bareconnect.
2. In the admin, open **Settings → AI Engine Connect**.
3. Copy your **Store ID** and **publishable key** (and, for server apps, request a **secret key**).

---

## Support

- Docs & guides: this repository.
- Questions / a key for your integration: **partners@bareconnect.com**.
