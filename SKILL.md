---
name: edenai
description: Use this skill whenever the user wants to call AI services through Eden AI — a unified API over 500+ models that routes to OpenAI, Anthropic, Google, AWS, Mistral, Cohere, Stability, ElevenLabs, Deepgram, Replicate, and many other providers with one key. Trigger for LLM chat/completion (OpenAI-compatible), text tasks (moderation, NER, topic extraction, spell check, AI-content detection, plagiarism), translation (text + document), image tasks (generation, object/face/logo detection, explicit content, deepfake detection, background removal, anonymization, face compare), speech (text-to-speech, speech-to-text with diarization), OCR & document parsing (invoices, receipts, IDs, resumes, tables), or video generation. Also trigger when the user mentions comparing AI providers, benchmarking the same input across providers, consolidating multi-vendor AI billing, avoiding vendor lock-in, building provider fallbacks, smart routing, BYOK, or names "Eden AI" / "edenai" directly. Prefer this skill over writing bespoke HTTP code against individual providers whenever a user hints at multi-provider needs.
---

# Eden AI

Eden AI is a unified API layer over 500+ models and many providers. One endpoint, one key, consistent request/response shape. This skill covers the two API surfaces, the canonical multi-provider request pattern, every task category with its endpoint, async job polling, and the non-obvious error-handling patterns.

## When this skill applies

- Any task where the user says "use Eden AI" or references `edenai`.
- Any task where the user wants the *same input* run through *multiple providers* (comparison, fallback, A/B, cost/quality tradeoff).
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

## Two API surfaces

Eden AI exposes two generations of endpoints:

- **`/v3/llm/...`** — modern, OpenAI-compatible LLM surface. Use for any chat/completion against any LLM (Claude, GPT, Gemini, Mistral, Llama, Cohere, DeepSeek, Qwen, etc.). Drop-in replacement for `/chat/completions`.
- **`/v2/...`** — task-specific endpoints. Use for everything that isn't a plain LLM call: OCR, image gen, TTS, STT, document parsing, video gen, translation, moderation, NER, etc.

When in doubt: chat/completion → `/v3/llm/`, anything else → `/v2/`.

## LLM chat — `/v3/llm/chat/completions`

OpenAI-compatible. The `model` field takes a `provider/model-id` string:

```bash
curl -X POST https://api.edenai.run/v3/llm/chat/completions \
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
    base_url="https://api.edenai.run/v3/llm",
)

resp = client.chat.completions.create(
    model="anthropic/claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello"}],
)
```

### Provider prefixes

Never hardcode a model list — the catalog changes frequently. Fetch it live:

```bash
curl -s https://api.edenai.run/v3/llm/models \
  -H "Authorization: Bearer $EDENAI_API_KEY"
```

Known prefixes, with a sample model for orientation:

| Prefix | Example model IDs |
|---|---|
| `anthropic/` | `anthropic/claude-opus-4-7`, `anthropic/claude-sonnet-4-6`, `anthropic/claude-haiku-4-5` |
| `openai/` | `openai/gpt-4o`, `openai/gpt-4o-mini` |
| `google/` | `google/gemini-2.5-pro`, `google/gemini-2.5-flash` |
| `mistral/` | `mistral/mistral-large-latest` |
| `cohere/` | `cohere/command-a-03-2025`, `cohere/command-r-plus-08-2024` |
| `amazon/` | Bedrock hub — e.g. `amazon/anthropic.claude-3-haiku-20240307-v1:0`, `amazon/amazon.nova-pro-v1:0`, `amazon/meta.llama3-70b-instruct-v1:0` |
| `bytedance/` | `bytedance/seed-2-0-pro-260328` |
| `databricks/` | `databricks/databricks-claude-opus-4-5` |
| `deepinfra/` | `deepinfra/deepseek-ai/DeepSeek-V3` |
| `cloudflare/` | `cloudflare/@cf/meta/llama-2-7b-chat-fp16` |
| `cerebras/` | `cerebras/llama3.1-8b` |

### LLM features

- **Streaming** — `"stream": true`.
- **Tool / function calling** — OpenAI format.
- **Vision inputs** — `image_url` content blocks.
- **Structured output** — `response_format` with a JSON schema.
- **Web search** — when the model supports it.
- **Smart routing** — Eden AI auto-picks a model based on constraints.
- **Fallback** — swap providers on failure without touching client code.
- **BYOK** — bring your own OpenAI/Anthropic/etc. key; Eden still handles routing and consolidated reporting.

## Expert models — `/v2/*` multi-provider pattern

Every `/v2/` task follows the same shape:

```json
{
  "providers": "amazon,google",
  "<task-specific fields>": "..."
}
```

- `providers` — comma-separated string (or list in some SDKs). Listing multiple providers runs them in parallel and returns all results.
- `fallback_providers` — if the first provider fails, Eden tries the next.
- Optional: `response_as_dict`, `show_original_response`, `show_base_64` (task-dependent).

Example — OCR with two providers in parallel:

```bash
curl -X POST https://api.edenai.run/v2/ocr/ocr \
  -H "Authorization: Bearer $EDENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "providers": "amazon,google",
    "file_url": "https://example.com/receipt.jpg",
    "language": "en"
  }'
```

Response shape (v2) — keyed by provider:

```json
{
  "amazon": {"status": "success", "cost": 0.0015, "text": "...", "bounding_boxes": [...]},
  "google": {"status": "success", "cost": 0.0015, "text": "...", "bounding_boxes": [...]}
}
```

Each provider returns independently. A provider can fail (`status: "fail"` with an `error` field) while others succeed — **check status per provider; a 200 HTTP does not mean every provider succeeded.**

### Feature catalog

All endpoints below live under `https://api.edenai.run/v2/`. For the live, authoritative inventory and the exact provider list per task, fetch `GET /v3/info/`.

**Text analysis**

| Endpoint | Description | Providers |
|---|---|---|
| `text/ai_detection` | AI-generated text vs. human | Sapling, WinstonAI |
| `text/moderation` | Offensive / harmful content scan | Google, Microsoft, OpenAI |
| `text/spell_check` | Spelling & grammar | ProWritingAid, Sapling |
| `text/topic_extraction` | Themes & topics | Google, OpenAI, TensTorrent |
| `text/named_entity_recognition` | Entities (people, places, orgs) | Amazon, Microsoft, OpenAI, TensTorrent |
| `text/plagia_detection` | Plagiarism | WinstonAI |

**Translation**

| Endpoint | Description | Providers |
|---|---|---|
| `translation/automatic_translation` | Translate text | Amazon, DeepL, Google, Microsoft, ModernMT, OpenAI |
| `translation/document_translation` | Translate a whole document | DeepL, Google |

**OCR & document parsing**

| Endpoint | Description | Providers |
|---|---|---|
| `ocr/ocr` | Generic text detection | Amazon, API4AI, Google, Microsoft, Mistral, SentiSight |
| `ocr/ocr_async` | Multi-page OCR (async) | Amazon, Microsoft, Mistral |
| `ocr/ocr_tables_async` | Structured table extraction (async) | Amazon, Google, Microsoft |
| `ocr/identity_parser` | ID document fields | Affinda, Amazon, Base64, Klippa, Microsoft, Mindee, OpenAI |
| `ocr/financial_parser` | Invoices, receipts, bank statements, bank checks | Affinda, Amazon, Base64, EagleDoc, Extracta, Google, Klippa, Microsoft, Mindee, OpenAI, TabScanner, Veryfi |
| `ocr/resume_parser` | Structured resume data | Affinda, Extracta, Klippa, OpenAI, SenseLoaf |

**Image**

| Endpoint | Description | Providers |
|---|---|---|
| `image/generation` | Text-to-image | ByteDance, Leonardo, MiniMax, OpenAI (DALL-E 2 & 3), Replicate, StabilityAI |
| `image/background_removal` | Strip background | API4AI, Clipdrop, PhotoRoom, PicsArt, SentiSight, StabilityAI |
| `image/ai_detection` | AI-generated image detection | WinstonAI |
| `image/object_detection` | Objects + bounding boxes | Amazon, API4AI, Clarifai, Google, Microsoft, SentiSight |
| `image/face_detection` | Faces & attributes | Amazon, API4AI, Clarifai, Google |
| `image/explicit_content` | NSFW / explicit flag | Amazon, Clarifai, Google, Microsoft, OpenAI, SentiSight |
| `image/anonymization` | Blur faces / PII | API4AI |
| `image/deepfake_detection` | Manipulated face detection | SightEngine |
| `image/face_compare` | Face similarity | Amazon, Base64, FacePlusPlus |
| `image/logo_detection` | Brand logo detection | API4AI, Clarifai, Google, Microsoft, OpenAI |

**Audio**

| Endpoint | Description | Providers |
|---|---|---|
| `audio/text_to_speech` | Text → audio | Amazon, Deepgram, ElevenLabs, Google, Lovo AI, Microsoft, OpenAI |
| `audio/speech_to_text_async` | Transcription + diarization (async) | Amazon, AssemblyAI, Deepgram, Gladia, Google, Microsoft, OpenAI |

**Video**

| Endpoint | Description | Providers |
|---|---|---|
| `video/generation_async` | Text / image → video (async) | Amazon, ByteDance, Google, MiniMax, OpenAI |

## Async jobs

Endpoints ending in `_async` use a two-step pattern:

1. `POST` the task — returns `{"public_id": "..."}` immediately.
2. `GET` the same endpoint with `/{public_id}` appended — returns `status` (`pending` / `processing` / `finished` / `failed`) and, once finished, the `results`.

Poll with backoff — start at ~2 s, back off to ~30 s. Don't hammer.

```python
import os, time, requests

headers = {"Authorization": f"Bearer {os.environ['EDENAI_API_KEY']}"}
job = requests.post(
    "https://api.edenai.run/v2/audio/speech_to_text_async",
    headers=headers,
    json={"providers": "openai", "file_url": "https://example.com/meeting.mp3"},
).json()

public_id = job["public_id"]
delay = 2
while True:
    r = requests.get(
        f"https://api.edenai.run/v2/audio/speech_to_text_async/{public_id}",
        headers=headers,
    ).json()
    if r["status"] in ("finished", "failed"):
        break
    time.sleep(delay)
    delay = min(delay * 2, 30)
```

**Webhooks** are an alternative — pass a webhook URL in the POST and Eden AI pings you when the job finishes. Prefer webhooks over polling when the caller can receive inbound HTTP.

## OpenAI-compatible moderation — `/v3/moderation`

Drop-in for OpenAI's `/moderations` endpoint. Same request and response shape.

## File management

Upload files once and reuse them by ID across calls — avoids re-uploading large inputs.

- `POST /v3/files` — upload
- `GET /v3/files` — list
- `DELETE /v3/files/{id}` — delete one
- `DELETE /v3/files` — delete all

## Platform info endpoints

- `GET /v3/info/` — full feature + provider catalog (authoritative).
- `GET /v3/llm/models` — live LLM catalog.
- Consumption, credits, and token-management endpoints live under `/v3/` — see https://edenai.co/docs.

## Error handling

Two distinct failure modes, both important:

1. **HTTP-level failure** (401, 403, 429, 5xx) — auth, quota, rate limit, Eden outage. Retry with backoff on 429/5xx; 4xx auth errors are not retriable.
2. **Per-provider failure inside a 200 response** — the HTTP call succeeded, but one of the listed providers returned `status: "fail"` with an `error`. This is *normal* — a provider might not support a language, might be temporarily down, or might reject the input. Always iterate and check per provider.

```python
for provider_name, result in response.json().items():
    if result.get("status") == "fail":
        log.warning(f"{provider_name} failed: {result.get('error')}")
        continue
    # use result
```

If you specified `fallback_providers`, Eden handles this and returns only the successful one.

## Cost tracking

Every successful v2 provider result includes a `cost` field (USD). When the user cares about cost comparison — and they often do, it's half the reason Eden AI exists — surface this in the response you return.

The v3 LLM endpoint returns usage in the OpenAI format (`prompt_tokens`, `completion_tokens`, `total_tokens`). Dollar cost for v3 LLM calls is visible on the Eden AI dashboard and via the consumption endpoints.

## Guardrails and good defaults

- **Never log or echo the API key.** Read from `EDENAI_API_KEY`; pass in headers; done.
- **Start with one provider, not all.** Multi-provider calls multiply cost. Only fan out when the user explicitly wants comparison or redundancy.
- **Prefer specialized endpoints.** `ocr/financial_parser` returns structured line items / totals / vendor; generic `ocr/ocr` returns raw text. Don't regex line items out of raw OCR.
- **Surface `cost`** when comparing providers.
- **Return results structured by provider.** Don't silently pick a winner — show both and let the user decide, unless they've asked otherwise.
- **Use `fallback_providers` for production reliability** instead of multi-provider fan-out.

## Quick recipes

**"Translate this and moderate the output."** → `/v2/translation/automatic_translation` then `/v2/text/moderation`.

**"Parse this invoice PDF and get the line items."** → `/v2/ocr/financial_parser`. Do NOT use plain `/v2/ocr/ocr` and regex.

**"Which LLM is cheapest for this prompt?"** → Call `/v3/llm/chat/completions` once per candidate model and compare dashboard cost, or use smart routing with a cost constraint.

**"Transcribe this meeting and identify speakers."** → `/v2/audio/speech_to_text_async` with speaker diarization. Poll until finished, or register a webhook.

**"Generate a product image."** → `/v2/image/generation` with one of the image-gen providers (OpenAI, Stability, Replicate, MiniMax, ByteDance, Leonardo).

**"Fallback from OpenAI to Anthropic if OpenAI is down."** → `/v3/llm/chat/completions` with smart routing, or any `/v2/*` endpoint with `fallback_providers: "openai,anthropic"`.

---

For anything not covered here, the live docs at https://edenai.co/docs are authoritative — Eden AI adds providers and endpoints often, so treat this skill's catalog as a starting map, not an exhaustive list.
