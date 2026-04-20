# Responses API — `POST /v3/responses`

OpenAI-Responses-compatible. Same `provider/model-id` format as `/v3/chat/completions`. Pick this endpoint over `/v3/chat/completions` when you want Eden AI to **store the conversation server-side** so subsequent turns don't resend the full history.

Model IDs below are illustrative — always fetch the current catalog from `GET /v3/models`.

```bash
curl -X POST https://api.edenai.run/v3/responses \
  -H "Authorization: Bearer $EDENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-4o",
    "instructions": "You are a terse assistant.",
    "input": "Summarize the Krebs cycle in one line."
  }'
```

Eden-specific extras carry over from universal-ai: pass `fallbacks` for sequential retry across providers, and read the `cost` field (USD) from the response.

## Conversation chaining with `previous_response_id`

Set `store: true` on turn 1, capture the returned `id`, then pass it as `previous_response_id` on turn 2 — the server threads history, you only send the new input.

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["EDENAI_API_KEY"],
    base_url="https://api.edenai.run/v3",
)

t1 = client.responses.create(
    model="anthropic/claude-opus-4-7",
    input="My favorite color is periwinkle.",
    store=True,
)

t2 = client.responses.create(
    model="anthropic/claude-opus-4-7",
    input="What did I just tell you?",
    previous_response_id=t1.id,
)
```

Stored responses can be fetched with `GET /v3/responses/{id}` and removed with `DELETE /v3/responses/{id}`.
