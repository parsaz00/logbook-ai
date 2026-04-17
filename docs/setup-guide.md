# Setup Guide

Follow these steps in order after running `docker compose up -d`.

---

## Step 1 — Create your n8n owner account

Open [http://localhost:5678](http://localhost:5678). Fill in the owner registration form (email + password). This is just for the local UI — use any credentials you'll remember.

---

## Step 2 — Add credentials in n8n

Go to **Credentials** (left sidebar) → **Add Credential** for each of the three below.

### 2a. Anthropic API Key

- Credential type: **Header Auth**
- Name: `Anthropic API Key`
- Name field: `x-api-key`
- Value field: your Anthropic API key (`sk-ant-...`)

### 2b. Google Sheets + Gmail (same OAuth app)

Both Sheets and Gmail use the same Google Cloud OAuth client — you only authorize once.

**Google Sheets:**
- Credential type: **Google Sheets OAuth2 API**
- Name: `Google Sheets OAuth2`
- Paste your GCP Client ID and Client Secret
- Click **Connect** → a browser popup will ask you to authorize — approve it

**Gmail:**
- Credential type: **Gmail OAuth2 API**
- Name: `Gmail OAuth2`
- Same GCP Client ID and Client Secret
- Click **Connect** → authorize again (same Google account)

> If n8n shows a "redirect URI mismatch" error during OAuth, confirm you added exactly `http://localhost:5678/rest/oauth2-credential/callback` to your GCP OAuth client's Authorized redirect URIs.

---

## Step 3 — Set up the Google Sheet

Create a new Google Sheet at [sheets.google.com](https://sheets.google.com).

**Tab 1 — rename to:** `logbook`

Add these column headers in row 1 (exact spelling matters):
```
date  shift_type  staffing_json  sales_json  raw_notes  ai_formatted_entry  action_items  follow_ups  created_at
```

**Tab 2 — rename to:** `errors`

Add these column headers in row 1:
```
timestamp  workflow  step  error_message
```

The Sheet ID (`1mop7_bAjSq9ize2QNE_SsV8YBkvd3tCpY-v6D0BdJao`) is already embedded in both workflow JSON files. If you create a new sheet with a different ID, update the `documentId.value` field in both workflow JSON files before importing, or update it inside n8n after importing.

---

## Step 4 — Import the workflows

1. In n8n, go to **Workflows** → **Import from file**
2. Import `workflows/nightly-logbook.json`
3. Import `workflows/weekly-brief.json`

After importing each workflow:

- n8n will show **credential warnings** on nodes — click each flagged node and select the matching credential you created in Step 2
- The Google Sheets nodes need: `Google Sheets OAuth2`
- The Claude HTTP Request nodes need: `Anthropic API Key` (Header Auth)
- The Gmail nodes need: `Gmail OAuth2`

---

## Step 5 — Test Workflow 1 (Nightly Logbook)

1. Open the **Nightly Logbook Entry** workflow
2. Click **Activate** (toggle in the top right)
3. Click the **Form Trigger** node → copy the **Test URL** shown
4. Open that URL in a browser — you'll see the logbook form
5. Fill it in using one of the samples from `docs/sample-inputs.md`
6. Submit the form
7. Watch the execution in the n8n workflow view

**Expected result:**
- A row appears in your Google Sheet (`logbook` tab)
- A message appears in your Slack `#logbook` channel
- An email arrives at pzehtab287@gmail.com

If a node fails, click it in the execution view to see the error detail.

---

## Step 6 — Test Workflow 2 (Weekly Brief)

The weekly brief runs on a Sunday cron, but you can trigger it manually for testing:

1. Open the **Weekly GM Brief** workflow
2. Click **Activate**
3. To test immediately: click the **Weekly Cron** node → click **Execute Node**
   - This reads whatever is currently in your Sheet (you need at least one row from Step 5)
4. The brief should post to `#gm-brief` Slack and arrive by email

---

## Common issues

| Problem | Fix |
|---|---|
| OAuth redirect URI mismatch | Add `http://localhost:5678/rest/oauth2-credential/callback` to GCP authorized redirect URIs exactly |
| Claude returns 401 | The `x-api-key` header name must be lowercase; check the credential name field |
| Google Sheets append fails with column error | The Sheet tab must be named exactly `logbook` with the exact column headers from Step 3 |
| Slack returns 404 | The webhook URL is no longer valid; regenerate it in your Slack app settings |
| Weekly brief: "No entries in last 7 days" | The `created_at` column must be populated — check that the nightly workflow ran and wrote to the Sheet |
