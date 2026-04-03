---
audit_author: SwarmWorker_001
audit_date: 2025-07-20
audit_version: 7f8a9c2e1d4b3a6f5e0c8d7b9a2f1e4d3c5b6a7e8f9d0c1b2a3e4f5d6c7b8a9
---
# pyproject.toml

## Overview
This `pyproject.toml` file defines the project configuration and dependencies for **autodoc-swarm**, an automated documentation generation system built on LangChain and deep agent frameworks. It specifies the Python version requirement (>=3.12), core package dependencies, and project metadata including name, version, and description.

## Architecture Diagram
@startuml
package "autodoc-swarm (v0.1.0)" {
  [Python >=3.12] as python

  package "Core Dependencies" {
    [deepagents >=0.4.12] as deep
    [pydantic >=2.12.5] as pydantic
    [tenacity >=9.1.4] as retry
    [typer >=0.24.1] as typer
  }

  package "AI Providers" {
    [langchain-anthropic >=1.4.0] as anthropic
    [langchain-google-genai >=4.2.1] as google
    [langchain-openai >=1.1.12] as openai
  }

  package "Development" {
    [pytest >=9.0.2] as testing
    [python-dotenv >=1.2.2] as dotenv
  }

  python --> deep
  python --> pydantic
  python --> retry
  python --> typer
  python --> anthropic
  python --> google
  python --> openai
  python --> testing
  python --> dotenv

  deep --> anthropic
  deep --> google
  deep --> openai
  deep --> pydantic
  deep --> retry

  pydantic -[hidden]- testing
  dotenv -[hidden]- typer
}
@enduml

## Package Information

### Project Metadata

| Field            | Value           |
|------------------|-----------------|
| **Name**         | autodoc-swarm   |
| **Version**      | 0.1.0           |
| **Description**  | Add your description here |
| **Python**       | >=3.12          |
| **Readme**       | README.md       |

### Dependencies

| Package                | Min Version | Category        |
|------------------------|-------------|-----------------|
| deepagents             | 0.4.12      | Core Agent      |
| langchain-anthropic    | 1.4.0       | AI Provider     |
| langchain-google-genai | 4.2.1       | AI Provider     |
| langchain-openai       | 1.1.12      | AI Provider     |
| pydantic               | 2.12.5      | Data Validation |
| pytest                 | 9.0.2       | Testing         |
| python-dotenv          | 1.2.2       | Configuration   |
| tenacity               | 9.1.4       | Retry Logic     |
| typer                  | 0.24.1      | CLI Framework   |
