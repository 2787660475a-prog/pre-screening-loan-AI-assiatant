# First-Loan AI Pre-Review Assistant

An engineering-oriented MVP for first-time small and micro business loan applications. The system covers customer and application management, document uploads with OCR, CSV transaction imports, metric calculation, a rule DSL, AI business analysis, pre-review reports, a human approval gate, LMS archiving, in-app notifications, and audit logs. AI is used only to assist analysis; every acceptance action must be completed by a human operator.

## Current Highlights

- Six-stage controlled workflow: document parsing → rule screening → business analysis → report generation → human gate → archiving and notifications.
- Explainable transaction analysis: monthly inflows and outflows, net cash flow, counterparty concentration, round-amount and related-party transactions, and suspected bridge-funding or circular transfers.
- Clear separation between rules and AI: deterministic rule findings are presented first, followed by an evidence-based AI business profile.
- Retrieval-augmented report generation: a versioned local policy knowledge base supplies traceable citations, excerpts, relevance scores, and verification questions.
- Two-step confirmation at the human gate: Confirm Acceptance, Edit and Accept, and Return for Additional Documents all require a human comment and are recorded in the audit log.
- Chinese-language pre-review PDF: structured sections, risk disclaimer, audit notice, footer, and page numbers, with Noto CJK fonts installed in the container.
- Sensitive information such as phone numbers is masked by default, and unnecessary sensitive fields are excluded from model analysis.

## Quick Start

```bash
cp .env.example .env
docker compose up -d --build
docker compose exec api python -m app.scripts.seed
```

- Web application: http://localhost:3000
- API documentation: http://localhost:8000/docs
- MinIO console: http://localhost:9001

All demo accounts use the password `Demo123!`:

- `rm_demo` — Relationship Manager
- `reviewer_demo` — Reviewer
- `admin_demo` — Administrator

The seed data includes three scenarios: normal acceptance, enhanced verification, and return for additional documents.

To run without Docker, install the dependencies under `apps/api`, then run:

```bash
python -m app.scripts.seed
uvicorn app.main:app --reload
```

Under `apps/web`, run:

```bash
npm ci
npm run dev
```

The local API uses SQLite by default.

## Demo Workflow

1. Sign in and open an existing demo application to review its risks, recommendation, and report.
2. Create a new application and upload a business license, identity document, and bank statements.
3. Use `sample-transactions.csv` from the repository root to call `POST /cases/{id}/transactions/import` and import transaction data.
4. Submit the application for pre-review, then inspect the rules, metrics, AI business profile, and report.
5. Complete the human gate using Confirm Acceptance, Edit and Accept, or Return for Additional Documents.

## Architecture and Project Structure

```text
apps/web    Next.js 15 administration portal
apps/api    FastAPI, SQLAlchemy, rules and metrics, adapters, and tests
apps/api/app/knowledge    Versioned local RAG knowledge base
docs        Design documentation
```

Docker Compose orchestrates the Web application, API, PostgreSQL, Redis, MinIO, and Mock Worker. Adapters are located under `apps/api/app/adapters`. Mock OCR, Mock LLM, and Mock LMS implementations are enabled by default, allowing the complete workflow to run without external API keys.

## Retrieval-Augmented Report Generation

The pre-review workflow retrieves relevant guidance after deterministic rules have run and before AI analysis and report assembly:

```text
Customer, application, materials, metrics, and rule findings
→ Build a retrieval query
→ Retrieve policy chunks from the local knowledge base
→ Add citations and verification guidance to AI analysis
→ Persist the RAG context and audit event
→ Generate the structured report and PDF with source references
→ Human review gate
```

The MVP uses a deterministic BM25-like sparse retriever implemented in `apps/api/app/services/rag.py`. Its knowledge chunks are stored in `apps/api/app/knowledge/credit_policy.json`. This keeps the complete workflow operational without an embedding API or vector database while preserving an adapter-shaped result that can later be replaced by pgvector, Milvus, Elasticsearch, or another retrieval service.

Each report records:

- Retrieval mode and knowledge-base version
- Retrieval query
- Citation IDs and relevance scores
- Source document, section, and excerpt
- Knowledge-grounded verification questions
- A dedicated retrieval audit event

RAG provides policy explanations and verification leads only. It cannot override blocking rules or make the final acceptance decision.

Configuration:

```env
RAG_ENABLED=true
RAG_TOP_K=4
RAG_KNOWLEDGE_PATH=
```

Leave `RAG_KNOWLEDGE_PATH` empty to use the bundled demo knowledge base.

## Testing

```bash
cd apps/api && python -m pytest
cd apps/web && npm run typecheck && npm run build
```

The backend test suite includes a complete API workflow:

```text
Sign in
→ Create a customer and application
→ Upload the minimum required documents
→ Import six months of transaction data
→ Run the pre-review workflow
→ Complete the human gate
→ Archive to the LMS
→ Inspect the audit timeline
→ Download the Chinese-language PDF
```

Core security controls include JWT authentication, server-side RBAC, a 20 MB upload limit, a MIME-type allowlist, SHA-256 hashing, path isolation, structured model-output validation, trusted local RAG sources, and rule execution without `eval`. Demo thresholds, knowledge content, and scores do not represent actual bank policies.

