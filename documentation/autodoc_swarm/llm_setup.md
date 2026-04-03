---
audit_author: SwarmWorker_001
audit_date: 2025-07-15
audit_version: 4f3e5c8d7a1b2e6f9c0d3a8b5e7f1c2d
---
# llm_setup

## Overview
The `llm_setup.py` module is a centralized factory for instantiating LangChain-compatible chat models across multiple LLM providers. It exposes a single public function, `get_llm`, which accepts a provider name and a model identifier, validates the required API key from environment variables, and returns the corresponding `BaseChatModel` instance.

Supported providers:
- **openrouter** — Uses `ChatOpenAI` from `langchain_openai` with OpenRouter's base URL.
- **anthropic** — Uses `ChatAnthropic` from `langchain_anthropic`.
- **openai** — Uses `ChatOpenAI` from `langchain_openai` with OpenAI's default endpoint.
- **google** — Uses `ChatGoogleGenerativeAI` from `langchain_google_genai`.

If an unsupported provider is specified or a required API key environment variable is missing, a `ValueError` is raised with a descriptive message.

## Architecture Diagram
@startuml
skinparam backgroundColor #FFFFFF
skinparam Shadowing false
skinparam RoundCorner 10
skinparam SequenceArrowThickness 2

actor Caller
participant "get_llm(provider, model)" as Factory
database "os.environ" as Env
participant "ChatOpenAI" as COAI
participant "ChatAnthropic" as CAnth
participant "ChatGoogleGenerativeAI" as CGem
participant "BaseChatModel" as Return

Caller -> Factory: get_llm(provider, model)

alt provider == "openrouter"
    Factory -> Env: os.environ.get("OPENROUTER_API_KEY")
    Env --> Factory: api_key
    alt api_key missing
        Factory -> Caller: raise ValueError
    else
        Factory -> COAI: ChatOpenAI(base_url=..., api_key, model)
        COAI --> Return: BaseChatModel
    end
elseif provider == "anthropic"
    Factory -> Env: os.environ.get("ANTHROPIC_API_KEY")
    Env --> Factory: api_key
    alt api_key missing
        Factory -> Caller: raise ValueError
    else
        Factory -> CAnth: ChatAnthropic(api_key, model_name)
        CAnth --> Return: BaseChatModel
    end
elseif provider == "openai"
    Factory -> Env: os.environ.get("OPENAI_API_KEY")
    Env --> Factory: api_key
    alt api_key missing
        Factory -> Caller: raise ValueError
    else
        Factory -> COAI: ChatOpenAI(api_key, model)
        COAI --> Return: BaseChatModel
    end
elseif provider == "google"
    Factory -> Env: os.environ.get("GOOGLE_API_KEY")
    Env --> Factory: api_key
    alt api_key missing
        Factory -> Caller: raise ValueError
    else
        Factory -> CGem: ChatGoogleGenerativeAI(google_api_key, model)
        CGem --> Return: BaseChatModel
    end
else
    Factory -> Caller: raise ValueError("Unsupported provider")
end

Return --> Caller: BaseChatModel instance
@enduml

## Functions / Methods

### `get_llm(provider: str, model: str) -> BaseChatModel`

Factory function that returns a LangChain-compatible chat model for the given provider.

**Parameters:**

| Name     | Type | Description                                              |
|----------|------|----------------------------------------------------------|
| provider | str  | Name of the LLM provider. Must be one of: `openrouter`, `anthropic`, `openai`, or `google`. The comparison is case-insensitive. |
| model    | str  | The model identifier string for the chosen provider (e.g., `"gpt-4"`, `"claude-3-sonnet-20240229"`). |

**Returns:**

| Type         | Description                                           |
|--------------|-------------------------------------------------------|
| BaseChatModel | An instantiated LangChain chat model for the provider. |

**Raises:**

| Exception   | Condition                                           |
|-------------|-----------------------------------------------------|
| ValueError  | If `provider` is not one of the supported providers. |
| ValueError  | If the required `*_API_KEY` environment variable is not set. |

**Provider Mapping:**

| Provider   | Imported Class          | Environment Variable    | Additional Configuration                  |
|------------|-------------------------|-------------------------|--------------------------------------------|
| openrouter | `ChatOpenAI`            | `OPENROUTER_API_KEY`    | `base_url="https://openrouter.ai/api/v1"` |
| anthropic  | `ChatAnthropic`        | `ANTHROPIC_API_KEY`     | —                                          |
| openai     | `ChatOpenAI`            | `OPENAI_API_KEY`        | —                                          |
| google     | `ChatGoogleGenerativeAI`| `GOOGLE_API_KEY`        | —                                          |

**Usage Example:**

```python
from autodoc_swarm.llm_setup import get_llm

# OpenAI
llm = get_llm("openai", "gpt-4")

# Anthropic
llm = get_llm("anthropic", "claude-3-sonnet-20240229")

# OpenRouter
llm = get_llm("openrouter", "google/gemini-1.5-pro")

# Google
llm = get_llm("google", "gemini-1.5-flash")
```
