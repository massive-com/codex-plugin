<img src="images/logo_new.png" alt="Massive" width="100%"/>

# Massive Codex Plugin

This plugin helps Codex and similar coding assistants use Massive correctly when you're building or debugging integrations. It bundles Massive-specific guidance, reusable workflows, and MCP-based API lookup so the model can check the current API instead of guessing.

## What ships

When you install this plugin, Codex gets:

- **Stable Massive coding guidance via `AGENTS.md`.** Ticker formats, SDK naming, pagination behavior, auth patterns, plan tiers, and common pitfalls.
- **Live API grounding via MCP.** Through the configured [Massive MCP server](https://github.com/massive-com/mcp_massive), Codex can check current endpoint docs, validate parameters and response fields, and test real calls.
- **Five focused skills.**

| Skill | Use it for |
|---|---|
| `massive-scaffold` | Create a Massive project with the right dependencies, `.env` setup, and working boilerplate. |
| `massive-discover` | Find the right Massive endpoint, parameters, plan tier, and SDK example for a data task. |
| `massive-debug` | Diagnose Massive API errors, empty results, rate limits, and SDK misuse. |
| `massive-options` | Build options strategies and analysis projects using Massive options data. |
| `massive-dashboard` | Scaffold a Streamlit dashboard backed by Massive APIs. |

## How It Stays Accurate

This plugin is set up so Codex relies less on memory and more on real Massive sources:

- Use `plugins/massive/AGENTS.md` for stable facts that do not change often.
- Use the bundled Massive MCP server for current endpoint, parameter, and response-shape details instead of relying on memory.
- If live endpoint lookup is unavailable, the skills should say so rather than inventing current routes, fields, or plan access details.
- Skill names are prefixed with `massive-` to reduce selector collisions with other plugins' generic skills.

## When It Helps

Use this when you want the coding assistant to help with implementation, not just answer questions.

- Use it to help generate Massive client code that matches the real SDK and API surface.
- Use it to pick the right endpoint and parameters for a feature before writing application code.
- Use it to debug 401, 403, 404, rate limit, pagination, and empty-result problems faster.
- Use it to validate assumptions against live Massive tooling when response fields or endpoint behavior matter.

The point is not "show live prices in Codex." The point is to help the coding assistant write and debug Massive integrations more reliably.

## Install locally

Validated on `2026-04-23` with `codex-cli 0.123.0`.

1. Export your Massive API key:

```bash
export MASSIVE_API_KEY=your_api_key_here
```

2. Add this repository as a Codex plugin marketplace from the repo root:

```bash
codex plugin marketplace add .
```

3. Restart Codex.
4. Open the plugin browser:

```text
codex
/plugins
```

5. Install `Massive` from the `Massive` marketplace, then start a new thread.

## How Codex Uses It

This repository does not contain or deploy the `mcp_massive` implementation itself. What it does package is the Codex-side wiring:

- `.agents/plugins/marketplace.json` exposes the plugin to Codex as a marketplace entry.
- `plugins/massive/.codex-plugin/plugin.json` tells Codex that this plugin bundles skills and MCP configuration.
- `plugins/massive/.mcp.json` tells Codex how to launch the Massive MCP server and which environment variable it needs.

The actual activation flow in Codex is:

1. `codex plugin marketplace add .` registers the marketplace.
2. You install `Massive` from `/plugins`. Registering the marketplace alone is not enough.
3. Codex copies the installed plugin into its plugin cache and records the plugin as enabled.
4. In a new thread, Codex can use the bundled skills and make the Massive MCP tools available.
5. When a task needs those tools, Codex launches the MCP command from `.mcp.json`.

So yes, this repo should be responsible for activating Massive inside Codex in the configuration sense, by shipping the marketplace entry, plugin manifest, and `.mcp.json`. It should not be responsible for hosting or deploying the upstream `mcp_massive` server implementation itself.

## Install from GitHub

Once this repo is published with the marketplace file at the repository root, the Git-backed install flow should be:

```bash
codex plugin marketplace add github.com/massive-com/codex-plugin
```

Then restart Codex and install the `Massive` plugin from `/plugins`.

## Local development note

Codex uses an installed copy of the plugin from its plugin cache rather than reading your working tree live during normal use. If you change public skill names, manifests, or other install-surface files, reinstall the plugin from `/plugins`. In practice, bumping the plugin version is the safest way to make sure Codex refreshes the installed copy.

## Quick smoke test

After install, try prompts like:

- `Use $massive-discover to find the right Massive endpoint for daily AAPL aggregates in Python.`
- `Use $massive-scaffold to create a new Massive project. Args: demo rest python`
- `Use $massive-debug to diagnose a Massive API error or unexpected response.`
- `Build a Massive integration for daily aggregates in TypeScript and verify the endpoint and fields before writing the code.`

## Requirements

- OpenAI Codex CLI
- [uv](https://docs.astral.sh/uv/) for `uvx`
- Python 3.12+ for the bundled MCP server
- A Massive API key from [massive.com/dashboard](https://massive.com/dashboard)

## Status

Validated so far:

- local marketplace registration works
- plugin install and MCP use work in an installed Codex profile
- Massive MCP tools successfully answered live `search_endpoints`, `get_endpoint_docs`, and `call_api` requests during smoke testing

Remaining work is mostly publish polish, screenshots/icons, and a final reinstall check after the `0.2.0` version bump.

## License

MIT.

## Disclaimer

Educational material, not investment advice or a recommendation to buy or sell any security. Massive is a market data provider, not a broker-dealer, exchange, or investment adviser. Market data may originate from third-party exchanges and data providers, or may be derived or calculated by Massive; in either case, it is subject to the terms of your Massive subscription. The data and code samples are provided as-is, without warranty of accuracy, completeness, or timeliness. You're responsible for your use of the data and for compliance with all applicable laws and data licensing terms.
