# AI Message Classifier with Fallback & Audit Logging

An n8n workflow that classifies incoming messages into structured categories using Google Gemini — with a catch. It doesn't trust the AI on the first try. If confidence is low or the response fails validation, it escalates through deeper analysis, retries, and finally logs persistent failures to a database for human review.

---

## What It Does

Takes a message like:

> *"Hi, I'm Sarah from Bright Dental, sarah@brightdental.com, we need help automating our appointment reminders, fairly urgent."*

And classifies it into exactly one category:

- `booking_request`
- `price_inquiry`
- `complaint`
- `general_question`

The workflow routes through multiple AI passes depending on how confident the model is, and validates every response against a strict allow-list before accepting it.

---

## Workflow Architecture
Input → Cleanup → Gemini Fast Classifier → Parse & Validate
↓
┌─────────────────────────┘
↓
Confidence Check (high + valid?) ──Yes──→ Return Result
↓ No
Gemini Deep Reclassifier → Parse & Validate
↓
Validation Check (passed?) ──Yes──→ Return Result
↓ No
Gemini Retry → Parse & Validate
↓
Validation Check (passed?) ──Yes──→ Return Result
↓ No
Log to Neon PostgreSQL → Return Failure Response
plain

---

## Node Breakdown

| Node | Purpose |
|------|---------|
| **Edit Fields** | Sets the raw `message` input for testing. |
| **Code: Cleanup** | Trims whitespace and capitalizes the first letter for consistency. |
| **Gemini Fast Classifier** | First-pass AI. Quick classification with confidence score (`high` / `medium` / `low`). |
| **Parse Gemini** | Strips markdown fences, parses JSON, validates the category against an allow-list (`booking_request`, `price_inquiry`, `complaint`, `general_question`). |
| **Confidence Check** | Routes low-confidence or failed validations to the deep reclassifier. High-confidence valid results return immediately. |
| **Gemini Deep Reclassifier** | Second-pass AI with a stricter prompt. Receives the original message plus the fast classifier's reason for uncertainty. Forces `high` confidence. |
| **Validation IF** | Checks if the deep classification passed validation. If yes, returns the result. If no, triggers the retry node. |
| **Gemini Retry** | Third-pass AI with the strictest prompt. Last attempt to get a valid category. |
| **Parse Retry Code** | Parses and validates the retry response. Packages everything into a structured result object plus raw AI response metadata. |
| **validatiom** | Final validation gate. If passed, returns the result. If failed, routes to the database logger. |
| **insert to Neon** | Persists failed classifications to a PostgreSQL database (`failed_classifications` table) with full context for manual review. |
| **Respond to Webhook** | Returns clean JSON at every exit point (fast success, deep success, retry success, failure). |

---

## Example Outputs

### Fast Path — High Confidence, Valid
```json
{
  "status": "classified",
  "message": "Hi, I'm Sarah from Bright Dental...",
  "category": "booking_request",
  "confidence": "high",
  "reason": "Message explicitly requests appointment automation help",
  "source": "gemini-fast"
}
Fallback Path — Deep Reclassifier
JSON
{
  "status": "classified",
  "message": "Hi, I'm Sarah from Bright Dental...",
  "category": "booking_request",
  "confidence": "high",
  "reason": "Re-analyzed and confirmed booking intent",
  "source": "gemini-deep"
}
Failure Path — All Attempts Failed, Logged
JSON
{
  "status": "classified",
  "message": "Hi, I'm Sarah from Bright Dental...",
  "category": "general_question",
  "confidence": "low",
  "reason": "AI returned invalid response",
  "source": "gemini-fast-failed"
}
Requirements
n8n — Self-hosted or n8n Cloud
Google Gemini API Key — For all three AI nodes
Neon PostgreSQL — For the failed_classifications audit table
Basic familiarity with n8n expressions, JSON, and SQL
Setup Instructions
1. Import the Workflow
Download the workflow JSON from this repo.
In n8n, go to Workflows → Import from File.
Select the downloaded JSON.
2. Configure Credentials
Open each Google Gemini node.
Under Credentials, add your Google Gemini API key (or separate keys if you want to rotate between them).
Open the insert to Neon node.
Add your Neon PostgreSQL connection credentials.
3. Create the Database Table
Run this SQL in your Neon database before executing:
sql
CREATE TABLE IF NOT EXISTS failed_classifications (
  id SERIAL PRIMARY KEY,
  original_message TEXT NOT NULL,
  original_category TEXT,
  original_confidence TEXT,
  original_reason TEXT,
  original_source TEXT,
  validation_passed BOOLEAN,
  raw_ai_response_object JSONB,
  failed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
4. Test the Workflow
Click Execute Workflow.
The workflow runs using the sample message in the Edit Fields node.
Check the output of the relevant Respond to Webhook node depending on which path the message took.
Why Multi-Pass?
Fast AI is cheap and fast, but not always right. This workflow treats classification as a pipeline:
Fast pass handles the clear cases instantly.
Deep pass handles ambiguity with more context.
Retry pass is the last resort.
Database logging ensures nothing disappears into a black hole when all AI attempts fail.
The validation layer is hardcoded — if Gemini invents a category that isn't in the allow-list, the workflow rejects it and retries instead of passing garbage downstream.
File Structure
plain
.
├── README.md              # This file
├── workflow.json          # Full n8n workflow export
└── workflow-screenshot.png
Tech Stack
n8n — Workflow automation
Google Gemini — LLM for classification
Neon — Serverless PostgreSQL for audit logging
JavaScript — Custom parsing, validation, and data transformation
