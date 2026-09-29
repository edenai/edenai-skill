# Anthropic Messages: `POST /v3/v1/messages`

This is a drop-in for Anthropic's `/v1/messages`. It takes Anthropic request bodies (`model`, `messages`, `system`, `max_tokens`, `tools`, `tool_choice`, `thinking`, `stop_sequences`, `top_k`, `stream` and `metadata`) and returns Anthropic-shaped responses.

- **The path:** `/v3/v1/messages` is correct: Anthropic's `/v1/messages` mounted under `/v3`, so an SDK with `base_url=https://api.edenai.run/v3` finds it.
- **Any model:** it serves any chat model in the catalog, not only Claude.
- **Eden AI fields:** `fallbacks`, `routing`, `router_candidates`, `session_id` and `tags` work here as they do on chat completions.

Model ids below are examples; check `GET /v3/models` for the live catalog.

```bash
curl -s -X POST https://api.edenai.run/v3/v1/messages \
  -H "x-api-key: $EDENAI_API_KEY" -H "anthropic-version: 2023-06-01" -H "Content-Type: application/json" \
  -d '{
    "model": "anthropic/claude-sonnet-latest",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

Both auth styles work, since we tested both in September 2026: `x-api-key: <key>` as the Anthropic SDK's `api_key=` sends it, and `Authorization: Bearer <key>` as `auth_token=` sends it.

```python
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["EDENAI_API_KEY"], base_url="https://api.edenai.run/v3")
resp = client.messages.create(
    model="anthropic/claude-sonnet-latest",
    max_tokens=256,
    messages=[{"role": "user", "content": "Hello"}],
)
print(resp.content[0].text)
```

## Claude Code on Eden AI

This follows [Eden AI's Claude Code guide](https://www.edenai.co/docs/v3/integrations/claude-code):

```bash
export ANTHROPIC_BASE_URL="https://api.edenai.run/v3"
export ANTHROPIC_API_KEY="$EDENAI_API_KEY"
export ANTHROPIC_MODEL="anthropic/claude-opus-latest"
export ANTHROPIC_DEFAULT_OPUS_MODEL="anthropic/claude-opus-latest"
export ANTHROPIC_DEFAULT_SONNET_MODEL="anthropic/claude-sonnet-latest"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="anthropic/claude-haiku-latest"
claude
```

- **Model names:** use full `provider/model` strings. A bare `claude-opus-4-7` isn't found, and any catalog model with tool calling can be set as `ANTHROPIC_MODEL`.
- **Prompt caching:** Eden AI reads the `x-claude-code-session-id` header, so routed requests from one Claude Code session stay on the provider holding its prompt cache.
- **Expert models as tools:** add the Eden AI MCP server with `claude mcp add --transport http edenai https://mcp.edenai.run/mcp --header "Authorization: Bearer $EDENAI_API_KEY"`. See [mcp-server.md](mcp-server.md).
- **Tags:** to tag Claude Code's usage, add `export ANTHROPIC_CUSTOM_HEADERS="X-EdenAI-Tags: client=acme,project=invoices"`.

## Prompt caching with `cache_control`

On this endpoint, add `cache_control: {"type": "ephemeral", "ttl": "5m" | "1h"}` to the last block of the reusable prefix.
- **Usage fields:** the response reports `usage.cache_creation_input_tokens` and `usage.cache_read_input_tokens`.
- **Chat completions:** Eden AI adds cache marks automatically for Claude there. Any explicit `cache_control` turns that off.

## Token counting: `POST /v3/v1/messages/count_tokens`

It takes `model`, `messages`, `system`, `tools` and `tool_choice`, and returns the input token count without running the model. It's useful for budgeting and context-window checks.
