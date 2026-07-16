# Email to Todoist — n8n Workflow

An n8n workflow that serves as the data-fetching and task-creation backbone for an OpenClaw AI agent. OpenClaw calls this workflow via webhook (MCP): it fetches a normalized feed of emails from multiple accounts across Gmail and Outlook, decides which ones need a Todoist task, then hands its decisions back to n8n — which resolves projects, creates the tasks natively, acknowledges what it processed, and posts a run summary to Telegram.

---

## How It Works

The workflow runs as two webhook calls per cycle, plus an optional standalone notifier.

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
      +-- [Get Open Tasks]  (Todoist — all open tasks, runs once)
      |
      | (all 9 branches run in parallel)
      v
  [Tag with account + source]  →  [Merge all 8 mail streams]  →  [Normalize, Deduplicate, Sort]
      v
  [Filter New]  (READ Seen Message IDs — drop already-acked message IDs)
      v
  [Filter Dup Content]  (READ Seen Message IDs — drop already-acked content keys)
      v
  [Aggregate New] --------------------+
                                      |
       [Pipeline Tasks Only] --------+--> [Combine Response] → [Shape Response] → [Respond to Webhook]
                                                                                          |
                                                                                          v
                                                                             OpenClaw (AI Agent)
  — reads the new-email list + openTasks (currently-open tasks this pipeline created)
  — decides which emails need a task, plus title/description/priority/due date/project for each
  — skips any email whose action is already covered by an open task, even if worded differently
  — decides which remaining emails to skip (no task needed)
  — POSTs all of it in one call to Step 2 (Commit)
```

Shape Response always returns a well-formed `{count, generatedAt, messages, openTasks}` payload — including on a zero-new-mail day, when `messages` is simply empty.

### Step 2 — Commit

OpenClaw no longer talks to Todoist directly — it just describes what it wants, and n8n creates the tasks natively. This is a single webhook call that resolves projects, creates tasks, acknowledges everything it successfully handled, and sends a Telegram summary.

```
OpenClaw (AI Agent)
      |
      | HTTP POST /webhook/email-tasks-commit
      | body: { "tasks": [...], "skipped": [...], "received": N }
      v
  [Commit Webhook]
      |
      +----------------------------+
      |                            |
      v                            v
  [Prepare Tasks]              [Skipped Rows]
  (resolve project name          (pull messageId/subject/
   -> Todoist project ID,         contentKey for emails
   fallback Inbox; map            OpenClaw decided didn't
   p1-p4 -> priority 4-1)         need a task)
      |                            |
      v                            |
  [Create Todoist Task]            |
  (native Todoist node,            |
   continues past errors)          |
      |                            |
   success     failure             |
      |           |                |
      v           v                |
  [Task Ack   [Failed              |
   Fields]     Fields]             |
      |           |                |
      +-----+-----+                |
            v                      |
      [Merge Acks] <---------------+
            |
            v
  [Record Committed]  (WRITE Seen Message IDs — created + skipped only; failures are left out)
            |
            v
         [Sync]
            |
            v
     [Build Summary]  (groups new tasks by project, lists any failures)
            |
            v
     [Send Summary]  (Telegram)
```

Tasks that fail to create (e.g. Todoist briefly unavailable) are **not** acked, so they reappear in the next Fetch call's message list and get retried automatically — nothing is silently dropped.

> **Note:** an older, simpler acknowledgment path (`/webhook/email-ack`, `Ack Webhook` → `Split Ack` → `Record Acked`, taking just `{ "messageIds": [...] }`) still exists in the workflow for backward compatibility, but OpenClaw no longer calls it — task creation, acking, and summary notification are now all handled by the Commit step above.

### Step 3 — Notify (optional)

A standalone webhook for pushing ad-hoc alerts (e.g., errors) to Telegram, using the same bot as the Commit step's run summaries. It is not wired into the fetch/commit flow — any caller can trigger it independently.

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
| Get Open Tasks | Todoist | Fetches all Todoist tasks; runs once per webhook call, in parallel with the mail fetches |
| Pipeline Tasks Only | Code (JS) | Filters to tasks this pipeline created that are still open, identified by the `From: ... \| Account: ...` line the Commit step writes into each task's description |
| Combine Response | Merge | Pairs the new-message list with the open-tasks list by position |
| Shape Response | Code (JS) | Builds the final `{count, generatedAt, messages, openTasks}` payload — always well-formed, even with zero new messages |
| Respond to Webhook | Respond to Webhook | Returns the combined payload to OpenClaw |

**Commit workflow (`/webhook/email-tasks-commit`)**

| Node | Type | Purpose |
|---|---|---|
| Commit Webhook | Webhook | Entry point; receives OpenClaw's task decisions (`tasks`) and skipped emails (`skipped`) |
| Prepare Tasks | Code (JS) | Resolves each task's `project` name to a Todoist project ID (fuzzy, case/emoji-insensitive match; falls back to Inbox), maps `p1`-`p4` to Todoist's 4–1 priority scale |
| Skipped Rows | Code (JS) | Extracts `messageId`/`subject`/`contentKey` from the `skipped` array so those emails also get acked without creating a task |
| Create Todoist Task | Todoist | Creates each task natively in its resolved project; continues past errors instead of failing the run |
| Task Ack Fields | Set | Captures identifying fields for successfully created tasks |
| Failed Fields | Set | Captures the same fields plus the error message for tasks that failed to create |
| Merge Acks | Merge | Combines successful creates with skipped rows (failed creates are excluded) |
| Record Committed | n8n Data Table | Write: records `message_id`/`content_key` for everything successfully handled, so it's suppressed on future fetch runs |
| Sync | Merge | Waits for the data-table write to finish before building the summary |
| Build Summary | Code (JS) | Groups newly created tasks by project and lists any failures into a formatted message |
| Send Summary | Telegram | Sends the grouped summary to Telegram |

**Legacy Ack workflow (`/webhook/email-ack`) — no longer called by OpenClaw**

| Node | Type | Purpose |
|---|---|---|
| Ack Webhook | Webhook | Receives a plain list of message IDs (`{ "messageIds": [...] }`) |
| Split Ack | Split Out | Splits the array into individual items |
| Record Acked | n8n Data Table | Write: records each acked message's `message_id` and `content_key` so it is suppressed on future fetch runs |

Kept for backward compatibility; the Commit workflow above now owns acking as part of task creation.

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

`contentKey` is what makes cross-mailbox and cross-run dedup possible even when the same message shows up under a different `messageId` — e.g., a forward, a re-send, or the same email delivered separately to two mailboxes. But it's an exact match on the normalized subject line, so it only catches identical subjects. A reminder email days later about the same underlying request usually has different wording (e.g. "Update Acme Vendor Subscription" today, "Please renew your Acme Vendor plan" next week) — `contentKey` can't see that it's the same request. Catching that needs semantic judgment, not string matching, which is what the third layer below is for.

Three layers of deduplication work together to ensure no email generates more than one Todoist task:

### 1. In-run deduplication (Normalize, Sort node)

Within a single webhook call, the same email can arrive from multiple inboxes (e.g., a reply-all that lands in two of your accounts). The code node sorts newest-first, then collapses items using a `Set` keyed on `contentKey` (falling back to `messageId` if no content key could be computed). The newest copy is kept; all others are discarded. The response includes a `duplicatesRemoved` count.

### 2. Cross-run deduplication (Data Table + Commit)

The fetch window is 2 days, so the same email appears on multiple consecutive runs. The **Seen Message IDs** data table prevents repeat task creation across runs, but rows are only written to it **after n8n has actually created (or skipped) each item** — inside the Commit workflow. This makes the system crash-safe:

- **Fetch run**: two data-table lookups filter the candidate list before responding — first by `message_id` (**Filter New**), then by `content_key` (**Filter Dup Content**), so a message is dropped if either identifier was already acked. Nothing is written at this stage.
- **OpenClaw**: receives the filtered list, decides which emails need tasks (with project/priority/due date) and which to skip, then POSTs everything to `/webhook/email-tasks-commit`.
- **Commit run**: creates each task natively via the Todoist node, then writes `message_id`/`content_key` to the table only for tasks that were **successfully created** plus emails that were explicitly **skipped**. Tasks whose creation failed are left unacked on purpose.

If a run fails partway (Todoist is unavailable, a single task creation errors, etc.), only the items that actually succeeded get acked. The rest remain unacked, reappear in the next 2-day fetch window, and get retried automatically. Nothing is lost and nothing is double-counted.

### 3. Semantic deduplication (OpenClaw + `openTasks`)

`messageId` and `contentKey` only catch exact re-sends and identical-subject repeats — neither can tell that a differently-worded reminder is about the same underlying request. To close that gap, the Fetch response also includes `openTasks`: every currently-open Todoist task this pipeline has created (via **Get Open Tasks** → **Pipeline Tasks Only**, matched by the `From: ... | Account: ...` line the Commit step writes into each task's description).

OpenClaw's decide step is required to check each candidate email against `openTasks` before turning it into a task — if an open task already covers the same action, even worded differently, the email is skipped instead of creating a duplicate task. This is judgment-based rather than identifier-based, so it catches cases the first two layers miss.

Once a task is completed in Todoist, it drops out of `openTasks`, so a later reminder about that same thing is free to create a fresh task — that's the intended behavior, since at that point it's a genuinely new open item.

#### Data Table Schema

| Column | Type | Description |
|---|---|---|
| `message_id` | String | Prefixed ID (`gmail:…` or `outlook:…`) |
| `subject` | String | Subject line of the acked message |
| `content_key` | String | Normalized `sender | subject` key, used to catch content duplicates across differing message IDs |
| `processed_at` | String | ISO 8601 timestamp of when the row was recorded |

---

## Output Schema

The webhook returns a single, always-well-formed JSON object — including on a zero-new-mail day, when `messages` is simply an empty array (it used to hang instead of responding in that case):

```json
{
  "count": 42,
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
  ],
  "openTasks": [
    {
      "title": "Renew Acme Vendor subscription",
      "project": "Project 5",
      "createdAt": "2026-01-01"
    }
  ]
}
```

`messageId` is prefixed with `gmail:` or `outlook:` to prevent cross-provider ID collisions. `contentKey` is the normalized `sender|subject` pair used for content-based deduplication. `openTasks` lists every currently-open Todoist task this pipeline has created, and is what OpenClaw uses for semantic duplicate detection (see [Deduplication](#deduplication)).

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

**Todoist API:**
- One credential for the Commit workflow's Create Todoist Task node

**Telegram (Bot API):**
- One credential, used by both the Commit workflow's Send Summary node and the standalone Notify workflow's Send Telegram node
- Set the target `chatId` on each of those nodes to your own chat/group ID

### Data Table

Create a data table in n8n named **Seen Message IDs** with the columns defined in the schema above. The workflow reads from and writes to this table on every run.

### Import the Workflow

1. In n8n, go to **Workflows → Import**
2. Upload `Email-to-Todoist.json`
3. Assign your credentials to each Gmail, Outlook, Todoist, and Telegram node
4. Create the **Seen Message IDs** data table
5. In the **Prepare Tasks** code node, replace the placeholder `PROJECTS` map with your own Todoist project names → project IDs (keep an `inbox` entry as the fallback)
6. Activate the workflow

### Configure OpenClaw

Configure two tools in OpenClaw:

**Fetch** — call first to get new emails:
```
POST https://<your-n8n-host>/webhook/email-tasks
```

**Commit** — call last, once you've decided what to do with each email:
```
POST https://<your-n8n-host>/webhook/email-tasks-commit
Content-Type: application/json

{
  "received": 5,
  "tasks": [
    {
      "title": "Follow up on invoice",
      "description": "...",
      "priority": "p2",
      "dueString": "tomorrow",
      "project": "Project 1",
      "messageId": "gmail:abc123...",
      "subject": "Invoice #123",
      "contentKey": "sender@example.com|invoice 123"
    }
  ],
  "skipped": [
    { "messageId": "outlook:xyz456...", "subject": "Newsletter", "contentKey": "..." }
  ]
}
```

OpenClaw must always commit before ending its run — task creation, acking, and the Telegram summary all happen as a side effect of this one call. If it never commits, the fetched emails stay unacked and simply reappear next run.

---

## Notes

- The workflow fetches emails from the **last 2 days** on every trigger. Adjust the `minus(2, 'day')` expressions in the Gmail/Outlook nodes to change the lookback window. The window should always be longer than the maximum expected gap between runs so unacked emails are never missed.
- The workflow is exposed as an MCP tool (`availableInMCP: true`), which is how OpenClaw discovers and invokes it.
- The data table grows over time. Old entries can be pruned periodically without affecting correctness, as long as you only remove rows with a `processed_at` older than your lookback window.
- Project name matching in **Prepare Tasks** is case-insensitive, strips emoji/smart-quotes, and falls back to a substring match before defaulting to Inbox — so OpenClaw doesn't need to send an exact project name.
- The notify webhook (`/webhook/email-tasks-notify`) is independent of the fetch/commit pair and has no `Respond to Webhook` node — it's fire-and-forget. Use it for out-of-band alerts rather than as part of the core task pipeline.
- The Fetch webhook always responds, even when there's no new mail. Previously, a zero-new-mail day left the response chain unfired and the webhook call would hang; `Combine Response` → `Shape Response` now always produces a `{count, messages, openTasks}` payload regardless of how many (if any) new messages survived filtering.
- Content-based filtering means two emails with the same sender and subject are treated as the same message, even across different mailboxes or with different provider IDs. If you legitimately expect repeat emails with identical subjects from the same sender (e.g., recurring automated reports), only the first will produce a task per acked cycle.
