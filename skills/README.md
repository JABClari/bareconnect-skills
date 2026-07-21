# Skills

A **skill** is an Agent Skills instruction module an AI coding agent fetches and follows. Each skill
is a folder with a `SKILL.md` (YAML frontmatter + guide) and optional `references/` deep playbooks.
You don't run these yourself — you point your AI builder at one.

## Available skills

| Skill | Folder | Key type | Status |
|-------|--------|----------|--------|
| **AI Engine Connect** — build a client-only storefront (catalog + cart + hosted checkout) | [`ai-engine-connect/`](./ai-engine-connect) | **publishable** (read + cart only) | live · v1 |
| Headless — build your own server-side stack | `headless/` (see [`../headless`](../headless)) | secret (server only) | reference |
| UCP Connect — make a store agent-shoppable | `ucp-connect/` | — | planned |

Machine-readable API specs live in [`../openapi/`](../openapi) (`storefront.v1.yaml`).

## Delivery — how an agent gets the skill

Point your builder at this repo's raw `SKILL.md`:
```
https://raw.githubusercontent.com/JABClari/bareconnect-skills/main/skills/ai-engine-connect/SKILL.md
```
(Or paste the contents of `SKILL.md` directly into the agent.)

## Per-tool connect prompts (customize per agent)

Different AI builders take the instruction differently. Give yours the matching prompt, filling in
your **store reference** (`bcs_…`) and **publishable key** (`bcpk_live_…`) from **Settings → Connect**
in the Bareconnect admin.

**Lovable / Bolt / v0 (build-from-scratch tools):**
```
Build a storefront for my Bareconnect store. Fetch and follow exactly:
https://raw.githubusercontent.com/JABClari/bareconnect-skills/main/skills/ai-engine-connect/SKILL.md
Store reference: <STORE_REF>
Publishable key: <PUBLISHABLE_KEY>
When done, tell me to open my Bareconnect admin to connect payments and go live.
```

**Cursor / Windsurf (in an existing repo):**
```
Wire this frontend to my Bareconnect store, following the AI Engine Connect skill.
Use publishable key <PUBLISHABLE_KEY> and store reference <STORE_REF>. Read + cart only —
checkout must redirect to Bareconnect-hosted checkout. Do not build any admin.
```

**Claude / ChatGPT (agent with fetch):**
```
Fetch the AI Engine Connect SKILL.md and follow it to connect a storefront to Bareconnect.
Store reference <STORE_REF>, publishable key <PUBLISHABLE_KEY>.
```

> These share ONE skill body — only the wording that suits each tool differs. Never duplicate the
> skill content per tool; only the invocation prompt changes.

## Hard rules for skills in this repo
1. **Publishable key only** — skills never ask for a secret key or perform admin/order-writing.
2. **Hosted checkout** — a skill always hands off to Bareconnect-hosted checkout; it never collects
   payment.
3. **No mocking** — empty/failed calls surface a real state, never fake data.
4. **Self-contained** — each `SKILL.md` includes auth, the run, a verify step, and the final
   "open your admin" instruction. Deep endpoint detail goes in `references/`.
