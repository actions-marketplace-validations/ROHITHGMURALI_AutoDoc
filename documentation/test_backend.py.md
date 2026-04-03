---
audit_author: SwarmWorker_01
audit_date: 2025-12-20
audit_version: 3fa7b5c5e2c12f0f3c1a0d7eab3e6b1d3b4e8e7f
---
# test_backend.py

## Overview
This module contains pytest-based integration tests for two core components of the `autodoc_swarm` package:

1. **`SecureFilesystemBackend`** — Validates that the backend correctly blocks read and write operations against sensitive files (e.g., `.env`), raising a `PermissionError` for each restricted access attempt.
2. **`check_file_freshness`** — Verifies the file freshness check utility by asserting the expected boolean return value when a source file is older or newer than its corresponding documentation file.

Both tests use pytest's `tmp_path` fixture to create isolated, temporary directories for each test run.

## Architecture Diagram
@startuml
package "autodoc_swarm.backend" {
  class SecureFilesystemBackend {
    +read(path: str): str
    +write(path: str, content: str): None
  }
}

package "autodoc_swarm.tools" {
  function check_file_freshness(source_path: str, doc_path: str): bool
}

package "test_backend.py" {
  class TestBackend {
    +test_secure_backend(tmp_path)
    +test_freshness(tmp_path)
  }
}

TestBackend ..> SecureFilesystemBackend : instantiates and tests
TestBackend ..> check_file_freshness : invokes and asserts

note right of TestBackend::test_secure_backend
  Asserts PermissionError when
  reading or writing .env files
end note

note right of TestBackend::test_freshness
  Asserts freshness returns
  False when doc is newer,
  True when source is newer
end note
@enduml

## Functions / Methods

### `test_secure_backend(tmp_path)`
**Purpose:** Ensures `SecureFilesystemBackend` enforces file-level access restrictions.

| Parameter | Type | Description |
|-----------|------|-------------|
| `tmp_path` | `pathlib.Path` | pytest-built-in fixture providing a temporary directory. |

**Flow:**
1. Instantiates `SecureFilesystemBackend` rooted at `tmp_path`.
2. Creates a `.env` file inside `tmp_path` containing `SECRET=123`.
3. Calls `backend.read(".env")` — expects `PermissionError`.
4. Calls `backend.write(".env", "HACK")` — expects `PermissionError`.

**Return:** None (test asserts raised exceptions).

---

### `test_freshness(tmp_path)`
**Purpose:** Validates the `check_file_freshness` utility correctly compares modification timestamps of source and documentation files.

| Parameter | Type | Description |
|-----------|------|-------------|
| `tmp_path` | `pathlib.Path` | pytest-built-in fixture providing a temporary directory. |

**Flow:**
1. Creates `source.py` inside `tmp_path` with content `"a"`.
2. Sleeps 0.1 seconds to ensure timestamp separation.
3. Creates `doc.md` inside `tmp_path` with content `"b"`.
4. Asserts `check_file_freshness(source.py, doc.md)` returns **`False`** (doc is newer, so no refresh needed).
5. Sleeps 0.1 seconds again.
6. Overwrites `source.py` with content `"c"` (making it newer than the doc).
7. Asserts `check_file_freshness(source.py, doc.md)` returns **`True`** (source is newer, so refresh needed).

**Return:** None (test asserts boolean return values).
