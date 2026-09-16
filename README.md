# Upwork AI Screening

An [n8n](https://n8n.io) automation that watches Upwork for new jobs, uses AI to
filter out the noise, and only pings me on Slack for jobs that are actually
worth applying to.

## Why this exists

Scrolling Upwork all day looking for good jobs is a waste of time — most
posts aren't a fit. This automation does the scrolling for me:

1. **[Vibeworker](https://vibeworker.com)** watches Upwork (RSS/job feed) and
   fires a webhook every time a new job is posted.
2. That webhook lands in **n8n**, which asks an **AI model** whether the job
   is actually relevant to the kind of work I do.
3. If it's a real match, n8n **posts it to Slack** with a "Mark Done" button.
4. I check Slack whenever I have a free moment, apply to anything that looks
   good, and hit **Mark Done** — no need to babysit Upwork all day.

## How it works

```
Upwork job posted
       │
       ▼
Vibeworker (RSS watcher)
       │  POST /vibeworker-job-alert
       ▼
┌─────────────────────────────────────────────────────────┐
│                        n8n workflow                      │
│                                                           │
│  Vibeworker Webhook                                       │
│        │                                                  │
│        ├──▶ Acknowledge Webhook (responds 200 right away) │
│        │                                                  │
│        ▼                                                  │
│  Parse Webhook Payload   (pulls title/link/description    │
│        │                  out of the Vibeworker payload)  │
│        ▼                                                  │
│  Screen with AI          (GPT-4o-mini decides: relevant   │
│        │                  or not, and why)                │
│        ▼                                                  │
│  Keep Only Relevant      (drops anything AI marked false) │
│        │                                                  │
│        ▼                                                  │
│  Send Slack Message      (posts job + "Mark Done" button) │
└─────────────────────────────────────────────────────────┘
       │
       ▼
   Slack channel  ──▶  I read it, apply, click "Mark Done"
       │
       ▼ (button click)
POST /upwork-done
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│         n8n workflow (Slack interaction handler)          │
│                                                           │
│  Slack Interaction Webhook                                │
│        │                                                  │
│        ▼                                                  │
│  Handle Slack Verification (handles Slack's one-time URL   │
│        │                    verification handshake)       │
│        ▼                                                  │
│  Is Challenge? ──yes──▶ Respond With Challenge             │
│        │no                                                 │
│        ├──▶ Acknowledge Slack Event (200 OK to Slack)      │
│        │                                                  │
│        ▼                                                  │
│  Parse Button Click                                        │
│        │                                                  │
│        ▼                                                  │
│  Update Slack Message   (edits the message to show ✅ Done)│
└─────────────────────────────────────────────────────────┘
```

Two independent triggers live in the same workflow:

| Trigger | Path | Purpose |
|---|---|---|
| **Job alert** | `POST /vibeworker-job-alert` | Vibeworker calls this every time it finds a new Upwork job matching your Vibeworker filter. |
| **Slack interaction** | `POST /upwork-done` | Slack calls this whenever someone clicks a button on a message the bot posted (used for "Mark Done"). |

## Repo structure

```
.
├── README.md
└── workflows/
    └── upwork-ai-screening.json   # importable n8n workflow
```

## Node-by-node breakdown

**Job alert pipeline**

- **Vibeworker Webhook** — receives the `job.matched` event from Vibeworker.
- **Acknowledge Webhook** — immediately replies `{"status": "received"}` so
  Vibeworker doesn't time out while the AI screening runs.
- **Parse Webhook Payload** — pulls `title`, `link`, `content` (description),
  `pubDate`, and client info (payment verified, spend, rating, etc.) out of
  the raw Vibeworker payload. Drops anything without a title.
- **Screen with AI** — sends the job title + description to GPT-4o-mini with
  a prompt describing what kind of work is a fit (n8n/Zapier/Make automation,
  CRM integrations, Voice AI, backend/API dev) and what to reject (VA work,
  marketing, sales-closer roles, spreadsheet-only gigs, anything too vague).
  Returns strict JSON: `{"relevant": true|false, "reason": "..."}`.
- **Keep Only Relevant** — parses the AI's JSON response and drops any job
  marked `relevant: false`.
- **Send Slack Message** — posts the surviving jobs to a Slack channel via
  `chat.postMessage`, with the job title, a link to the Upwork post, the
  AI's reason for matching, and a **✅ Mark Done** button.

**Slack "Mark Done" handler**

- **Slack Interaction Webhook** — the Request URL you give Slack for
  Interactivity (button clicks) and Event Subscriptions (URL verification).
- **Handle Slack Verification** — branches based on payload shape: Slack's
  one-time `url_verification` challenge, a `block_actions` button click
  (form-encoded, JSON string in `payload`), or anything else.
- **Is Challenge?** — routes the verification handshake to
  **Respond With Challenge**, everything else to **Acknowledge Slack Event** +
  **Parse Button Click**.
- **Parse Button Click** — pulls the action id, the job link (button value),
  the Slack `response_url`, and who clicked it out of the interaction
  payload.
- **Update Slack Message** — POSTs to Slack's `response_url` with
  `replace_original: true`, swapping the message text for
  "✅ Done — marked by \<you\>" so the channel shows what's been handled.

## Setup

### 1. Prerequisites

- An n8n instance (cloud or self-hosted) with the
  `@n8n/n8n-nodes-langchain` community/AI nodes installed.
- A [Vibeworker](https://vibeworker.com) account with an Upwork filter set
  up (this is what watches Upwork and decides what counts as a "new job").
- A Slack app with:
  - A **Bot Token** with the `chat:write` scope.
  - **Interactivity & Shortcuts** turned on.
- An OpenAI (or n8n AI Gateway) credential for the screening step.

### 2. Import the workflow

1. In n8n, go to **Workflows → Import from File** and select
   `workflows/upwork-ai-screening.json`.
2. Open **Screen with AI** and attach your OpenAI/AI Gateway credential.
3. Open **Send Slack Message** and **Update Slack Message**, and attach a
   Header Auth credential with:
   - Header name: `Authorization`
   - Header value: `Bearer <your-slack-bot-token>`
4. In **Send Slack Message**, replace the `"channel"` value in the JSON body
   with your own Slack channel ID.
5. Activate the workflow. n8n will generate the two webhook URLs — copy them
   for the next steps.

### 3. Point Vibeworker at n8n

In your Vibeworker filter/notification settings, set the webhook URL to the
**Vibeworker Webhook** node's production URL (path: `/vibeworker-job-alert`).
Vibeworker will POST a `job.matched` event to it every time it finds a job
matching your filter.

### 4. Point Slack at n8n

In your Slack app config:
- **Interactivity & Shortcuts → Request URL**: the **Slack Interaction
  Webhook** node's production URL (path: `/upwork-done`).
- If you also enable **Event Subscriptions**, use the same URL as the
  Request URL — the workflow already handles Slack's `url_verification`
  challenge automatically.

### 5. Try it

Trigger a test job match in Vibeworker (or send a test POST to the
`/vibeworker-job-alert` webhook with a sample payload). If the job is a
match, it should show up in your Slack channel within a few seconds with a
**✅ Mark Done** button. Click it — the message should update to show it's
done.

## Customizing what counts as "relevant"

Edit the prompt in the **Screen with AI** node. It currently screens for:

- n8n / Zapier / Make.com workflow building
- CRM integrations (Attio, HubSpot, GoHighLevel, etc.)
- Voice AI / chatbot backend work
- Custom automation scripting / backend API dev

...and rejects VA/data-entry work, marketing/content/design, sales-closer
roles, spreadsheet-only tasks, and anything too vague to judge. Adjust the
bullet points to match your own skills and what you actually want to see.

## Notes / things to double check on your own instance

- **Slack request verification**: the interaction handler currently trusts
  any POST to `/upwork-done` — it doesn't verify Slack's signing secret
  (`x-slack-signature` / `x-slack-request-timestamp` headers). If your
  webhook URL could be guessed or shared, consider adding a signature
  verification step before **Parse Button Click**.
- **Credentials aren't in this file**: n8n stores actual API keys/tokens in
  its own credential store, not in the workflow JSON, so this export is
  safe to keep in git. You'll need to re-attach your own credentials after
  importing (see Setup step 2).
- `meta.instanceId` in the JSON is a placeholder — n8n will assign your own
  on import.
