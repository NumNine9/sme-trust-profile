# Architecture & Design Patterns

This section describes the architecture and design patterns selected for the SME Verification and Opportunity Matching platform. 

---

## 1. Architectural Style: Modular Monolith + Async Worker

### Rationale

The platform has clearly separable domains (intake, extraction, validation, readiness, consent, matching, feedback) but a **single delivery team**, **modest expected traffic**, and a **hard timeline**. A microservices architecture would introduce operational overhead—service discovery, network failure modes, distributed tracing, multiple deploy pipelines—without solving any problem the project currently has.

A **modular monolith** keeps one deployable codebase with strict internal module boundaries, so any module can later be extracted into a service **if and only if** real usage proves the need.

The one non-negotiable split is between **synchronous request handling** and **long-running document processing**. OCR, extraction, validation, and embedding take 10–90 seconds and must never block an HTTP request. This is handled by an **async worker process** running the same codebase.

### Architecture Diagram

```text
┌─────────────────┐
│   React PWA     │  SME / Buyer / YES views
│  (mobile-first) │  device-side compression, resumable uploads
└────────┬────────┘
         │ HTTPS
┌────────▼─────────────────────────────────────────────────┐
│                  FastAPI API (sync)                      │
│  ┌──────────┬──────────┬──────────┬──────────┬────────┐  │
│  │  Intake  │ Profile  │ Consent  │ Matching │Feedback│  │
│  └──────────┴──────────┴──────────┴──────────┴────────┘  │
└────┬───────────────────────────────────────┬─────────────┘
     │ enqueue                               │ read/write
┌────▼──────────────┐              ┌─────────▼──────────────┐
│  Job Queue        │              │  PostgreSQL            │
│  (Postgres-backed │              │  + pgvector            │
│   or Redis)       │              │  row-level security    │
└────┬──────────────┘              │  append-only audit     │
     │ dequeue                     └─────────▲──────────────┘
┌────▼──────────────────────────────────────┐│
│              Worker (async)                ││
│  ┌─────────┬─────────┬─────────┬────────┐ ││
│  │Classify │  OCR    │ Extract │Validate│ ││
│  ├─────────┼─────────┼─────────┼────────┤ ││
│  │Financial│Readiness│ Embed   │ Flag   │ ││
│  └─────────┴─────────┴─────────┴────────┘ ││
└────┬───────────────────────────┬──────────┘│
     │                           │            │
┌────▼──────────────┐   ┌────────▼─────────┐ │
│  AI Gateway       │   │  Azure Blob      │ │
│  (Adapter+Strategy│   │  Storage         │ │
│   + fallback)     │   │  (source docs,   │ │
│  ┌──────────────┐ │   │   short-lived    │ │
│  │Doc Intell.   │ │   │   SAS links)     │ │
│  │Gemini/Mistral│ │   └──────────────────┘ │
│  │Voyage/TEI-BGE│ │                        │
│  └──────────────┘ │                        │
└───────────────────┘                        │
                                             │
                    ┌────────────────────────┘
                    │
              ┌─────▼──────────────┐
              │  Human Review Queue│
              │  (Postgres table + │
              │   reviewer view)   │
              └────────────────────┘
```

### Component Responsibilities

| Component | Responsibility |
|---|---|
| **React PWA** | SME, buyer, YES views; mobile-first uploads; consent UI; explained shortlists |
| **FastAPI API (sync)** | Auth, profile reads, consent grants, shortlist views, reviewer actions |
| **Worker (async)** | OCR, classification, extraction, validation, financial calc, embedding, readiness generation, flagging |
| **AI Gateway** | Single internal interface for all AI calls; routing; fallback; redaction; audit |
| **PostgreSQL + pgvector** | Profiles, policies, decisions, consent, audit, embeddings |
| **Azure Blob Storage** | Source documents, encrypted at rest, short-lived links |
| **Human Review Queue** | Low-confidence and flagged items; reviewer confirms verification |
| **Job Queue** | Decouples API from long-running processing |

### Deployment Topology (MVP / Pilot)

```text
Azure South Africa North
├── Container App: FastAPI API          (1–2 instances)
├── Container App: Worker               (1 instance, scale on queue depth)
├── Container App: TEI (bge-large, CPU) (optional, self-hosted embeddings)
├── Azure Database for PostgreSQL Flexible Server (+ pgvector)
├── Azure Blob Storage
└── Azure Key Vault  (provider keys, DB credentials)
```

- Single region satisfies data-residency requirements.
- Secrets injected from Key Vault as environment variables—never in code or frontend.
- One resource group for the pilot; add staging only when a second organisation is onboarded.
- Self-hosted TEI on CPU is sufficient for MVP; migrate to GPU or hosted embedding API only when matching latency becomes user-visible.

---

## 2. Design Patterns

The patterns below are selected because each one directly addresses a stated requirement. Patterns not listed (CQRS, event sourcing, saga orchestration, service mesh) were deliberately rejected as unnecessary complexity at this stage.

| Pattern | Where applied | Why |
|---|---|---|
| **Adapter** | AI Gateway: one adapter per provider (Azure Document Intelligence, Gemini, Mistral, Voyage, TEI/BGE) | Enables provider swap and fallback without touching business logic |
| **Strategy** | Model routing: choose model per task, configurable per environment | Encapsulates "which model for which task"; supports primary + fallback |
| **Pipeline / Chain of Responsibility** | Document processing: classify → OCR → extract → validate → calculate → embed → readiness → flag | Each step is small, testable, retryable; failures isolated |
| **Repository** | Data access for profiles, consent, decisions, audit | Keeps SQL out of business logic; enforces row-level security and consistent audit writes |
| **Command** | Human review actions: approve, correct, verify, dispute, resolve | Every reviewer action becomes a logged, replayable command |
| **Policy / Configuration as Data** | Opportunity requirements, AirStudent credit policy, funder rules, buyer criteria | Corporate requirements configurable per organisation without code changes |
| **Transactional Outbox (lightweight)** | Writing audit and consent records in the same transaction as a decision | Guarantees no decision is recorded without its audit entry; no message broker needed |
| **Feature Flags (simple)** | Toggle AI provider per task, per environment | Demo with one provider, pilot with another, no branching code |
| **Explainable Recommendation** | Readiness assessment, match reasons, improvement actions | AI output cites profile fields only; recommendations traced to evidence |
| **Redaction Gateway** | Pre-processing before every model call | Personal identifiers masked before model calls where extraction allows |

### AI and Deterministic Logic Separation

A core architectural principle is that **AI and deterministic logic never overlap in responsibility**:

- **AI** performs: OCR, classification, structured extraction, semantic matching, plain-language explanation drafting.
- **Deterministic rules** perform: format validation, expiry checks, cross-document consistency, duplicate detection, financial calculations, eligibility screening.
- **Humans** perform: verification of flagged inconsistencies, final shortlist approval, final credit decisions, dispute resolution.

The AI Gateway is a **backend-only module**. No AI provider is ever exposed to the frontend or the user directly.

---

## 3. Why This Architecture Fits the Requirements

| Requirement group | How the architecture satisfies it |
|---|---|
| **SME document intake** | PWA handles compression and resumable uploads; API enqueues documents immediately so the user is never blocked |
| **AI extraction, validation, review** | Pipeline pattern isolates each step; low-confidence output routes to human review queue via a logged command |
| **Deterministic financial capacity** | Runs as a versioned, unit-tested Python module inside the worker—no LLM touches the arithmetic |
| **Explained readiness and SME guidance** | Readiness generation is a discrete pipeline step whose output is validated against the profile schema before reaching the UI |
| **Consent, reuse, data governance** | Repository pattern enforces row-level security; consent checks run in API before any profile read; audit is append-only |
| **Opportunity intake and integration** | CSV, web form, and upload ingestion share the same policy ingestion module; phase-2 API and email ingestion plug in without schema changes |
| **Matching and human decision-making** | Policy-as-data holds each organisation's rules; hard rules run before semantic matching; recommendations generated but final decisions require an explicit human command |
| **Feedback, disputes, YES insights** | Command pattern logs every feedback and dispute action; YES dashboard reads aggregated views only, with minimum group sizes enforced in the repository layer |
| **Future repayment signal** | Modelled as an additional evidence source flowing through the same profile and consent layers—no architectural change required |
| **Hackathon demonstration** | Full pipeline runs on a single Container App + worker + Postgres + Blob Storage, deployable in a day and demoable in 90 seconds |
| **Mobile-first multi-view product** | One PWA with role-based routes; one backend serving all three views |
| **AI/rules separation and defensible calculations** | AI Gateway isolated from deterministic rule engine and financial module; provider adapters make AI calls replaceable and auditable |
| **Append-only audit logging** | Repository + Command + Outbox patterns ensure every consent, access, review, and decision writes an immutable record |
| **Privacy, consent, security** | Row-level security, redaction gateway, short-lived blob links, Key Vault secret management |
| **Fairness and documentation monitoring** | Documentation level stored as a first-class profile attribute; readiness and shortlist outcomes compared in aggregate without exposing individual records |
| **Configurability, safe scale, multi-partner reuse** | Policy-as-data and organisation-scoped repositories allow a second funder or credit partner to onboard without code changes |

---

## 4. What Was Deliberately Excluded

| Excluded option | Reason |
|---|---|
| Microservices | Single team, modest traffic, tight timeline—operational overhead outweighs benefit |
| Kafka / RabbitMQ / event streaming | Volumes do not require it; Postgres-backed or Redis queues suffice for hackathon and pilot |
| Separate vector database (Pinecone, Weaviate) | pgvector in the existing Postgres meets scale and keeps one data store |
| CQRS, event sourcing, saga orchestration | No requirement justifies the complexity; Command pattern plus append-only audit achieves the same traceability |
| Serverless functions for document processing | Cold starts harm the 90-second demo target; local queue + worker is easier to reason about |
| Dedicated review service | A Postgres table plus reviewer view inside the same app is sufficient |
| GPU-hosted BGE from day one | CPU-hosted TEI or a hosted embedding API is adequate for MVP; migrate when latency or cost demands |

---

## 5. Summary

The platform is built as a **modular monolith with an async worker**, using a small set of patterns—**Adapter, Strategy, Pipeline, Repository, Command, Policy-as-data, lightweight Outbox, Feature Flags, Explainable Recommendation, and Redaction Gateway**—each mapped to a specific functional or non-functional requirement.

This gives the simplest architecture that still satisfies explainability, consent, audit, provider flexibility, and human oversight—and leaves a clean, low-cost path to extract services or swap AI providers only when real usage proves the need.
