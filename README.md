# P12 - AI Output Reviewer / Approval Gate

> n8n automation workflow that reviews AI-generated content using a two-phase evaluation engine — deterministic rule checks followed by an LLM quality judge (Groq Llama 3.1) — calculates a confidence score, auto-sends high-confidence outputs, routes borderline drafts to a Slack human approval queue, rejects low-quality content with detailed feedback, and maintains a complete audit trail in Google Sheets.

---

## 📌 Problem Statement

Zapier's 2026 survey found that **58% of enterprise AI users spend 3+ hours per week revising or redoing AI outputs** — even when they say AI helps productivity. The real problem is no longer just doing work, but **checking AI work**.

Most teams either:
- Blindly trust AI output and send it (risky)
- Manually review every single output (defeats automation)

**This workflow solves that by building an intelligent approval gate** that automatically reviews AI-generated content, scores it, and decides what to do — with humans only stepping in when genuinely needed.

---

## 🎯 What This Workflow Does

```
AI draft submitted
        ↓
Phase 1 — Rule Check (instant, zero LLM cost)
        ↓
Phase 2 — LLM Quality Judge (Groq Llama 3.1)
        ↓
Confidence Score calculated (Rule 40% + Quality 60%)
        ↓
score ≥ 85  → Auto-send to destination
score 60-84 → Slack human review queue (Approve / Reject links)
score < 60  → Auto-reject with detailed feedback to requester
        ↓
Full audit trail logged to Google Sheets
```

---

## 🏗️ Workflow Architecture

### Workflow 1 — AI Output Review Pipeline (11 Nodes)

```
[N1]  Webhook Trigger         → Receives AI draft via POST
[N2]  Prepare Input           → Normalizes fields, generates review_id
[N3]  Rule Checker            → Phase 1: deterministic checks
[N4]  Critical Violation?     → Gates: violation → reject, clean → LLM
[N5]  LLM Quality Judge       → Phase 2: Groq scores quality
[N6]  Confidence Score Engine → Calculates final score + decision
[N7]  Decision Router         → 3-way: AUTO_SEND / HUMAN_REVIEW / AUTO_REJECT
[N8A] Gmail Auto Send         → Sends approved draft to destination
[N8B] Slack Human Review      → Posts draft + score + approve/reject links
[N8C] Gmail Rejection Notice  → Notifies requester with violation details
[N9]  Audit Log               → Appends full record to Google Sheets
```

### Workflow 2 — Human Approval Handler (7 Nodes)

```
[N1]  Approval Webhook        → Receives approve/reject click from Slack
[N2]  Lookup Review Record    → Fetches original record from Google Sheets
[N3]  Approve or Reject?      → Routes based on action param
[N4A] Gmail Send Approved     → Sends approved draft to destination
[N4B] Gmail Notify Rejection  → Notifies requester of human rejection
[N5A] Update Log Approved     → Updates Google Sheets → Decision: APPROVED
[N5B] Update Log Rejected     → Updates Google Sheets → Decision: REJECTED
```

---

## 📸 Screenshots

### Workflow 1 — AI Output Review Pipeline
![Workflow 1](screenshots/p12_ai_output_reviewer_approval_gate_workflow1.png)

### Workflow 2 — Human Approval Handler
![Workflow 2](screenshots/p12_ai_output_reviewer_approval_handler_workflow2.png)

---

## 🧠 How the Scoring Works

### Phase 1 — Rule Checker (Deterministic)

| Check | Pass Condition | Penalty |
|---|---|---|
| Word Count | ≥ 10 words | -25 points |
| Greeting | Starts with Hello / Hi / Dear | -25 points |
| Prohibited Words | No banned phrases found | -25 points |

**Rule Score = 100 − (violations × 25)**

> If any violation found → `critical_violation: true` → skips LLM entirely → straight to rejection

### Phase 2 — LLM Quality Judge (Groq Llama 3.1)

Groq evaluates: tone, clarity, brand alignment, coherence
Returns: `quality_score (0–100)` + `reasoning`

### Final Confidence Score

```
final_score = (rule_score × 0.4) + (quality_score × 0.6)
```

| Score | Decision | Action |
|---|---|---|
| ≥ 85 | AUTO_SEND | Sent immediately to destination |
| 60–84 | HUMAN_REVIEW | Posted to Slack for human approval |
| < 60 | AUTO_REJECT | Requester notified with reasons |

---

## 🗂️ Google Sheets Structure

### Sheet Name: `Reviews`

| Column | Description |
|---|---|
| Review ID | Unique ID e.g. `rev_20260612143022` |
| Content Type | email / slack_message / report / social_post |
| Rule Score | Phase 1 score (0–100) |
| Quality Score | Phase 2 LLM score (0–100) |
| Final Score | Weighted final score (0–100) |
| Decision | APPROVED / PENDING_REVIEW / REJECTED |
| Requester | Email of person who submitted the draft |
| Destination | Where to send if approved |

---

## 📁 Repository Structure

```
n8n-problem-12-ai-output-reviewer-approval-gate/
│
├── README.md
├── p12_ai_output_reviewer_approval_gate_workflow1.json
├── p12_ai_output_reviewer_approval_handler_workflow2.json
│
└── screenshots/
    ├── p12_ai_output_reviewer_approval_gate_workflow1.png
    └── p12_ai_output_reviewer_approval_handler_workflow2.png
```

---

## 🔧 Prerequisites

Before setting up, make sure you have:

- [ ] n8n installed and running (v2.17.7+)
- [ ] ngrok or a public URL for your n8n instance
- [ ] Groq account with API key → [console.groq.com](https://console.groq.com)
- [ ] Gmail account connected to n8n via OAuth2
- [ ] Slack workspace with a review channel (e.g. `#ai-review-queue`)
- [ ] Google Sheets with the correct column structure (see above)

---

## ⚙️ Setup Instructions

### Step 1 — Create Google Sheet

1. Go to [Google Sheets](https://sheets.google.com) → create a new spreadsheet
2. Name it: `AI_Content_Review_Audit`
3. Rename Sheet 1 to: `Reviews`
4. Add these column headers in Row 1 exactly as written:

```
Review ID | Content Type | Rule Score | Quality Score | Final Score | Decision | Requester | Destination
```

5. Copy the Sheet ID from the URL:
```
https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID_IS_HERE/edit
```

---

### Step 2 — Set Up Credentials in n8n

Go to **n8n → Settings → Credentials** and create:

**Gmail OAuth2**
- Type: Gmail OAuth2
- Follow the OAuth2 flow to connect your Google account
- Name it: `Gmail account`

**Google Sheets OAuth2**
- Type: Google Sheets OAuth2
- Use the same Google account
- Name it: `Google Sheets account`

**Slack OAuth2**
- Type: Slack OAuth2
- Connect your Slack workspace
- Name it: `Slack account`
- Make sure the app has `chat:write` permission

---

### Step 3 — Import Workflow 1

1. In n8n → click **+** → **Import from file**
2. Select `p12_ai_output_reviewer_approval_gate_workflow1.json`
3. Click **Import**

---

### Step 4 — Configure Workflow 1 Placeholders

Open each node and replace the placeholders:

**LLM Quality Judge node:**
```
YOUR_GROQ_API_KEY → your actual Groq API key from console.groq.com
```

**N8B - Slack Human Review node:**
```
YOUR_N8N_WEBHOOK_URL  → your ngrok URL e.g. https://abc123.ngrok-free.app
YOUR_SLACK_CHANNEL_ID → your actual Slack channel ID
YOUR_SLACK_CHANNEL_NAME → your actual Slack channel name
```

**Audit Log node:**
```
YOUR_GOOGLE_SHEET_ID → your Sheet ID copied in Step 1
```

**All credential fields:**
- Assign Gmail account, Google Sheets account, Slack account from Step 2

---

### Step 5 — Import Workflow 2

1. In n8n → click **+** → **Import from file**
2. Select `p12_ai_output_reviewer_approval_handler_workflow2.json`
3. Click **Import**

---

### Step 6 — Configure Workflow 2 Placeholders

**Lookup Review Record + both Update nodes:**
```
YOUR_GOOGLE_SHEET_ID → same Sheet ID from Step 1
```

**All credential fields:**
- Assign Gmail account and Google Sheets account from Step 2

---

### Step 7 — Activate Both Workflows

1. Open Workflow 1 → toggle **Active** (top right) → confirm
2. Open Workflow 2 → toggle **Active** → confirm

> ⚠️ Both workflows must be active before testing

---

### Step 8 — Note Your Webhook URLs

After activating, your webhook URLs will be:

**Workflow 1:**
```
POST https://YOUR_N8N_URL/webhook/ai-review
```

**Workflow 2:**
```
GET https://YOUR_N8N_URL/webhook/approval-action
```

Make sure the Slack node in Workflow 1 uses your exact Workflow 2 URL in the Approve and Reject links.

---

## 🧪 Testing

### Test 1 — Happy Path (should AUTO_SEND or HUMAN_REVIEW)

Send POST request to `https://YOUR_N8N_URL/webhook/ai-review`:

```json
{
  "content": "Dear customer, thank you for reaching out. We have reviewed your request and will process it within 2 business days. Please do not hesitate to contact us if you have any questions. Best regards, Support Team.",
  "content_type": "email",
  "rules_profile": "support",
  "destination": "customer@example.com",
  "requester_email": "agent@yourcompany.com"
}
```

**Expected result:**
- Score ≥ 85 → email sent to `destination`
- Score 60–84 → Slack message posted with Approve / Reject links
- Google Sheets row added in both cases

---

### Test 2 — Critical Violation Path (should AUTO_REJECT instantly)

```json
{
  "content": "Click here for guaranteed free money now!",
  "content_type": "email",
  "rules_profile": "support",
  "destination": "customer@example.com",
  "requester_email": "agent@yourcompany.com"
}
```

**Expected result:**
- LLM never called
- Rejection email sent to `requester_email`
- Google Sheets row added with Decision: REJECTED

---

### Test 3 — Human Approval Flow

1. Send Test 1 payload and get a HUMAN_REVIEW result
2. Open your Slack `#ai-review-queue` channel
3. Click the ✅ **Approve** link
4. Verify Gmail sends approved draft to `destination`
5. Verify Google Sheets Decision column updates to `APPROVED`

Repeat with ❌ **Reject** link:
- Verify rejection email sent to `requester_email`
- Verify Google Sheets Decision column updates to `REJECTED`

---

## 📥 Input Payload Reference

| Field | Required | Type | Description |
|---|---|---|---|
| `content` | ✅ | string | The AI-generated draft text to review |
| `content_type` | ✅ | string | `email` / `slack_message` / `report` / `social_post` |
| `rules_profile` | ✅ | string | `support` / `marketing` / `internal` |
| `destination` | ✅ | string | Email address to send approved content to |
| `requester_email` | ✅ | string | Email of person who submitted the draft |

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| n8n (v2.17.7) | Workflow automation engine |
| Groq API (llama-3.1-8b-instant) | LLM quality judge |
| Gmail | Send approved drafts + rejection notices |
| Slack | Human review queue with approve/reject links |
| Google Sheets | Full audit trail + review record storage |
| Webhook | Entry point for AI draft submission |

---

## 💡 Key Design Decisions

**Why two-phase review?**
Rule checks catch obvious failures instantly at zero cost. LLM only runs on drafts that already passed basics — saving API calls and reducing latency.

**Why 40/60 weighting?**
Rules are binary and precise. LLM scoring is probabilistic. Giving LLM slightly more weight allows nuanced quality judgments to influence the final score more than structural checks.

**Why 3-way routing instead of binary?**
Binary routing either over-trusts AI or creates human bottlenecks. Three-way routing means only genuinely borderline content requires human attention.

**Why store context in Google Sheets?**
Workflow 2 needs the original draft data to send the approved email. Storing in Sheets makes the system stateless and resilient — no dependency on live execution data.

---

## ⚠️ Security Notes for Production

Before deploying to production:

- [ ] Move Groq API key to n8n HTTP Header Auth credential — never hardcode
- [ ] Replace ngrok URL with a stable domain
- [ ] Add webhook authentication to both webhooks
- [ ] Restrict Google Sheets access to service account only

---

## 👩‍💻 Author

S Jagadeesh
