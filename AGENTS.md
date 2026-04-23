# AGENTS.md

## Repository expectations

- Treat the OpenAI Codex docs as the source of truth for plugin packaging, skill metadata, and `AGENTS.md` conventions.
- Before calling the local install flow complete, validate marketplace registration with `CODEX_HOME=$(mktemp -d) codex plugin marketplace add .`.
- Keep install-facing copy accurate. Do not leave placeholder commands in `README.md` or manifest prompts.
- Keep each skill focused on one job. When changing a skill name or trigger description, update `SKILL.md`, `agents/openai.yaml`, `README.md`, and plugin prompts together.
- Prefer live Massive MCP and official Massive docs for current endpoint details. Do not document guessed endpoints, parameters, or plan access rules.
