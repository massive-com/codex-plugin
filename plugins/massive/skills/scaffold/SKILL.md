---
name: massive-scaffold
description: Scaffold a new Massive API project with dependency files, .env setup, and boilerplate code. Use when creating a new app, demo, or example that will call Massive APIs. Do not use for unrelated project scaffolding.
---

# Scaffold a new Massive project

Create a new project called `$0` of type `$1` (default: `rest` if not specified).
Language: `$2` (default: `python` if not specified). Infer from context if the user mentions a language or SDK.

Prefer official Massive SDK packages and methods from this skill and `AGENTS.md`. Do not invent unofficial packages, endpoints, or auth flows.

**Keep the entry point minimal.** Copy the example from the language reference as-is. Do not add env-var configuration tables (`MARKETS`, `FEEDS`, subscription parsers), helper utilities (`require_env`, `message_to_dict`, `first_available`, `choose_option`), or abstractions the user didn't ask for. A scaffold is a starting point, not a framework — the user will add what they need.

## Project types

- **rest** (default): CLI script for data fetching and analysis.
- **websocket**: Real-time streaming script with a handler callback.
- **streamlit**: Interactive dashboard with Plotly charts and cached API calls (Python only).

## Plan tier check (run BEFORE scaffolding)

Surface the minimum plan requirement to the user **before** writing any files. This prevents the frustrating case where a user scaffolds a WebSocket project on the Basic tier and cannot connect.

- **rest** on Basic (free): works for end-of-day aggregates, reference data, technical indicators. 5 calls/min cap.
- **websocket**: requires **Starter plan ($29-49/mo)** minimum. Basic tier cannot connect to any WebSocket feed. Confirm the user has Starter or above before scaffolding.
- **streamlit**: the default template uses snapshots, which require **Starter plan minimum**. Warn the user; offer to use end-of-day aggregates only if they are on Basic.

If the user requests `websocket` or `streamlit` and has not indicated their plan, ask one clarifying question: "This project type requires a Starter plan or above ($29-49/mo). Are you on Starter or higher?" If they say no, either:

1. Scaffold anyway with a clear README note about the plan requirement, or
2. Suggest a `rest` project on Basic with end-of-day aggregates.

## Language-specific setup

After resolving language and project type, read the matching reference for SDK packages, dependency files, entry-point code, and SDK quirks:

- **Python** (all project types including streamlit) → [references/python.md](references/python.md)
- **JavaScript / TypeScript** → [references/javascript.md](references/javascript.md)
- **Go** → [references/go.md](references/go.md)
- **Kotlin / JVM** → [references/kotlin.md](references/kotlin.md)

Only read the reference for the chosen language.

## Required files (all languages)

### .env.example

```
MASSIVE_API_KEY=your_api_key_here
```

### .gitignore

Always include:

```
.env
data/
```

Plus language-specific entries listed in the language reference.

### README.md

1. One-line description of what the project does
2. Quickstart: `cp .env.example .env`, add your key, install deps, run
3. Link to Massive docs: https://massive.com/docs

## Plan guidance

After scaffolding, remind the user which plan tier they will need:

- **Basic (free):** Enough for testing with end-of-day aggregates, reference data, technical indicators. Limited to 5 calls/min.
- **Starter ($29-49/mo):** Required for WebSocket streaming, snapshots, flat files, and second aggregates. Options Starter adds Greeks/IV.
- **Developer ($79/mo):** Required for trade data access.
- **Advanced ($99-199/mo):** Required for real-time data, quotes, and financials/ratios (Stocks Advanced).
- **Business ($999+/mo):** Required for commercial use, building products, or redistributing data.

For WebSocket and Streamlit project types, the user will need at least a Starter plan.

## Steps

1. **Plan check first.** For `websocket` or `streamlit` types, confirm the user is on Starter or above (see "Plan tier check" above). Do not silently scaffold a project that will fail on their plan.
2. Resolve language and project type, then read the matching language reference listed above.
3. Create the project directory: `$0/`.
4. Write the dependency file, entry point, `.env.example`, `.gitignore`, and `README.md` using the language reference.
5. Confirm the structure and provide the quickstart:

   ```
   cd $0
   cp .env.example .env
   # Add your Massive API key to .env
   ```

   Then the language-specific install and run commands from the reference.
6. State the minimum plan tier needed for the project type (repeat even if already mentioned in step 1).
7. Point to follow-up skills: `$massive-discover` to find endpoints, `$massive-debug` if errors arise.
