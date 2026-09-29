# OpenAI-compatible media endpoints

Eden AI serves the OpenAI shapes for embeddings, images, audio, video and moderation, so the official OpenAI SDK works with `base_url="https://api.edenai.run/v3"` and the Eden AI key.

- **Model strings:** these endpoints take `provider/model` strings. Each has its own public model list, and all support `?view=models`.
- **Extras:** responses add Eden AI's `cost` (USD) and `provider`. Tags go in the `tags` body field, or in the `X-EdenAI-Tags` header on multipart endpoints.
- **Versus Universal AI:** the same providers are mostly reachable through `/v3/universal-ai` too. Use these endpoints when the code already speaks OpenAI; use Universal AI for `fallbacks`, `provider_params` on more features, and one shape for everything.

## Embeddings: `POST /v3/embeddings`

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.environ["EDENAI_API_KEY"], base_url="https://api.edenai.run/v3")
out = client.embeddings.create(model="mistral/mistral-embed", input=["first text", "second text"])
vectors = [d.embedding for d in out.data]          # data[i].index matches the input order
```

- **Models:** `GET /v3/embeddings/models` lists them (29 in September 2026: OpenAI, Google, Cohere, Mistral, Databricks…).
- **Fields:** `input` takes a string, a list of strings or token ids. `encoding_format` is `float` or `base64`. `dimensions` works only on models that support it, such as `openai/text-embedding-3-*`.
- **Unknown fields** are forwarded to the provider. `metadata` and `user` are accepted but aren't tags.
- **Batching:** send a list `input`, which is much cheaper than one call per text. Identical requests are served free from the response cache.

## Images: `POST /v3/images/generations` and `POST /v3/images/edits`

```bash
curl -s https://api.edenai.run/v3/images/generations \
  -H "Authorization: Bearer $EDENAI_API_KEY" -H "Content-Type: application/json" \
  -d '{"model": "vertex/gemini-2.5-flash-image", "prompt": "A ceramic mug on a wooden table, soft window light", "n": 1, "size": "1024x1024"}'
```

- **Models:** `GET /v3/images/models` lists them, for example `vertex/gemini-3.1-flash-image`, `openai/gpt-image-2.5-flare` and `stabilityai/sd3.5-large`.
- **Fields:**
  - `prompt`, `n` (1–10), `size` and `quality`, where the accepted values depend on the model.
  - `response_format`: `url` or `b64_json`.
  - Unknown fields are forwarded.
- **Response:** `data: [{"b64_json" | "url"}]` plus `cost` and `provider`. Hosted URLs can expire, so download them.
- **Edits:** the same fields plus `images: [{"file_id": "..."} or {"image_url": "https://... or data:image/...;base64,..."}]` and an optional mask. Send JSON for that form, or multipart for the OpenAI SDK's file-upload form. `openai/gpt-image-*` needs a `data:` URL or a `file_id`, not a remote URL.
- **Limits:**
  - There are no `fallbacks` and no streaming.
  - `/v3/images/variations` doesn't exist.
  - Sending an image model to `/v3/chat/completions` returns 400.
  - In our test, an edit on `vertex/gemini-3.1-flash-image` with a `file_id` returned 400 `This feature is currently not available`. Universal AI `image/generation/google/gemini-2.5-flash-image` with `input.reference_images` did work, keeping a character consistent across images.

## Audio

**Text-to-speech: `POST /v3/audio/speech`**
- **Request:** JSON `{model, input, voice, response_format, speed, instructions}`.
- **Response:** raw audio bytes. The `x-edenai-cost` and `x-edenai-provider` response headers carry the cost and provider.
- **Models:** `GET /v3/audio/speech/models` lists them, for example `openai/tts-1` and `microsoft/tts-1`.
- **Voices:** `GET /v3/audio/voices?model=...` lists them.

**Speech-to-text: `POST /v3/audio/transcriptions`**
- **Request:** multipart (the OpenAI SDK shape: a `file` part plus fields), or JSON with `file_id` or `file_url` (an https URL or a base64 data URL).
- **Fields:** `language`, `prompt`, `response_format` (`json`, `text`, `srt`, `verbose_json` or `vtt`), `timestamp_granularities` and `temperature`.
- **Models:** `GET /v3/audio/transcriptions/models` lists 65, across OpenAI, Microsoft, Deepgram and others.

For diarization with a speaker count, Universal AI `audio/speech_to_text_async` with `input.speakers` is the richer option.

## Video: `/v3/videos`

A facade over Universal AI `video/generation_async`: the same providers, prices and jobs, in OpenAI's video shape.

```python
video = client.videos.create_and_poll(model="pruna/p-video", prompt="A paper boat sails down a rainy street", seconds=5)
```

- **Routes:**
  - `POST /v3/videos` takes JSON, or multipart with an `input_reference` file part.
  - `GET /v3/videos/{id}` returns the status. `status` goes `queued`, then `in_progress`, then `completed` or `failed`.
  - `GET /v3/videos/{id}/content` returns the mp4, as a signed-link redirect for 7 days and the bytes after that.
  - `GET /v3/videos` lists jobs, `DELETE /v3/videos/{id}` deletes a finished one, and `GET /v3/videos/models` lists the models.
- **Fields:**
  - `model` must be `provider/model`; a bare name returns 400.
  - `prompt`, `seconds` and `size` (`WIDTHxHEIGHT`).
  - `input_reference: {"file_id": ...}` or `{"image_url": "https://..."}` for image-to-video. Base64 data URLs are refused.
  - Eden AI extras: `seed`, `provider_params` (allow-listed per provider), `webhook_receiver` and `user_webhook_parameters`.
- **Limits:**
  - A JSON body rejects unknown fields with 422, so there are no `fallbacks`, no `@edenai` and no `tags` field. Use the `X-EdenAI-Tags` header instead.
  - `cost` stays 0 until the job finishes.
  - The video id is also a Universal AI job id.
  - Video models aren't in `GET /v3/models`.

## Moderation: `POST /v3/moderations`

OpenAI's moderation shape: `{"model": "openai/omni-moderation-latest", "input": "..."}`. `GET /v3/moderations/models` lists the models, which were OpenAI's only in September 2026. For Google or Microsoft moderation, use Universal AI `text/moderation/{provider}`.

## Decisions (alpha): `POST /v3/alpha/decisions`

A decision model answers typed questions about some `state`, returning numbers instead of prose, in one fast request. It's billed on input only, at about $0.042 per million input tokens for `typesafe/jev-latest`. The endpoint is alpha, so its shape and model ids can change without notice. Keep it behind a small adapter.

```json
{
  "model": "typesafe/jev-latest",
  "state": {"subject": "Duplicate charge", "message": "I was charged twice and nobody answers!"},
  "questions": {
    "is_urgent":   {"type": "noul", "instructions": "Does this message express urgency?"},
    "department":  {"type": "choice", "instructions": "Which team should handle this?",
                    "criteria": {"billing": "Payments, refunds", "technical": "Bugs, outages", "sales": "Plans, pricing"}},
    "frustration": {"type": "score", "instructions": "How frustrated is the customer?",
                    "criteria": ["Calm", "Frustrated but civil", "Very angry"]}
  }
}
```

- **Question types:**
  - `noul` returns a probability.
  - `choice` returns the winner, per-option probabilities and a confidence, for up to 255 options.
  - `score` returns a position on your ordered rubric.
- **Batching is nearly free:** ask several questions in one request, because the state is read once.
- **Unsuitable tasks:** prose, extraction, counting and arithmetic.
- **Models:** `GET /v3/alpha/decisions/models` lists them (`typesafe/jev-latest`, `typesafe/jev-preview`, `qwen/decision-model-preview`).
