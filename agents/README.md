# Harness Adapters

Harness-specific metadata for installing this skill. The actual skill content
lives in `../SKILL.md` and `../references/`; everything here is just the
small surface a given agent harness expects (display name, default prompt,
invocation policy).

Adapters present today:

- `openai.yaml` — OpenAI skill metadata for Codex and ChatGPT Skills. Used when
  this directory is installed under `~/.codex/skills/`, `.codex/skills/`, or
  uploaded to a ChatGPT workspace that supports Skills.

Claude Code does not need an adapter file: it reads `SKILL.md` directly from
the skill directory. AGENTS.md-aware tools (Cursor, Aider, etc.) discover the
skill via a pointer in the consuming project's `AGENTS.md`; see the top-level
`README.md` for the pointer template.

Add a new adapter file here when porting to another harness rather than
duplicating skill content. Keep adapters thin — they should only carry
harness-specific install metadata, not packaging guidance.
