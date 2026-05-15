---
name: open-inference
description: use this to write code to call a large language model using LiteLLM via OpenRouter, with OpenInference as the inference provider.
author: Eric Huang
version: 0.1.0
---

# Calling an LLM via OpenInference

These instructions allow you write code to call an LLM with OpenInference specified as the inference provider.  
This method uses LiteLLM and OpenRouter.

## Setup

The OPENROUTER_API_KEY must be set in the .env file and loaded in as an environment variable.

The uv project must include litellm and pydantic.
`uv add litellm pydantic`

## Code snippets

Use code like these examples in order to use OpenInference.

### Imports and constants

```python
from litellm import completion
MODEL = "openrouter/openai/gpt-oss-120b:free"
EXTRA_BODY = {"provider": {"order": ["open-inference/int8"]}}
```

### Code to call via OpenInference for a text response

```python
response = completion(model=MODEL, messages=messages, reasoning_effort="low", extra_body=EXTRA_BODY)
result = response.choices[0].message.content
```

### Code to call via OpenInference for a Structured Outputs response

```python
response = completion(model=MODEL, messages=messages, response_format=MyBaseModelSubclass, reasoning_effort="low", extra_body=EXTRA_BODY)
result = response.choices[0].message.content
result_as_object = MyBaseModelSubclass.model_validate_json(result)
```
