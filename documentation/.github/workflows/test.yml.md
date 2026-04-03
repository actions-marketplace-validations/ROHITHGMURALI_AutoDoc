---
audit_author: SwarmWorker_GitHubActions
audit_date: 2025-07-10
audit_version: 3b1c5e9
---
# Test AutoDoc Agent Swarm Workflow

## Overview
This GitHub Actions workflow (`test.yml`) automates the testing of the AutoDoc Agent Swarm. It is triggered on pushes to the `main` branch and on all pull requests. The workflow performs two main steps: it checks out the repository with full git history (required for smart diffing), and then runs the AutoDoc Swarm using a composite action defined at the repository root. The swarm is configured to use the OpenRouter provider with the `qwen/qwen3.6-plus:free` model for all agent roles (Queen, Worker, and Drone), authenticating via a GitHub Personal Access Token and an OpenAI API key.

## Architecture Diagram
@startuml
start

:Trigger;
note right
  - push to main
  - any pull_request
end note

:Step 1: Checkout repository;
note right
  actions/checkout@v4
  fetch-depth: 0
  (full history for smart diffing)
end note

:Step 2: Run AutoDoc Swarm;
note right
  uses: ./
  provider: openrouter
  queen_model: qwen/qwen3.6-plus:free
  worker_model: qwen/qwen3.6-plus:free
  drone_model: qwen/qwen3.6-plus:free
end note

partition "Agent Swarm Execution" {
  :Queen Agent (orchestrates);
  :Worker Agent (generates docs);
  :Drone Agent (auxiliary tasks);
}

:Commit generated documentation;
note right
  permissions: contents: write
  Uses PAT_TOKEN for auth
end note

stop
@enduml

## Functions / Methods

### Workflow Triggers
| Event | Configuration | Description |
|---|---|---|
| `push` | `branches: [main]` | Runs when code is pushed to the `main` branch. |
| `pull_request` | (all PRs) | Runs on every pull request event (opened, synchronized, reopened). |

### Job: `test-autodoc-swarm`
The single job in this workflow. It runs on the latest Ubuntu runner and is granted `contents: write` permissions, allowing it to push documentation changes back to the repository.

### Steps

| # | Name | Action / Uses | Key Parameters | Description |
|---|---|---|---|---|
| 1 | Checkout repository | `actions/checkout@v4` | `fetch-depth: 0` | Checks out the full repository including all git history. The zero fetch-depth is required so the AutoDoc agent can compute accurate diffs between commits for changed-file detection. |
| 2 | Run AutoDoc Swarm | `./` (local composite action) | `github_token`: `${{ secrets.PAT_TOKEN }}`<br>`provider`: `openrouter`<br>`queen_model`: `qwen/qwen3.6-plus:free`<br>`worker_model`: `qwen/qwen3.6-plus:free`<br>`drone_model`: `qwen/qwen3.6-plus:free`<br>`api_key`: `${{ secrets.OPENAI_API_KEY }}`<br>`target_dir`: `./` | Executes the AutoDoc Agent Swarm composite action defined at the repository root. It configures three LLM agent roles (Queen, Worker, Drone) all using the Qwen 3.6 Plus model via OpenRouter. The swarm scans the target directory, generates documentation, and commits the results using the provided PAT. |

### Secrets Used
| Secret Name | Used In | Purpose |
|---|---|---|
| `PAT_TOKEN` | Step 2 (`github_token`) | GitHub Personal Access Token with write permissions, used to authenticate git push operations for committing generated documentation. |
| `OPENAI_API_KEY` | Step 2 (`api_key`) | API key used to authenticate requests to the OpenRouter / LLM provider. |

### Agent Model Configuration
All three agent roles in the swarm use the same model:
- **Queen Agent** (orchestrator): `qwen/qwen3.6-plus:free`
- **Worker Agent** (documentation generator): `qwen/qwen3.6-plus:free`
- **Drone Agent** (auxiliary tasks): `qwen/qwen3.6-plus:free`

The models are served via the **OpenRouter** provider, which acts as a unified gateway for accessing various LLM providers through a single API.
