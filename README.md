<img src="images/logo_new.png" alt="Massive Market Data" width="100%"/>

# Massive Market Data Codex Plugin

A Codex plugin for building and debugging integrations against the Massive market data API. You get five workflow skills, the Massive API knowledge file, and the Massive MCP server — all in one install.

## What you get

Five skills for common integration tasks:

- `massive-scaffold` — new Massive project with SDK dependencies, `.env`, and working boilerplate
- `massive-discover` — find the right endpoint, parameters, plan tier, and SDK example for a data task
- `massive-debug` — diagnose API errors, empty results, rate limits, and SDK misuse
- `massive-options` — options strategies and analysis using Massive options data
- `massive-dashboard` — scaffold a Streamlit dashboard backed by Massive APIs

An `AGENTS.md` knowledge file covering ticker formats, SDK naming, pagination, auth, plan tiers, and the common pitfalls.

The Massive MCP server, bundled via `uvx`. It exposes three tools to Codex: `search_endpoints`, `call_api`, and `query_data`. Codex uses them for live endpoint discovery, API calls, and SQL over stored DataFrames. If the server can't launch, the skills fall back to `AGENTS.md` and say so.

## Install

Three steps. No restarts.

### Before you start

- [**OpenAI Codex CLI**](https://developers.openai.com/codex/quickstart) drives the two `codex` commands below. Install with `npm install -g @openai/codex` or `brew install codex`. Runs on macOS, Windows, and Linux.
- **OpenAI Codex desktop app** is where you install the plugin and chat. Available on macOS and Windows from [developers.openai.com/codex](https://developers.openai.com/codex/quickstart). Linux users can use the Codex TUI (`codex`) instead — see the note at the end.
- [**uv**](https://docs.astral.sh/uv/) launches the bundled MCP server. Install with `curl -LsSf https://astral.sh/uv/install.sh | sh` (macOS or Linux), `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"` (Windows), or `pip install uv` (any platform). Python 3.12+ is fetched automatically by `uvx` on first MCP launch.
- A [Massive API key](https://massive.com/dashboard). The free Basic tier is enough to start.

### 1. Add the marketplace

```bash
codex plugin marketplace add massive-com/codex-plugin
```

For local development from a checkout of this repo, use `codex plugin marketplace add .` instead.

### 2. Install the plugin from the Codex app

Open the Codex desktop app, go to the plugins screen, find **Massive Market Data** in the drop down, and Install or add it to Codex.

### 3. Register your API key if using the MCP server.

```bash
codex mcp add massive --env MASSIVE_API_KEY=YOUR_MASSIVE_API_KEY -- uvx --from git+https://github.com/massive-com/mcp_massive mcp_massive
```

This writes a user-level entry to `~/.codex/config.toml` that overrides the plugin's placeholder with your key. The next chat you start will pick it up — the MCP list in the app may not refresh until you reopen the application the next time, but calls from chat work right away. Rotate later with `codex mcp remove massive` followed by the same `add` command with the new value.

*If you'd rather keep the key in your shell environment, skip this step. The bundled MCP reads `MASSIVE_API_KEY` via `env_vars`. For the macOS desktop app, run `launchctl setenv MASSIVE_API_KEY your_key` so the GUI inherits the value.*

*Linux (TUI) users: after step 1, run `codex`, type `/plugins`, install **Massive Market Data**, then `/exit` to leave cleanly before running step 3.*

### Verify

```bash
codex mcp list
```

Should show `massive` as `enabled`. In the Codex app, start a new chat and try:

> Use $massive-discover to find the right Massive endpoint for daily AAPL aggregates in Python.

If the MCP is working, Codex will call `search_endpoints` against the live server. If it isn't, the skill falls back to `AGENTS.md` and tells you so.

## Accuracy model

`AGENTS.md` holds stable facts (ticker formats, SDK conventions, plan tiers) that don't change often. The MCP server is the source of truth for live endpoint metadata, parameters, and response shapes — skills prefer it over memory. When live lookup fails, skills are expected to say so, not invent routes or fields. Skill names are prefixed `massive-` to avoid collisions with generic skills from other plugins.

## Local development

Codex caches an installed copy of the plugin rather than reading your working tree. If you edit skills, manifests, or other install-surface files, bump the version in `plugins/massive/.codex-plugin/plugin.json` and reinstall via `/plugins`. That's the reliable way to get Codex to pick up your changes.

## License

MIT.

## Disclaimer

Educational material, not investment advice or a recommendation to buy or sell any security. Massive is a market data provider, not a broker-dealer, exchange, or investment adviser. Market data may originate from third-party exchanges and data providers, or may be derived or calculated by Massive; in either case, it is subject to the terms of your Massive subscription. The data and code samples are provided as-is, without warranty of accuracy, completeness, or timeliness. You're responsible for your use of the data and for compliance with all applicable laws and data licensing terms.
