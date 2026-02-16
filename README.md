
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

It is not a single prompt — it is a stateful AI workflow.

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
