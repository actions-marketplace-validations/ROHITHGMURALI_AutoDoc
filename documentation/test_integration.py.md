---
audit_author: SwarmWorker_1
audit_date: 2025-12-23
audit_version: d5a2f3c1b0e9d8a7f6e5c4b3a2d1e0f9a8b7c6d5
---
# test_integration.py

## Overview
This file contains integration tests for the `autodoc_swarm` package. It validates three core behaviors:
1. **Swarm initialization** — ensures that `create_swarm` correctly instantiates three LLM clients (for queen, worker, and drone roles), creates a deep agent, and spawns two sub-agents.
2. **File-freshness logic** — verifies that the `check_file_freshness` utility returns the correct boolean when comparing source and documentation file timestamps (including the case where the doc does not yet exist).
3. **Mirrored directory structure** — confirms that `SecureFilesystemBackend.write` can create files nested in arbitrarily deep directory hierarchies, automatically generating any missing intermediate directories.

The tests use `pytest` fixtures (e.g., `tmp_path`) and `unittest.mock` for patching external dependencies.

## Architecture Diagram
@startuml
actor Tester

rectangle "test_integration.py" as tests {
  [test_swarm_initialization] as T1
  [test_check_file_freshness_logic] as T2
  [test_mirrored_directory_structure] as T3
}

rectangle "autodoc_swarm.agent" as agent {
  [create_swarm] as CS
  [get_llm] as GLLM
  [create_deep_agent] as CDA
  [SubAgent] as SA
}

rectangle "autodoc_swarm.tools" as tools {
  [check_file_freshness] as CFF
}

rectangle "autodoc_swarm.backend" as backend {
  [SecureFilesystemBackend] as SFS
}

Tester --> T1 : runs
Tester --> T2 : runs
Tester --> T3 : runs

T1 ..> CS : imports & calls
CS --> GLLM : calls ×3 (queen, worker, drone)
CS --> CDA : calls once
CS --> SA : instantiates ×2

T2 ..> CFF : imports & calls (3 scenarios)

T3 ..> SFS : instantiates
SFS --> "Filesystem" : creates nested dirs + writes file
@enduml

## Functions / Methods

### `test_swarm_initialization(mock_subagent, mock_create_deep_agent, mock_get_llm)`
- **Purpose**: Validates that `create_swarm` initializes the full agent hierarchy with the correct provider and model names.
- **Parameters**:
  - `mock_subagent` (`MagicMock`): Patch for `autodoc_swarm.agent.SubAgent`.
  - `mock_create_deep_agent` (`MagicMock`): Patch for `autodoc_swarm.agent.create_deep_agent`.
  - `mock_get_llm` (`MagicMock`): Patch for `autodoc_swarm.agent.get_llm`.
- **Return type**: `None`
- **Key assertions**:
  | Assertion | Expected Value |
  |---|---|
  | `mock_get_llm` called with `("openai", "gpt-4o")` | `yes` |
  | `mock_get_llm` called with `("openai", "gpt-4o-mini")` | `yes` |
  | `mock_get_llm` called with `("openai", "gpt-3.5-turbo")` | `yes` |
  | `mock_get_llm.call_count` | `3` |
  | `mock_create_deep_agent.call_count` | `1` |
  | `mock_subagent.call_count` | `2` |

### `test_check_file_freshness_logic()`
- **Purpose**: Exercises `check_file_freshness` across three timestamp scenarios to ensure correct staleness detection.
- **Parameters**: None (uses local side-effect setup/teardown).
- **Return type**: `None`
- **Test scenarios**:
  1. **Doc does not exist** → expects `True` (docs need to be generated).
  2. **Doc is newer than source** → expects `False` (docs are up-to-date).
  3. **Source is newer than doc** → expects `True` (docs are stale and need regeneration).

### `test_mirrored_directory_structure(tmp_path)`
- **Purpose**: Verifies that `SecureFilesystemBackend` can write a file into a deeply nested path, automatically creating all intermediate directories.
- **Parameters**:
  - `tmp_path` (`pathlib.Path`): Pytest fixture providing an isolated temporary directory.
- **Return type**: `None`
- **Key assertions**:
  | Assertion | Expected Value |
  |---|---|
  | `full_path.exists()` | `True` |
  | `full_path.read_text()` | `"# Routes\n"` |
