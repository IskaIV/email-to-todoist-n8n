# Email to Todoist — n8n Workflow

An n8n workflow that serves as the data-fetching backbone for an OpenClaw AI agent. OpenClaw calls this workflow via webhook (MCP), receives a normalized feed of emails from multiple accounts across Gmail and Outlook, then uses its own AI reasoning to organize that mail into Todoist tasks.

---

## How It Works

```
OpenClaw (AI Agent)
      |
      | HTTP POST /webhook/email-tasks
      v
  [n8n Webhook]
      |
      +-- [Get Account 1 Mail]  (Outlook)
      +-- [Get Account 2 Mail]  (Outlook)
      +-- [Get Account 3 Mail]  (Gmail)
      +-- [Get Account 4 Mail]  (Gmail)
      +-- [Get Account 5 Mail]  (Gmail)
      +-- [Get Account 6 Mail]  (Gmail)
      +-- [Get Account 7 Mail]  (Gmail)
      +-- [Get Account 8 Mail]  (Gmail)
      |
      | (all 8 run in parallel)
      v
  [Tag with account + source]
      v
  [Merge all 8 streams]
      v
  [Normalize, Deduplicate, Sort]
      v
  [Respond to Webhook]
      |
      v
OpenClaw (AI Agent)
  — reads email list
  — decides priority, project, due dates
  — creates Todoist tasks
```

---

## Nodes

| Node | Type | Purpose |
|---|---|---|
| Webhook | Webhook | Entry point; triggered by OpenClaw at `/webhook/email-tasks` |
| Get \* Mail (×8) | Gmail / Outlook | Fetches all emails received in the last 24 hours |
| Edit Fields (×8) | Set | Tags each email with its `account` name and `source` (Gmail or Outlook) |
| Merge | Merge | Combines all 8 parallel streams into one |
| Normalize, Sort | Code (JS) | Normalizes Gmail vs Outlook schemas, deduplicates, sorts newest-first |
| Respond to Webhook | Respond to Webhook | Returns the JSON payload back to OpenClaw |

---

## Output Schema

The webhook returns a single JSON object:

```json
{
  "count": 42,
  "duplicatesRemoved": 3,
  "generatedAt": "2026-01-01T14:00:00.000Z",
  "messages": [
    {
      "messageId": "gmail:abc123...",
      "account": "Account 3",
      "source": "Gmail",
      "from": "sender@example.com",
      "subject": "Example Subject",
      "snippet": "Preview of the email body...",
      "date": "2026-01-01T13:45:00.000Z"
    }
  ]
}
```

`messageId` is prefixed with `gmail:` or `outlook:` to prevent cross-provider ID collisions. Deduplication collapses the same message when it arrives in multiple inboxes, keeping the newest copy.

---

## Email Accounts

| Account | Provider |
|---|---|
| Account 1 | Outlook |
| Account 2 | Outlook |
| Account 3 | Gmail |
| Account 4 | Gmail |
| Account 5 | Gmail |
| Account 6 | Gmail |
| Account 7 | Gmail |
| Account 8 | Gmail |

---

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- OpenClaw with MCP/webhook integration configured

### Credentials Required

Configure the following credentials in n8n before activating the workflow:

**Outlook (Microsoft OAuth2):**
- One credential per Outlook account (2 total)

**Gmail (Google OAuth2):**
- One credential per Gmail account (6 total)

### Import the Workflow

1. In n8n, go to **Workflows → Import**
2. Upload `Email-to-Todoist.json`
3. Assign your credentials to each Gmail and Outlook node
4. Activate the workflow

### Configure OpenClaw

Point OpenClaw's email-fetch tool at:

```
POST https://<your-n8n-host>/webhook/email-tasks
```

OpenClaw receives the normalized email list, applies AI reasoning to determine task content, project placement, priority, and due dates, then creates the tasks directly in Todoist.

---

## Notes

- The workflow fetches emails from the **last 24 hours** on every trigger. Adjust the `minus(1, 'day')` expressions in the Gmail/Outlook nodes to change the lookback window.
- The workflow is exposed as an MCP tool (`availableInMCP: true`), which is how OpenClaw discovers and invokes it.
- Deduplication is keyed on `messageId` when available, or falls back to a composite of sender address + subject + snippet prefix.
