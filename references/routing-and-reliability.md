# Routing, fallbacks and caching

This covers how Eden AI picks the model and the provider, how it retries, and how to see what it did. Source pages: [Provider Routing](https://www.edenai.co/docs/v3/llms/provider-routing), [Fallback](https://www.edenai.co/docs/v3/general/fallback), [Request Metadata](https://www.edenai.co/docs/v3/llms/request-metadata) and [Prompt Caching](https://www.edenai.co/docs/v3/llms/prompt-caching).

## Four ways to name an LLM

| You send | What happens | `edenai_metadata.strategy` |
|---|---|---|
| `openai/gpt-latest` | That provider, that model. No routing | `direct` or `alias` |
| `gpt-oss-120b` (no provider) | Eden AI picks one of the providers serving that model, and fails over between them | `routed` |
| `gpt-oss-120b:latency` | The same, with an objective written into the name | `routed` |
| `@edenai` | Eden AI picks the model too | `auto` |

Two suffixes add to these:
- **A region:** `<model>@eu` or `<model>@us` pins where the model runs. Each fallback entry carries its own region.
- **Routable names:** `GET /v3/models?view=models` lists one entry per routable name, with `endpoint_count` and every provider endpoint nested under it, each with its own price, capabilities and regions. In September 2026, `gpt-oss-120b` had 17 providers and `kimi-k3` had 12.

## Provider routing (a bare model name)

The default picks the cheapest provider for the request's shape. The choice is weighted, so traffic spreads across providers and failing providers are ranked last. Change it with a `routing` object:

```json
{
  "model": "gpt-oss-120b",
  "messages": [{"role": "user", "content": "hi"}],
  "routing": {
    "sort": "latency",
    "allowed_providers": ["groq", "cerebras", "together_ai"],
    "allow_fallbacks": true,
    "sticky": true
  }
}
```

- **`sort`** takes one of four values. Naming one turns off the traffic spreading.
  - `cost` (the default): the cheapest provider.
  - `speed`: the most tokens per second.
  - `latency`: the fastest time to first token.
  - `exact`: the best instruction-following, for tools and structured output.
  - With too little performance data, it ranks by price.
- **`allowed_providers`** limits the pool, failover included. If none of the listed providers serves the model, the request fails.
- **`allow_fallbacks: false`** stops failover to other providers of the *same* model. It never removes the models you listed in `fallbacks`.
- **`sticky`** is on by default. It keeps a conversation on the provider that holds its prompt cache, for models whose providers discount cache reads. The affinity lasts 10 minutes by default after each cache read or write. Identify the conversation in one of two ways:
  - The `session_id` body field, the same on every turn: a thread id, not a fresh UUID and not a user id.
  - One of the headers `x-session-id`, `x-claude-code-session-id`, `x-session-affinity` or `session-id`.

`session_id` and `routing` are accepted on `/v3/chat/completions`, `/v3/responses` and `/v3/v1/messages`. `routing` is also accepted on `/v3/audio/speech` and `/v3/audio/transcriptions`.

## The `@edenai` model router

```json
{
  "model": "@edenai",
  "router_candidates": ["openai/gpt-latest", "anthropic/claude-haiku-latest", "gpt-oss-120b"],
  "routing": {"quality_cost": 7},
  "messages": [{"role": "user", "content": "Classify this ticket: ..."}]
}
```

- **`router_candidates`:** up to 64 entries. A bare name has its provider routed afterwards; a `provider/model` entry is served by exactly that provider. Without candidates, the router picks from a default pool of catalog models.
- **`quality_cost`:** 0 asks for the best model for the request, 10 for the cheapest model that can still handle it.
- **Dropped entries:** candidates it can't rank are listed under `edenai_metadata.routing.auto.dropped`.
- **Not everywhere:** `/v3/videos` JSON bodies accept neither `@edenai` nor `fallbacks`, and `/v3/images/*` doesn't accept `fallbacks`.

## Fallbacks (models you name)

```json
{"model": "vertex/gemini-flash-latest", "fallbacks": ["openai/gpt-latest", "anthropic/claude-sonnet-latest"], "messages": [...]}
{"model": "text/moderation/microsoft", "fallbacks": ["text/moderation/google"], "input": {"text": "..."}}
```

- **Limits:** at most 3 entries, and more returns 422. Entries are tried in order, and a repeated entry is really tried again.
- **Universal AI:** every entry must be the same feature and subfeature. Fallbacks aren't allowed on `audio/tts` or `image/face_recognition`, because a voice or a face collection belongs to one provider.
- **Billing:** only the attempt that answered is billed.
- **EU endpoint:** every entry must be EU-eligible, or the whole call returns 451.
- **Comparing providers:** fallbacks are sequential, not a comparison. To compare providers side by side, send parallel requests yourself.

## Seeing what happened: `x-edenai-metadata: enabled`

Add the header and the response gains an `edenai_metadata` block. Nothing else in the response changes.

```json
"edenai_metadata": {
  "requested": "gpt-oss-120b",
  "strategy": "routed",
  "region": "global",
  "summary": "available=15, served=groq/openai/gpt-oss-120b",
  "attempt": 1,
  "is_byok": false,
  "attempts": [{"provider": "groq", "model": "groq/openai/gpt-oss-120b", "status": 200, "region": "global"}],
  "tags": {"project": "demo"}
}
```

- **Coverage:** it's returned on all the chat dialects, embeddings, moderations, decisions, images, transcriptions, videos, synchronous Universal AI, and the async job-creation response. It's not returned on `GET /v3/universal-ai/async/{id}` or on cached responses.
- **Text-to-speech:** `/v3/audio/speech` returns raw audio, so read the `x-edenai-provider` and `x-edenai-cost` response headers instead.
- **Streaming:** the block rides the first chunk. A later error frame carries a corrected block.
- **Universal AI:** the block has `mode` (`sync` or `async`) instead of `strategy` and `endpoints`.

## Response caching and prompt caching

| | Response caching | Prompt caching |
|---|---|---|
| What it does | Returns an earlier answer to an identical request, for free | The provider reuses a cached prompt prefix and still generates a new answer |
| Best for | Embeddings, moderation, OCR, NER | Long system prompts, tools, documents, agent loops |
| Control | On by default; toggled per project in the dashboard. Tags don't make requests different | Automatic on OpenAI and DeepSeek. Eden AI adds cache marks for Claude. You can also set explicit marks |

**Prompt caching controls:**
- **OpenAI:** `prompt_cache_key`, `prompt_cache_retention` (`in_memory` or `24h`), and on GPT-5.6 and newer, `prompt_cache_breakpoint` and `prompt_cache_options`.
- **Claude:** `cache_control` with `ttl` of `5m` or `1h` on `/v3/v1/messages`. Any explicit `cache_control` turns off Eden AI's automatic marks.
- **Gemini:** `cache_control` on a content block.

**Where to read cached tokens:**
- **Chat Completions:** `usage.prompt_tokens_details.cached_tokens` and `cache_write_tokens`.
- **Responses:** `usage.input_tokens_details.*`.
- **Messages:** `usage.cache_read_input_tokens` and `cache_creation_input_tokens`.
- The response `cost` already includes cache pricing.
