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
      v
  [Filter New]  (READ Seen Message IDs — drop already-acked message IDs)
      v
  [Filter Dup Content]  (READ Seen Message IDs — drop already-acked content keys)
      v
  [Aggregate New]
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
| Normalize, Sort | Code (JS) | Normalizes Gmail vs Outlook schemas, computes a content key, deduplicates within the run, sorts newest-first |
| Filter New | n8n Data Table | Read: filters out messages whose `message_id` was already acked |
| Filter Dup Content | n8n Data Table | Read: filters out messages whose `content_key` was already acked (catches re-sends/forwards with a new message ID but identical content) |
| Aggregate New | Aggregate | Collects the surviving messages into a single `messages` array |
| Respond to Webhook | Respond to Webhook | Returns the filtered email list to OpenClaw |

**Ack workflow (`/webhook/email-ack`)**

| Node | Type | Purpose |
|---|---|---|
| Ack Webhook | Webhook | Receives the list of message IDs OpenClaw successfully tasked |
| Seen Message IDs | n8n Data Table | Write: records each acked message's `message_id` and `content_key` so it is suppressed on future fetch runs |
| Respond to Webhook | Respond to Webhook | Confirms receipt to OpenClaw |

**Notify workflow (`/webhook/email-tasks-notify`)**

| Node | Type | Purpose |
|---|---|---|
| Notify Webhook | Webhook | Entry point for ad-hoc notifications; accepts `{ "text": "..." }` |
| Send Telegram | Telegram | Forwards the `text` field to a configured Telegram chat |

---

## Deduplication

Every message carries two identifiers, both computed in the Normalize, Sort node:

- **`messageId`** — the provider-native ID (`gmail:<id>` or `outlook:<id>`)
- **`contentKey`** — `<sender email> | <normalized subject>`, with `Re:`/`Fwd:`/`Fw:` prefixes and case/whitespace differences stripped

`contentKey` is what makes cross-mailbox and cross-run dedup possible even when the same message shows up under a different `messageId` — e.g., a forward, a re-send, or the same email delivered separately to two mailboxes.

Two layers of deduplication work together to ensure no email generates more than one Todoist task:

### 1. In-run deduplication (Normalize, Sort node)

Within a single webhook call, the same email can arrive from multiple inboxes (e.g., a reply-all that lands in two of your accounts). The code node sorts newest-first, then collapses items using a `Set` keyed on `contentKey` (falling back to `messageId` if no content key could be computed). The newest copy is kept; all others are discarded. The response includes a `duplicatesRemoved` count.

### 2. Cross-run deduplication (Data Table + Ack)

The fetch window is 2 days, so the same email appears on multiple consecutive runs. The **Seen Message IDs** data table prevents repeat task creation across runs, but rows are only written to it **after OpenClaw confirms tasks were created** — via a POST to the ack endpoint. This makes the system crash-safe:

- **Fetch run**: two data-table lookups filter the candidate list before responding — first by `message_id` (**Filter New**), then by `content_key` (**Filter Dup Content**), so a message is dropped if either identifier was already acked. Nothing is written at this stage.
- **OpenClaw**: receives the filtered list, creates Todoist tasks, then POSTs the message IDs to `/webhook/email-ack`.
- **Ack run**: writes each message's `message_id` and `content_key` to the table, marking it as processed.

If a run fails between fetch and ack (OpenClaw crashes, Todoist is unavailable, etc.), no rows are written. The emails remain unacked, reappear in the next 2-day fetch window, and get processed again cleanly. Nothing is lost and nothing is double-counted.

#### Data Table Schema

| Column | Type | Description |
|---|---|---|
| `message_id` | String | Prefixed ID (`gmail:…` or `outlook:…`) |
| `subject` | String | Subject line of the acked message |
| `content_key` | String | Normalized `sender | subject` key, used to catch content duplicates across differing message IDs |
| `processed_at` | String | ISO 8601 timestamp of when the row was recorded |

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
      "contentKey": "sender@example.com|example subject",
      "from": "sender@example.com",
      "subject": "Example Subject",
      "snippet": "Preview of the email body...",
      "date": "2026-01-01T13:45:00.000Z"
    }
  ]
}
```

`messageId` is prefixed with `gmail:` or `outlook:` to prevent cross-provider ID collisions. `contentKey` is the normalized `sender|subject` pair used for content-based deduplication (see [Deduplication](#deduplication)).

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
- The data table grows over time. Old entries can be pruned periodically without affecting correctness, as long as you only remove rows with a `processed_at` older than your lookback window.
- The notify webhook (`/webhook/email-tasks-notify`) is independent of the fetch/ack pair and has no `Respond to Webhook` node — it's fire-and-forget. Use it for out-of-band alerts (e.g., failures) rather than as part of the core task pipeline.
- Content-based filtering means two emails with the same sender and subject are treated as the same message, even across different mailboxes or with different provider IDs. If you legitimately expect repeat emails with identical subjects from the same sender (e.g., recurring automated reports), only the first will produce a task per acked cycle.
