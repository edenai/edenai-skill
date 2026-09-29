# Eden AI skill for Claude Code

An agent skill that teaches [Claude Code](https://claude.com/claude-code) to use [Eden AI](https://edenai.co), an AI gateway with one API key for 500+ models from OpenAI, Anthropic, Google, Mistral, AWS, Azure and 50+ other providers.

Once installed, Claude reaches for Eden AI whenever you ask for:

- **LLM chat** against any provider, through the OpenAI-compatible, OpenAI Responses or Anthropic Messages APIs. This includes routing by price or latency, fallbacks, and the `@edenai` model router.
- **Embeddings, images, audio, video and moderation**, through OpenAI-compatible endpoints.
- **OCR and document parsing:** invoices, receipts, IDs, resumes and tables.
- **Web search, scraping, crawling and deep research** for agents, from Tavily, Firecrawl and Linkup.
- **Speech:** text-to-speech, and speech-to-text with speaker labels.
- **Other expert models:** image and video generation, image analysis, translation, moderation, and AI-content and deepfake detection.
- **Eden AI's MCP server**, which gives an agent the expert models as tools.
- **Account work:** cost tracking per customer with tags, sandbox keys for tests, usage reports and key management through the Management API, BYOK and the EU endpoint.

You don't need to name the skill — Claude matches it from the task.

## Install

### With the `skills` CLI (recommended)

One-liner via the [open agent skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add https://github.com/edenai/edenai-skill
```

Useful flags:

```bash
# Install globally (available in every session) to Claude Code only
npx skills add https://github.com/edenai/edenai-skill -g -a claude-code -y

# Install into the current project only
npx skills add https://github.com/edenai/edenai-skill -a claude-code -y

# Target another supported agent (Cursor, Codex, OpenCode, Gemini CLI, …)
npx skills add https://github.com/edenai/edenai-skill -a cursor
```

`-g` installs to the user scope (`~/.claude/skills/`); without it the skill is installed to the project scope (`./.claude/skills/`). See `npx skills add --help` for the full list of supported agents and options.

### With `git clone`

User scope (available in every Claude Code session):

```bash
git clone https://github.com/edenai/edenai-skill.git ~/.claude/skills/edenai
```

Project scope (available only in one repo) — from inside the project directory:

```bash
mkdir -p .claude/skills
git clone https://github.com/edenai/edenai-skill.git .claude/skills/edenai
```

### Manual

If you'd rather copy the file, place `SKILL.md` at either:

- `~/.claude/skills/edenai/SKILL.md` — user scope
- `<project>/.claude/skills/edenai/SKILL.md` — project scope

Claude Code picks up skills from both locations on startup.

## Configuration

Set your Eden AI API key as an environment variable:

```bash
export EDENAI_API_KEY="your-key-here"
```

Put it in your shell profile (`~/.bashrc`, `~/.zshrc`) so it's available in every session. You can grab a key from the Eden AI dashboard at https://edenai.co.

## Usage

Just ask. Claude will load the skill, build the right request, call Eden AI, and return results grouped by provider (with cost when relevant).

Examples:

> *"Transcribe this meeting recording and identify speakers: `https://example.com/meeting.mp3`"*

> *"Compare GPT and Claude on this prompt and show me both responses with their cost."*

> *"Parse the line items out of this invoice PDF, and fall back to another parser if the first one fails."*

> *"Give my agent web search and page scraping through Eden AI's MCP server."*

> *"Generate three product photos for a ceramic mug with Gemini and GPT Image."*

> *"Tag every call with the customer id so I can see what each customer costs."*

## What's in the skill

- **`SKILL.md`** is the map. It covers:
  - every v3 endpoint, and which one to use for a task;
  - model strings and the live catalogs that list them;
  - chat with Eden AI's extensions;
  - Universal AI's request and response shape, and its feature catalog;
  - async jobs, files and the MCP server;
  - errors, cost, and quick recipes.
- **`references/routing-and-reliability.md`** covers provider routing, `@edenai`, fallbacks, request metadata, and response and prompt caching.
- **`references/universal-ai.md`** has providers and inputs per feature, output shapes, `provider_params`, async jobs, signed webhooks and face-recognition collections.
- **`references/openai-compatible-media.md`** covers embeddings, images, audio, video, moderation, and decisions (alpha).
- **`references/mcp-server.md`** covers connecting a client, the tools, and using them from any LLM.
- **`references/account-and-governance.md`** covers key types, sandbox, tags, the Management API, usage, pricing and limits, BYOK, guardrails and the EU endpoint.
- **`references/responses-api.md`** and **`references/anthropic-messages.md`** cover the other two chat dialects, and Claude Code on Eden AI.

Claude reads a reference file only when a task needs it.

## Updating

```bash
cd ~/.claude/skills/edenai && git pull
```

## Uninstalling

### With the `skills` CLI

```bash
# Remove the skill (prompts for scope)
npx skills remove edenai

# Remove from user scope, Claude Code only, no prompts
npx skills remove edenai -g -a claude-code -y

# Interactive picker across all installed skills
npx skills remove
```

### Manual

```bash
# User scope
rm -rf ~/.claude/skills/edenai

# Project scope
rm -rf .claude/skills/edenai
```

## Links

- Eden AI: https://edenai.co
- API docs: https://edenai.co/docs, with an index for LLMs at https://www.edenai.co/docs/llms.txt
- OpenAPI spec: https://api.edenai.run/v3/docs/openapi.json
- Live LLM catalog (public): `GET https://api.edenai.run/v3/models`
- Live expert-model catalog (public): `GET https://api.edenai.run/v3/info`
- MCP server: `https://mcp.edenai.run/mcp`

## Contributing

The skill is `SKILL.md` plus deeper docs under `references/` that Claude reads on demand. Edits welcome — open a PR on https://github.com/edenai/edenai-skill.

## License

MIT
