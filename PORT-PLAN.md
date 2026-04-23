# Massive Codex Plugin — Port Plan

This repo is a port of the [Massive Claude Code plugin](https://github.com/massive-com/claude-code-plugin) to OpenAI Codex. The skill structure, MCP server config, and API knowledge translate almost 1:1.

## Why target Codex specifically

OpenAI has four surfaces where Massive could ship developer context, ranked by fit:

| Option | Fit | Effort |
|---|---|---|
| **OpenAI Codex plugin** | **Best match.** Same skill format, MCP support, CLI install model. | ~1 day port + validation. |
| ChatGPT Apps SDK (MCP Apps) | Different product: interactive UI inside consumer ChatGPT. Requires a hosted MCP server plus a web UI. | Weeks. Separate project. |
| ChatGPT consumer Skills (emerging) | Reportedly Claude-compatible folder format; not GA for third-party authoring as of April 2026. | Wait-and-see. |
| ChatGPT Custom GPTs | Text-only, no code execution, cannot bundle MCP. | Not worth it. |

Codex is OpenAI's direct competitor to Claude Code: a coding CLI with plugins, skills, and MCP support. The rest of this plan assumes Codex.

## Verified conventions (from Codex CLI 0.123.0 and bundled reference plugins)

### Marketplace layout (NESTED)

The marketplace root contains:

```
<marketplace-root>/
├── .agents/plugins/marketplace.json    # Marketplace catalog
└── plugins/
    └── <plugin-name>/
        ├── .codex-plugin/plugin.json    # Plugin manifest
        ├── .mcp.json                    # MCP server config (optional)
        ├── .app.json                    # App/connector config (optional)
        ├── AGENTS.md                    # Plugin-scoped agent instructions
        └── skills/
            └── <skill-name>/
                ├── SKILL.md             # Required
                └── agents/openai.yaml   # Optional UI metadata
```

`marketplace.json` entries reference plugins via `source.path: "./plugins/<name>"` (paths relative to the marketplace root).

Install a local marketplace with: `codex plugin marketplace add /path/to/repo`

### plugin.json schema (resolved)

Required: `name` (kebab-case), `version`, `description`.

Optional: `author`, `homepage`, `repository`, `license`, `keywords`, `skills`, `mcpServers`, `apps`, `interface`.

The `interface` block holds UI metadata: `displayName`, `shortDescription`, `longDescription`, `developerName`, `category`, `capabilities` (e.g., `["Interactive", "Write"]`), `websiteURL`, `privacyPolicyURL`, `termsOfServiceURL`, `logo`, `defaultPrompt` (array), `brandColor`, `screenshots`.

**There is no `userConfig` field.** For API-key auth, use env-var interpolation in `.mcp.json` (e.g., `${MASSIVE_API_KEY}`) and document the env-var requirement in the README.

For OAuth-backed connectors (like the bundled GitHub plugin), auth happens via `.app.json` with a `connector_*` ID provisioned by OpenAI. That path requires OpenAI infrastructure and does not apply to third-party API-key plugins.

### SKILL.md frontmatter (resolved)

Only `name` and `description` are supported in frontmatter. Claude Code's `argument-hint`, `disable-model-invocation`, and `allowed-tools` are not part of the Codex spec and have been stripped from our skills.

UI metadata (display name, icons, default prompts) goes in `skills/<name>/agents/openai.yaml`:

```yaml
interface:
  display_name: "Scaffold"
  short_description: "..."
  icon_small: "./assets/icon-small.svg"
  icon_large: "./assets/icon.png"
  default_prompt: "Use $scaffold to create a new Massive project."
```

### Skill invocation syntax

Users invoke skills with `$<skill-name>` (e.g., `$scaffold`, `$discover`). All cross-references in our SKILL.md files have been rewritten from `/massive:X` to `$X`.

Inter-skill routing uses relative paths: `../<other-skill>/SKILL.md`.

## File-by-file mapping from Claude Code

| Claude Code plugin | Codex plugin |
|---|---|
| `.claude-plugin/plugin.json` | `plugins/massive/.codex-plugin/plugin.json` |
| `.claude-plugin/marketplace.json` | `.agents/plugins/marketplace.json` |
| `.claude/CLAUDE.md` | `plugins/massive/AGENTS.md` |
| `skills/<name>/SKILL.md` | `plugins/massive/skills/<name>/SKILL.md` (+ optional `agents/openai.yaml`) |
| `.mcp.json` | `plugins/massive/.mcp.json` |
| `cross-tool/` | *moved to standalone `massive-ai-rules` repo* |

## Remaining work

1. **Enable the plugin end-to-end.** `codex plugin marketplace add .` registers the marketplace; verify the plugin enables and that `$scaffold`, `$discover`, `$debug`, `$options`, `$dashboard` all appear. (Currently the marketplace is registered locally; need to complete the enable flow.)
2. **Live API call test.** Export `MASSIVE_API_KEY` and confirm the MCP server spawns and answers an end-to-end call.
3. **Polish.** Add plugin logo and icons to the `interface` block. Decide whether to prefix skill names (`massive-scaffold` etc.) to avoid collisions with other plugins' generic-name skills.
4. **Bump to v1.0.0** in `plugin.json` once tests pass.
5. **Publish to GitHub.** Push to `github.com/massive-com/codex-plugin`.

## Source-of-truth strategy

The AGENTS.md in this plugin contains SDK patterns, gotchas, and plan tiers (the stable stuff). For current endpoint details, Codex fetches `https://massive.com/docs/rest/llms-full.txt`. Same strategy as the `massive-ai-rules` repo — drift across tool-specific files drops to near-zero because the website is the authoritative source.

## References

- [Codex Plugins overview](https://developers.openai.com/codex/plugins)
- [Build plugins – Codex](https://developers.openai.com/codex/plugins/build)
- [Agent Skills – Codex](https://developers.openai.com/codex/skills)
- [OpenAI Skills Catalog](https://github.com/openai/skills)
- [AGENTS.md spec](https://agents.md/)
- [Source repo: Massive Claude Code plugin](https://github.com/massive-com/claude-code-plugin)
