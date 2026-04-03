---
audit_author: SwarmWorker_001
audit_date: 2025-12-14
audit_version: 4f8a92b3c7e6d1f0a5b9c3e7d2f6a8b1c4d5e9f0
---
# Agent (create_swarm)

## Overview
This module defines the `create_swarm` factory function, which instantiates a hierarchical multi-agent Swarm system for automated documentation generation. It wires together three distinct agent roles:

- **Queen** — the orchestrator (`create_deep_agent`) that receives the user's request, manages subagents, and has access to a secure filesystem backend and a file-freshness tool.
- **Worker** — a Technical Writer subagent responsible for reading source code and producing documentation.
- **Drone** — a Devil's Advocate / QA subagent that critiques generated documentation and validates that requirements are met.

Each agent receives its own language-model instance (potentially targeting different model names) via the shared `get_llm` helper, and all filesystem access is sandboxed under the provided `target_dir`.

## Architecture Diagram
```
@startuml
skinparam packageStyle rectangle

actor User

package "Swarm System" {
  [get_llm] as llm_factory

  queue llm_Queen
  queue llm_Worker
  queue llm_Drone

  component "SecureFilesystemBackend" as backend
  component "check_file_freshness" as freshness_tool

  component "Queen Agent" as queen {
    [orchestrator]
  }

  component "Worker SubAgent" as worker {
    [Technical Writer]
  }

  component "Drone SubAgent" as drone {
    [QA / Devil's Advocate]
  }
}

User --> queen : invokes
llm_factory --> llm_Queen : (provider, queen_model)
llm_factory --> llm_Worker : (provider, worker_model)
llm_factory --> llm_Drone : (provider, drone_model)

llm_Queen --> queen
llm_Worker --> worker
llm_Drone --> drone

backend --> queen : root_dir = target_dir
freshness_tool --> queen : tool

queen --> worker : delegates
queen --> drone : delegates

@enduml
```

## Functions / Methods

### `create_swarm`

```python
def create_swarm(
    target_dir: str,
    provider: str,
    queen_model: str,
    worker_model: str,
    drone_model: str,
) -> DeepAgent
```

**Description**  
Factory function that constructs and returns the Queen agent fully wired with its subagents (Worker and Drone), a secure filesystem backend, and a file-freshness tool.

**Parameters**

| Parameter      | Type   | Description                                                                 |
|----------------|--------|-----------------------------------------------------------------------------|
| `target_dir`   | `str`  | Absolute path to the directory the filesystem backend should be rooted at.  |
| `provider`     | `str`  | LLM provider identifier passed to `get_llm` for instantiating model clients.|
| `queen_model`  | `str`  | Model name for the Queen orchestrator agent.                                |
| `worker_model` | `str`  | Model name for the Worker (Technical Writer) subagent.                      |
| `drone_model`  | `str`  | Model name for the Drone (QA) subagent.                                     |

**Returns**  
The fully configured Queen `DeepAgent` instance ready to receive prompts and delegate tasks to its subagents.

**Behavior**
1. Calls `get_llm(provider, <model>)` three times to obtain LLM instances for the Queen, Worker, and Drone.
2. Creates a `SecureFilesystemBackend` rooted at `target_dir`.
3. Formats the `QUEEN_SYSTEM_PROMPT_TEMPLATE` with `target_dir`.
4. Instantiates Worker and Drone `SubAgent` objects with their respective system prompts (`WORKER_SYSTEM_PROMPT`, `DRONE_SYSTEM_PROMPT`) and LLM instances.
5. Calls `create_deep_agent` with the Queen's prompt, LLM, backend, the list of subagents, and the `check_file_freshness` tool.
6. Returns the resulting Queen agent.

**Dependencies**

- `deepagents.create_deep_agent`, `deepagents.SubAgent` — framework primitives for agent creation.
- `autodoc_swarm.prompts` — system prompt templates (`QUEEN_SYSTEM_PROMPT_TEMPLATE`, `WORKER_SYSTEM_PROMPT`, `DRONE_SYSTEM_PROMPT`).
- `autodoc_swarm.backend.SecureFilesystemBackend` — sandboxed filesystem access.
- `autodoc_swarm.tools.check_file_freshness` — tool exposed to the Queen agent.
- `autodoc_swarm.llm_setup.get_llm` — factory for provider-specific LLM instances.
