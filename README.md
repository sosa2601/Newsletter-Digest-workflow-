# Newsletter Intelligence System
### VysionAI · n8n Self-Hosted · Saswati Gorai

A two-workflow system that scrapes Gmail newsletters on AI and GTM marketing, stores them in Google Sheets, and sends a personalized AI digest every 3–4 days.

---

## System Overview

```
WF1: Newsletter Data Dump Creation
Gmail (labeled emails) → Google Sheets (Newsletter_Dump) → Config update

WF2: Newsletter Delta Check + AI Digest
Google Sheets (unprocessed rows) → Claude via OpenRouter → Gmail digest → Mark processed
```

**Google Sheet:** `Newsletter Dump` (ID: `1EEFcyItS4QHrpHy8mk0qknQ55aWqkbN5k5DKERf921s`)
- Sheet tabs: `Newsletter_Dump`, `Config`

---

## Workflow 1 — Newsletter Data Dump Creation

**n8n ID:** `VtGlyFjAvkHP9r2P`

**What it does:**
Fetches all emails from Gmail with labels `AI-Content` or `GTM`, cleans the body text, deduplicates, writes new rows to the `Newsletter_Dump` sheet, and updates the `last_scraped_date` in the `Config` sheet so next run only pulls new emails.

**Node-by-node:**

| Node | Purpose |
|---|---|
| Schedule Trigger | Triggers the workflow on a set schedule |
| Read Config Sheet | Reads `last_scraped_date` from Config tab — used as the Gmail fetch cutoff |
| Set Start Date | Parses the date into `YYYY-MM-DD` format |
| Gmail Fetch Labeled Emails Only | Fetches emails with label `AI-Content` OR `GTM`, received after start date |
| Deduplicate and Clean Emails | Strips HTML, removes unsubscribe footers, truncates body at 8,000 chars, deduplicates by `email_id` |
| Write to Newsletter Dump | Appends cleaned rows to `Newsletter_Dump` sheet |
| Count and Prep Config Update | Counts rows written, preps today's date |
| Update Config Date to Today | Updates `last_scraped_date` in Config — prevents re-fetching on next run |

**Sheet columns written:**
`email_id`, `date`, `sender`, `subject`, `body`, `source_type`, `source_channel`, `processed`, `scraped_at`

**Gmail label requirement:**
Emails must be labeled `AI-Content` or `GTM` in Gmail for this workflow to pick them up. Set up filters in Gmail to auto-label newsletters from your subscribed senders.

---

## Workflow 2 — Newsletter Delta Check + AI Digest

**n8n ID:** `5PfouC5H3aHQxYp3`

**What it does:**
Reads unprocessed rows from `Newsletter_Dump`, filters to the last 10 days, builds a context-aware prompt using your professional profile, sends it to Claude (via OpenRouter), emails you the formatted digest, and marks all processed emails in the sheet.

**Node-by-node:**

| Node | Purpose |
|---|---|
| Schedule Daily 11am | Runs on Mon / Wed / Sat at 1am (configurable) |
| Read Unprocessed Emails from Dump | Filters `Newsletter_Dump` where `processed = false` |
| Context + Build Prompt — EDIT HERE | Filters to last N days, builds the full Claude prompt with your context |
| Claude Generate Digest | Calls OpenRouter → `anthropic/claude-haiku-4.5`, max 4096 tokens |
| Send Digest Email | Sends styled HTML digest to `contact@vysionai.com` |
| Prep Processed IDs | Extracts all `email_id` values from this batch |
| Mark as Processed in Sheet | Updates `processed = true` for each email — prevents re-inclusion in next digest |

**Digest format (7 sections Claude produces):**
1. New AI Tools — name | what it does | why relevant
2. GTM & Growth Trends — key insights, interview flags
3. How Companies Are Running GTM — team structures, hiring signals
4. Automation & Workflow Ideas — VysionAI offering / portfolio angles
5. Interview & Outreach Language — exact phrases and frameworks
6. Content Ideas — 3–5 trending content angles
7. Top Sources This Round — highest-signal newsletters

---

## Configuration

### To change the lookback window (WF2)
In the `Context + Build Prompt — EDIT HERE` code node, update:
```js
const DAYS_BACK = 10; // change to any number
```

### To update your context/profile (WF2)
Edit the `MY_CONTEXT` object in the same node:
```js
const MY_CONTEXT = {
  name: 'Saswati Gorai',
  role: '...',
  goal: '...',
  // etc.
};
```

### To change the digest delivery email (WF2)
In the `Send Digest Email` Gmail node, update the `sendTo` field.

### To change the AI model (WF2)
In the `Claude Generate Digest` HTTP node, update:
```json
"model": "anthropic/claude-haiku-4.5"
```
Options: `anthropic/claude-sonnet-4-5`, `anthropic/claude-opus-4` (higher cost, higher quality).

### To change scrape schedule (WF1)
Update the `Schedule Trigger` node interval. Default is every hour (no specific schedule set — configure as needed).

### To change digest schedule (WF2)
Update `Schedule Daily 11am` — currently set to trigger on days 1, 3, 6 (Mon/Wed/Sat) at 1am.

---

## Google Sheet Setup

The sheet `Newsletter Dump` needs two tabs:

**Tab: `Config`**
| key | value | notes |
|---|---|---|
| last_scraped_date | 2026-01-01 | Set this to your desired backfill start date on first run |

**Tab: `Newsletter_Dump`**
Columns (in order):
`email_id` | `date` | `sender` | `subject` | `body` | `source_type` | `source_channel` | `matched_keywords` | `processed` | `scraped_at`

---

## Gmail Label Setup

For WF1 to pick up your newsletters, emails must have one of these labels:
- `AI-Content`
- `GTM`

**Recommended setup:**
Go to Gmail → Settings → Filters and Blocked Addresses → Create filter for each newsletter sender → Apply label `AI-Content` or `GTM`.

---

## Dependencies

| Credential | Used In | Type |
|---|---|---|
| Google Sheets account | WF1 + WF2 | OAuth2 (`gFHCIe3QxodFLCCc`) |
| Gmail account VysionAI | WF1 + WF2 | OAuth2 (`ab1Q0SaIWIwIUR86`) |
| OpenRouter API key | WF2 | API key in HTTP header |

**OpenRouter key** is hardcoded in the `Claude Generate Digest` HTTP node header (`x-api-key`). Rotate this if the workflow is shared or moved.

---

## First Run Checklist

- [ ] Create the `Newsletter Dump` Google Sheet with `Config` and `Newsletter_Dump` tabs
- [ ] Add `last_scraped_date` row in Config tab with your desired backfill start date
- [ ] Set up Gmail filters and labels (`AI-Content`, `GTM`)
- [ ] Import both workflows into n8n
- [ ] Verify Google Sheets and Gmail credentials are connected
- [ ] Run WF1 manually once to backfill existing emails
- [ ] Confirm rows appear in `Newsletter_Dump` with `processed = false`
- [ ] Run WF2 manually once to test digest generation and email delivery
- [ ] Activate both workflows for scheduled runs

---

## Troubleshooting

**No emails fetched (WF1)**
- Check Gmail labels exist and are applied to newsletters
- Check `last_scraped_date` in Config isn't set to a future date

**Digest email not sending (WF2)**
- Check Gmail OAuth credentials are still valid (re-authorize if expired)
- Check `processed = false` rows exist in `Newsletter_Dump`

**Empty digest / "No emails in last N days"**
- WF1 may not have run recently — trigger it manually
- Increase `DAYS_BACK` in the code node

**Claude returns no content**
- Check OpenRouter API key is valid and has credits
- Check model name is correct (`anthropic/claude-haiku-4.5`)

---

*Built by Saswati Gorai · VysionAI · vysionai.com*
