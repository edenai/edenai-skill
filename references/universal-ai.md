# Universal AI: expert models in detail

`POST /v3/universal-ai` (sync) and `POST /v3/universal-ai/async` (for `_async` subfeatures) run every non-LLM feature with one request shape. Read this for the providers of a feature, its input and output fields, `provider_params`, webhooks and face-recognition collections.

## Request and response

```json
{
  "model": "feature/subfeature/provider[/model]",
  "input": {"...": "the feature's own fields"},
  "fallbacks": ["feature/subfeature/other_provider"],
  "provider_params": {"...": "native provider options"},
  "show_original_response": false,
  "tags": {"project": "invoices"}
}
```

A synchronous response is HTTP 200 whenever the request itself was valid:

```json
{
  "status": "success",
  "cost": "0.0015",
  "provider": "google",
  "feature": "ocr",
  "subfeature": "ocr",
  "output": {"text": "...", "bounding_boxes": [...]},
  "error": null
}
```

- **`status` is `success` or `fail`.** A provider failure comes back as HTTP 200 with `"status": "fail"` and `error.message`, so always check `status`.
- **`cost`** is a string of USD.
- **`original_response`** is the provider's raw reply, present only with `show_original_response: true`.
- **HTTP errors** mean the request never reached a provider: 401 or 403 for keys, 402 for credits, 422 for a bad `model` or `input`, 429 for rate limits, and 451 for a non-EU model on the EU endpoint.

## Providers and inputs (September 2026)

This is a snapshot of `GET /v3/info/{feature}/{subfeature}`. That endpoint is public and returns the live providers, the full `provider/model` strings, pricing, `input_schema` and `output_schema`. A `*` marks a required input.

| Feature | Providers | Input |
|---|---|---|
| `text/ai_detection` | sapling, winstonai | text* |
| `text/moderation` | google, microsoft, openai | text*, language |
| `text/anonymization` | amazon, microsoft | text*, language |
| `text/spell_check` | prowritingaid, sapling | text*, language |
| `text/topic_extraction` | google, openai, tenstorrent | text*, language |
| `text/named_entity_recognition` | amazon, microsoft, openai, tenstorrent | text*, language |
| `text/plagia_detection` | winstonai | text*, title |
| `web/search` | firecrawl, linkup, tavily | query*, max_results, depth |
| `web/scraping` | firecrawl, linkup, tavily | url* |
| `web/map` | firecrawl, tavily | url* |
| `web/crawl_async` | firecrawl, tavily | url* |
| `web/batch_scrape_async` | firecrawl | urls* |
| `web/structured_extraction_async` | firecrawl | urls*, prompt, schema |
| `web/research_async` | firecrawl, linkup, tavily | prompt*, reasoning_depth, schema |
| `ocr/ocr` | amazon, api4ai, google, ionos, microsoft, mistral, sentisight | file*, language |
| `ocr/ocr_async` | amazon, microsoft, mistral | file* |
| `ocr/ocr_tables_async` | amazon, google, microsoft | file* |
| `ocr/financial_parser` | affinda, amazon, base64, eagledoc, extracta, google, klippa, microsoft, mindee, openai, tabscanner, veryfi | file*, language, document_type |
| `ocr/identity_parser` | affinda, amazon, base64, klippa, microsoft, mindee, openai | file* |
| `ocr/resume_parser` | affinda, extracta, klippa, openai, senseloaf | file* |
| `image/generation` | bytedance, google, leonardo, minimax, openai, replicate, stabilityai | text*, resolution, num_images, reference_images, mask |
| `image/background_removal` | api4ai, clipdrop, photoroom, picsart, sentisight, stabilityai | file* |
| `image/object_detection` | amazon, api4ai, google, microsoft, sentisight | file* |
| `image/face_detection` | amazon, api4ai, google | file* |
| `image/face_compare` | amazon, base64, facepp | file1*, file2* |
| `image/face_recognition` | amazon | collection_id*, file* |
| `image/explicit_content` | amazon, google, microsoft, openai, sentisight | file* |
| `image/logo_detection` | api4ai, google, microsoft, openai | file* |
| `image/anonymization` | api4ai | file* |
| `image/ai_detection` | resemble, winstonai | file* |
| `image/deepfake_detection` | resemble, sightengine | file* |
| `translation/automatic_translation` | amazon, deepl, google, microsoft, modernmt, openai | text*, target_language*, source_language |
| `translation/document_translation` | deepl, google | file*, target_language*, source_language |
| `audio/tts` | amazon, deepgram, elevenlabs, google, gradium, lovoai, microsoft, openai | text*, voice, speed, audio_format, speaking_pitch, speaking_volume |
| `audio/speech_to_text_async` | amazon, assembly, deepgram, gladia, google, gradium, microsoft, openai | file*, language, speakers, profanity_filter, vocabulary |
| `video/generation_async` | amazon, bytedance, google, minimax, openai, pixverse, pruna, xai | text*, file, duration, dimension, seed |
| `video/deepfake_detection_async` | resemble | file* |

**Model string details:**
- **Some providers need a model suffix.** OpenAI text-to-speech is `audio/tts/openai/tts-1` or `.../gpt-4o-mini-tts`, not a bare `audio/tts/openai`. PixVerse video is `video/generation_async/pixverse/v6`. The `models` list in `/v3/info/{feature}/{subfeature}` gives the exact strings.
- **AssemblyAI** is `assembly` in model strings.
- **Voices** are listed by `GET /v3/info/audio/tts/voices`.
- **File inputs** (`file`, `file1`, `file2`, `reference_images`) take a public URL or a `file_id` from `POST /v3/upload`.

## Output shapes worth knowing

| Feature | Where the result is |
|---|---|
| `image/generation` | `output.items[i].image` (base64) and `output.items[i].image_resource_url` |
| `audio/tts` | `output.audio_resource_url` |
| `audio/speech_to_text_async` | `output.text`, plus `output.diarization.entries[]` with `speaker`, `start_time`, `end_time` |
| `video/generation_async` | `output.video_resource_url` |
| `ocr/ocr` | `output.text` and `output.bounding_boxes` |
| `web/search` | `output.results[]` with `title`, `url`, `content`, `score`, plus `output.answer` when the provider gives one |
| `web/research_async` | `output.answer` and `output.sources[]` |
| `translation/automatic_translation` | `output.text` |

Resource URLs are signed CDN links that expire, so download anything you want to keep. They don't need the Eden AI key.

## `provider_params`

`provider_params` forwards native options to the provider alongside `input`:

```json
{
  "model": "video/generation_async/pixverse/v6",
  "input": {"text": "A lighthouse in a storm, slow push-in", "duration": 5},
  "provider_params": {"generate_audio_switch": true, "aspect_ratio": "16:9", "quality": "720p"}
}
```

- **Only for that provider:** the options apply to the provider in `model`, so switching provider usually means changing them.
- **Not validated by Eden AI, but sometimes restricted:** a bad value comes back as a provider error. Some providers accept only an allow-list of keys, and a key outside it is refused with an error that names it.
- **Examples in the docs:**
  - `quality` and `style` for OpenAI image generation.
  - `language_code` and `prompt` for Gemini text-to-speech.

## Async jobs

1. **Create the job:** `POST /v3/universal-ai/async` with the usual body, plus optional `webhook_receiver` and `user_webhook_parameters`. It returns HTTP 202 with `public_id`, `status: "processing"`, `created_at` and `provider`.
2. **Poll it:** `GET /v3/universal-ai/async/{public_id}` returns the same object. `status` goes from `processing` to `success` or `fail`. `output` and `cost` are filled on success, and `error` on fail.
3. **List or delete:** `GET /v3/universal-ai/async` lists your jobs, and `DELETE /v3/universal-ai/async/{public_id}` deletes one.

Poll with backoff, starting around 2 s and capping around 30 s. Video and deep-research jobs take one to several minutes.

## Webhooks

With `webhook_receiver` set, Eden AI POSTs this to your HTTPS URL when the job ends:

```json
{
  "event": "async_job_completed",
  "job_id": "550e8400-...",
  "status": "success",
  "feature": "ocr",
  "subfeature": "ocr_async",
  "provider": "amazon",
  "model": null,
  "created_at": "2026-04-21T12:00:00+00:00",
  "finished_at": "2026-04-21T12:01:30+00:00",
  "output": {"raw_text": "..."},
  "error": null,
  "user_parameters": {"internal_ref": "order-42"}
}
```

**Headers:**
- `X-Edenai-Webhook: true`
- `X-Edenai-Signature`: a hex RSA PKCS1 v1.5 signature.
- `X-Edenai-Hash-Algorithm: SHA256`

**Verifying the signature:**
1. Parse the body.
2. Re-serialize it as canonical JSON: sorted keys, 2-space indent, UTF-8.
3. Take the SHA-256 **hex digest**.
4. Verify the signature over that hex string with Eden AI's public key, which you get from support (`webhook_rsa.pub.pem`).

```python
import hashlib, orjson
from Crypto.Hash import SHA256
from Crypto.PublicKey import RSA
from Crypto.Signature import PKCS1_v1_5

KEY = RSA.import_key(open("edenai_webhook_rsa.pub.pem", "rb").read())

def verify(raw_body: bytes, signature_hex: str) -> bool:
    canonical = orjson.dumps(orjson.loads(raw_body), option=orjson.OPT_INDENT_2 | orjson.OPT_SORT_KEYS)
    digest = SHA256.new(hashlib.sha256(canonical).hexdigest().encode())
    try:
        return PKCS1_v1_5.new(KEY).verify(digest, bytes.fromhex(signature_hex))
    except ValueError:
        return False
```

## Face-recognition collections

`image/face_recognition` matches a face against a collection you build first. A collection is tied to one provider, so fallbacks aren't allowed.

```
POST   /v3/universal-ai/collections                       {"model": "image/face_recognition/amazon", "name": "staff"}  -> collection id
POST   /v3/universal-ai/collections/{id}/items            {"file": "<file_id or URL>"}  -> provider-side face ids
GET    /v3/universal-ai/collections[/{id}[/items]]        list or inspect
DELETE /v3/universal-ai/collections/{id}[/items/{item_id}]
POST   /v3/universal-ai  {"model": "image/face_recognition/amazon", "input": {"collection_id": "<id>", "file": "<image>"}}
                     -> output.items[] with face_id and confidence
```
