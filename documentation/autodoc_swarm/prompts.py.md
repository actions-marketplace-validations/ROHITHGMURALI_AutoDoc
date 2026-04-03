---
audit_author: SwarmWorker_004
audit_date: 2025-06-24
audit_version: a8e42f7
---
# autodoc_swarm/prompts.py

## Overview
The `prompts.py` module defines three system prompt templates that power the AutoDoc Agent Swarm — a multi-agent system for automatically generating and validating code documentation. These prompts configure three distinct subagent roles:

- **Swarm Queen** (`QUEEN_SYSTEM_PROMPT_TEMPLATE`): The orchestrator that scans a target code repository, evaluates documentation freshness via the `check_file_freshness` tool, builds a TODO list of stale or missing docs, and delegates generation and review tasks to Worker and Drone subagents. It enforces a maximum of 3 revision retries per document before escalating to human review.
- **Swarm Worker** (`WORKER_SYSTEM_PROMPT`): The Technical Writer subagent that reads source code and produces structured Markdown documentation with embedded PlantUML architecture diagrams, YAML frontmatter audit blocks, and detailed function/method descriptions.
- **Swarm Drone** (`DRONE_SYSTEM_PROMPT`): The QA subagent that acts as a "devil's advocate," critiquing Worker-generated docs against strict validation criteria (schema compliance, PlantUML correctness, coverage completeness) and returning a strict JSON `PASS`/`FAIL` verdict with actionable feedback.

Together, these prompts form a closed-loop documentation pipeline: the Queen orchestrates discovery and task delegation, the Worker generates documentation, and the Drone validates quality, feeding failures back to the Worker through the Queen for iterative refinement.

## Architecture Diagram
@startuml
actor Developer as Dev
participant "Queen\n(Orchestrator)" as Queen
participant "Worker\n(Technical Writer)" as Worker
participant "Drone\n(QA Inspector)" as Drone

Dev -> Queen: Provide target_dir
Queen -> Queen: Deep scan repository\n(ls_info, read)
Queen -> Queen: Evaluate freshness per file\n(check_file_freshness)
Queen -> Queen: Build TODO list\n(files where source > doc)

loop For each stale file
    Queen -> Worker: Delegate doc generation\n(source_path, doc_path)
    Worker --> Queen: Return generated Markdown

    Queen -> Drone: Delegate doc review\n(source_path, doc_path)
    Drone --> Queen: JSON verdict\n{status, feedback}

    alt status == "FAIL" and retries < 3
        Queen -> Worker: Re-delegate with\nDrone feedback
        Worker --> Queen: Revised Markdown
        Queen -> Drone: Re-review
        Drone --> Queen: JSON verdict
    else status == "FAIL" and retries >= 3
        Queen -> Queen: Mark as\nFAILED_REQUIRES_HUMAN
    else status == "PASS"
        Queen -> Queen: Mark as Complete
    end
end

@enduml

## Functions / Methods

This module does not define executable functions or classes. It exposes three **string constant templates** used by the swarm orchestration layer to configure LLM subagent system prompts.

### `QUEEN_SYSTEM_PROMPT_TEMPLATE`

**Type:** `str` (template with `{target_dir}` injection point)

**Purpose:** Configures the Swarm Queen agent. Defines the orchestration logic for repository-wide documentation management, including:

- **Path resolution rules:** Distinguishes between filesystem backend tools (paths relative to `target_dir`) and the `check_file_freshness` tool (paths including `target_dir` prefix), and documents the mirroring convention (`src/api/routes.py` → `documentation/src/api/routes.md`).
- **Core workflow (5 directives):**
  1. **Deep Scan:** Recursively scan the entire repository, skipping standard noise directories (`.git/`, `node_modules/`, `venv/`, `__pycache__/`, `.env`).
  2. **Evaluate Freshness:** For each discovered source file, use `check_file_freshness(source_path, doc_path)` to determine if the doc is stale or missing.
  3. **Plan Tasks:** Maintain a TODO list of files requiring documentation.
  4. **Orchestrate Workers and Drones:** Delegate doc generation to the Worker, then delegate review to the Drone upon completion.
  5. **Enforce Retries:** Track rejection counts per document. Max 3 retries; on exhaustion, mark as `FAILED_REQUIRES_HUMAN` and proceed to the next file. On `PASS`, mark as `Complete`.

**Parameters (template variable):**
- `target_dir` (str): The absolute path of the code repository to document.

---

### `WORKER_SYSTEM_PROMPT`

**Type:** `str` (static template)

**Purpose:** Configures the Swarm Worker agent. Instructs it to read source code and produce documentation adhering to a strict Markdown schema:

- **YAML frontmatter:** Must include `audit_author` (e.g., `SwarmWorker_<ID>`), `audit_date` (YYYY-MM-DD), and `audit_version` (hash of source file).
- **Document structure:**
  - `# <Component Name>`: Title of the documented component.
  - `## Overview`: Clear, concise description of the file's purpose.
  - `## Architecture Diagram`: Embedded PlantUML block (`@startuml` / `@enduml`) representing architecture, classes, or logic flow.
  - `## Functions / Methods`: Detailed descriptions of public functions/methods, including parameters and return types.
- **Feedback correction:** If Drone feedback indicates failures, the Worker must correct the specified issues before resubmitting.

**Parameters:** None (agent receives `source_path` and `doc_path` at invocation time via the Queen).

---

### `DRONE_SYSTEM_PROMPT`

**Type:** `str` (static template)

**Purpose:** Configures the Swarm Drone QA agent. Defines validation criteria and output formatting for documentation review:

- **Validation Criteria:**
  1. **Schema Check:** Verifies the exact YAML frontmatter audit block is present (`audit_author`, `audit_date`, `audit_version`).
  2. **PlantUML Syntax:** Ensures `@startuml` and `@enduml` tags are correctly placed and PlantUML syntax is logically sound.
  3. **Coverage Check:** Compares source code against documentation to ensure no public methods, classes, or critical logic flows are omitted.
- **Output Requirement:** Returns a strict JSON payload with no additional text. Two valid formats:
  - **Pass:** `{"status": "PASS", "feedback": ""}`
  - **Fail:** `{"status": "FAIL", "feedback": "Detailed required fixes here..."}`

The Drone's JSON response is consumed by the Queen to decide whether to accept the document or request further revisions from the Worker.
