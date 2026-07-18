+++
date = '2026-07-18T16:39:13+04:00'
draft = false
title = 'Practical Advice for Developers Integrating AI'
+++

> **AI Disclosure:** This text was originally written manually. AI was used only for minor style and grammar refinements.

After working on several projects built around AI services, I have collected a number of patterns that cover both the technical and creative sides of LLM integration. In this article, we will discuss how to make your AI application more robust, debuggable, configurable, and genuinely creative. The code examples provided will be in Python.

## The Technical Side

### Name Your Queries

If you are writing in Python, you are likely using the `openai` library. During the early prototyping stage, it is common to hardcode the provider API URL, model, temperature, `top_k`, and `top_p`. However, as your project grows and you experiment with different models, you will need to change these values frequently. Furthermore, as LLM costs increase, you may need to use different providers for different types of queries.

Here is how your code should look in a more mature state:

```python
from enum import Enum
from pathlib import Path

class RequestType(Enum):
    ART_DIRECTOR = 'art_director'

# The client handles configuration based on the request type
client = OpenAiClient(request_type=RequestType.ART_DIRECTOR)
system_prompt = read_file(Path('prompts/art_director.md'))
response = await client.chat(system_prompt, messages)
```

Note the `RequestType` enum: `ART_DIRECTOR` is the unique identifier for this specific LLM request. This is the core idea: **each distinct LLM query should have its own name.** This name should be used in configurations and logs, and it should ideally match the system prompt filename associated with that query type.

Once each query has a unique name, configuration becomes straightforward:

```bash
# .env configuration
ART_DIRECTOR_PROVIDER=openrouter
ART_DIRECTOR_MODEL=google/gemini-pro-1.5
ART_DIRECTOR_TEMPERATURE=1.5
ART_DIRECTOR_TOP_P=0.95
ART_DIRECTOR_TOP_K=64
```

Ensure that every entry in `RequestType` is reflected in your settings or configuration class. In Python, you can write a test to enforce this; in Rust, you can handle this at compile time using macros.

### Log Your Queries

A significant part of your job will involve analyzing LLM output to understand why your application is behaving unexpectedly. Consequently, you should log every request and response.

If you are building at scale, I recommend using ClickHouse for logging. For smaller projects, a standard PostgreSQL table is sufficient. You likely won't need ClickHouse until your `llm_log` table exceeds 10 million records.

Here is a recommended schema for your logging table:

```
+-------------------+-----------------------------+-------------------------------------------------------+
| Column            | Type                        | Modifiers                                             |
|-------------------+-----------------------------+-------------------------------------------------------|
| id                | integer                     | not null default nextval('llm_log_id_seq'::regclass)  |
| request_type      | character varying           |                                                       |
| provider          | character varying           | not null                                              |
| model             | character varying           | not null                                              |
| request           | jsonb                       | not null                                              |
| response          | jsonb                       |                                                       |
| status            | character varying           | not null                                              |
| error_message     | text                        |                                                       |
| finish_reason     | character varying           |                                                       |
| duration_ms       | integer                     | not null                                              |
| prompt_tokens     | integer                     |                                                       |
| completion_tokens | integer                     |                                                       |
| total_tokens      | integer                     |                                                       |
| cached_tokens     | integer                     |                                                       |
| meta              | jsonb                       |                                                       |
| created_at        | timestamp without time zone | not null default timezone('utc'::text, now())         |
+-------------------+-----------------------------+-------------------------------------------------------+
```

This table will be invaluable for debugging and basic analytics. It helps answer questions like: Which providers or models are slow or unreliable? Where are we consuming the most tokens?

The `meta` field is used to store query context, such as IDs for users, projects, characters, or specific jobs.

### Cached Queries

Another benefit of maintaining a central log is the ability to cache responses. If you have a complex pipeline orchestrating several LLMs across multiple expensive steps, you don't want to waste time and money repeating those steps if the pipeline is restarted.

You can solve this by adding an optional foreign key or unique identifier to the `llm_log` table that points to a specific job ID. In your code, it might look like this:

```python
# When a cache_key is provided, the client checks for a successful 
# existing record with that key before making a new request.
client = OpenAiClient(
    request_type=RequestType.ART_DIRECTOR, 
    cache_key={'job_id': job_id}
)
```

This approach gives you granular control over which request types should be cached and which should always remain fresh. In this example, the cache key is a combination of `request_type` and `job_id`. If you modify your system prompt and need to redo certain steps, you can simply delete the relevant entry from `llm_log`.

### JSON Responses and Schema Validation

Whenever possible, prefer structured responses. Most modern LLMs are trained to produce valid JSON. Including a schema definition in your system prompt helps significantly:

```text
# Response Schema
{
    "names": [array of strings], // as described in the NAMES section
    "styles": [array of strings] // as described in the STYLES section
}

Your response MUST be valid JSON, containing only the object with no explanations or Markdown code fences.
```

However, experienced LLM developers know that this is not foolproof. Even the best models occasionally prepend text like "Here is the JSON you requested" or wrap the output in ```json ... ``` fences.

The solution is to ensure your `OpenAiClient` expects a JSON response and can robustly extract data if parsing fails:

```python
client = OpenAiClient(request_type=RequestType.ART_DIRECTOR, response_format='json')

async def chat(self, ..., response_format: str | None = None):
    # ... call the LLM ...
    if response_format == 'json':
        return self.parse_json_response(raw_response)
```

**Note:** The `openai` library supports a `response_format={'type': 'json_object'}` parameter. This can be a "potential pitfall": it is passed directly to the provider, and some model/provider combinations do not support it. Instead of silently ignoring it, the service may return a `400 Bad Request`. Use this feature at your own risk, or wrap it in a configuration option.

### Retries and Fallbacks

The modern LLM landscape is often unstable. Providers are frequently overloaded and may struggle to keep up with demand. While flakiness is common, providing a smooth user experience requires handling these failures gracefully through retries and fallbacks.

Retries are effective for network lag or transient issues. I recommend a maximum of two retries with a delay of less than one second. Excessive retries and long delays result in poor user experiences. If a service is truly down, your best option is to fall back to a different model or provider:

```python
client = OpenAiClient(
    request_type=RequestType.ART_DIRECTOR,
    cache_key={'job_id': job_id},
    fallback=[
        OpenAiClient(provider='openrouter', model='anthropic/claude-3-haiku'),
        OpenAiClient(provider='novita', model='deepseek/deepseek-v3'),
    ]
)
```

In this setup, `OpenAiClient` will attempt to call the primary provider/model specified in the configuration. In the event of an error, it will automatically repeat the request using the fallback clients.

## Conclusion

As your project scales, the logic surrounding your LLM requests will naturally grow in complexity. Rather than trying to anticipate every future requirement, design your code to be extensible and adaptable to changing technical or business needs.

This article concludes Part 1, covering the technical aspects of LLM integration. In Part 2, we will explore the more creative and nuanced side of working with LLMs. We will discuss techniques for ensuring LLMs don't repeat themselves and consistently generate interesting, fresh content.

Thank you for reading, and stay tuned!

