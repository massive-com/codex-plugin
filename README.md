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

Validated on `2026-04-23` with `codex-cli 0.123.0`.

### Install the plugin

Git-backed install once the repo is public:

```bash
codex plugin marketplace add massive-com/codex-plugin
```

For local development from a checkout of this repo:

```bash
codex plugin marketplace add .
```

Then restart Codex, open the plugin browser with `/plugins`, and install `Massive Market Data` from the `Massive Market Data` marketplace.

### MCP setup

The plugin bundles the Massive MCP server declaration. Installing the plugin registers the `massive` MCP automatically; the server itself is fetched and cached by `uvx` on first launch. You only need to provide your API key.

**Prerequisite:** `uv` must be installed and on your `PATH` (see [Requirements](#requirements)).

After installing the plugin, register your Massive API key so the MCP can authenticate. This writes a user-level override to `~/.codex/config.toml` that takes precedence over the plugin's placeholder:

```bash
# macOS / Linux
codex mcp add massive --env MASSIVE_API_KEY=YOUR_MASSIVE_API_KEY -- uvx --from git+https://github.com/massive-com/mcp_massive mcp_massive
```

```powershell
# Windows (PowerShell)
codex mcp add massive --env MASSIVE_API_KEY=YOUR_MASSIVE_API_KEY -- uvx --from git+https://github.com/massive-com/mcp_massive mcp_massive
```

The key persists across shells and reboots. Rotate it later with `codex mcp remove massive` followed by the same `add` command with the new value.

If you prefer to set the key via your shell environment instead (useful when running `codex` from a terminal where `MASSIVE_API_KEY` is already exported), the bundled `.mcp.json` uses `env_vars: ["MASSIVE_API_KEY"]` and will pick up the shell value automatically. On macOS GUI launches of Codex, use `launchctl setenv MASSIVE_API_KEY your_key` to make the value visible to the app.

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

- OpenAI Codex CLI
- [uv](https://docs.astral.sh/uv/), used by the plugin's bundled `.mcp.json` to launch the Massive MCP server via `uvx`. Check with `uv --version`. Install with `curl -LsSf https://astral.sh/uv/install.sh | sh` (macOS or Linux), `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"` (Windows), or `pip install uv` (any platform).
- Python 3.12+ (fetched automatically by `uvx` on first MCP launch if not already present)
- A Massive API key from [massive.com/dashboard](https://massive.com/dashboard)

## License

MIT.

## Disclaimer

Educational material, not investment advice or a recommendation to buy or sell any security. Massive is a market data provider, not a broker-dealer, exchange, or investment adviser. Market data may originate from third-party exchanges and data providers, or may be derived or calculated by Massive; in either case, it is subject to the terms of your Massive subscription. The data and code samples are provided as-is, without warranty of accuracy, completeness, or timeliness. You're responsible for your use of the data and for compliance with all applicable laws and data licensing terms.
