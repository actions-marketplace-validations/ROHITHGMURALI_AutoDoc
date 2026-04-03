---
audit_author: SwarmWorker_002
audit_date: 2025-07-18
audit_version: 8c3e6f2b9a1d4e7c5f0a3b8d2e6f1a4c9d7e0b3a
---
# AutoDoc Agent Swarm — README

## Overview
The AutoDoc Agent Swarm (DeepAgents Edition) is an advanced, hierarchical multi-agent system built with [LangChain](https://github.com/langchain-ai/langchain) and [DeepAgents](https://github.com/deepagents/deepagents). It autonomously analyzes, templates, generates, and verifies documentation for any given code repository. The system orchestrates three agent roles — the Swarm Queen (orchestrator), the Swarm Worker (technical writer), and the Swarm Drone (QA evaluator) — to produce high-quality Markdown documentation enriched with PlantUML diagrams while maintaining strict schema compliance.

## Features

- **Hierarchical Swarm Architecture**: Clear separation of concerns across Planning (Queen), Writing (Worker), and Reviewing (Drone) agent roles.
- **Multi-Provider LLM Support**: Configure the swarm to use OpenRouter, Anthropic, Google, or OpenAI out-of-the-box.
- **Per-Agent Model Configuration**: Assign distinct, optimized models for each agent — e.g., use a high-intelligence model for the Queen and faster/cheaper models for the Worker and Drone.
- **Secure Filesystem Backend**: Implements a strict `SecureFilesystemBackend` preventing the LLM from inadvertently accessing or leaking sensitive data (`.env`, `.pem`, secrets files, `.git`, `node_modules`).
- **Smart Incremental Updates**: Built-in freshness checks so the Swarm only documents files that have changed since the last run, saving LLM context window and costs.
- **Mirrored Output Directory**: Generated documentation elegantly mirrors the exact directory structure of the source code under a `documentation/` folder.

## Architecture Diagram
@startuml
actor Developer as D
participant GitHub_Action_or_CLI as CLI
participant Swarm_Queen as Queen
database Codebase as Repo
participant Swarm_Worker as Worker
participant SecureFilesystemBackend as FSBackend
participant Swarm_Drone as Drone

D -> CLI : Triggers (push event or local command)
CLI -> Queen : Initialize swarm with target_dir & model config
Queen -> Repo : ls / read_file (filtered scan)
Queen -> Queen : Evaluate freshness (OS mtime comparison)
Queen -> Queen : Build task plan (files needing docs)

loop For each stale file
  Queen -> Worker : Delegate documentation task
  Worker -> Repo : Read source code
  Worker -> FSBackend : Write draft Markdown + PlantUML
  Worker -> Queen : Draft complete

  Queen -> Drone : Delegate verification
  Drone -> FSBackend : Read draft document
  Drone -> Queen : Return PASS or FAIL with feedback

  alt FAIL (retries < 3)
    Queen -> Worker : Reassign with Drone feedback
  else FAIL (max retries reached)
    Queen -> D : Escalate to human intervention
  else PASS
    Queen -> Queen : Mark task complete
  end
end

Queen -> CLI : All tasks complete
CLI -> D : Docs committed to documentation/ directory
@enduml

## Installation
This project uses `uv` for lightning-fast dependency management.

1. **Install `uv`** (if not already installed):
   Follow instructions at https://github.com/astral-sh/uv to install uv.

2. **Clone and Initialize**:
   ```bash
   git clone <your-repo-url>
   cd autodoc-swarm
   uv sync
   ```

3. **Configure Environment Variables**:
   Copy the example environment file and add your preferred API keys:
   ```bash
   cp .env.example .env
   ```
   Edit `.env` to include your `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, etc. Optional LangSmith tracing configurations are also included in the example file.

## GitHub Action Usage

### Prerequisites

1. **API key secret** — Add your LLM provider API key to your repository secrets:
   Go to `Settings → Secrets and variables → Actions → New repository secret` and create the appropriate secret (e.g., `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_API_KEY`).

2. **Token with write access** — The action commits generated docs back to the branch. You can use either:
   - **Personal Access Token (PAT)** — Store a PAT as a secret (e.g., `PAT_TOKEN`). This is recommended because commits made with a PAT will trigger downstream workflow runs.
   - **`secrets.GITHUB_TOKEN`** — The built-in GitHub token. Simpler to use, but commits made with it will not trigger further workflow runs.

3. **Workflow permissions** — The job must have `contents: write` permission to allow the action to commit and push documentation back to the repository.

### Minimal Setup

Create `.github/workflows/autodoc.yml` in your repository:

```yaml
name: AutoDoc

on:
  push:
    branches:
      - main

jobs:
  autodoc:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # required for smart diffing

      - uses: ROHITHGMURALI/AutoDoc@main
        with:
          github_token: ${{ secrets.PAT_TOKEN }}
          api_key: ${{ secrets.OPENROUTER_API_KEY }}
          provider: openrouter
          queen_model: anthropic/claude-3.5-sonnet
          worker_model: anthropic/claude-3.5-sonnet
          drone_model: anthropic/claude-3.5-sonnet
          target_dir: ./
```

### All Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `github_token` | Yes | — | Token used to commit and push generated docs back to the branch |
| `api_key` | Yes | — | API key for the chosen LLM provider |
| `provider` | Yes | `openrouter` | LLM provider: `openrouter`, `anthropic`, `openai`, or `google` |
| `queen_model` | Yes | `anthropic/claude-3.5-sonnet` | Model for the Swarm Queen (orchestrator) |
| `worker_model` | Yes | `anthropic/claude-3.5-sonnet` | Model for the Swarm Worker (writer) |
| `drone_model` | Yes | `anthropic/claude-3.5-sonnet` | Model for the Swarm Drone (QA reviewer) |
| `target_dir` | Yes | `.` | Directory within the repo to scan and document |

### Provider & Model Examples

**OpenRouter (cheapest for testing):**
```yaml
provider: openrouter
queen_model: anthropic/claude-3.5-sonnet
worker_model: qwen/qwen3.6-plus:free
drone_model: qwen/qwen3.6-plus:free
api_key: ${{ secrets.OPENROUTER_API_KEY }}
```

**Anthropic direct:**
```yaml
provider: anthropic
queen_model: claude-opus-4-6
worker_model: claude-sonnet-4-6
drone_model: claude-haiku-4-5-20251001
api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

**OpenAI:**
```yaml
provider: openai
queen_model: gpt-4o
worker_model: gpt-4o-mini
drone_model: gpt-4o-mini
api_key: ${{ secrets.OPENAI_API_KEY }}
```

**Google:**
```yaml
provider: google
queen_model: gemini-2.0-flash
worker_model: gemini-2.0-flash
drone_model: gemini-2.0-flash
api_key: ${{ secrets.GOOGLE_API_KEY }}
```

### How It Works

1. On every push, the action checks which files changed (smart diffing via `git diff`).
2. The Swarm Queen scans the `target_dir`, evaluates documentation freshness, and builds a task list.
3. The Swarm Worker generates Markdown documentation with PlantUML diagrams for each file.
4. The Swarm Drone reviews each doc and returns `PASS` or `FAIL` with feedback. Failed docs are retried up to 3 times.
5. All generated docs are committed back to the same branch under `<target_dir>/documentation/`, mirroring the source tree structure.

### Output Structure

Given a `target_dir` of `./src`, generated docs appear at:
```
src/
├── api/
│   └── routes.py
documentation/
└── src/
    └── api/
        └── routes.md
```

## CLI Usage

A Typer CLI is provided for running the swarm locally during development or for ad-hoc documentation passes.

### Basic Execution

```bash
uv run python run_swarm.py --target ./src
```

### Advanced Execution

```bash
uv run python run_swarm.py \
    --target ./my_backend_service \
    --provider openai \
    --queen-model gpt-4o \
    --worker-model gpt-4o-mini \
    --drone-model gpt-4o-mini
```

### CLI Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `--target` | `str` | `.` | Target repository path to scan and document |
| `--force-update` | `bool` | `False` | Force update all documentation, ignoring freshness checks (currently simulated via prompt adjustments) |
| `--provider` | `str` | `openrouter` | LLM provider: `openrouter`, `anthropic`, `openai`, or `google` |
| `--queen-model` | `str` | `anthropic/claude-3.5-sonnet` | Model name for the Swarm Queen (orchestrator) |
| `--worker-model` | `str` | `anthropic/claude-3.5-sonnet` | Model name for the Swarm Worker (technical writer) |
| `--drone-model` | `str` | `anthropic/claude-3.5-sonnet` | Model name for the Swarm Drone (QA reviewer) |

## Development & Testing

To run the integration and unit test suite:

```bash
uv run pytest
```

Tests include validations for:
- The `SecureFilesystemBackend` successfully blocking access to `.env` files.
- The `check_file_freshness` logic accurately assessing modification timestamps.
- Successful initializations and distinct LLM assignments for the Queen and subagents.
- Documentation filesystem mirroring behavior.

## Functions / Methods

The README itself does not define programmatic functions or methods; it documents the user-facing interfaces and configuration contract of the AutoDoc Agent Swarm system. The key entry points are:

**GitHub Action Entry Point (`entrypoint.py` / `action.yml`)**
- **Description:** AutoDoc Swarm triggers on `push` events (or `pull_request`) and runs as a GitHub Action via `ROHITHGMURALI/AutoDoc@main`. It uses a Docker container to execute the swarm, with inputs passed as environment variables.
- **Key internal functions in `entrypoint.py`:**
  - `get_changed_files(target_dir)` — Returns a list of files changed in the current PR or commit for smart diffing. Files under `documentation/` and common ignore-files are excluded.
  - `configure_git()` — Configures Git user name/email and sets the safe directory for the GitHub workspace.
  - `commit_and_push(target_dir)` — Stages, commits, and pushes generated documentation to the target branch using the provided token.
  - `write_step_summary(content)` — Appends content to the GitHub Actions step summary file.
  - `main()` — Orchestrates the GitHub Action flow: reads inputs, runs the swarm, and commits results.
- **Return Type:** Generated Markdown documentation is committed to the same branch under `<target_dir>/documentation/`, mirroring the source tree structure.

**CLI Entry Point — `run_swarm.py`**
- **Description:** A Typer-based CLI for running the swarm locally during development or ad-hoc documentation passes.
- **Key function:** `run()` — The single Typer command that initializes the swarm with the provided configuration, invokes the Queen agent with a kick-off message, and outputs the final result.
- **Parameters:** See the CLI Parameters table above.
- **Return Type:** Generated Markdown documentation written to `<target_dir>/documentation/` with the mirrored directory structure.
