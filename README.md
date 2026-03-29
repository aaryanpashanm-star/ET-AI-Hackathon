# ClinicalBridge — AI Healthcare Compliance Agent

> **ET GenAI Hackathon 2026 · Problem Statement 5**  
> Domain-Specialized AI Agent with Compliance Guardrails (Healthcare)

[![Python](https://img.shields.io/badge/Python-3.11+-blue)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-green)](https://fastapi.tiangolo.com)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.1-orange)](https://langchain-ai.github.io/langgraph/)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

---

## What It Does

ClinicalBridge is a **multi-agent AI system** that automates medical prior authorization and clinical coding workflows for Indian healthcare providers. It navigates ICD-10/CPT rule sets, validates against payer-specific policies, and produces a **fully auditable decision trail** — all with compliance guardrails built in from the ground up.

### The Problem
- Prior authorization takes 1–3 days per case, causing treatment delays
- Medical coding errors cost Indian hospitals ₹2,400 Cr+ annually in claim rejections
- 68% of denied claims are never resubmitted — pure revenue leakage
- Health workers and billers lack real-time access to evolving payer guidelines

### The Solution
A 4-agent pipeline that handles the full prior auth + coding workflow:

```
Patient Case Input → [Intake Agent] → [Coding Agent] → [Auth Agent] → [Audit Agent] → Decision + Audit Trail
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    ClinicalBridge Agent System               │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │  Intake     │───▶│  Coding     │───▶│   Auth      │    │
│  │  Agent      │    │  Agent      │    │   Agent     │    │
│  │             │    │ ICD-10/CPT  │    │  Payer RAG  │    │
│  └─────────────┘    └─────────────┘    └──────┬──────┘    │
│                                               │            │
│                          ┌────────────────────▼──────┐    │
│                          │      Audit Agent           │    │
│                          │  Immutable Decision Log    │    │
│                          └───────────────────────────┘    │
│                                                             │
│  RAG Layer: ChromaDB  │  LLM: Claude Sonnet  │  FastAPI   │
└─────────────────────────────────────────────────────────────┘
```

---

## Agents

| Agent | Role | Tools Used |
|-------|------|-----------|
| **Intake Agent** | Parses clinical notes, extracts symptoms, meds, procedures | NER, ICD-10 lookup |
| **Coding Agent** | Assigns ICD-10 diagnosis + CPT procedure codes with confidence | RAG over ICD-10/CPT DB |
| **Auth Agent** | Checks payer-specific prior auth rules, flags requirements | Payer policy RAG, rule engine |
| **Audit Agent** | Logs every decision, confidence score, and source citation | Immutable audit store |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Agent Orchestration | LangGraph (stateful multi-agent graph) |
| LLM | Claude Sonnet 4 (via Anthropic API) |
| Vector Store | ChromaDB |
| RAG Knowledge Base | ICD-10-CM, CPT codes, MOHFW guidelines, sample payer policies |
| API Layer | FastAPI + Pydantic |
| Frontend Demo | React + Tailwind |
| Audit Store | SQLite (append-only, hash-chained) |
| Language | Python 3.11+ |

---

## Quickstart

### Prerequisites
- Python 3.11+
- Node.js 18+ (for frontend)
- Anthropic API key

### Installation

```bash
# 1. Clone the repo
git clone https://github.com/aaryanpasha/clinicalbridge.git
cd clinicalbridge

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set environment variables
cp .env.example .env
# Add your ANTHROPIC_API_KEY to .env

# 5. Initialize the vector store (loads ICD-10 + CPT + payer policy data)
python scripts/init_rag.py

# 6. Start the API server
uvicorn src.api.main:app --reload --port 8000

# 7. (Optional) Start the frontend demo
cd frontend && npm install && npm run dev
```

### Run a Demo Case

```bash
python scripts/demo_case.py
```

This runs a sample prior auth request through the full pipeline and prints the decision + audit trail.

---

## API Reference

### `POST /api/v1/authorize`

Submit a case for prior authorization.

**Request:**
```json
{
  "patient_id": "P-1234",
  "clinical_notes": "Patient presents with chest pain, shortness of breath...",
  "proposed_procedure": "Coronary angiography",
  "payer": "PMJAY",
  "provider_id": "HOSP-456"
}
```

**Response:**
```json
{
  "case_id": "CB-20260329-001",
  "decision": "APPROVED_WITH_CONDITIONS",
  "icd10_codes": ["I25.10", "R07.9"],
  "cpt_codes": ["93454"],
  "confidence": 0.91,
  "conditions": ["Echocardiogram report required within 48h"],
  "audit_trail_id": "AUD-20260329-001",
  "processing_time_ms": 2340
}
```

### `GET /api/v1/audit/{audit_trail_id}`

Retrieve the full immutable audit trail for a case decision.

### `GET /api/v1/health`

Health check endpoint.

---

## Guardrails & Compliance

ClinicalBridge is built with compliance-first architecture:

- **Confidence Gating**: Any coding or auth decision below 75% confidence is automatically escalated to a human reviewer — never auto-approved
- **RAG-Constrained Outputs**: LLM responses are strictly grounded in ICD-10, CPT, and payer policy documents — hallucination surface is minimized
- **Immutable Audit Trail**: Every agent decision is hash-chained and timestamped — tamper-evident by design
- **DPDP Act 2023 Compliance**: No patient PII is stored in vector indices; all PHI is encrypted at rest (AES-256)
- **Human-in-the-Loop**: All decisions above a configurable financial threshold require physician/admin sign-off
- **Disclaimer Enforcement**: System outputs always carry a "support tool, not clinical prescription" marker

---

## Evaluation Metrics (Target)

| Metric | Target | Rationale |
|--------|--------|-----------|
| Coding Accuracy | ≥ 92% | vs. certified medical coder gold labels |
| Auth Decision Accuracy | ≥ 88% | vs. manual payer policy review |
| Hallucination Rate | < 2% | GPT-4 factuality judge on 200 outputs |
| End-to-end Latency | < 5 sec | Full pipeline on standard cloud instance |
| Audit Coverage | 100% | Every decision logged, no exceptions |
| Escalation Recall | 100% | Every low-confidence case reaches a human |

---

## Project Structure

```
clinicalbridge/
├── src/
│   ├── agents/
│   │   ├── intake_agent.py       # Clinical note parsing + NER
│   │   ├── coding_agent.py       # ICD-10/CPT assignment
│   │   ├── auth_agent.py         # Prior authorization logic
│   │   └── audit_agent.py        # Immutable audit logging
│   ├── tools/
│   │   ├── icd10_lookup.py       # ICD-10-CM code lookup tool
│   │   ├── cpt_lookup.py         # CPT code lookup tool
│   │   ├── payer_policy.py       # Payer-specific rule checker
│   │   └── confidence_gate.py    # Confidence threshold enforcer
│   ├── rag/
│   │   ├── vector_store.py       # ChromaDB wrapper
│   │   ├── ingestion.py          # Document ingestion pipeline
│   │   └── retriever.py          # RAG retrieval logic
│   ├── api/
│   │   ├── main.py               # FastAPI app entrypoint
│   │   ├── routes.py             # API route definitions
│   │   └── schemas.py            # Pydantic request/response models
│   └── utils/
│       ├── audit_store.py        # Hash-chained audit log
│       ├── crypto.py             # AES-256 encryption helpers
│       └── config.py             # Config + env management
├── data/
│   ├── icd10/                    # ICD-10-CM code files (open access)
│   ├── cpt/                      # CPT code descriptions
│   └── payer_policies/           # Sample payer policy documents
├── tests/
│   ├── test_agents.py
│   ├── test_api.py
│   └── test_audit.py
├── scripts/
│   ├── init_rag.py               # One-time RAG initialization
│   └── demo_case.py              # Demo script
├── frontend/                     # React demo UI
├── docs/
│   ├── architecture.md
│   └── impact_model.md
├── .env.example
├── requirements.txt
└── README.md
```

---

## Datasets Used

All datasets are open access:

| Dataset | Source | Use |
|---------|--------|-----|
| ICD-10-CM 2026 | CMS.gov | Diagnosis coding RAG |
| CPT Code Descriptions | AMA (open subset) | Procedure coding RAG |
| MOHFW Clinical Protocols | mohfw.gov.in | Indian clinical guideline RAG |
| PMJAY Package List | pmjay.gov.in | Payer policy rules |
| MedQA (USMLE) | HuggingFace | Agent fine-tuning eval |

---

## Hackathon Context

Built for **ET GenAI Hackathon 2026**, Problem Statement 5: *Domain-Specialized AI Agents with Compliance Guardrails*.

**By Aaryan Pasha**  
AI Engineer | Ex-EY Tech Risk & Compliance Consultant

---

## License

MIT License — see [LICENSE](LICENSE)
