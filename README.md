# Agentic AI Screening Demo  
AI-Powered Candidate Screening Workflow using AWS Step Functions + Lambda + Bedrock

---

## Overview

This project demonstrates a production-style **Agentic AI screening workflow** for high-volume frontline hiring.

It combines:

- Durable orchestration with **AWS Step Functions**
- Modular workers implemented as **AWS Lambda functions**
- Deterministic business rules for scoring
- LLM-powered structured extraction and rationale generation using **Amazon Bedrock (Anthropic Claude Sonnet 4 via Inference Profile)**
- S3-based data persistence
- A lightweight web UI (hostable on GitHub Pages)

The goal is not perfect hiring accuracy, but to demonstrate:

- Agentic orchestration patterns  
- Hybrid AI (LLM + deterministic policy)  
- Auditability and traceability  
- Scalable screening architecture  

---

## Purpose

High-volume hiring (e.g., hourly workforce, frontline roles) requires:

- Fast triage of applications  
- Clear scoring logic  
- Transparent decisions  
- Ability to handle incomplete submissions  
- Repeatable and auditable workflow  

This demo shows how to build a structured, production-ready screening pipeline using serverless AWS components.

---

## Context

Traditional screening systems often suffer from:

- Manual CV reading  
- Inconsistent evaluation criteria  
- Black-box AI decisions  
- Poor traceability  
- Lack of governance  

This solution introduces:

- Deterministic scoring logic  
- Clear separation between extraction and policy  
- LLM only where it adds value  
- Full execution trace via Step Functions  

---

# Architecture

## High-Level Architecture

[ Web UI (HTML) ]
|
v
[ Lambda Function URL (API) ]
|
v
[ AWS Step Functions ]
|
+--> 4A Load Job Context
|
+--> 4B Ingest Application Event
|
+--> 4C Normalize & Dedupe
|
+--> 4D Structured Extraction (Bedrock)
|
+--> 4E Fit Scoring (Deterministic + Bedrock rationale)
|
+--> 4F Next Best Action
|
+--> 4G Update ATS (Simulated)
|
+--> 4H Manage Candidate Communication (Simulated)
|
+--> 4I Schedule Interviews (Simulated)
|
+--> 4J Output Metrics (Simulated)
|
+--> 4K Write Back to S3


---

## Design Principles

- Modular worker Lambdas  
- Deterministic business scoring  
- LLM used only for extraction and explanation  
- Clear separation of responsibilities  
- Durable execution state  
- Fully observable and auditable  

---

# Business Logic: Extraction, Scoring, and Decision

The workflow begins with a **Structured Extraction** step powered by Claude (via Amazon Bedrock, temperature = 0). It converts the unstructured job description and candidate application into a strict JSON schema:

- `years_experience` (number)  
- `has_required_certification` (true/false)  
- `education_level` (string)  
- `skills` (array of strings)  
- `availability` (string)  
- `confidence` (number, model self-reported extraction reliability)  

This structured output is passed to a fully deterministic **Fit Scoring** engine.

## Deterministic Scoring Rules

- **+40 points** if `years_experience >= 2`  
  - Otherwise → `"Minimum 2 years experience"`  
- **+30 points** if `has_required_certification == true`  
  - Otherwise → `"Required certification"`  
- **+20 points** if `availability` is confirmed  
  - Otherwise → `"Availability confirmation"`  
- **+10 bonus points** if `confidence > 70`  

**Maximum score: 100**

## Decision Logic

- **Score ≥ 75 AND no missing required items → "Interview Scheduled"**  
- **Score ≥ 40 → "Missing Information / Clarification Required"**  
- **Score < 40 → "Rejected"**  

All decisions are rule-based, ensuring explainability, auditability, and compliance-ready behavior.

---

# End-to-End Flow

## 1. User Input (Web UI)

The user pastes:

- Candidate CV (text)  
- Candidate responses to screening questions  
- Job description  

The UI:

- Calls backend API (Lambda Function URL)  
- Starts a Step Function execution  
- Polls for completion  
- Displays:
  - Application Status  
  - Score  
  - Confidence  
  - Explanation  
  - Missing Information (if applicable)  
  - Execution ARN  

---

## 2. Step Function Workflow

### Step 4A — Load Job Context

Prepares the evaluation configuration for the workflow.  
Receives `jobDescriptionText`, `jobId`, and `locationId`.  
Builds a `requirementsSchema` based on detected keywords (e.g., certifications, must-have skills).  
Generates rule metadata (`roleId`, `locationId`, `jdVersion`, `eeoc_audit_enabled`).  

This step does not process candidate data.

---

### Step 4B — Ingest Application Event

Standardizes raw submission data into a consistent `application` object:

- `applicationId`  
- `cvText`  
- `screeningAnswersText`  
- `jobDescriptionText`  
- `jobId`  
- `locationId`  
- `availability`  

Ensures structural consistency only.

---

### Step 4C — Normalize & Dedupe

Ensures repeated submissions can be consistently identified.

#### Deterministic Normalization

`normalized = re.sub(r"\s+", " ", (cv + " " + answers).strip().lower())`

This:

- Removes leading/trailing whitespace  
- Converts text to lowercase  
- Replaces multiple spaces and line breaks with a single space  

Example:

`"John DOE\n\n3 Years Experience"`  
→ `"john doe 3 years experience"`

#### Dedupe Key Generation

`dedupe_key = hashlib.sha256(normalized.encode("utf-8")).hexdigest()[:16]`

Properties:

- Fully deterministic  
- Same content → same key  
- Any change → different key  
- No LLM involved  

Demo mode always returns `"isDuplicate": False`.

---

### Step 4D — Structured Extraction (LLM)

Uses **Amazon Bedrock (Claude Sonnet 4 via Inference Profile)** to extract structured fields.

Confidence is a model-generated reliability estimate reflecting extraction certainty (clear dates and explicit certifications yield higher confidence; vague phrasing yields lower confidence).

Important:

- `temperature = 0`  
- Strict JSON output enforced  
- Markdown removed automatically  
- JSON parsed robustly  

In production, the LLM remains limited to semantic normalization.  
All decisions remain deterministic.

---

### Step 4E — Fit Scoring (Deterministic + LLM Rationale)

Deterministic score calculation as defined above.

Score bands:

- High ≥ 75  
- Medium ≥ 40  
- Low < 40  

LLM generates a short explanation (max 4 sentences).  
The LLM does **not** decide the outcome.

---

### Step 4F — Next Best Action

Applies deterministic policy based on:

- `score`  
- `missingInfoList`  
- `confidence`  

Returns:

- `finalDecision`  
- Policy label  
- Optional rejection reason code  

Low confidence (< 0.6) may trigger rejection.

---

### Step 4G — Update ATS (Simulated)

Maps `finalDecision` to an ATS status (e.g., `InvitedToInterview`, `Rejected`).  
Returns `SIMULATED_OK`.

---

### Step 4H — Manage Candidate Communication (Simulated)

Generates simulated candidate messages:

- Interview link  
- Clarification request  
- Rejection notice  

Returns structured communication metadata.

---

### Step 4I — Schedule Interviews (Simulated)

Represents scheduling integration.  
Returns simulated scheduling result.

---

### Step 4J — Output Metrics (Simulated)

Generates demo KPIs:

- `timeToFirstDecision`  
- `inviteToScheduleConversionRate`  
- `scheduleCompletionRate`  
- `humanReviewRate`  
- `queueLatency`  
- `throughputPerMinute`  

Marked with `"metricsStatus": "SIMULATED"`.

---

### Step 4K — Write Back to S3

Persists final summary to S3:

- Raw input  
- Extracted fields  
- Score  
- Confidence  
- Explanation  
- Final decision  
- Execution ARN  
- Metrics  

Updates status to `COMPLETED`.

---

# LLM Integration Details

**Model:** Anthropic Claude Sonnet 4  
**Invocation via Bedrock Inference Profile:**

`arn:aws:bedrock:eu-west-3:<ACCOUNT_ID>:inference-profile/eu.anthropic.claude-sonnet-4-20250514-v1:0`

Why inference profile?

- Required for newer Anthropic models  
- Handles throughput routing  
- Avoids on-demand invocation errors  

---

## IAM Requirements

Lambda role must include:

- `bedrock:InvokeModel`  
- `bedrock:InvokeModelWithResponseStream`  
- AWS Marketplace permissions (first invocation only):
  - `aws-marketplace:Subscribe`  
  - `aws-marketplace:ViewSubscriptions`  
  - `aws-marketplace:Unsubscribe`  

---

# Key AWS Resources

- AWS Step Functions (STANDARD workflow)  
- Lambda (modular workers)  
- Lambda Function URL (API)  
- Amazon S3 (persistence)  
- Amazon Bedrock  
- IAM roles (least privilege)  
- CloudWatch Logs (observability)  

---

# Observability & Governance

- Unique execution ARN per run  
- Full state history in Step Functions  
- Deterministic scoring ensures reproducibility  
- LLM limited to extraction and explanation  
- Policy-driven decision layer  

---

# Security Model

- No direct Bedrock access from frontend  
- LLM invoked only from backend Lambda  
- IAM roles scoped to required permissions  
- No secrets in frontend  
- S3 write-back for traceability  

---

# Why This Is Agentic

This workflow demonstrates:

- Orchestrated multi-step reasoning  
- Task decomposition  
- Structured intermediate memory  
- Clear execution boundaries  
- Hybrid AI + deterministic policy  
- Next Best Action logic  

It is not a single prompt.  
It is a stateful AI workflow.

---

# Scalability

Supports:

- High concurrency (serverless)  
- Adding new criteria  
- Swapping models  
- Replacing scoring logic  
- Human-in-the-loop gates  
- Real ATS integration  

---

# FAQ: Example — John Doe Case

## Normalization, De-Duplication, and Extraction

| Step | What It Does | Example (John Doe) | Output |
|------|--------------|--------------------|--------|
| Ingest | Normalize structure | `{ "applicationId": "123", "cvText": "JOHN DOE...", "jobId": "warehouse-role" }` | Clean JSON |
| Normalize / Dedupe | Canonicalize text + generate fingerprint | `"JOHN DOE\n\nWorked..."` → `"john doe worked..."`<br>`a3f91c7b2e4d8f10` | Clean string + hash |
| LLM Extraction | Semantic parsing | `{ "years_experience": 3, "has_required_certification": true, "confidence": 82 }` | Structured features |

---

# Disclaimer

This is a demonstration system.  
It does not replace human hiring judgment.

Real-world implementation must include:

- Legal review  
- Bias auditing  
- Equal opportunity compliance  
- Data protection safeguards  
- Human oversight  

---

If you found this project useful, feel free to fork and extend it.
