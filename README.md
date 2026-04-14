# claude-newsletter-digest

A Claude Code skill that fetches Gmail Forum-tab newsletters from the past 24h, analyzes each in parallel via a sub-agent, and sends an HTML digest via the Resend API.

## Contents

- `.claude/skills/newsletter-digest/SKILL.md` — the main skill recipe
- `.claude/skills/newsletter-digest/scripts/digest_template.html` — HTML template reference
- `.claude/agents/newsletter-analyzer.md` — sub-agent definition (one instance per email)
- `CLAUDE.md` — project context

## Local usage

1. Open this folder in Claude Code
2. Copy `.env.example` to `.env` and fill in your Resend credentials
3. Run `/newsletter-digest`

## Remote usage (Anthropic cloud scheduled trigger)

This repo is designed to be cloned by a remote Claude Code trigger:

1. Attach the Gmail MCP connector to the trigger
2. Point the trigger `sources` at this repo
3. Pass `RESEND_API_KEY`, `RESEND_FROM`, `RESEND_TO` inline in the trigger prompt (the remote environment has no `.env` file)
4. Trigger prompt invokes `/newsletter-digest` after the clone

See `SKILL.md` for the full workflow.
