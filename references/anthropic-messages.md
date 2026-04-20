# Anthropic-compatible messages — `POST /v3/v1/messages`

Drop-in for Anthropic's native `/v1/messages`. Accepts the same body shape — `model`, `messages`, `system`, `max_tokens`, `tools`, `tool_choice`, `stream` — and returns Anthropic-shaped responses. Other Anthropic-documented fields (`metadata`, `stop_sequences`, `top_k`, beta headers) pass through; check live docs for beta-header support.

> The `/v3/v1/messages` path is intentional — Eden AI mounts Anthropic's `/v1/messages` under the `/v3/` prefix. Not a typo.

Model IDs below are illustrative — always fetch the current catalog from `GET /v3/models`.

```bash
curl -X POST https://api.edenai.run/v3/v1/messages \
  -H "Authorization: Bearer $EDENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "anthropic/claude-opus-4-7",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

The Anthropic Python SDK works by swapping base URL and auth token:

```python
from anthropic import Anthropic
import os

client = Anthropic(
    auth_token=os.environ["EDENAI_API_KEY"],
    base_url="https://api.edenai.run/v3",
)

resp = client.messages.create(
    model="anthropic/claude-sonnet-4-6",
    max_tokens=256,
    messages=[{"role": "user", "content": "Hello"}],
)
```

> Use `auth_token=` (not `api_key=`). `auth_token` sends `Authorization: Bearer`, which is what Eden AI expects; `api_key` sends `x-api-key` and will fail at this endpoint.

## Using Eden AI as a Claude Code backend

Because `/v3/v1/messages` is drop-in Anthropic-compatible, Claude Code itself can be pointed at Eden AI — no code changes, just two env vars:

```bash
export ANTHROPIC_BASE_URL="https://api.edenai.run/v3"
export ANTHROPIC_AUTH_TOKEN="$EDENAI_API_KEY"
claude
```

Claude Code's existing Anthropic SDK calls hit `/v3/v1/messages` unchanged. You get BYOK, consolidated billing, and provider-level fallback routing. The same pattern works for any app built on the Anthropic SDK.

For the current list of Claude Code proxy env vars, see https://docs.claude.com/en/docs/claude-code/settings.

## Token counting — `POST /v3/v1/messages/count_tokens`

Takes the same body shape as `/v3/v1/messages` (drop `max_tokens`) and returns a token count. Useful for pre-flight budgeting, context-window checks, and cost estimation.
