# Claude Gmail

## Purpose
Gmail newsletter digest automation. Fetches newsletters from the Gmail Forum tab, analyzes them with parallel sub-agents, and delivers a formatted digest email to the inbox every day at 7 AM.

## Key Skills Used
- `newsletter-digest` — the main workflow (fetch → analyze → email)
- `gmail-inbox` — general Gmail management
- `gmail-label` — labeling and organizing emails
- `migrate-subscriptions` — moving newsletter subscriptions between accounts
- `subscription-discovery` — finding active subscriptions

## Key Sub-agents Used
- `newsletter-analyzer` — one instance per email, runs in parallel

## MCP Tools Used
- `mcp__claude_ai_Gmail__search_threads` — find Forum-tab emails from past 24h
- `mcp__claude_ai_Gmail__get_thread` — read full email content
- `mcp__claude_ai_Gmail__create_draft` — send the digest to inbox

## Accounts
- Primary newsletter account: `xingjian.zh@gmail.com`
- Stanford account (subscription discovery only): `xjz@stanford.edu`

## Schedule
Daily at 7:00 AM via `/schedule` skill (Anthropic cloud trigger).

## Notes
- Gmail Forum tab (`category:forums`) automatically catches newsletters — no manual filter needed
- Sub-agents only have Read/Write tools; skill pre-writes email content to `.tmp/emails/` before spawning them
- Digest output: HTML email with columns: Date | Sender | Subject | Summary | Key Statistics | Link
