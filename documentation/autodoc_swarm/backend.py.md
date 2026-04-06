---
audit_author: SwarmWorker_002
audit_date: 2025-09-09
audit_version: autodoc_swarm/backend.py
---
# SecureFilesystemBackend

## Overview

`SecureFilesystemBackend` is a secured subclass of `deepagents.backends.FilesystemBackend` that wraps file system operations (read, write, list) with multiple layers of access control. Its primary purpose is to prevent LLM-driven agents from accessing sensitive files or directories. It achieves this through path normalization, denial of absolute-path escape, path-traversal rejection (`..`), directory-level blocklists (e.g. `.git`, `node_modules`), and filename-pattern blocklists (e.g. `.env*`, `*.pem`, any file containing "credentials" or "secret", and any `*ignore` files).

## Architecture Diagram

@startuml
skinparam classBackgroundColor white
skinparam classBorderColor #1a73e8
skinparam classFontColor #1a1a1a
skinparam classFontSize 16

class FilesystemBackend {
  + read(path, *args, **kwargs) : str
  + write(path, content) : Any
  + ls_info(path) : list[dict]
}

class SecureFilesystemBackend {
  - _root_dir : str
  - blocked_patterns : List[re.Pattern]
  - blocked_dirs : Set[str]
  + __init__(root_dir, virtual_mode, *args, **kwargs)
  - _normalize_path(path_str) : str
  - _is_allowed(path_str) : bool
  + read(path, *args, **kwargs) : str
  + write(path, content) : Any
  + ls_info(path) : list[dict]
}

SecureFilesystemBackend --|> FilesystemBackend : extends

package "Security Layers" {
  [Path Normalization\n_strip absolute prefix,\n_strip root_dir parts] as normalize
  [Access Control\n_block absolute paths,\n_block .. traversal,\n_block dirs,\n_block patterns] as acl
}

SecureFilesystemBackend ..> normalize : uses in\nread / write / ls_info
SecureFilesystemBackend ..> acl : enforces after normalization

note right of acl
  Blocked dirs: .git, node_modules,
  venv, .venv, __pycache__

  Blocked patterns: .env*, *.pem,
  *credentials*, *secret*, *ignore
end note

note right of normalize
  Converts:
  "/dummy_repo/src/api.py"        -> "src/api.py"
  "workspace/repo/src/api.py"     -> "src/api.py"
end note
@enduml

## Functions / Methods

### `__init__(self, root_dir: str = ".", virtual_mode: bool = False, *args, **kwargs)`

Initializes the backend by delegating to the parent `FilesystemBackend`, then configures security rules.

- **Parameters:**
  - `root_dir` (`str`): Base directory confinement root. Defaults to `"."` (current working directory).
  - `virtual_mode` (`bool`): Passed through to the parent backend to enable virtual filesystem mode. Defaults to `False`.
  - `*args, **kwargs`: Forwarded to `FilesystemBackend.__init__`.
- **Instance Attributes Created:**
  - `_root_dir` (`str`): Stores the confinement root for later path normalization.
  - `blocked_patterns` (`List[re.Pattern]`): Compiled regex patterns matching restricted filenames:
    - `\.env.*` — environment files (`.env`, `.env.local`, etc.)
    - `.*\.pem$` — private key files (`.pem`)
    - `.*credentials.*` (case-insensitive) — credential stores
    - `.*secret.*` (case-insensitive) — secret files
    - `.*ignore$` (case-insensitive) — ignore files (`.gitignore`, `.dockerignore`, etc.)
  - `blocked_dirs` (`Set[str]`): Directory names to reject anywhere in the path: `.git`, `node_modules`, `venv`, `.venv`, `__pycache__`.

---

### `_normalize_path(self, path_str: str) -> str`

Cleans and standardizes an incoming path string so it can be safely used relative to `_root_dir`. Strips leading absolute prefixes, removes the `root_dir` prefix wherever it appears in the path, and handles redundant `./` segments.

- **Parameters:**
  - `path_str` (`str`): Raw path string that may be absolute, contain the full workspace prefix, or be relative.
- **Returns:**
  - `str` — A cleaned, relative path string. Examples:
    | Input | Output |
    |---|---|
    | `"/dummy_repo/src/api.py"` | `"src/api.py"` |
    | `"github/workspace/dummy_repo/src/api.py"` | `"src/api.py"` |
    | `"./src/main.py"` | `"src/main.py"` |

---

### `_is_allowed(self, path_str: str) -> bool`

Evaluates whether a normalized path is permitted under the security policy. Returns `False` (denying access) if any check fails.

- **Parameters:**
  - `path_str` (`str`): The file path to evaluate (should already be normalized).
- **Returns:**
  - `bool` — `True` if access is permitted; `False` otherwise.
- **Security Checks (in order):**
  1. **Absolute path rejection** — blocks any path that `Path.is_absolute()` considers absolute.
  2. **Path traversal rejection** — blocks if `..` appears in any part of the path.
  3. **Blocked directory check** — iterates over all path parts; returns `False` if any part is in `blocked_dirs`.
  4. **Blocked pattern check** — tests the filename against every pattern in `blocked_patterns`; returns `False` on first match.

---

### `read(self, path: str, *args, **kwargs) -> str`

Reads and returns the contents of a file after applying normalization and access-control checks.

- **Parameters:**
  - `path` (`str`): Path to the file.
  - `*args, **kwargs`: Forwarded to the parent `FilesystemBackend.read`.
- **Returns:**
  - `str` — The file contents.
- **Raises:**
  - `PermissionError` — if `_is_allowed` returns `False`.
- **Flow:**
  1. Normalize `path` via `_normalize_path`.
  2. Validate access via `_is_allowed`; raise `PermissionError` on denial.
  3. Delegate to `super().read(path, *args, **kwargs)` and return result.

---

### `write(self, path: str, content: str) -> Any`

Writes content to a file after applying normalization and access-control checks.

- **Parameters:**
  - `path` (`str`): Destination file path.
  - `content` (`str`): The text content to write.
- **Returns:**
  - `Any` — Return value from the parent `FilesystemBackend.write`.
- **Raises:**
  - `PermissionError` — if `_is_allowed` returns `False`.
- **Flow:**
  1. Normalize `path` via `_normalize_path`.
  2. Validate access via `_is_allowed`; raise `PermissionError` on denial.
  3. Delegate to `super().write(path, content)`.

---

### `ls_info(self, path: str = ".") -> list[dict]`

Lists directory contents, returning metadata for each item that passes the security filter. Unlike `read` and `write`, this method does **not** `_is_allowed` the directory path itself; instead, it retrieves results from the parent backend and filters them item by item.

- **Parameters:**
  - `path` (`str`): Directory path to list. Defaults to `"."`.
- **Returns:**
  - `list[dict]` — A list of item metadata dictionaries (each expected to contain at least a `"path"` key) for non-blocked entries.
- **Raises:**
  - `Exception` — Propagates any non-`FileNotFoundError` exceptions from the underlying `super().ls_info` call after logging.
- **Flow:**
  1. Normalize `path` via `_normalize_path`.
  2. Call `super().ls_info(path)` inside a `try` block.
  3. Iterate returned items and skip any whose path contains a `blocked_dirs` component or whose filename matches a `blocked_patterns` regex.
  4. Collect allowed items and return them.
  5. If `FileNotFoundError` is raised, return an empty list `[]`.
  6. On any other exception, log a message and re-raise.
