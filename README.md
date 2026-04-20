# Eden AI skill for Claude Code

An agent skill that teaches [Claude Code](https://claude.com/claude-code) to use [Eden AI](https://edenai.co) — a unified API for 500+ AI models across OpenAI, Anthropic, Google, AWS, Mistral, Cohere, Stability, ElevenLabs, Deepgram, Replicate, and many more providers.

Once installed, Claude will reach for Eden AI automatically whenever you ask for:

- LLM chat / completion against any provider (OpenAI-compatible)
- OCR & document parsing — invoices, receipts, IDs, resumes, tables
- Image generation, object / face / logo detection, background removal, deepfake detection
- Text-to-speech and speech-to-text with speaker diarization
- Video generation
- Translation, moderation, NER, topic extraction, AI-content detection
- Provider comparison, fallback, or cost benchmarking on the same input

You don't need to name the skill — Claude matches it from the task.

## Install

### User scope (available in every Claude Code session)

```bash
git clone https://github.com/edenai/edenai-skill.git ~/.claude/skills/edenai
```

### Project scope (available only in one repo)

From inside the project directory:

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

> *"Compare GPT-4o and Claude Opus on this prompt and show me both responses with cost."*

> *"Parse the line items out of this invoice PDF."*

> *"Generate three product photos for a ceramic mug using Stability and DALL-E 3."*

> *"Moderate this user comment across Google, Microsoft, and OpenAI and flag the strictest verdict."*

## What's in the skill

- The two Eden AI surfaces (`/v3/llm/*` OpenAI-compatible + `/v2/*` task-specific).
- The full feature catalog by category — text, image, audio, video, OCR, documents, translation — with endpoints and supporting providers.
- Provider prefixes for the 500+ model LLM catalog.
- Canonical multi-provider request shape, per-provider response handling, cost field.
- Async polling pattern (with webhook alternative).
- Error handling — HTTP-level vs. per-provider `status: "fail"` inside a 200.
- Good-defaults guardrails (never log the key, start with one provider, prefer specialized endpoints, surface cost, use `fallback_providers` for production reliability).

## Updating

```bash
cd ~/.claude/skills/edenai && git pull
```

## Uninstalling

```bash
rm -rf ~/.claude/skills/edenai
```

## Links

- Eden AI: https://edenai.co
- API docs: https://edenai.co/docs
- Live LLM catalog: `GET https://api.edenai.run/v3/llm/models`
- Full feature inventory: `GET https://api.edenai.run/v3/info/`

## Contributing

The skill is a single `SKILL.md` file. Edits welcome — open a PR on https://github.com/edenai/edenai-skill.

## License

MIT
