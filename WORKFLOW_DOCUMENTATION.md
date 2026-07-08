# Lead Qualification & Human Approval Workflow
## Complete Setup and Configuration Guide

---

## What This Workflow Does

This workflow automates the process of receiving, qualifying, and responding to new leads. When a lead is added to your Google Sheet, the system instantly picks it up, validates the data, generates an AI summary, scores the lead, drafts a personalized follow-up email, and sends all of that to Slack for your approval. Nothing is sent to the lead without your explicit click. You approve or reject directly from Slack and the sheet updates automatically.

---

## Architecture Overview

```
New lead added to Google Sheet
        ↓
Apps Script fires webhook to n8n instantly
        ↓
Flatten Lead Data (normalize fields)
        ↓
Write "Pending" to sheet
        ↓
Validate required fields
        ↓                    ↘
AI Summary → AI Score        Error Log (missing fields)
        ↓
AI Draft Follow-Up Message
        ↓
Slack — Send and Wait for Approval  ← HUMAN DECISION POINT
        ↓                    ↘
Approve                      Reject
   ↓                           ↓
Write "Approved"           Write "Rejected"
   ↓                           ↓
Send Email to Lead         Slack Confirm Rejected
   ↓
Slack Confirm Approved
```

---

## Prerequisites

Before configuring this workflow you need the following accounts and credentials ready.

- An n8n Cloud account (or self-hosted n8n instance)
- A Google account with access to Google Sheets and Gmail
- A Slack workspace where you are an admin
- A Google AI Studio account for the Gemini API key

---

## Step 1 — Google Sheet Setup

Your main leads sheet must have these exact columns in this order. Column names are case sensitive and some have trailing spaces — copy them exactly.

| Column | Notes |
|--------|-------|
| Name | Lead full name |
| Emai | Lead email address (note: one l) |
| Phone | Lead phone number |
| Company | Company or business name |
| Adress | Address or location (note: one d) |
| Message | Lead's message or inquiry |
| Status  | Leave blank — workflow writes here (has trailing space) |
| Submitted  | Leave blank — workflow writes timestamp here (has trailing space) |
| Lead_ID | Leave blank — auto-generated |
| row_number | Leave blank — read-only, managed by Apps Script |

Create a second tab in the same spreadsheet called **Error Log** with these columns: Timestamp, Lead Name, Email, Error Type, Details, Status.

---

## Step 2 — Google Apps Script Setup

This script fires the webhook the moment a new row is added to your sheet.

Open your Google Sheet. Click **Extensions** then **Apps Script**. Delete any existing code and paste the following:

```javascript
function sendToN8N(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var lastRow = sheet.getLastRow();
  var headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  var data = sheet.getRange(lastRow, 1, 1, sheet.getLastColumn()).getValues()[0];

  var payload = {};
  headers.forEach(function(header, index) {
    payload[header] = data[index];
  });
  payload['row_number'] = lastRow;
  payload['Lead_ID'] = 'LEAD-' + lastRow + '-' + Date.now();

  var options = {
    method: 'POST',
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };

  var response = UrlFetchApp.fetch('YOUR_N8N_WEBHOOK_URL', options);
  Logger.log('Response: ' + response.getResponseCode());
}
```

Replace `YOUR_N8N_WEBHOOK_URL` with your n8n Webhook node's production URL.

Save the script. Click the clock icon on the left sidebar. Click **Add Trigger** and configure as follows:
- Function to run: **sendToN8N**
- Event source: **From spreadsheet**
- Event type: **On edit**

Save and authorize when Google prompts you.

---

## Step 3 — n8n Credentials Setup

In n8n go to **Credentials** and add the following. Never share these credential IDs or keys with anyone.

**Google Sheets OAuth2**
- Click Add Credential
- Search for Google Sheets OAuth2
- Click Connect with Google and authorize

**Gmail OAuth2**
- Click Add Credential
- Search for Gmail OAuth2
- Click Connect with Google and authorize

**Google Gemini API**
- Go to aistudio.google.com
- Click Get API Key and create a new key
- In n8n click Add Credential
- Search for Google PaLM API
- Paste your API key

**Slack OAuth2**
- In n8n click Add Credential
- Search for Slack OAuth2
- Follow the OAuth flow to connect your workspace
- Make sure the bot has been added to your approval channel

---

## Step 4 — Workflow Configuration

After importing the workflow JSON update the following placeholders. Search for each one and replace with your actual values.

| Placeholder | Replace with |
|-------------|--------------|
| YOUR_GOOGLE_SHEET_ID | The ID from your sheet URL between /d/ and /edit |
| YOUR_GOOGLE_SHEETS_CREDENTIAL_ID | The credential ID from n8n Credentials page |
| YOUR_GEMINI_CREDENTIAL_ID | The credential ID from n8n Credentials page |
| YOUR_SLACK_CREDENTIAL_ID | The credential ID from n8n Credentials page |
| YOUR_GMAIL_CREDENTIAL_ID | The credential ID from n8n Credentials page |

The Slack channel ID `C0BFZC59G04` is already set to your channel. If you want to change the channel right-click the channel in Slack, click Copy Link, and extract the ID from the end of the URL.

---

## Step 5 — Activate the Workflow

In n8n click the **Inactive** toggle in the top right corner of your workflow to turn it **Active**. The toggle turns green when active. The production webhook URL is now live and will receive requests from your Apps Script immediately.

---

## How the Human Approval Works

When a lead arrives and passes through the AI steps, n8n sends a Slack message to your approval channel using the **Send and Wait** operation. This is n8n's built-in human-in-the-loop feature. It pauses the workflow indefinitely and waits for your response.

The Slack message contains:
- Lead name, email, phone, company, address, and Lead ID
- AI-generated lead summary
- Qualification score with reason
- AI-drafted follow-up email message

Two buttons appear at the bottom of the message: **Approve and Send Email** and **Reject**.

Clicking **Approve and Send Email** resumes the workflow, writes Approved to the sheet, sends the drafted email to the lead via Gmail, and confirms in Slack.

Clicking **Reject** resumes the workflow, writes Rejected to the sheet, and confirms in Slack. No email is sent to the lead.

---

## Error Handling

**Missing required fields**: If a lead arrives without a name, email, or phone number the Validate Lead Fields node catches it and routes to the Error Log. A row is appended to your Error Log tab with the timestamp, available lead info, and which fields were missing. The lead's Status is not updated and no Slack message is sent.

**Gemini API failure**: If the AI steps fail n8n marks the execution as failed in the executions list. Check the executions log in n8n for the error detail. The lead will remain in Pending status in your sheet.

**Slack delivery failure**: If the Slack message fails to send the execution fails before reaching the Wait node. The lead remains in Pending status. Check n8n executions for the error.

**Gmail send failure**: If the email fails after approval the sheet is already updated to Approved but the email was not sent. Check executions and resend manually from Gmail.

---

## Checking Executions

All production workflow runs appear in the **Executions** list in n8n, accessible from the left sidebar. Each execution shows which nodes ran and which failed. Click any execution to see the data at each step. This is your primary debugging tool.

---

## Node Reference

| Node | Purpose |
|------|---------|
| Webhook | Receives POST request from Apps Script |
| Flatten Lead Data | Extracts body fields into clean variables |
| Write Pending Status | Marks lead as Pending in sheet immediately |
| Validate Lead Fields | Checks name, email, phone are present |
| Lead Summarize AI | Gemini writes a 2-3 sentence lead summary |
| Lead Score AI | Gemini scores the lead 1-10 with justification |
| Draft Follow Up Msg AI | Gemini writes personalized follow-up email |
| Slack — Send and Wait | Sends approval message and pauses workflow |
| Approved or Rejected? | Routes based on Slack button clicked |
| Write Approved Status | Updates sheet to Approved with timestamp |
| Send Email to Lead | Sends AI-drafted email via Gmail |
| Slack — Confirm Approved | Notifies you email was sent |
| Write Rejected Status | Updates sheet to Rejected with timestamp |
| Slack — Confirm Rejected | Notifies you lead was rejected |
| Error Log | Logs missing-field errors to Error Log tab |

---

## AI Model

All three AI steps use **Google Gemini 2.0 Flash Lite** (`models/gemini-2.0-flash-lite`). This model is fast, free-tier generous at 1500 requests per day, and produces high quality output for summarization and drafting tasks. No OpenAI API key is required.

---

## Handover Checklist

Before handing this workflow over confirm the following are complete.

- [ ] Google Sheet created with correct column names
- [ ] Error Log tab created in the same spreadsheet
- [ ] Apps Script installed with correct webhook URL and trigger set to On edit
- [ ] All four credentials connected in n8n
- [ ] All placeholder values replaced in the workflow JSON
- [ ] Workflow set to Active
- [ ] Test lead added to sheet and full end-to-end flow confirmed
- [ ] Approve button tested and email received by lead
- [ ] Reject button tested and status updated in sheet

---

## Support

For any issues check the n8n Executions list first. It shows exactly which node failed and why. Most issues are credential expiry, wrong sheet column names, or the workflow being set to Inactive.
