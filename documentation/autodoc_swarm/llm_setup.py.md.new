---
audit_author: SwarmWorker_003
audit_date: 2025-07-16
audit_version: llm_setup.py
---
# LLM Setup

## Overview
The `llm_setup.py` module provides a centralized factory for initializing LangChain-compatible chat models across multiple LLM providers. The `get_llm` function accepts a provider name and model identifier, validates the corresponding API key from environment variables, and returns a properly configured `BaseChatModel` instance. This design enables seamless switching between providers (OpenRouter, Anthropic, OpenAI, Google) without modifying downstream code.

## Architecture Diagram
@startuml
start
:get_llm(provider, model);
:Normalize provider to lowercase;

if (provider == openrouter?) then (yes)
  :Check OPENROUTER_API_KEY;
  if (key present?) then (no)
    :raise ValueError;
    stop
  else (yes)
    :import ChatOpenAI;
    :Instantiate ChatOpenAI(
      base_url=https://openrouter.ai/api/v1,
      api_key, model
    );
  endif
elseif (provider == anthropic?) then (yes)
  :Check ANTHROPIC_API_KEY;
  if (key present?) then (no)
    :raise ValueError;
    stop
  else (yes)
    :import ChatAnthropic;
    :Instantiate ChatAnthropic(
      api_key, model_name=model
    );
  endif
elseif (provider == openai?) then (yes)
  :Check OPENAI_API_KEY;
  if (key present?) then (no)
    :raise ValueError;
    stop
  else (yes)
    :import ChatOpenAI;
    :Instantiate ChatOpenAI(
      api_key, model
    );
  endif
elseif (provider == google?) then (yes)
  :Check GOOGLE_API_KEY;
  if (key present?) then (no)
    :raise ValueError;
    stop
  else (yes)
    :import ChatGoogleGenerativeAI;
    :Instantiate ChatGoogleGenerativeAI(
      google_api_key, model
    );
  endif
else (no)
  :raise ValueError(
    Unsupported provider
  );
  stop
endif

:return BaseChatModel instance;
stop
@enduml

## Functions / Methods

### `get_llm(provider: str, model: str) -> BaseChatModel`

Returns the appropriate LangChain `BaseChatModel` instance based on the specified provider.

**Parameters:**
| Name     | Type   | Description                                      |
|----------|--------|--------------------------------------------------|
| provider | `str`  | Name of the LLM provider (`openrouter`, `anthropic`, `openai`, `google`). Case-insensitive. |
| model    | `str`  | Model identifier string specific to the provider (e.g., `gpt-4o`, `claude-3-5-sonnet-20241022`). |

**Returns:**
| Type           | Description                                                |
|----------------|------------------------------------------------------------|
| `BaseChatModel` | A configured LangChain chat model instance ready for use. |

**Raises:**
- `ValueError` — If the required `*_API_KEY` environment variable is not set for the chosen provider.
- `ValueError` — If the provider name is not one of the supported providers.

### Supported Providers and Environment Variables

| Provider   | LangChain Class          | Required Environment Variable | Instantiation Details                                      |
|------------|--------------------------|-------------------------------|------------------------------------------------------------|
| `openrouter` | `ChatOpenAI` (from `langchain_openai`) | `OPENROUTER_API_KEY`  | Uses custom `base_url="https://openrouter.ai/api/v1"` |
| `anthropic`  | `ChatAnthropic` (from `langchain_anthropic`) | `ANTHROPIC_API_KEY` | Passes `model_name` parameter |
| `openai`     | `ChatOpenAI` (from `langchain_openai`) | `OPENAI_API_KEY`    | Standard OpenAI endpoint |
| `google`     | `ChatGoogleGenerativeAI` (from `langchain_google_genai`) | `GOOGLE_API_KEY` | Passes `google_api_key` parameter |
