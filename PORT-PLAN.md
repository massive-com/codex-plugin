# Massive Codex Plugin — Port Plan

This repo is a port of the [Massive Claude Code plugin](https://github.com/massive-com/claude-code-plugin) to OpenAI Codex. The skill structure, MCP server config, and API knowledge from the Claude Code plugin map almost 1:1 to Codex's plugin format.

## Why target Codex specifically

OpenAI has four surfaces where Massive could ship developer context, ranked by fit:

| Option | Fit | Effort |
|---|---|---|
| **OpenAI Codex plugin** | **Best match.** Same skill format, MCP support, CLI install model. | ~1-2 days of porting and verification. |
| ChatGPT Apps SDK (MCP Apps) | Different product: interactive UI inside consumer ChatGPT. Requires a hosted MCP server plus a web UI. | Weeks. Separate project. |
| ChatGPT consumer Skills (emerging) | Reportedly Claude-compatible folder format; not GA for third-party authoring as of April 2026. | Wait-and-see. |
| ChatGPT Custom GPTs | Text-only, no code execution, cannot bundle MCP. | Not worth it. |

Codex is OpenAI's direct competitor to Claude Code: a coding CLI/agent with plugins, skills, and MCP support. The rest of this plan assumes Codex.

## What translates directly

- **Skill folder format.** Codex reads `skills/<name>/SKILL.md` with `name` + `description` frontmatter, exactly like Claude Code. Same progressive-disclosure pattern.
- **MCP server.** `.mcp.json` configures MCP servers the same way. The Massive MCP server (`uvx mcp_massive@v0.9.0`) works as-is.
- **API knowledge.** The content of `CLAUDE.md` becomes `AGENTS.md` at the repo root. AGENTS.md is a multi-vendor standard (Codex, Kilo Code, Roo, Warp, Factory, OpenCode).
- **Plugin manifest concept.** `.codex-plugin/plugin.json` is the Codex equivalent of `.claude-plugin/plugin.json`. Most fields carry over.

## File-by-file mapping

| Claude Code plugin | Codex plugin | Notes |
|---|---|---|
| `.claude-plugin/plugin.json` | `.codex-plugin/plugin.json` | Renamed dir. Schema similar but drop `userConfig` (not in Codex's documented schema) and add `interface` for UI metadata. |
| `.claude-plugin/marketplace.json` | `.agents/plugins/marketplace.json` | Different path. Plugin source schema changes from `"source": "./"` to `"source": {"source": "local", "path": "./"}`. |
| `.claude/CLAUDE.md` | `AGENTS.md` (repo root) | Moved up one directory. Content unchanged. |
| `skills/<name>/SKILL.md` | `skills/<name>/SKILL.md` | Identical path. Minor frontmatter cleanup (see below). |
| `.mcp.json` | `.mcp.json` | Env interpolation syntax differs. See "API key" below. |
| `cross-tool/` | *(moved out)* | These become the standalone `massive-ai-rules` repo (Cursor, Copilot, Windsurf, Gemini, etc.). |
| `README.md` | `README.md` | Rewritten for Codex install flow. |

## Known differences that need verification

### 1. API key prompting (`userConfig` replacement)
The Claude Code plugin uses `userConfig` in `plugin.json` to prompt on install and interpolates the key via `${user_config.massive_api_key}` in `.mcp.json`.

Codex's plugin.json schema does not publicly document a `userConfig` equivalent. The marketplace.json policy field `"authentication": "ON_INSTALL"` implies prompting exists, but the exact schema and interpolation variable are unclear.

**Current placeholder:** This repo's `.mcp.json` uses `${MASSIVE_API_KEY}` env-var interpolation. Users must `export MASSIVE_API_KEY=...` before running Codex.

**To resolve:** Install Codex locally, run the built-in `$plugin-creator` scaffolder against a hypothetical API-key-requiring plugin, and inspect what it emits.

### 2. Skill frontmatter
Our SKILL.md files include `argument-hint`, `disable-model-invocation`, and `allowed-tools`. Codex only documents `name` and `description` as required.

**Current state:** The extra fields are retained (Codex may ignore them silently).

**To resolve:** Determine if Codex accepts these fields. If not, move relevant metadata to `skills/<name>/agents/openai.yaml` (Codex's per-skill UI metadata file).

### 3. Skill invocation syntax in cross-references
Claude Code invokes skills as `/massive:discover`. Codex uses `$skill-name` or a `/skills` picker. The SKILL.md files here still reference `/massive:discover`, `/massive:debug`, etc. in prose.

**To resolve:** Confirm Codex's invocation syntax, then global find-replace across all skill files.

### 4. Marketplace layout
The Codex docs show marketplace entries like:
```json
{"source": {"source": "local", "path": "./plugins/my-plugin"}}
```
with paths relative to the marketplace root. If marketplace.json lives at `.agents/plugins/marketplace.json`, `./` resolves to that directory, not the repo root.

Two viable layouts:
- **Flat (current):** Plugin files at repo root, marketplace path `"./"`. Simpler, matches Claude Code.
- **Nested:** Plugin files under `.agents/plugins/massive/`, marketplace path `"./massive/"`. More aligned with Codex examples.

**To resolve:** Test both layouts with local Codex install.

## Work remaining

1. Verify points 1-4 above against a real Codex install.
2. Port skill files' inter-references and adjust frontmatter.
3. Rewrite `README.md` with the verified Codex install command and quickstart.
4. Update any "Claude Code" or "Claude" phrasing in AGENTS.md and skills to be assistant-agnostic.
5. Test end-to-end: install locally, confirm skills appear, confirm MCP server spawns and answers a live API call.
6. Publish: push to `github.com/massive-com/codex-plugin`, update the plugin's `repository` field, and follow Codex marketplace submission.

## Source-of-truth strategy

Rather than duplicating the full API surface in AGENTS.md (and in every other AI tool's rules file), treat `https://massive.com/docs/rest/llms-full.txt` and `https://massive.com/docs/llms.txt` as authoritative. The AGENTS.md in this repo covers SDK patterns, gotchas, and plan tiers (things that don't change per endpoint). When Codex needs current endpoint details, it should fetch `llms-full.txt`.

This is the same strategy used in the planned `massive-ai-rules` repo. Once llms.txt is referenced uniformly, drift across tool-specific rule files drops to near-zero.

## References

- [Codex Plugins overview](https://developers.openai.com/codex/plugins)
- [Build plugins – Codex](https://developers.openai.com/codex/plugins/build)
- [Agent Skills – Codex](https://developers.openai.com/codex/skills)
- [OpenAI Skills Catalog](https://github.com/openai/skills)
- [AGENTS.md spec](https://agents.md/)
- [Source repo: Massive Claude Code plugin](https://github.com/massive-com/claude-code-plugin)
