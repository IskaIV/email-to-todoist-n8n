# Email to Todoist — n8n Workflow

An n8n workflow that serves as the data-fetching backbone for an OpenClaw AI agent. OpenClaw calls this workflow via webhook (MCP), receives a normalized feed of emails from multiple accounts across Gmail and Outlook, then uses its own AI reasoning to organize that mail into Todoist tasks.

---

## How It Works

The workflow runs as two separate webhook calls per cycle.

### Step 1 — Fetch

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
      |
      +--> [Data Table: Seen Message IDs]  (READ — filter already-acked IDs)
      v
  [Respond to Webhook]
      |
      v
OpenClaw (AI Agent)
  — reads email list
  — decides priority, project, due dates
  — creates Todoist tasks
```

### Step 2 — Acknowledge

```
OpenClaw (AI Agent)
      |
      | HTTP POST /webhook/email-ack  (called last, after all tasks are created)
      | body: { "messageIds": ["gmail:abc...", "outlook:xyz..."] }
      v
  [n8n Ack Webhook]
      |
      v
  [Data Table: Seen Message IDs]  (WRITE — record IDs as processed)
      |
      v
  [Respond to Webhook]
```

### Step 3 — Notify (optional)

A standalone webhook for pushing ad-hoc alerts (e.g., errors, run summaries) to Telegram. It is not wired into the fetch/ack flow — any caller can trigger it independently.

```
Caller
      |
      | HTTP POST /webhook/email-tasks-notify
      | body: { "text": "message to send" }
      v
  [Notify Webhook]
      |
      v
  [Send Telegram]
```

---

## Nodes

**Fetch workflow (`/webhook/email-tasks`)**

| Node | Type | Purpose |
|---|---|---|
| Webhook | Webhook | Entry point; triggered by OpenClaw |
| Get \* Mail (×8) | Gmail / Outlook | Fetches all emails received in the last 2 days |
| Edit Fields (×8) | Set | Tags each email with its `account` name and `source` (Gmail or Outlook) |
| Merge | Merge | Combines all 8 parallel streams into one |
| Normalize, Sort | Code (JS) | Normalizes Gmail vs Outlook schemas, deduplicates within the run, sorts newest-first |
| Seen Message IDs | n8n Data Table | Read: filters out already-acked message IDs before responding |
| Respond to Webhook | Respond to Webhook | Returns the filtered email list to OpenClaw |

**Ack workflow (`/webhook/email-ack`)**

| Node | Type | Purpose |
|---|---|---|
| Ack Webhook | Webhook | Receives the list of message IDs OpenClaw successfully tasked |
| Seen Message IDs | n8n Data Table | Write: records each acked ID so it is suppressed on future fetch runs |
| Respond to Webhook | Respond to Webhook | Confirms receipt to OpenClaw |

**Notify workflow (`/webhook/email-tasks-notify`)**

| Node | Type | Purpose |
|---|---|---|
| Notify Webhook | Webhook | Entry point for ad-hoc notifications; accepts `{ "text": "..." }` |
| Send Telegram | Telegram | Forwards the `text` field to a configured Telegram chat |

---

## Deduplication

Two layers of deduplication work together to ensure no email generates more than one Todoist task:

### 1. In-run deduplication (Normalize, Sort node)

Within a single webhook call, the same email can arrive from multiple inboxes (e.g., a reply-all that lands in two of your accounts). The code node collapses these using a `Set` keyed on:

- `messageId` (`gmail:<id>` or `outlook:<id>`) when available
- A composite of **sender address + subject + snippet prefix** as a fallback

The newest copy is kept; all others are discarded. The response includes a `duplicatesRemoved` count.

### 2. Cross-run deduplication (Data Table + Ack)

The fetch window is 2 days, so the same email appears on multiple consecutive runs. The **Seen Message IDs** data table prevents repeat task creation across runs, but IDs are only written to it **after OpenClaw confirms tasks were created** — via a POST to the ack endpoint. This makes the system crash-safe:

- **Fetch run**: reads the table and filters out already-acked IDs before responding. Nothing is written.
- **OpenClaw**: receives the filtered list, creates Todoist tasks, then POSTs the message IDs to `/webhook/email-ack`.
- **Ack run**: writes those IDs to the table, marking them as processed.

If a run fails between fetch and ack (OpenClaw crashes, Todoist is unavailable, etc.), no IDs are written. The emails remain unacked, reappear in the next 2-day fetch window, and get processed again cleanly. Nothing is lost and nothing is double-counted.

#### Data Table Schema

| Column | Type | Description |
|---|---|---|
| `message_id` | String | Prefixed ID (`gmail:…` or `outlook:…`) |
| `account` | String | Account label the message was fetched from |
| `source` | String | `Gmail` or `Outlook` |
| `seen_at` | String | ISO 8601 timestamp of when the ID was recorded |

---

## Output Schema

The webhook returns a single JSON object containing only emails not previously seen:

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

`messageId` is prefixed with `gmail:` or `outlook:` to prevent cross-provider ID collisions.

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

**Telegram (Bot API):**
- One credential for the notify workflow's Send Telegram node
- Set the target `chatId` on that node to your own chat/group ID

### Data Table

Create a data table in n8n named **Seen Message IDs** with the columns defined in the schema above. The workflow reads from and writes to this table on every run.

### Import the Workflow

1. In n8n, go to **Workflows → Import**
2. Upload `Email-to-Todoist.json`
3. Assign your credentials to each Gmail and Outlook node
4. Create the **Seen Message IDs** data table
5. Activate the workflow

### Configure OpenClaw

Configure two tools in OpenClaw:

**Fetch** — call first to get new emails:
```
POST https://<your-n8n-host>/webhook/email-tasks
```

**Ack** — call last, after all Todoist tasks are created:
```
POST https://<your-n8n-host>/webhook/email-ack
Content-Type: application/json

{ "messageIds": ["gmail:abc123...", "outlook:xyz456..."] }
```

OpenClaw must always ack before ending its run. If it acks before tasks are fully created, emails may be silently dropped on failures.

---

## Notes

- The workflow fetches emails from the **last 2 days** on every trigger. Adjust the `minus(2, 'day')` expressions in the Gmail/Outlook nodes to change the lookback window. The window should always be longer than the maximum expected gap between runs so unacked emails are never missed.
- The workflow is exposed as an MCP tool (`availableInMCP: true`), which is how OpenClaw discovers and invokes it.
- The data table grows over time. Old entries can be pruned periodically without affecting correctness, as long as you only remove IDs with a `seen_at` older than your lookback window.
- The notify webhook (`/webhook/email-tasks-notify`) is independent of the fetch/ack pair and has no `Respond to Webhook` node — it's fire-and-forget. Use it for out-of-band alerts (e.g., failures) rather than as part of the core task pipeline.
