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

Traditional screening systems suffer from:

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

```
[ Web UI (HTML) ]
        |
        v
[ Lambda Function URL (API) ]
        |
        v
[ AWS Step Functions ]
        |
        +--> 4A Load Input data from S3 bucket
        |
        +--> 4B Ingest historical applicants data from S3 bucket
        |
        +--> 4C Normalize & Dedupe
        |
        +--> 4D Structured Extraction (Bedrock)
        |
        +--> 4E Fit Scoring (Deterministic + Bedrock rationale)
        |
        +--> 4F Update ATS
        |
        +--> 4G Manage communication with candidate
        |
        +--> 4H Schedule interviews
        |
        +--> 4I Output metrics 
        |
        +--> 4J Write Back to S3
```

---

## Design Principles

- Modular worker Lambdas
- Deterministic business scoring
- LLM only for extraction + explanation
- Clear separation of responsibilities
- Durable execution state
- Observable and auditable

---



## Business logic: Extraction, Scoring, and Decision Logic

The workflow begins with a **Structured Extraction** step powered by Claude (via Amazon Bedrock, temperature = 0), which converts the unstructured job description and candidate application into a strict JSON schema containing the following fields:

- `years_experience` (number)  
- `has_required_certification` (true/false)  
- `education_level` (string)  
- `skills` (array of strings)  
- `availability` (string)  
- `confidence` (number, model self-reported extraction reliability)

This structured output is passed to a fully deterministic **Fit Scoring** engine. The scoring rubric is explicitly defined as follows:

- **+40 points** if `years_experience >= 2`  
  - Otherwise: record missing item → `"Minimum 2 years experience"`
- **+30 points** if `has_required_certification == true`  
  - Otherwise: record missing item → `"Required certification"`
- **+20 points** if `availability` is confirmed  
  - Otherwise: record missing item → `"Availability confirmation"`
- **+10 bonus points** if `confidence > 70`

Maximum possible score: **100 points**

The final decision is rule-based:

- **Score ≥ 75 AND no missing required items → "Interview Scheduled"**
- **Score ≥ 40 → "Missing Information / Clarification Required"**
- **Score < 40 → "Rejected"**

All decisions are driven by explicit weighted criteria tied directly to job requirements, ensuring explainability, auditability, and compliance-ready behavior.

# End-to-End Flow

## 1. User Input (Web UI)

User pastes:

- Candidate CV (text)
- Candidate responses to screening questions
- Job description

The UI:

- Calls backend API (Lambda Function URL)
- Starts a Step Function execution
- Polls for completion
- Displays real results:
  - Application Status
  - Score
  - Confidence
  - Explanation
  - Missing Information (if applicable)
  - Execution ARN

---

## 2. Step Function Workflow

### Step 4A — Normalize & Dedupe
- Cleans input structure
- Prevents duplicate execution (optional)
- Prepares consistent application object

---

### Step 4B — Load Job Context
- Loads job metadata from S3
- Adds structured job context to execution state

---

### Step 4C — Structured Extraction (LLM)

Uses **Amazon Bedrock (Claude Sonnet 4 via inference profile)** to extract structured JSON:

```json
{
  "years_experience": number,
  "has_required_certification": true/false,
  "education_level": "string",
  "skills": ["string"],
  "availability": "string",
  "confidence": number
}
```

Important:

- Temperature = 0
- Prompt enforces strict JSON output
- JSON is sanitized and parsed robustly
- Markdown fences removed automatically
- First JSON object extracted safely

This step transforms unstructured text into structured evaluation fields.

---

### Step 4D — Fit Scoring (Deterministic + LLM Rationale)

This is hybrid logic:

## Deterministic Score Calculation

Scoring logic:

| Criteria | Points |
|----------|--------|
| ≥ 2 years experience | +40 |
| Required certification present | +30 |
| Availability provided | +20 |
| High confidence (>70) | +10 |

Total possible score: 100

---

## Score Bands

- High ≥ 75
- Medium ≥ 40
- Low < 40

---

## Decision Logic (NextBestAction Step)

Based on score and missing fields:

- High + no missing → Interview Scheduled
- Medium → Missing Information / Clarification Required
- Low → Rejected

---

## Bedrock Rationale

LLM generates:

- Short explanation (max 4 sentences)
- Why score is what it is
- What is missing (if anything)

The LLM does NOT decide outcome.  
It only explains deterministic results.

This ensures:

- Governance
- Predictability
- No hallucinated decisions

---

### Step 4E — Next Best Action

Maps score + missing info into:

- Final Decision
- Status label for UI
- Reason codes (optional)

---

### Step 4F — Write Back to S3

Stores:

- Raw input
- Extracted structured fields
- Score
- Explanation
- Final decision
- Execution ARN

Ensures full auditability.

---

# LLM Integration Details

## Model

Anthropic Claude Sonnet 4  
Invoked via Bedrock Inference Profile:

```
arn:aws:bedrock:eu-west-3:<ACCOUNT_ID>:inference-profile/eu.anthropic.claude-sonnet-4-20250514-v1:0
```

Why inference profile?

- Required for newer Anthropic models
- Handles throughput routing
- Avoids on-demand invocation errors

---

## IAM Requirements

Lambda role must include:

- bedrock:InvokeModel
- bedrock:InvokeModelWithResponseStream
- AWS Marketplace permissions (first invocation only):
  - aws-marketplace:Subscribe
  - aws-marketplace:ViewSubscriptions
  - aws-marketplace:Unsubscribe

Once subscribed, invocation works account-wide.

---

# Key AWS Resources

- AWS Step Functions (STANDARD workflow)
- Lambda (multiple workers)
- Lambda Function URL (API endpoint)
- Amazon S3 (data persistence)
- Amazon Bedrock (Claude Sonnet 4)
- IAM roles (least privilege recommended)
- CloudWatch Logs (observability)

---

# Observability & Governance

- Every execution has unique ARN
- Full state history available in Step Functions
- Deterministic scoring ensures reproducibility
- LLM used only for:
  - Extraction
  - Explanation
- Final decision logic remains policy-driven

---

# Security Model

- No direct Bedrock access from frontend
- All LLM calls from backend Lambda
- IAM role scoped to required services
- No secrets stored in frontend
- S3 write-back for traceability

---

# UI Behavior

The UI:

- Shows animated step progression (cosmetic)
- Displays real backend results
- Supports:
  - Approved / Interview Scheduled
  - Missing Information
  - Rejected
- Displays:
  - Score
  - Confidence
  - Explanation
  - Execution ID

---

# Why This Is “Agentic”

This workflow demonstrates agentic principles:

- Orchestrated multi-step reasoning
- Task decomposition
- Structured intermediate memory
- Clear execution boundaries
- Hybrid AI + deterministic policy
- Next Best Action decisioning

It is not a single prompt.  
It is a stateful AI workflow.

---

# Scalability

The architecture supports:

- High concurrency (serverless)
- Adding new evaluation criteria
- Swapping models
- Replacing scoring logic
- Adding human-in-the-loop gates
- Integration with ATS systems

---

# Extensibility Ideas

- Add bias detection checks
- Add historical hiring data scoring
- Add human approval gate before scheduling
- Add feedback loop for model improvement
- Store embeddings for semantic search
- Add dashboard analytics

---

# What This Demo Proves

- You can combine Bedrock + deterministic rules safely
- You can build auditable AI hiring workflows
- You can use inference profiles correctly
- You can separate extraction from decision logic
- You can build a production-style orchestration layer

---

# Disclaimer

This is a demonstration system.  
It does not replace human hiring judgment.  
Real-world implementations must include:

- Legal review
- Bias auditing
- Equal opportunity compliance
- Data protection safeguards
- Human oversight

---

# Author

Designed as a demonstration of modern Agentic AI orchestration patterns  
Using AWS Serverless + Bedrock + deterministic governance logic.

---

If you found this project useful, feel free to fork and extend it.
