---
name: edenai
description: Use this skill whenever the user wants to call AI services through Eden AI — a unified API over 500+ models that routes to OpenAI, Anthropic, Google, AWS, Mistral, Cohere, Stability, ElevenLabs, Deepgram, Replicate, and many other providers with one key. Trigger for LLM chat/completion (OpenAI-compatible `/v3/chat/completions`), and for every non-LLM AI task through the unified `/v3/universal-ai` endpoint — text tasks (moderation, NER, topic extraction, spell check, AI-content detection, plagiarism), translation (text + document), image tasks (generation, object/face/logo detection, explicit content, deepfake detection, background removal, anonymization, face compare), speech (text-to-speech, speech-to-text with diarization), OCR & document parsing (invoices, receipts, IDs, resumes, tables), or video generation. Also trigger when the user mentions comparing AI providers, benchmarking inputs across providers, consolidating multi-vendor AI billing, avoiding vendor lock-in, building provider fallbacks, smart routing, BYOK, or names "Eden AI" / "edenai" directly. Prefer this skill over writing bespoke HTTP code against individual providers whenever a user hints at multi-provider needs.
---

# Eden AI

Eden AI is a unified API layer over 500+ models and many providers. One endpoint, one key, consistent request shape. This skill covers the v3 API surface, the universal-ai `model` string, the fallbacks pattern, every task category, async job polling, and the non-obvious error-handling patterns.

## When this skill applies

- Any task where the user says "use Eden AI" or references `edenai`.
- Any task where the user wants to compare or build fallbacks across multiple providers.
- Any task in a supported category (LLM, text, image, audio, video, OCR, documents, translation) where the user hasn't committed to a specific provider's native SDK.
- When the user wants consolidated billing / cost visibility across providers, or a single key to many models.

If the user has already committed to a specific provider's native SDK (e.g. "use the OpenAI Python SDK"), don't force Eden AI — use the native SDK. Eden AI's value is breadth, not depth.

## Setup

Eden AI uses a single API key. Store it in `EDENAI_API_KEY`. Never hardcode.

```bash
export EDENAI_API_KEY="your-key-here"
```

Base URL: `https://api.edenai.run`

Auth header on every request:

```
Authorization: Bearer $EDENAI_API_KEY
Content-Type: application/json
```

## The v3 API surface

Every AI call goes through one of these v3 endpoints:

- **`POST /v3/chat/completions`** — OpenAI-compatible LLM chat. Use for any chat/completion against any LLM (Claude, GPT, Gemini, Mistral, Llama, Cohere, DeepSeek, Qwen, etc.). Drop-in replacement for OpenAI's `/chat/completions`.
- **`POST /v3/responses`** — OpenAI-Responses-compatible. Same provider/model catalog as `/v3/chat/completions`, plus optional server-side conversation state (`store` + `previous_response_id`) so multi-turn chats don't resend full history.
- **`POST /v3/v1/messages`** — Anthropic-Messages-compatible drop-in. Accepts native Anthropic request bodies and returns Anthropic-shaped responses. Also usable as a Claude Code backend via `ANTHROPIC_BASE_URL`.
- **`POST /v3/universal-ai`** — every non-LLM AI feature (OCR, image gen, TTS, STT, document parsing, moderation, NER, translation, etc.). Single endpoint; the `model` field routes to the right feature and provider.
- **`POST /v3/universal-ai/async`** — same thing for long-running tasks (video generation, speech-to-text, multi-page OCR). Returns a `job_id` you poll on `GET /v3/universal-ai/async/{job_id}`.

Plus a few support endpoints: `POST /v3/moderations` (OpenAI-compatible), `POST /v3/v1/messages/count_tokens` (Anthropic-compatible token counter), `POST /v3/upload` (file management), `GET /v3/info` (feature catalog), `GET /v3/models` (LLM catalog).

> **v2 is legacy.** Only cost monitoring and user-token management remain on `/v2/*` — supported through end of 2026. Never use v2 for AI calls.

## LLM chat — `POST /v3/chat/completions`

OpenAI-compatible. The `model` field takes a `provider/model-id` string:

```bash
curl -X POST https://api.edenai.run/v3/chat/completions \
  -H "Authorization: Bearer $EDENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "anthropic/claude-opus-4-7",
    "messages": [{"role": "user", "content": "Explain photosynthesis in one sentence."}],
    "max_tokens": 200
  }'
```

Because this endpoint is OpenAI-compatible, the official OpenAI SDK works by swapping base URL and key:

```python
from openai import OpenAI
import os

client = OpenAI(
    api_key=os.environ["EDENAI_API_KEY"],
    base_url="https://api.edenai.run/v3",
)

resp = client.chat.completions.create(
    model="anthropic/claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello"}],
)
```

### Finding models

The LLM catalog changes often — do NOT rely on any hardcoded list. Always fetch the current catalog before picking a model when the user hasn't named one explicitly:

```bash
curl -s https://api.edenai.run/v3/models \
  -H "Authorization: Bearer $EDENAI_API_KEY"
```

Model IDs follow a `provider/model-id` shape. Common prefixes include `anthropic/`, `openai/`, `google/`, `mistral/`, `cohere/`, `amazon/` (Bedrock hub), `databricks/`, `deepinfra/`, `cloudflare/`, `cerebras/`, `bytedance/`. A few concrete examples for orientation: `anthropic/claude-opus-4-7`, `openai/gpt-4o`, `google/gemini-2.5-pro`. This list is illustrative — new providers appear often; check `GET /v3/models` for truth.

### LLM features

- **Streaming** — `"stream": true`.
- **Tool / function calling** — OpenAI format.
- **Vision inputs** — `image_url` content blocks.
- **Structured output** — `response_format` with a JSON schema.
- **Web search** — when the model supports it.
- **Smart routing** — Eden AI auto-picks a model based on constraints.
- **Fallback** — swap providers on failure without touching client code.
- **BYOK** — bring your own OpenAI/Anthropic/etc. key; Eden still handles routing and consolidated reporting.

## Expert models — `POST /v3/universal-ai`

One endpoint, one payload shape, every non-LLM feature:

```json
{
  "model": "category/feature/provider",
  "fallbacks": ["category/feature/other_provider"],
  "input": { "...feature-specific fields": "..." }
}
```

- `model` — a three-part string `"<category>/<feature>/<provider>"`, e.g. `"text/moderation/microsoft"` or `"ocr/financial_parser/openai"`.
- `fallbacks` — optional array (max 3) of alternate model strings tried **sequentially** if the primary fails. Eden returns the first success.
- `input` — feature-specific parameters (text, file, language, etc.).

Example — moderate some text, fall back to Google if Microsoft fails:

```bash
curl -X POST https://api.edenai.run/v3/universal-ai \
  -H "Authorization: Bearer $EDENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "text/moderation/microsoft",
    "fallbacks": ["text/moderation/google", "text/moderation/openai"],
    "input": {"text": "Some text to moderate"}
  }'
```

### Parallel comparison is client-side

v3 does **not** run multiple providers in parallel in a single call. If the user wants side-by-side provider comparison (e.g. cost/quality A/B), fire multiple `POST /v3/universal-ai` requests in parallel from your code and aggregate the results. Use `fallbacks` for *reliability* (sequential retry), multi-request fan-out for *comparison*.

### Finding features

The feature × provider matrix changes often — do NOT rely on any hardcoded list. Always fetch the authoritative catalog before picking a feature/provider pair:

```bash
curl -s https://api.edenai.run/v3/info \
  -H "Authorization: Bearer $EDENAI_API_KEY"
```

Every AI feature lives under one of six categories: `text`, `translation`, `ocr`, `image`, `audio`, `video`. Model strings follow `category/feature/provider`.

Illustrative examples (not exhaustive — call `GET /v3/info` for the full list and supported providers):

- **text** — `text/moderation/{provider}`, `text/named_entity_recognition/{provider}`, `text/topic_extraction/{provider}`
- **translation** — `translation/automatic_translation/{provider}`, `translation/document_translation/{provider}`
- **ocr** — `ocr/ocr/{provider}`, `ocr/financial_parser/{provider}`, `ocr/identity_parser/{provider}`, `ocr/resume_parser/{provider}`
- **image** — `image/generation/{provider}`, `image/object_detection/{provider}`, `image/background_removal/{provider}`, `image/face_detection/{provider}`
- **audio** — `audio/text_to_speech/{provider}`, `audio/speech_to_text_async/{provider}`
- **video** — `video/generation_async/{provider}`

Feature names ending in `_async` must use the async endpoint (`POST /v3/universal-ai/async`).

## Async jobs — `/v3/universal-ai/async`

Feature names ending in `_async` take too long for a synchronous response. Fire them through the async endpoint:

1. `POST /v3/universal-ai/async` with the usual `{model, fallbacks, input}` — returns `{"job_id": "..."}` (HTTP 202).
2. `GET /v3/universal-ai/async/{job_id}` — returns `status` (`pending` / `processing` / `finished` / `failed`) and, once finished, the `results`.
3. `GET /v3/universal-ai/async` — list your jobs.
4. `DELETE /v3/universal-ai/async/{job_id}` — cancel / delete.

Poll with backoff — start at ~2 s, back off to ~30 s. Don't hammer.

```python
import os, time, requests

headers = {"Authorization": f"Bearer {os.environ['EDENAI_API_KEY']}"}

job = requests.post(
    "https://api.edenai.run/v3/universal-ai/async",
    headers=headers,
    json={
        "model": "audio/speech_to_text_async/openai",
        "input": {"file": "https://example.com/meeting.mp3", "language": "en"},
    },
).json()

job_id = job["job_id"]
delay = 2
while True:
    r = requests.get(
        f"https://api.edenai.run/v3/universal-ai/async/{job_id}",
        headers=headers,
    ).json()
    if r["status"] in ("finished", "failed"):
        break
    time.sleep(delay)
    delay = min(delay * 2, 30)
```

**Webhooks** are an alternative — pass a webhook URL in the POST and Eden AI pings you when the job finishes. Prefer webhooks over polling when the caller can receive inbound HTTP.

## OpenAI-compatible moderation — `POST /v3/moderations`

Drop-in for OpenAI's `/moderations`. Same request and response shape. Useful when you already have OpenAI-moderation client code.

## File management — `/v3/upload`

Upload files once and reuse them by ID across calls — avoids re-uploading large inputs.

- `POST /v3/upload` — upload (returns a file ID)
- `GET /v3/upload` — list your files (optional `purpose` filter)
- `POST /v3/upload/delete` — delete specific files by ID
- `DELETE /v3/upload` — delete all your files (irreversible)

Reference an uploaded file in any `/v3/universal-ai` call by passing its ID in the `input.file` field.

## Platform info endpoints

- `GET /v3/info` — full feature + provider catalog. Authoritative.
- `GET /v3/models` — live LLM catalog.

## Legacy v2 (cost & tokens only)

The only remaining v2 paths — supported through end of 2026 — are for account-level concerns, not AI calls:

- `GET /v2/info/splitted-schema/cost_management/` — consumption
- `GET /v2/info/splitted-schema/cost_management/credits/` — remaining credits
- `GET|POST /v2/info/splitted-schema/user/custom_token/` — API token management

Do NOT use v2 for any AI request. Every AI call goes through `/v3/chat/completions` or `/v3/universal-ai`.

## Error handling

Two distinct failure modes:

1. **HTTP-level failure** (401, 403, 429, 5xx) — auth, quota, rate limit, Eden outage. Retry with backoff on 429/5xx; 4xx auth errors are not retriable.
2. **All fallbacks exhausted** — when the primary `model` *and* every provider in `fallbacks` fail, the response will surface the error. Always inspect the response body for an `error` field even on a 2xx; the returned JSON tells you which provider actually served the response.

For reliability in production, always set `fallbacks` to 1–3 alternate providers so transient provider outages don't break your request.

## Cost tracking

Every `/v3/universal-ai` response includes a `cost` field (USD) for the provider that served the request. The `/v3/chat/completions` endpoint returns usage in the standard OpenAI format (`prompt_tokens`, `completion_tokens`, `total_tokens`); dollar cost is available on the Eden AI dashboard and via the v2 cost-monitoring endpoints above.

When the user cares about cost comparison — and they often do, it's half the reason Eden AI exists — surface `cost` in the response you return.

## Guardrails and good defaults

- **Never log or echo the API key.** Read from `EDENAI_API_KEY`; pass in headers; done.
- **One provider first, fallbacks for reliability.** Use `fallbacks` for cascading retries; fan out client-side only when the user explicitly wants parallel comparison.
- **Prefer specialized features.** `ocr/financial_parser` returns structured line items / totals / vendor; generic `ocr/ocr` returns raw text. Don't regex line items out of raw OCR.
- **Surface `cost`** when comparing providers.
- **Use async for anything long-running.** Video, multi-page OCR, speech-to-text on meeting-length audio — always the async endpoint.
- **Upload reused files once.** Large files used in multiple calls should go through `/v3/upload` first.

## Other LLM endpoints — Responses API and Anthropic Messages

Two additional LLM endpoints cover narrower use cases. They are documented in full in `references/` — read the reference file only when the task calls for one.

- **`POST /v3/responses`** → server-side conversation state (`store: true` + `previous_response_id`) so multi-turn chats don't resend history. Full details, chaining example: [references/responses-api.md](references/responses-api.md).
- **`POST /v3/v1/messages`** → drop-in for Anthropic's native `/v1/messages`; accepts Anthropic request bodies verbatim. Main use cases: existing Anthropic-SDK code, or pointing **Claude Code itself** at Eden AI via `ANTHROPIC_BASE_URL` + `ANTHROPIC_AUTH_TOKEN`. Full details, SDK example, Claude Code setup, `count_tokens`: [references/anthropic-messages.md](references/anthropic-messages.md).

### Which LLM endpoint to pick

- Default to `/v3/chat/completions` — it's OpenAI-compatible and covers almost every LLM task.
- Switch to `/v3/responses` only when the user wants Eden AI to hold conversation state server-side.
- Switch to `/v3/v1/messages` only when the user is already on the Anthropic SDK, or wants to route Claude Code through Eden AI.

## Quick recipes

**"Translate this and moderate the output."** → Two `/v3/universal-ai` calls: `translation/automatic_translation/deepl` then `text/moderation/openai`.

**"Parse this invoice PDF and get the line items."** → `/v3/universal-ai` with `model: "ocr/financial_parser/mindee"` (or Veryfi, Klippa, etc.). Do NOT use generic OCR and regex.

**"Which LLM is cheapest for this prompt?"** → Fire N parallel `/v3/chat/completions` calls with different `model` values, compare usage/dashboard cost.

**"Transcribe this meeting and identify speakers."** → `/v3/universal-ai/async` with `model: "audio/speech_to_text_async/assemblyai"` (or Deepgram, Gladia). Poll until finished, or register a webhook.

**"Generate a product image."** → `/v3/universal-ai` with `model: "image/generation/stabilityai"` (or OpenAI, Replicate, MiniMax, ByteDance, Leonardo).

**"Fallback from OpenAI to Anthropic if OpenAI is down."** → `/v3/chat/completions` with smart routing, or `/v3/universal-ai` with `fallbacks: ["text/moderation/anthropic"]` etc.

**"Route Claude Code through Eden AI."** → Set `ANTHROPIC_BASE_URL` + `ANTHROPIC_AUTH_TOKEN` to point at `/v3/v1/messages`. See [references/anthropic-messages.md](references/anthropic-messages.md).

---

For anything not covered here, the live docs at https://edenai.co/docs are authoritative — Eden AI adds providers and endpoints often, so treat this skill's catalog as a starting map, not an exhaustive list.
