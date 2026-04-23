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

### 1. Install the Massive MCP server (optional)

For live endpoint verification, install the [Massive MCP server](https://github.com/massive-com/mcp_massive) once as a shared `uv` tool. It's reused across Codex, Claude Code, and any other MCP-aware harness:

```bash
uv tool install git+https://github.com/massive-com/mcp_massive
```

Upgrade later with `uv tool upgrade mcp-massive`. For advanced install options, see the [mcp_massive repo](https://github.com/massive-com/mcp_massive).

Skip steps 1 and 2 if you don't need live-API features. The plugin's knowledge and skills still work standalone.

### 2. Register the MCP server with Codex (optional)

```bash
read -rsp "Massive API key: " MASSIVE_API_KEY; echo
codex mcp add massive --env MASSIVE_API_KEY="$MASSIVE_API_KEY" -- mcp_massive
unset MASSIVE_API_KEY
```

This writes the server entry into `~/.codex/config.toml`. The key persists across shells and reboots. Rotate the key later with `codex mcp remove massive` followed by the same `add` command with the new value.

The plugin itself does not prompt for or store a Massive API key during install. Live API features use the separately registered MCP server and its `MASSIVE_API_KEY` environment value.

### 3. Install the plugin

Git-backed install once the repo is public:

```bash
codex plugin marketplace add github.com/massive-com/codex-plugin
```

For local development from a checkout of this repo:

```bash
codex plugin marketplace add .
```

Then restart Codex, open the plugin browser with `/plugins`, and install `Massive Market Data` from the `Massive Market Data` marketplace.

### 4. Verify

Inside a Codex session:

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
- [uv](https://docs.astral.sh/uv/), to install the Massive MCP server (only needed for live-API features)
- Python 3.12+ (required by the MCP server; the Python SDK itself supports 3.9+)
- A Massive API key from [massive.com/dashboard](https://massive.com/dashboard)

## License

MIT.

## Disclaimer

Educational material, not investment advice or a recommendation to buy or sell any security. Massive is a market data provider, not a broker-dealer, exchange, or investment adviser. Market data may originate from third-party exchanges and data providers, or may be derived or calculated by Massive; in either case, it is subject to the terms of your Massive subscription. The data and code samples are provided as-is, without warranty of accuracy, completeness, or timeliness. You're responsible for your use of the data and for compliance with all applicable laws and data licensing terms.
