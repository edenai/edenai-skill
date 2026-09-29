# Responses API: `POST /v3/responses`

This is the OpenAI Responses API. It takes the same `provider/model` strings as `/v3/chat/completions`. Use it over chat completions when you want Eden AI to **store the conversation server-side**, so later turns send only the new input. Source: [Responses](https://www.edenai.co/docs/v3/llms/responses).

Model ids here are examples; check `GET /v3/models` for the live catalog.

```bash
curl -s -X POST https://api.edenai.run/v3/responses \
  -H "Authorization: Bearer $EDENAI_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-latest",
    "instructions": "You are a terse assistant.",
    "input": "Summarize the Krebs cycle in one line."
  }'
```

## How it differs from chat completions

| | Chat Completions | Responses |
|---|---|---|
| System prompt | a `system` message | `instructions` |
| User input | `messages` | `input`: a string or a list of input items |
| Answer text | `choices[0].message.content` | `output[0].content[0].text` (`output_text` in the SDK) |
| Multi-turn | resend the history | `previous_response_id` |
| Persistence | none | stored by default (`store: true`) |
| Tokens | `prompt_tokens` / `completion_tokens` | `input_tokens` / `output_tokens` |

**Eden AI extras:**
- **In the request:** `fallbacks`, `routing`, `router_candidates` (with `model: "@edenai"`), `session_id` and `tags`, all working as they do on chat completions.
- **In the response:** `cost` (USD), `provider` and `provider_time`.
- **Status:** `status` is `completed`, `in_progress`, `incomplete` or `failed`.

## Chaining turns with `previous_response_id`

> **Test this before depending on it.** In September 2026, on our account, the first turn worked, but these calls failed:
> - A second turn with `previous_response_id` returned HTTP 500 `Missing dependency No module named 'apscheduler'` for both `anthropic/claude-haiku-latest` and `mistral/mistral-large-latest`.
> - `GET /v3/responses/{id}` returned 500 `GET responses is not supported for anthropic` (and the same for Mistral).
> - OpenAI models failed with a 401 from Eden AI's own OpenAI key.
>
> Until that's fixed, resend the history with `/v3/chat/completions` for multi-turn chat.

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["EDENAI_API_KEY"], base_url="https://api.edenai.run/v3")

t1 = client.responses.create(model="anthropic/claude-sonnet-latest", input="My favorite color is periwinkle.")
t2 = client.responses.create(
    model="anthropic/claude-sonnet-latest",
    input="What did I just tell you?",
    previous_response_id=t1.id,
)
print(t2.output_text)
```

**Managing stored responses:**
- **Read:** `GET /v3/responses/{id}` returns a stored response.
- **Delete:** `DELETE /v3/responses/{id}` removes it, after which it can't be used as a `previous_response_id`.
- **Opt out:** send `store: false` for a one-off call you don't want kept.

## Other fields

- **Standard Responses fields:** `max_output_tokens`, `temperature`, `top_p`, `tools` (functions and web search), `tool_choice`, `parallel_tool_calls`, `reasoning`, `text` (output format), `truncation` (`auto` or `disabled`), `include`, `background`, `metadata` and `user`.
- **Prompt caching:** `prompt_cache_key`, `prompt_cache_retention` and `prompt_cache_options` work here too, with `input_text` content blocks. Cached tokens are reported in `usage.input_tokens_details.cached_tokens`.
- **Streaming:** `stream: true` sends Server-Sent Events.
