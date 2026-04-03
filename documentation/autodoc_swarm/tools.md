---
audit_author: SwarmWorker_001
audit_date: 2025-07-18
audit_version: tools.py
---
# Tools

## Overview
The `tools` module provides utility functions for the autodoc_swarm system, primarily focused on documentation lifecycle management. Currently, it exposes a single function, `check_file_freshness`, which determines whether a source file is newer than its corresponding documentation file, signaling whether documentation needs to be regenerated.

## Architecture Diagram
@startuml
component "Client Code" as client {}
component "check_file_freshness" as func {
    [Validate Inputs] --> exists
    exists : os.path.exists(doc_path)
    [Exists?] --> |No (doc missing)| Return True
    [Exists?] --> |Yes| GetMtimes
    GetMtimes : os.path.getmtime(source_path)
    GetMtimes : os.path.getmtime(doc_path)
    GetMtimes --> Compare
    Compare : source_mtime > doc_mtime
    Compare --> |True| Return True (needs update)
    Compare --> |False| Return False (up-to-date)
    exists --> |Exception: FileNotFoundError| Return False
}
client --> func : Calls check_file_freshness(source_path, doc_path)
func --> client : Returns bool
@enduml

## Functions / Methods

### `check_file_freshness(source_path: str, doc_path: str) -> bool`

Determines whether a source file is newer than the documentation file, indicating that the documentation may be outdated and needs to be regenerated.

**Parameters:**
- `source_path` (`str`): Absolute path to the source code file whose freshness should be checked.
- `doc_path` (`str`): Absolute path to the documentation file to compare against.

**Return Type:**
- `bool` — Returns `True` in the following cases:
  - The documentation file does not exist yet (it needs to be created).
  - The source file's modification timestamp is newer than the documentation file's modification timestamp (documentation is outdated).
  
  Returns `False` in the following cases:
  - The documentation file is up-to-date (its modification timestamp is newer than or equal to the source file's).
  - The source file no longer exists (`FileNotFoundError`), meaning it was likely deleted and no longer requires documentation.

**Behavior Details:**
1. If the documentation file at `doc_path` does not exist, the function returns `True` immediately.
2. It retrieves the modification times (`mtime`) of both the source and documentation files using `os.path.getmtime()`.
3. It compares the two timestamps and returns `True` if the source file is newer.
4. If a `FileNotFoundError` is raised (e.g., the source file has been deleted), the function catches it and returns `False`.
