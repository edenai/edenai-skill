---
name: edenai
description: Use this skill whenever the user wants to call AI models through Eden AI (edenai), a gateway over 500+ models from OpenAI, Anthropic, Google, Mistral, AWS, Azure and 50+ other providers behind one API key. Covers LLM chat through the OpenAI-compatible /v3/chat/completions (plus the Responses and Anthropic Messages dialects), provider routing, fallbacks and the @edenai model router; OpenAI-compatible embeddings, images, audio, video and moderation endpoints; and every expert model through /v3/universal-ai, including OCR and document parsing (invoices, IDs, resumes, tables), web search, scraping, crawling and deep research, translation, speech, image and video generation, and moderation. Also covers async jobs, webhooks, file upload, the hosted MCP server, tags, cost tracking, sandbox and management keys, BYOK and the EU endpoint. Trigger when the user names Eden AI, or wants one key or one bill across providers, provider comparison, fallbacks or smart routing.
---

# Eden AI

Eden AI is an AI gateway: one API key, one bill and one request shape for 500+ models from 50+ providers. Every response reports what it cost. This file is the map; the `references/` files hold the details and are worth reading only when the task needs them.

The catalog changes weekly. Never trust a model or provider name from memory, including the ones in this file: check the live catalog first (see [Discovery](#discovery)). The docs are at https://www.edenai.co/docs, with a machine-readable index at https://www.edenai.co/docs/llms.txt.

## When this skill applies

- The user says "Eden AI" or "edenai", or the code already calls `api.edenai.run`.
- The user wants several providers behind one key: comparing models or providers, fallbacks, routing by price or speed, one bill, per-customer cost tracking.
- The task is an AI feature the user hasn't tied to a provider's own SDK: chat, embeddings, OCR, document parsing, web search, speech, translation, image or video generation, moderation.

If the user has chosen a provider's native SDK and wants only that provider, use the native SDK.

## Setup

```bash
export EDENAI_API_KEY="sk-eden-..."   # never hardcode it, never print it
```

- **Base URL:** `https://api.edenai.run/v3`. For EU data residency use `https://api.eu.edenai.run/v3` with the same key. It only routes to EU-cleared providers and returns HTTP 451 for anything else.
- **Auth:** `Authorization: Bearer $EDENAI_API_KEY` on every call.
- **Key types:**
  - `sk-eden-…` inference keys call models.
  - Sandbox keys (`sandbox_api_token`) return free mock responses in the real shape, for CI and development.
  - `mgmt-eden-…` management keys only call `/v3/manage/*`: creating keys and reading usage.
  - Details are in [references/account-and-governance.md](references/account-and-governance.md).

## Which endpoint

| Task | Endpoint | `model` format |
|---|---|---|
| Chat, vision, tools, structured output, web-grounded answers | `POST /v3/chat/completions` (OpenAI-compatible) | `provider/model` |
| Chat with server-side history | `POST /v3/responses` (OpenAI Responses) | `provider/model` |
| Anthropic SDK code, Claude Code | `POST /v3/v1/messages` (Anthropic Messages) | `provider/model` |
| Embeddings | `POST /v3/embeddings` | `provider/model` |
| Images, in the OpenAI shape | `POST /v3/images/generations`, `POST /v3/images/edits` | `provider/model` |
| Speech, in the OpenAI shape | `POST /v3/audio/speech`, `POST /v3/audio/transcriptions` | `provider/model` |
| Video, in the OpenAI shape | `POST /v3/videos`, then poll `GET /v3/videos/{id}` | `provider/model` |
| Moderation, in the OpenAI shape | `POST /v3/moderations` | e.g. `openai/omni-moderation-latest` |
| Typed yes/no, choice or score answers (alpha) | `POST /v3/alpha/decisions` | e.g. `typesafe/jev-latest` |
| Every other AI feature: OCR, parsing, web, translation, detection… | `POST /v3/universal-ai`, or `/v3/universal-ai/async` for `_async` features | `feature/subfeature/provider[/model]` |
| Files used by several calls | `POST /v3/upload` | |
| Expert models as tools for an agent | MCP server `https://mcp.edenai.run/mcp` | |

The OpenAI-compatible endpoints work with the official OpenAI SDK: set `base_url="https://api.edenai.run/v3"` and pass the Eden AI key as `api_key`. Details for the media endpoints are in [references/openai-compatible-media.md](references/openai-compatible-media.md).

## Discovery

These endpoints are public, so no key is needed. Check them before choosing a model:

```bash
curl -s https://api.edenai.run/v3/models                    # every LLM endpoint: pricing, context, capabilities, regions
curl -s "https://api.edenai.run/v3/models?view=models"      # one entry per routable model name, providers nested under it
curl -s https://api.edenai.run/v3/info                      # every expert feature and subfeature
curl -s https://api.edenai.run/v3/info/ocr/financial_parser # providers, model strings, pricing, input and output schema
```

- **LLM capabilities:** `capabilities` says what a model supports, for example `supports_function_calling`, `supports_response_schema`, `supports_web_search`, `supports_prompt_caching` and `input_modalities`. A missing key or `null` means not supported.
- **Per-surface lists:** each OpenAI-style surface has its own list: `/v3/embeddings/models`, `/v3/images/models`, `/v3/audio/speech/models`, `/v3/audio/transcriptions/models`, `/v3/videos/models`, `/v3/moderations/models` and `/v3/alpha/decisions/models`.
- **Text-to-speech voices:** `GET /v3/info/audio/tts/voices` lists them.

## LLM chat: `POST /v3/chat/completions`

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["EDENAI_API_KEY"], base_url="https://api.edenai.run/v3")
resp = client.chat.completions.create(
    model="anthropic/claude-sonnet-latest",
    messages=[{"role": "user", "content": "Explain photosynthesis in one sentence."}],
    extra_body={"fallbacks": ["openai/gpt-latest"], "tags": {"project": "demo"}},  # Eden AI extensions
)
print(resp.choices[0].message.content)
```

**Model names:**
- **`provider/model`** pins the provider: `anthropic/claude-sonnet-latest`, `openai/gpt-latest`, `vertex/gemini-flash-latest`. The catalog's `alias_of` shows the dated model an alias currently points to. Pin a dated id when you need reproducibility.
- **A bare model name routes it.** Send `gpt-oss-120b` and Eden AI picks one of the providers serving it, the cheapest by default, and fails over between them. You can steer this with `routing.sort` (`cost`, `speed`, `latency` or `exact`) or a suffix like `gpt-oss-120b:latency`. `routing.allowed_providers` and `routing.allow_fallbacks` restrict it.
- **`@edenai` picks the model itself** from optional `router_candidates`. `routing.quality_cost` runs from 0 (best model) to 10 (cheapest model that can still do it).
- **`<model>@eu`** pins a region.
- More in [references/routing-and-reliability.md](references/routing-and-reliability.md).

**What works beyond plain OpenAI:**
- **`fallbacks`:** up to 3 models, tried in order; more than 3 returns 422. Failed attempts aren't billed.
- **`tags`:** up to 10 key/value strings, to split cost per customer, project or environment. They can also go in an `X-EdenAI-Tags: client=acme,env=prod` header.
- **`session_id`:** keeps a routed conversation on the provider that holds its prompt cache.
- **Standard OpenAI parameters** pass through: `reasoning_effort`, `response_format` with `json_schema`, `tools`, `web_search_options` and `stream`. Claude also takes `thinking: {"type": "enabled", "budget_tokens": N}`.
- **Files:** attach one as a content block, `{"type": "file", "file": {"file_id": "<id from /v3/upload>"}}`.
- **Cost and provider:** every response carries `cost` (USD, a number) and `provider`. When streaming, send `stream_options: {"include_usage": true}` to get `cost` in the final usage chunk.
- **What Eden AI decided:** send `x-edenai-metadata: enabled` for an `edenai_metadata` block. It shows the requested model, the strategy, which provider served, every attempt with its status, the region, BYOK use and the recorded tags.

Stateful chat is in [references/responses-api.md](references/responses-api.md). In our September 2026 test, chaining with `previous_response_id` returned HTTP 500 for Claude and Mistral, so test it before relying on it. The Anthropic SDK and Claude Code are in [references/anthropic-messages.md](references/anthropic-messages.md).

## Expert models: `POST /v3/universal-ai`

One request shape for every non-LLM feature:

```bash
curl -s -X POST https://api.edenai.run/v3/universal-ai \
  -H "Authorization: Bearer $EDENAI_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "model": "ocr/financial_parser/mindee",
    "fallbacks": ["ocr/financial_parser/veryfi"],
    "input": {"file": "https://example.com/invoice.pdf"}
  }'
```

- **`model`:** `feature/subfeature/provider[/model]`, for example `text/moderation/openai` or `image/generation/openai/gpt-image-2`.
- **`input`:** the feature's own fields, such as `text`, `file`, `language`, `query` or `url`. `GET /v3/info/{feature}/{subfeature}` gives the exact schema. A file field takes a public URL or a `file_id` from `/v3/upload`.
- **`fallbacks`:** up to 3 more model strings for the same feature. They aren't allowed on `audio/tts` or `image/face_recognition`.
- **Optional fields:**
  - `provider_params` passes native provider options. It isn't validated, and it's sometimes restricted to an allow-list.
  - `show_original_response: true` adds the provider's raw reply.
  - `tags` works as it does for chat.

The response is **always HTTP 200 once the request is valid, even when the provider failed.** Check `status`:

```json
{"status": "success", "cost": "0.0100", "provider": "mindee", "feature": "ocr", "subfeature": "financial_parser", "output": {"...": "..."}}
{"status": "fail", "cost": "0", "provider": "mindee", "error": {"message": "..."}, "...": "..."}
```

`cost` is a string of USD here, so convert it with `float()`.

**Feature catalog (September 2026).** This is a snapshot; check `GET /v3/info` for the live list.

| Category | Subfeatures (`_async` ones use the async endpoint) |
|---|---|
| `text` | `moderation`, `ai_detection`, `named_entity_recognition`, `topic_extraction`, `spell_check`, `anonymization`, `plagia_detection` |
| `web` | `search`, `scraping`, `map`, `crawl_async`, `batch_scrape_async`, `structured_extraction_async`, `research_async` |
| `ocr` | `ocr`, `ocr_async`, `ocr_tables_async`, `financial_parser`, `identity_parser`, `resume_parser` |
| `image` | `generation`, `background_removal`, `object_detection`, `face_detection`, `face_compare`, `face_recognition`, `explicit_content`, `logo_detection`, `anonymization`, `ai_detection`, `deepfake_detection` |
| `translation` | `automatic_translation`, `document_translation` |
| `audio` | `tts`, `speech_to_text_async` |
| `video` | `generation_async`, `deepfake_detection_async` |

Providers per feature, output shapes and `face_recognition` collections are covered in [references/universal-ai.md](references/universal-ai.md).

## Async jobs: `/v3/universal-ai/async`

Subfeatures ending in `_async` run as jobs: speech-to-text, multipage OCR, tables, crawls, deep research and video.

```python
import os, time, requests

API = "https://api.edenai.run/v3"
H = {"Authorization": f"Bearer {os.environ['EDENAI_API_KEY']}"}

job = requests.post(f"{API}/universal-ai/async", headers=H, json={
    "model": "audio/speech_to_text_async/assembly",
    "input": {"file": "https://example.com/meeting.mp3", "language": "en", "speakers": 2},
}).json()                                       # HTTP 202: {"public_id": "...", "status": "processing", ...}

delay = 2
while True:
    body = requests.get(f"{API}/universal-ai/async/{job['public_id']}", headers=H).json()
    if body["status"] != "processing":          # "success" or "fail"
        break
    time.sleep(delay)
    delay = min(delay * 2, 30)
if body["status"] == "fail":
    raise RuntimeError(body["error"])
print(body["output"]["text"], body["cost"])
```

- **Job ids:** the id is `public_id`, not `job_id`. Statuses are `processing`, `success` and `fail`, and the result is in `output`.
- **Webhooks instead of polling:** add `"webhook_receiver": "https://…"` and optionally `user_webhook_parameters`. Eden AI then POSTs a signed `async_job_completed` payload when the job ends.
- **Managing jobs:** `GET /v3/universal-ai/async` lists jobs, and `DELETE /v3/universal-ai/async/{id}` deletes one.
- Signature checks are in [references/universal-ai.md](references/universal-ai.md).

## Files: `/v3/upload`

- **Upload:** `POST /v3/upload` takes multipart `file`, plus optional `expires_in_days` (1–30) and `purpose`. It returns a `file_id`. In our test, a file uploaded without `expires_in_days` came back with an `expires_at` 30 days out, although one docs page says 7. Set it explicitly when it matters.
- **Use it:** in Universal AI `input.file`, in a chat `file` content block, in image edits (`images: [{"file_id": …}]`) and in videos (`input_reference: {"file_id": …}`).
- **Manage files:** `GET /v3/upload` lists them. `POST /v3/upload/delete` with `{"file_ids": [...]}` deletes some, and `DELETE /v3/upload` deletes all.
- **Public URLs** work wherever a file is expected, so you only need to upload local files.

## MCP server

`https://mcp.edenai.run/mcp` uses streamable HTTP and `Authorization: Bearer $EDENAI_API_KEY`.
- **Tools:** one per expert feature (`ocr`, `ocr_financial_parser`, `web_search`, `web_research`, `audio_tts`, …), plus `upload_file`, `check_job` and `list_models`.
- **Claude Code:**

  ```bash
  claude mcp add --transport http edenai https://mcp.edenai.run/mcp --header "Authorization: Bearer $EDENAI_API_KEY"
  ```

- **Limitation:** tools take only the unified inputs. For provider options such as `provider_params`, call the REST API.
- More in [references/mcp-server.md](references/mcp-server.md).

## Errors and retries

| Signal | Meaning | What to do |
|---|---|---|
| Universal AI HTTP 200 with `"status": "fail"` | The provider failed | Add `fallbacks`, or read `error.message` |
| 400 | Bad request, refused before any provider is called and not billed. Examples are an invalid tag, or a file URL Eden AI couldn't fetch (`Unable to access file URL (status 404)`) | Fix the request, or upload the file |
| 401 / 403 | Bad or missing key, or an inference key used on `/v3/manage` | Don't retry |
| 402 | Not enough credits | Top up, or check the key's budget |
| 422 | Validation error, for example more than 3 fallbacks or a wrong `input` field | Check `GET /v3/info/...` for the schema |
| 429 | Rate limited. The default is 10 requests per second per account, shared by all its keys | Back off and retry |
| 451 | The model isn't allowed on the EU endpoint | Pick an EU-listed model from `api.eu.edenai.run/v3/models` |
| 5xx | Eden AI or provider trouble | Retry with backoff, and keep `fallbacks` set |

OpenAI-compatible endpoints return errors in OpenAI's `{"error": {...}}` shape. Universal AI and management endpoints use `{"detail": ...}`.

## Cost, usage and caching

- **Cost:** every response carries `cost` in USD.
  - Universal AI returns it as a string.
  - `/v3/audio/speech` returns raw audio, with the cost in the `x-edenai-cost` header.
  - A video's cost stays 0 until the job completes.
- **Pricing:** you pay the provider's price with no markup, plus a 5.5% platform fee on the self-serve plan.
- **Cost splits:** tag calls (`tags` or `X-EdenAI-Tags`) and see the split in the dashboard. Pull usage programmatically from `GET /v3/manage/usage/` with a management key. The old `/v2` cost endpoints are no longer documented, and the one we tried returned 405.
- **Response caching:** identical requests (same model, same input) are answered from Eden AI's cache for free. It's on by default and can be toggled per project in the dashboard. Tags don't make requests different.
- **Prompt caching** is separate: the provider still generates a fresh answer from a cached prefix. See [references/routing-and-reliability.md](references/routing-and-reliability.md).

## Good defaults

- **Read the key** from `EDENAI_API_KEY`, and never log it or put it in a URL.
- **Check the live catalog** before writing a model string, and prefer `-latest` aliases or bare routable names in long-lived code.
- **Set `fallbacks`** in production so a provider outage doesn't fail the request. To compare providers side by side, send parallel requests yourself: `fallbacks` is sequential.
- **Prefer specialized features.** `ocr/financial_parser` returns vendor, totals and line items. Don't regex them out of `ocr/ocr` text.
- **Check `status`** on every Universal AI response, not just the HTTP code.
- **Use the async endpoint** for `_async` features, and webhooks when the caller can receive them.
- **Report `cost`** when the user is comparing options, and tag calls when cost needs splitting.
- **Use a sandbox key** for tests and CI.

## Quick recipes

- **"Parse this invoice and get the line items."** Universal AI with `ocr/financial_parser/mindee`, falling back to `ocr/financial_parser/veryfi`.
- **"Transcribe this meeting and label the speakers."** Async `audio/speech_to_text_async/assembly` (or `/deepgram`, `/gladia`) with `input.speakers`. Or use `POST /v3/audio/transcriptions` for the OpenAI shape.
- **"Give my agent web search."** `web/search/tavily` (or `firecrawl`, `linkup`) with `{"query": …, "max_results": 5}`. Use `web/scraping/firecrawl` for one page, and async `web/research_async/tavily` for a cited research report. Or add the MCP server and let the agent call `web_search`.
- **"Which LLM is cheapest for this?"** Send a bare routable name, which routes to the cheapest provider by default. To compare different models, fire parallel `/v3/chat/completions` calls and compare `cost`.
- **"Fall back from OpenAI to Anthropic."** `{"model": "openai/gpt-latest", "fallbacks": ["anthropic/claude-sonnet-latest"]}` on `/v3/chat/completions`.
- **"Embed these documents for RAG."** `POST /v3/embeddings` with a list `input`. Pick the model from `GET /v3/embeddings/models`.
- **"Generate a product image."** `POST /v3/images/generations` with `vertex/gemini-2.5-flash-image`, or Universal AI `image/generation/openai/gpt-image-2`. Universal AI image generation also takes `reference_images`.
- **"Generate a video."** `POST /v3/videos` (OpenAI shape) or async `video/generation_async/{provider}` (for example `pixverse/v6` or `pruna`). Use `provider_params` for native audio or aspect ratio.
- **"Translate this, then moderate it."** `translation/automatic_translation/deepl`, then `text/moderation/microsoft` with `fallbacks: ["text/moderation/google"]`.
- **"Track cost per customer."** Put `tags: {"client": "acme"}` on every call, or set `default_tags` on that customer's API key.
- **"Run Claude Code through Eden AI."** Set `ANTHROPIC_BASE_URL=https://api.edenai.run/v3` and `ANTHROPIC_API_KEY=$EDENAI_API_KEY`. See [references/anthropic-messages.md](references/anthropic-messages.md).
