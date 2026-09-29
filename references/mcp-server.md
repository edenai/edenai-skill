# MCP server: expert models as agent tools

Eden AI hosts a Model Context Protocol server that exposes the expert-model catalog as tools: OCR, web search, speech, translation, image analysis and the rest. Any MCP client, or any function-calling LLM loop, can call them. Source: [MCP Server](https://www.edenai.co/docs/v3/expert-models/mcp-server).

| | |
|---|---|
| URL | `https://mcp.edenai.run/mcp` |
| Transport | Streamable HTTP |
| Auth | `Authorization: Bearer $EDENAI_API_KEY` |

- **Billing:** every tool call is a normal Eden AI call, billed to that key.
- **Visibility:** each call appears in monitoring.
- **Rules:** rate limits and data-governance rules apply as they do on the REST API.

## Connect a client

**Claude Code**, for yourself:

```bash
claude mcp add --transport http edenai https://mcp.edenai.run/mcp \
  --header "Authorization: Bearer $EDENAI_API_KEY"
```

**Claude Code**, shared with a team through the project's `.mcp.json`. Keep `"type": "http"`, or it's read as a stdio server. Run `/mcp` to check the connection.

```json
{
  "mcpServers": {
    "edenai": {
      "type": "http",
      "url": "https://mcp.edenai.run/mcp",
      "headers": {"Authorization": "Bearer ${EDENAI_API_KEY}"}
    }
  }
}
```

**Other clients:** the docs have ready-made blocks for Codex CLI (`~/.codex/config.toml`), Cline, Continue, OpenCode, Hermes Agent and OpenClaw.

## Tools

- **One tool per expert feature,** named `category_subfeature` with the `_async` suffix dropped. There were 40 tools in September 2026. Some examples:
  - `text_moderation`, `ocr_financial_parser` and `translation_automatic_translation`;
  - `web_search`, `web_scraping`, `web_research` and `web_crawl`;
  - `audio_tts`, `audio_speech_to_text`, `image_generation` and `video_generation`;
  - `ocr` and `ocr_async`, which are the exception.
- **`upload_file`** takes `content_base64`, `filename` and `expires_in_days` (1–30, default 30). It returns a `file_id`.
- **`check_job`** takes a `job_id`. It returns `status` (`success`, `fail` or still running) and the result.
- **`list_models`** lists providers, models and pricing, optionally for one tool: `{"tool": "web_search"}`.

**Conventions shared by every feature tool:**
- **`model` picks the provider:** `"provider"` or `"provider/model"`, for example `"amazon"` or `"tavily"`.
- **File parameters** take a public URL or a `file_id` from `upload_file`.
- **Long-running tools** (their description says "Long-running") return a `job_id`. Poll it with `check_job`.
- **Discovery:** every tool publishes a JSON Schema, so discover the catalog at runtime with `list_tools()` rather than hardcoding it.

**Limitations:**
- **No provider options:** the tools take only the unified inputs, not `provider_params`. For provider-specific options, such as PixVerse's native audio switch, call `/v3/universal-ai` directly.
- **Callers:** any model whose `capabilities.supports_function_calling` is true in `GET /v3/models` can call the tools.

## Using the tools without an MCP client

Fetch the catalog with the MCP Python SDK (`pip install "mcp>=2"`). Hand the schemas to any model as function tools through the Eden AI gateway, then run the calls the model asks for:

```python
from mcp import ClientSession
from mcp.client.streamable_http import create_mcp_http_client, streamable_http_client

async with (
    create_mcp_http_client(headers={"Authorization": f"Bearer {API_KEY}"}) as http,
    streamable_http_client("https://mcp.edenai.run/mcp", http_client=http) as (read, write),
    ClientSession(read, write) as mcp,
):
    await mcp.initialize()
    tools = [{"type": "function", "function": {"name": t.name, "description": t.description or "", "parameters": t.input_schema}}
             for t in (await mcp.list_tools()).tools if t.name in {"web_search", "ocr"}]
    # loop: client.chat.completions.create(model=..., messages=..., tools=tools)
    #       for each tool call: result = await mcp.call_tool(name, json.loads(arguments))
```

## Best practices

- **Send only the tools the task needs.** The full catalog is a lot of schema for one prompt.
- **Cap tool results at about 20,000 characters.** OCR and scraping output can be huge.
- **Rebuild assistant tool calls from the spec fields only:** `{id, type, function: {name, arguments}}`, with `""` rather than `null` content. Strict providers reject extra fields such as `index`.
- **Treat an `is_error` result as data.** Pass it back to the model, which usually retries with corrected arguments.
- **Rename a custom `web_search` tool** (for example to `internet_search`) on `/v3/v1/messages`, where it collides with some models' native web search.
- **Put file ids in a fenced code block** when you mention them in a prompt. Models sometimes mistype a UUID written in prose.
