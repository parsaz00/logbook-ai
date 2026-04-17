# Architecture Overview

## What this is

LogbookAI is an n8n-based workflow automation project that replaces a manual restaurant shift-log process with an AI-assisted pipeline. Shift leaders submit a structured form after each shift; the system formats their raw notes into a professional logbook entry and distributes it to the right channels automatically. At the end of each week, a second workflow reads the week's entries and produces a GM-facing summary brief.

## System components

```
Form submission (n8n Form Trigger)
        │
        ▼
Anthropic Claude API (HTTP Request node)
  • Nightly: formats raw shift data into structured logbook entry
  • Weekly: synthesizes 7 days of entries into a GM brief
        │
        ├──▶ Google Sheets (append row / read range)
        ├──▶ Slack (Incoming Webhook)
        └──▶ Gmail (send email)
```

Everything runs inside a single Docker container (n8n) with a persistent volume for workflow state. No external database, no custom backend.

## Key decisions

**n8n over custom code.** The entire integration layer — form trigger, HTTP calls, Sheets writes, Slack posts — is expressed as n8n workflow JSON. This keeps the project inspectable and modifiable without writing code, which matches real-world operations tooling. The only "custom" logic is in the LLM prompts.

**Claude via HTTP Request, not a dedicated node.** n8n has community AI nodes, but calling the Anthropic API directly via the HTTP Request node is more portable and doesn't depend on a third-party community package. The tradeoff is slightly more manual prompt templating.

**Google Sheets as the data store.** A spreadsheet is the right fidelity for this use case: the GM can open it, filter it, and share it with no infrastructure. A real production deployment would use a proper database (Postgres, Supabase) for query flexibility, backup guarantees, and scale.

**Incoming Webhooks for Slack (not a bot token).** Incoming webhooks require no OAuth scope negotiation and no bot user. The tradeoff is write-only — the system can post but not read or react to Slack messages. That's fine for a notification-only pattern.

**Single-location, single-sheet design.** Multi-location support would require either a sheet per location or a location column with filtered reads. Both are straightforward extensions but add surface area the portfolio project doesn't need.

## What's out of scope (and how to extend it)

| Out of scope | Production extension |
|---|---|
| Auth / access control | n8n Enterprise or a reverse proxy with SSO |
| Multi-location | Add a `location_id` field to the form and Sheet; filter on read |
| Historical trend queries | Migrate to Postgres; add a BI layer (Metabase, Looker) |
| Retry / dead-letter queue | n8n's built-in error workflow + a dedicated error Slack channel |
| Prompt versioning | Store prompts in a Git-backed config; reference by version in the workflow |
| Real-time dashboards | Stream Sheet data to a Looker Studio or Grafana dashboard |

## Data flow (nightly workflow)

1. Shift leader opens the n8n form URL and submits the entry.
2. n8n constructs a Claude API request: the system prompt (from `prompts/nightly-logbook-system.md`) plus a user prompt populated with all form fields.
3. Claude returns a formatted logbook entry as plain text.
4. n8n parses the response and:
   - Appends a row to the Google Sheet (`logbook` tab): raw fields + AI output + timestamp
   - Posts the formatted entry to the `#logbook` Slack channel
   - Sends an email copy to the manager's Gmail address
5. On any error, n8n catches it and appends a row to the `errors` Sheet tab with the timestamp, workflow step, and error message.

## Data flow (weekly workflow)

1. Cron trigger fires Sunday at 6pm PT.
2. n8n reads all rows from the `logbook` Sheet where `created_at` falls within the last 7 days.
3. The entries are concatenated and sent to Claude with the weekly-brief system prompt.
4. Claude returns a GM brief.
5. n8n posts the brief to `#gm-brief` and emails it to the manager.
