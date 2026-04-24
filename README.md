<img src="images/logo_new.png" alt="Massive Market Data" width="100%"/>

# Massive Market Data Codex Plugin

A Codex plugin for building and debugging Massive market data integrations. It packages Massive API guidance and reusable workflows, with optional live endpoint verification through the separately installed Massive MCP server.

## Included

The plugin includes:

- **Massive API guidance via `AGENTS.md`.** Ticker formats, SDK naming, pagination behavior, auth patterns, plan tiers, and common pitfalls.
- **Five focused workflows.**

| Skill | Use it for |
|---|---|
| `massive-scaffold` | Create a Massive project with the right dependencies, `.env` setup, and working boilerplate. |
| `massive-discover` | Find the right Massive endpoint, parameters, plan tier, and SDK example for a data task. |
| `massive-debug` | Diagnose Massive API errors, empty results, rate limits, and SDK misuse. |
| `massive-options` | Build options strategies and analysis projects using Massive options data. |
| `massive-dashboard` | Scaffold a Streamlit dashboard backed by Massive APIs. |

When paired with the [Massive MCP server](https://github.com/massive-com/mcp_massive) (installed separately, see Install below), Codex can use three live tools, `search_endpoints`, `call_api`, and `query_data`, for endpoint discovery, live calls, and SQL over stored DataFrames. The workflows list these tools as optional capabilities; without the MCP server, they fall back to the repo guidance in `AGENTS.md`.

## Accuracy Model

This repo separates stable integration guidance from live endpoint metadata:

- Use `plugins/massive/AGENTS.md` for stable facts that do not change often.
- Use the Massive MCP server (when registered) for current endpoint, parameter, and response-shape details instead of relying on memory.
- If live endpoint lookup is unavailable, workflows should say so rather than inventing current routes, fields, or plan access details.
- Skill names are prefixed with `massive-` to reduce selector collisions with other plugins' generic skills.

## Common Workflows

Use this plugin when implementation details matter:

- Generate Massive client code that matches the real SDK and API surface.
- Pick the right endpoint and parameters for a feature before writing application code.
- Debug 401, 403, 404, rate limit, pagination, and empty-result problems.
- Validate assumptions against live Massive tooling when response fields or endpoint behavior matter.

This is an integration-development plugin. Runtime market data access comes through your Massive API key and optional MCP registration.

## Install

Validated on `2026-04-24` with `codex-cli 0.123.0`. The flow is fully CLI-driven until the final launch — no mid-flow restarts.

**Prerequisites** (see full [Requirements](#requirements) below):
- **Codex CLI** — every install command below runs through it.
- **Codex desktop app** — where you use the plugin day-to-day (macOS/Windows). Linux users can use the Codex TUI instead.
- **uv** — installed and on your `PATH`. The plugin's bundled MCP is launched via `uvx`, which fetches and caches `mcp_massive` on first use.

### Step 1 — Add the marketplace

```bash
codex plugin marketplace add massive-com/codex-plugin
```

For local development from a checkout of this repo:

```bash
codex plugin marketplace add .
```

### Step 2 — Install the plugin

Launch the Codex TUI and use the plugin browser:

```bash
codex
```

Then type `/plugins`, select `Massive Market Data`, and press Install. When the install finishes, type `/exit` to leave the TUI (Ctrl-C or closing the terminal is not enough — `/exit` flushes Codex's state cleanly).

### Step 3 — Register your Massive API key

This writes a user-level override to `~/.codex/config.toml` that wins over the plugin's placeholder MCP entry:

```bash
# macOS / Linux / Windows (PowerShell) — same command
codex mcp add massive --env MASSIVE_API_KEY=YOUR_MASSIVE_API_KEY -- uvx --from git+https://github.com/massive-com/mcp_massive mcp_massive
```

The key persists across shells and reboots. Rotate it later with `codex mcp remove massive` followed by the same `add` command with the new value.

### Step 4 — Open Codex

Launch the Codex desktop app. On first open after the three CLI steps above, the plugin and MCP server both load together — no second open required. First MCP launch takes 5-10 seconds while `uvx` downloads and caches `mcp_massive`; subsequent launches start in ~1 second.

**Alternate key path:** if you'd rather supply `MASSIVE_API_KEY` via your shell environment, skip step 3. The bundled `.mcp.json` uses `env_vars: ["MASSIVE_API_KEY"]` and picks up the exported value. For macOS GUI launches that don't inherit your shell env, use `launchctl setenv MASSIVE_API_KEY your_key` so the Codex app can see it.

The plugin does not prompt for or store a Massive API key during install.

### Verify

Check the MCP server separately:

```bash
codex mcp list
```

The `massive` server should be enabled. In a new Codex session, the MCP tools are available as Massive tools such as `search_endpoints`, `call_api`, and `query_data`.

Check the plugin separately:

- Invoke a skill: `Use $massive-discover to find the right Massive endpoint for daily AAPL aggregates in Python.`
- If MCP is registered, Codex verifies endpoint metadata against the live server via `search_endpoints`.
- If MCP is not registered, Codex answers from AGENTS.md knowledge and notes that it couldn't verify the latest metadata.

## Quick smoke test

After install, try prompts like:

- `Use $massive-discover to find the right Massive endpoint for daily AAPL aggregates in Python.`
- `Use $massive-scaffold to create a new Massive project. Args: demo rest python`
- `Use $massive-debug to diagnose a Massive API error or unexpected response.`
- `Build a Massive integration for daily aggregates in TypeScript and verify the endpoint and fields before writing the code.`

## Store readiness

The manifest includes the install-surface metadata documented by OpenAI Codex plugins: display copy, website/privacy/terms links, default prompts, brand color, a composer icon, and the black Massive wordmark logo. A white wordmark variant is also stored in `plugins/massive/assets/` for future dark-surface use. Real screenshots should be added under `plugins/massive/assets/` and referenced from `plugins/massive/.codex-plugin/plugin.json` once you have screenshots that show Massive Market Data in use.

## Local development note

Codex uses an installed copy of the plugin from its plugin cache rather than reading your working tree live during normal use. If you change public skill names, manifests, or other install-surface files, reinstall the plugin from `/plugins`. Bumping the plugin version is the safest way to make sure Codex refreshes the installed copy.

## Requirements

- **[OpenAI Codex CLI](https://developers.openai.com/codex/quickstart)** — required for every install command below. Install with `npm install -g @openai/codex` or `brew install codex`. Runs on macOS, Windows, and Linux.
- **OpenAI Codex desktop app** — where the plugin's UI, skills, and MCP tools run in your day-to-day sessions. Download from [developers.openai.com/codex](https://developers.openai.com/codex/quickstart). Available on macOS and Windows; Linux users can use the CLI TUI (`codex` in a terminal) instead.
- [uv](https://docs.astral.sh/uv/), used by the plugin's bundled `.mcp.json` to launch the Massive MCP server via `uvx`. Check with `uv --version`. Install with `curl -LsSf https://astral.sh/uv/install.sh | sh` (macOS or Linux), `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"` (Windows), or `pip install uv` (any platform).
- Python 3.12+ (fetched automatically by `uvx` on first MCP launch if not already present)
- A Massive API key from [massive.com/dashboard](https://massive.com/dashboard)

## License

MIT.

## Disclaimer

Educational material, not investment advice or a recommendation to buy or sell any security. Massive is a market data provider, not a broker-dealer, exchange, or investment adviser. Market data may originate from third-party exchanges and data providers, or may be derived or calculated by Massive; in either case, it is subject to the terms of your Massive subscription. The data and code samples are provided as-is, without warranty of accuracy, completeness, or timeliness. You're responsible for your use of the data and for compliance with all applicable laws and data licensing terms.
