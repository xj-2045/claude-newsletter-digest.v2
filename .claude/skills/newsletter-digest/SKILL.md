---
name: newsletter-digest
description: Run a daily digest of newsletter emails in Gmail. Fetches emails via MCP Gmail tools from the past 24h, analyzes each one with a subagent, and delivers an HTML digest email via the Resend API. Use when user asks to run the newsletter digest, summarize newsletters, or review today's newsletters.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Task, Agent, mcp__claude_ai_Gmail__search_threads, mcp__claude_ai_Gmail__get_thread, mcp__claude_ai_Gmail__create_draft
---

# Newsletter Digest

## Goal
Fetch all newsletter emails from the past 24 hours via MCP Gmail tools, analyze each one in parallel using subagents, and deliver a formatted HTML digest email via the Resend API.

Only processes emails in the Forum tab (category:forums) — never touches job recruitment, personal, or other email types.

## Prerequisites
1. Gmail connected via MCP (claude.ai Gmail integration) — no OAuth tokens or `gmail_accounts.json` needed
2. `.env` in the project root with:
   - `RESEND_API_KEY` — from resend.com → API Keys
   - `RESEND_FROM` — verified sender (use `onboarding@resend.dev` for testing)
   - `RESEND_TO` — destination inbox
3. No manual Gmail filter needed — newsletters already go to the Forum tab automatically

## Subagent
- `newsletter-analyzer` — defined in `.claude/agents/newsletter-analyzer.md`
- Model: Sonnet (best quality for nuanced summarization)
- One subagent per email, all launched in parallel

## Flow

### Step 1: Fetch newsletter list via MCP
Use the MCP Gmail search_threads tool to find newsletters from the past 24 hours:
```
mcp__claude_ai_Gmail__search_threads:
  query: "category:forums newer_than:1d"
  pageSize: 50
```

Each returned thread includes a minimal message preview (subject, sender, snippet, date). Filter out any results whose `date` is older than 24 hours (threads may surface older messages).

### Step 2: Fetch full thread bodies via MCP
For each thread from Step 1, fetch the full body in parallel:
```
mcp__claude_ai_Gmail__get_thread:
  threadId: "<id from Step 1>"
  messageFormat: "FULL_CONTENT"
```

Launch all get_thread calls in a single parallel batch. For newsletters (which are almost always single-message threads), use the first message's `plaintextBody` as the body. Very large bodies (>60KB) may be persisted to disk — in that case, summarize from the preview plus snippet rather than reading the persisted file into context.

### Step 3: Write email JSON files
Create `.tmp/emails/` and `.tmp/analyzed/` directories.

For each email, write a JSON file to `.tmp/emails/email_N.json`:
```json
{
  "id": "gmail_message_id",
  "subject": "Newsletter subject line",
  "from": "Sender Name <email@domain.com>",
  "date": "email date string",
  "body_text": "plain text body from MCP response (truncate to ~4000 chars if needed)",
  "original_link": "web version URL if found in body, otherwise Gmail link"
}
```

**Important:** If an email body is very large (>4000 chars), summarize the key content yourself before writing the file, preserving all statistics and factual claims. The subagent only has Read/Write tools and cannot fetch from MCP.

### Step 4: Spawn newsletter-analyzer subagents in parallel
Spawn ALL subagents in a **single message** using the Agent tool with `run_in_background: true`:

For each email file:
```
Agent:
  subagent_type: "newsletter-analyzer"
  run_in_background: true
  prompt: "Read /abs/path/.tmp/emails/email_N.json, analyze it, write the result JSON to /abs/path/.tmp/analyzed/analyzed_N.json"
```

Launch all subagents at once for maximum parallelism.

### Step 5: Wait for all subagents to complete
You will receive task-notification messages as each subagent finishes. Wait until all are done.

### Step 6: Read analyzed results and compile digest
Read all `.tmp/analyzed/analyzed_N.json` files. Each contains:
```json
{
  "sender": "Clean sender name",
  "date": "YYYY-MM-DD",
  "summary_html": "<span style=\"...\">EMOJI TYPE</span><ul>...</ul>",
  "key_stats": ["stat 1", "stat 2"],
  "original_link": "URL or empty string"
}
```

Compile all results into `.tmp/digest.json` (array of all analyzed objects).

### Step 7: Generate HTML digest and deliver

Use the HTML template at `./scripts/digest_template.html` as reference.

**Two delivery paths — pick ONE based on runtime environment:**

- **7a (Local run):** `.env` file exists → send via Resend API with curl. Fast, lands in Inbox directly.
- **7b (Remote cloud run):** no `.env` file, `api.resend.com` is blocked by the Anthropic sandbox egress allowlist → send via Gmail MCP `create_draft` (the MCP host is always allowlisted). Lands in your Gmail Drafts folder.

Detect the path with a simple check at the start of the step:
```bash
[ -f .env ] && echo "local" || echo "remote"
```

**Table columns (no Date, no Subject):**
| Sender | Summary | Key Statistics | 🔗 | ✉️ |

**Column widths:** Sender 10% · Summary 43% · Key Statistics 40% · 🔗 3.5% · ✉️ 3.5%

**Summary column:** Use `summary_html` directly from the analyzed JSON — do NOT reformat it. The subagent already produces archetype-appropriate HTML (argument → thesis/against/evidence structure; roundup → one line per company; profile → who/achievement/detail; etc.). Bullet count scales with content.

**Font sizes:** td/sender: 15px · th/stats: 14px · icon links: 20px · footer: 14px

**Link columns:**
- 🔗 links to the original article URL
- ✉️ links to `https://mail.google.com/mail/u/0/#all/{messageId}`

#### Step 7a — Local send via Resend (when `.env` exists)

Write the HTML to `.tmp/newsletter_digest_YYYY-MM-DD.html`, then build a JSON payload file and POST it with curl. Use curl (not Python `urllib`) because the python.org Python 3.12 installer on macOS ships without a usable CA bundle and urllib hits `CERTIFICATE_VERIFY_FAILED` on `api.resend.com`. curl uses the system cert store and "just works".

```bash
set -a && source .env && set +a
DATE=$(date +%Y-%m-%d)

# Build payload with python (for safe JSON escaping of the HTML) — no network calls here.
python3 -c "
import json, pathlib, os
html = pathlib.Path(f'.tmp/newsletter_digest_{os.environ[\"DATE\"]}.html').read_text()
payload = {
    'from': os.environ['RESEND_FROM'],
    'to': [os.environ['RESEND_TO']],
    'subject': f'Newsletter Digest — {os.environ[\"DATE\"]}',
    'html': html,
}
pathlib.Path('.tmp/resend_payload.json').write_text(json.dumps(payload))
"

# POST to Resend via curl.
curl -sS -w '\nHTTP %{http_code}\n' -X POST 'https://api.resend.com/emails' \
  -H "Authorization: Bearer $RESEND_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @.tmp/resend_payload.json
```

A successful send returns `{"id":"<uuid>"}` and `HTTP 200`. If the response is a non-2xx status, log the error and continue — do not fail the whole run. Common causes: unverified sender domain (use `onboarding@resend.dev` for testing), invalid API key, or monthly free-tier limit (3k emails/mo).

#### Step 7b — Remote cloud send via Gmail MCP `create_draft` (when no `.env`)

The Anthropic sandbox blocks arbitrary outbound HTTPS (including `api.resend.com` → 403 "Host not in allowlist"), but MCP connector hosts are always allowlisted. Deliver the digest as a Gmail draft in the target inbox using `mcp__claude_ai_Gmail__create_draft`. The draft appears in Gmail Drafts where you can open it directly (or hit Send to forward it elsewhere).

**⚠ CRITICAL architectural rule for this path:**

The HTML digest is typically 50–150KB. The `Read` tool rejects files larger than ~10K tokens (~40KB), so the pattern "Write HTML to disk → Read it back → pass to create_draft" will FAIL with "exceeds maximum allowed tokens". This is exactly how the first cloud run broke.

**Fix:** assemble the HTML and emit the `create_draft` tool call in the **same assistant turn**, so the HTML string is still in your own output context when you reference it as the `htmlBody` argument. Do NOT route it through a `Write`→`Read` round-trip.

Concretely for 7b:

1. Compile `.tmp/digest.json` as usual — small, safe to Write/Read.
2. In a single assistant response, generate the complete HTML as one string in your reasoning, then immediately emit the `create_draft` tool call with that same string as `htmlBody`. Do not Write it to disk first.
3. **Optional** — after `create_draft` succeeds, you may Write the HTML to `.tmp/newsletter_digest_YYYY-MM-DD.html` as a post-hoc run artifact for inspection. This is safe because you no longer need to read it back.

Tool call shape (get recipient from the trigger prompt's `RESEND_TO` — the sandbox has no env vars):
```
mcp__claude_ai_Gmail__create_draft:
  to: ["xingjian.zh@gmail.com"]
  subject: "Newsletter Digest — YYYY-MM-DD (N newsletters)"
  htmlBody: "<!DOCTYPE html><html>...full digest HTML inline here..."
  body: "Daily newsletter digest — open in HTML view for the full table."
```

A successful call returns the Gmail draft ID. Log it and report success. On failure, log the error message and continue — do not fail the whole run.

**Gmail HTML rendering quirks to mind when writing the HTML in 7b:**

- **Gmail strips `<style>` blocks.** The existing template puts table/td styles in a `<style>` block at the top — for 7b, inline those styles as `style="..."` attributes on each `<table>`, `<th>`, `<td>`, and `<tr>` element. The per-row `summary_html` already uses inline styles, so most of the work is done.
- **Gmail clips messages over ~102KB** with "[Message clipped]". For 20–30 newsletters the body is usually comfortable; if you approach the limit, trim long summary bullets or drop optional `key_stats` entries.
- **Drafts land in Gmail → Drafts**, not Inbox. Tell the user to check there.

## HTML Template Reference
The HTML template lives at `./scripts/digest_template.html`. Key features:
- Clean Google-style design: blue header (#1a73e8), hover states (#f8f9ff), rounded corners
- Fixed table layout with 5 columns (no Date or Subject)
- Summary cells use structured HTML: bold topic label + bullet list
- All font sizes are +2 from browser defaults for readability

## Output
An HTML email delivered to your inbox with columns:
| Sender | Summary (bold topic + bullets) | Key Statistics | 🔗 | ✉️ |

## Gmail Forum Tab
No filter setup needed. Gmail's Forum tab automatically categorizes mailing lists and newsletters.
The MCP search uses `category:forums newer_than:1d` to query this tab directly.

## Performance
- Fetch (MCP): ~5–15s depending on email count
- Analysis: ~15–30s (parallel subagents, bottleneck is longest email)
- HTML generation: ~1s
- Total for 20 newsletters: ~25–50s

## Learnings
- MCP Gmail tools work without any OAuth setup — no `gmail_accounts.json` needed
- Gmail `category:forums` matches the Forum tab where newsletters land
- Some old thread messages appear in search results — filter by `internalDate` to exclude stale items
- Very large email bodies (>60KB) from MCP may get persisted to disk — use the preview/snippet plus your own summarization for the email JSON files
- The `newsletter-analyzer` subagent only has Read/Write tools (no MCP access), so all email content must be pre-written to JSON files
- Resend's free tier allows 3k emails/month, plenty for a daily personal digest
- `onboarding@resend.dev` works out of the box for testing; for a custom sender, verify a domain in the Resend dashboard
- The Resend API call is a plain HTTPS POST — no third-party automation platform required
- Always use curl (not python `urllib.request`) to call api.resend.com. python.org's Python 3.12 on macOS ships without the CA bundle and fails with `CERTIFICATE_VERIFY_FAILED`; curl uses the system cert store
- The MCP Gmail tools exposed by the claude.ai integration are `search_threads` (not `gmail_search_messages`) and `get_thread` (not `gmail_read_message`). The thread-shaped API returns minimal previews from search and full `plaintextBody` from get — newsletters are almost always single-message threads, so just use the first message
- **Anthropic's remote Claude Code sandbox has an outbound HTTPS egress allowlist.** Arbitrary third-party hosts like `api.resend.com` are blocked with a 403 "Host not in allowlist"; only Anthropic APIs and allowlisted MCP connector hosts are reachable. This is the opposite of the local Claude Code CLI, which has no egress restrictions. Hence the two-path Step 7 split.
- **Never Write-then-Read large HTML for a tool call.** The `Read` tool's ~10K-token file cap will reject files >~40KB, so assembling HTML as an `.html` file and then trying to Read it back to pass into `create_draft` (or any other MCP call) is a trap. Keep the HTML in your own output context and emit the tool call in the same turn.
