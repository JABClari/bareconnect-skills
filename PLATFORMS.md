# Platforms — per-agent packaging

The principle: **share one skill body, package it per agent.** Never fork the skill content per
tool. Keep the skill in `skills/<name>/SKILL.md` and add thin, per-platform manifests that tell
each agent how to discover/install/invoke it:

| Agent | Manifest | Notes |
|-------|----------|-------|
| Claude / Claude Code | `.claude-plugin/` | Agent Skills-compatible plugin config |
| Cursor | `.cursor-plugin/` | |
| Codex | `.codex-plugin/` | |
| Gemini | `gemini-extension.json` | |
| Generic / MCP | `.mcp.json`, `plugin.json` | any agent supporting the Agent Skills spec |

Delivery: **today**, agents fetch the skill from this repo's raw `SKILL.md` (see `skills/README.md`).
**Later:** stand up a thin service at `https://bareconnect.com/skills/<name>` that pulls the markdown
from `main`, edge-cached and invalidated on push, so connect-prompt URLs are branded and stable. Not
required for the skill to work — it already works via the raw repo URL.

## The two rules
1. **One skill body, many manifests.** Only the manifest and the per-tool *invocation prompt*
   (see `skills/README.md`) change per agent — never the skill content itself.
2. **Version via the repo, pin via the delivery layer.** Follow semver on the repo; a saved
   connect-prompt keeps working because it points at a stable path.

## Status
- [x] Storefront API is **live** (v1) — see `openapi/storefront.v1.yaml`; `ai-engine-connect` skill
      is out of draft.
- [x] Raw-repo delivery documented in `skills/README.md`.
- [ ] Author the per-agent manifests (Claude/Cursor/Codex/Gemini/MCP) — confirm each platform's
      current plugin schema before writing, then add thin manifests that reference the shared skill.
- [ ] Stand up the branded `bareconnect.com/skills/<name>` delivery service (optional polish).
- [ ] Add `yaml/` evals (token-budget scenarios per skill) for CI.
