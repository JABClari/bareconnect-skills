# UCP — Universal Commerce Protocol

UCP lets **AI agents** (assistants, marketplaces, shopping bots) discover Bareconnect stores and
their products, and complete purchases — programmatically, on a shopper's behalf.

Where the [skills](../skills) help a human's AI *build a storefront*, and the
[headless API](../headless) lets a developer *build their own stack*, **UCP is how an autonomous
agent shops** a Bareconnect store directly.

## What it enables
- **Discovery** — an agent can find stores and query their catalogs.
- **Product access** — structured product data an agent can reason over (attributes, variants,
  price, availability).
- **Purchase** — an agent can place an order through the protocol.

## How it relates to the rest
| Surface | Actor | Purpose |
|---------|-------|---------|
| Skills | a human's AI coding tool | build a storefront frontend |
| Headless API | a developer's server | build your own commerce stack |
| **UCP** | an autonomous AI agent | discover & buy, no human UI |

A store connected via a publishable key is *also* reachable by agents through UCP — so the same
store can be sold by a human-built frontend **and** shopped by AI agents.

## Status & reference
> **TODO:** port the public UCP specification here from the internal `docs/UCP.md` (endpoints,
> token tiers, the store directory, product schema, security model). Sanitise before publishing.

## MCP
An MCP (Model Context Protocol) server exposes a store's UCP capabilities to MCP-speaking agents
(e.g. Claude, ChatGPT). A `connect-mcp` skill is planned in [skills](../skills).
