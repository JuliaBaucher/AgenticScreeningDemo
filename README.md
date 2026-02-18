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


```

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

```

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
# 30-Minute Oral Presentation Script (with live demo)

For this interview challenge, I designed and built a working prototype of an **agentic screening system** for high-volume hourly hiring. The goal is to show a production-style workflow that can handle **10,000+ applications per week across 50+ locations**, triage candidates quickly, remain **auditable and compliant (EEOC / fair hiring)**, and preserve a strong candidate experience — while acknowledging recruiters cannot manually review every application.

### 1) Problem framing and what “good” looks like
In this context, the system must do four things well:  
First, **speed** — reduce time-to-first-decision so qualified candidates move quickly through the funnel.  
Second, **consistency** — apply the same criteria every time to avoid subjective drift across locations.  
Third, **compliance and auditability** — every decision must be explainable with clear reason codes and an execution trace.  
Fourth, **candidate experience** — if information is missing, we should request clarification instead of rejecting prematurely.

### 2) What the agent has access to (and what I implemented in the demo)
The problem statement mentions access to resumes, job descriptions, historical hiring outcomes, screening answers, scheduling, and an ATS. In this prototype, I implemented the realistic structure of those integrations while keeping the demo lightweight:
- The system ingests **CV text**, **screening answers**, and the **job description**.
- It runs **structured extraction** using an LLM to transform unstructured text into a consistent schema.
- It performs **deterministic fit scoring** and a **policy-based next best action** decision.
- It includes **stub integrations** for ATS update, candidate communications, and scheduling — returning realistic contract-compliant responses without calling external systems.
- Finally, it writes a complete **audit package** to S3 so the output is durable and reviewable.

### 3) System design and high-level architecture
At a high level, the system has three layers:
1) A lightweight **web UI** where a user pastes the CV, answers, and job description, and starts a run.  
2) A serverless orchestration backend based on **AWS Step Functions**, with modular **Lambda workers** per step.  
3) An observability and audit layer using **Step Functions execution history**, **CloudWatch logs/metrics**, and **S3 write-back** for final outcomes.

The key design decision is that this is not a single prompt. It’s a **stateful multi-step workflow** that creates structured intermediate artifacts, applies deterministic policies, and logs everything for governance.

### 4) Walkthrough of the workflow steps (what happens after I click “Start”)
When I submit an application, Step Functions executes a sequence of worker steps:

- **Load Job Context:** prepares job rules and a requirements schema derived from the job description, with versioning and audit flags.  
- **Ingest Application Event:** normalizes the payload into a consistent `application` object for downstream steps.  
- **Normalize & Dedupe:** canonicalizes the text and generates a deterministic dedupe key, which is how you would avoid reprocessing duplicates at scale.  
- **Structured Extraction (LLM):** uses Bedrock / Claude to extract a strict JSON schema like years of experience, certification presence, skills, availability, and a confidence estimate (based on user inputs understanding). Temperature is set close to 0 for stable outputs.  
- **Fit Scoring (Deterministic):** assigning points for experience, required certification, availability, and a confidence factor. Final application status is determined by rule-based thresholds, accompanied by an LLM-generated summary explaining the outcome. This is important: the model does not decide the outcome — scoring is deterministic and auditable.  
- **Next Best Action:** converts the score + missing info into one of three operational outcomes: interview scheduled, missing information request, or rejection. If rejected, it returns a structured **rejection reason code**.  
- **ATS / Comms / Scheduling (Simulated):** these are stub steps that keep the contracts realistic without external dependencies.  
- **Write Back to S3:** stores all outcomes (decision, score, reasons, execution ARN) in the application package for auditability.
- **Output Metrics (execution and funnel)** are computed in real time from Step Functions execution history and S3-stored decision data

### 5) Why this approach is compliant and production-oriented
A major risk in hiring systems is “black box” decisioning. In this prototype:
- The **LLM is constrained to extraction and explanation**, not eligibility decisions.  
- The **decision logic is deterministic** and mapped to explicit requirement checks.  
- Every rejection includes a **reason code** and a human-readable explanation aligned to the rubric.  
- Step Functions provides an immutable execution trace, and S3 stores the final package to support audits and reviews.

### 6) Candidate experience design
Instead of rejecting on missing information, the policy explicitly supports “Missing Information / Clarification Required.” This improves conversion, reduces false negatives, and matches how recruiters work in reality. Even in demo mode, communications and scheduling are modeled so we preserve the end-to-end funnel logic.

### 7) Prompting approach and iteration
I used prompting in two places:
- **Structured extraction prompt:** designed to output strictly valid JSON with a defined schema so it’s machine-consumable and safe.  
- **Rationale generation prompt:** produces a short explanation that references the deterministic scoring outcome and missing items, capped to a few sentences.

During iteration, the key improvements were enforcing strict JSON output, adding robust parsing, and keeping the LLM out of the decision boundary so the system remains predictable.

### 8) Demo (screen share)
Now I’ll show the prototype. I will:
1) Paste a sample job description and candidate application.  
2) Click “Start Demo” to launch an execution.  
3) Show the live status while the workflow runs.  
4) When completed, review the displayed output: execution ARN, structured extraction, score band, decision, and rejection code if applicable.  
5) Optionally open Step Functions to show the execution graph and the intermediate state transitions for auditability.

### 9) Success metrics (what we’d measure in production)
In production, the key success metrics would be:
- **Time to first decision** and end-to-end screening latency  
- **Throughput** (applications processed per hour/day)  
- **Success/failure rate** and error categories  
- **Conversion rates**: invite-to-schedule, schedule completion  
- **Human review rate** (bounded)  
- **Fairness / compliance monitoring**: distribution of outcomes across locations and cohorts, plus audit sampling

In this demo, I included simulated metrics to show how the KPI layer plugs in without building a full analytics pipeline.

### 10) Roadmap: what’s built vs what’s next
What’s built today is a complete end-to-end workflow with durable orchestration, deterministic decisioning, compliance-friendly outputs, and realistic integration stubs.  
Next steps for a production rollout would be:
- Replace stubs with real ATS, comms, and scheduling integrations  
- Add historical hiring outcome signals and calibration of scoring thresholds  
- Add human-in-the-loop gates for edge cases and low-confidence extractions  
- Add automated fairness and bias monitoring dashboards  
- Add scalable ingestion (S3/SQS) for true high-volume asynchronous processing

### Closing
To summarize: this prototype demonstrates a practical agentic screening architecture that is fast, modular, auditable, and designed for high-volume hourly hiring. The key is the separation of concerns: **LLM for semantic extraction**, **deterministic scoring and policy for decisions**, and **Step Functions for traceability and governance**. I’m happy to dive deeper into any component or walk through specific execution traces.

# Presentation Script 
For this interview challenge, I designed and built a working prototype of an agentic screening system for high-volume hourly hiring. The goal is to demonstrate a production-style workflow that can handle 10,000+ applications per week across 50+ locations, identify qualified candidates and move them through the funnel, comply with EEOC and fair hiring regulations, and preserve a strong candidate experience — while acknowledging that recruiters cannot manually review every application.

Problem
The challenge is to design an architecture that handles volume, speed, compliance, and user experience (for both recruiters and candidates).

How it works:
The user enters the job description, candidate CV, and answers to interview questions in the UI, and the system evaluates the application and provides a final decision: approved for interview, missing information, or rejected with a rejection reason. To facilitate the demo, there are predefined examples for each scenario. The system also returns execution and business metrics.

I selected a serverless architecture with a lightweight front end hosted on GitHub and an AWS-based backend with storage on S3, orchestration through Step Functions, and LLM integration via Bedrock.

Load Job Context: prepares job rules and a requirements schema derived from the job description, including versioning and audit flags.
Ingest Application Event: normalizes the payload into a consistent application object for downstream steps.
Normalize & Dedupe: canonicalizes the text and generates a deterministic dedupe key, which is how duplicates would be avoided at scale.
Structured Extraction (LLM): uses Bedrock / Claude to extract a strict JSON schema including years of experience, certification presence, skills, availability, and a confidence estimate (based on the model’s understanding of user inputs). Temperature is set close to 0 for stable outputs.
Fit Scoring (Deterministic): assigns points for experience, required certification, availability, and a confidence factor. The final application status is determined by rule-based thresholds, accompanied by an LLM-generated summary explaining the outcome. This is important: the model does not decide the outcome — scoring is deterministic and auditable.
Next Best Action: converts the score and missing information into one of three operational outcomes: interview scheduled, missing information request, or rejection. If rejected, it returns a structured rejection reason code.
ATS / Comms / Scheduling (Simulated): stub steps that keep contracts realistic without external dependencies.
Write Back to S3: stores all outcomes (decision, score, reasons, execution ARN) in the application package for auditability.
Output Metrics: execution and funnel metrics are computed in real time from Step Functions execution history and S3-stored decision data.

This approach is compliant and production-oriented. It handles high volume thanks to AWS serverless architecture and supports multiple languages through LLM-based extraction. It provides deterministic evaluation of candidates based on scoring, and any rejection is accompanied by a structured reason code to comply with EEOC standards. The architecture anticipates integration with external systems for candidate communication, ATS updates, and interview scheduling.

It is aligned with orchestration principles: execution authority through predefined steps; clear boundaries for LLM usage (the LLM is constrained to specific steps); durable memory and state management using S3 and Step Functions; fully traceable, controlled autonomy with the final decision available for human oversight; and full measurability with execution and business metrics available in the UI and in CloudWatch.

Next steps: integrate external systems (scheduling, candidate communication, ATS), support voice input, implement richer business logic, add SQS for burst handling, introduce a database for configuration and persistence, and incorporate analytics tools.


