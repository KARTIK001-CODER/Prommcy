# EvalForge — 10-Phase Engineering Development Roadmap

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Execution (Phase 0 Complete) |
| **Phases** | Phase 0 through Phase 10 |
| **Governance** | Strict Phase-Gate Approval (No premature phase initiation) |

---

## 1. Executive Roadmap Overview

EvalForge is developed through a disciplined, ten-phase evolutionary engineering methodology. Each phase produces a fully tested, functional, and self-contained increment of capability.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EvalForge Phase Horizon                           │
├─────────┬─────────────────────────────────────────────────┬─────────────────┤
│ Phase   │ Description                                     │ Target State    │
├─────────┼─────────────────────────────────────────────────┼─────────────────┤
│ **0**   │ Product Definition, Reference Analysis & Arch   │ COMPLETED       │
│ **1**   │ Backend Foundation, PostgreSQL, Auth & Projects │ Sprint 1        │
│ **2**   │ LLM Provider Abstraction & Adapters             │ Sprint 2        │
│ **3**   │ Core Evaluation Engine, Datasets & Assertions   │ Sprint 3        │
│ **4**   │ Model Benchmarking Suite & Cost Accounting      │ Sprint 4        │
│ **5**   │ RAG Evaluation Engine & pgvector Integration    │ Sprint 5        │
│ **6**   │ AI Red Teaming & Vulnerability Detection        │ Sprint 6        │
│ **7**   │ Evaluation History, Analytics & Regression      │ Sprint 7        │
│ **8**   │ Frontend Web Dashboard & Interactive Visualizer │ Sprint 8        │
│ **9**   │ Python CLI & GitHub Actions Integration         │ Sprint 9        │
│ **10**  │ Zero-Cost Cloud Deployment & Production Launch  │ Sprint 10       │
└─────────┴─────────────────────────────────────────────────┴─────────────────┘
```

---

## 2. Detailed Phase Specifications

---

### Phase 0: Product Definition, Reference Analysis & Technical Architecture
- **Objectives**: Establish foundational product vision, deconstruct Promptfoo reference architecture, define database schema, API contracts, security model, and zero-cost deployment plan.
- **Key Deliverables**: 
  - Complete architectural documentation suite (`PRODUCT_REQUIREMENTS.md`, `REFERENCE_ANALYSIS.md`, `SYSTEM_ARCHITECTURE.md`, `DATABASE_DESIGN.md`, `API_DESIGN.md`, `TECH_STACK.md`, `DEPLOYMENT_STRATEGY.md`, `SECURITY_STRATEGY.md`, `TESTING_STRATEGY.md`, `PHASE_ROADMAP.md`, `PHASE_1_PLAN.md`).
  - Core Architectural Decision Records (ADR 001 - 005).
- **Dependencies**: None.
- **Technical Challenges**: Balancing enterprise capabilities (multi-tenancy, vector search, durability) with strict zero-cost hosting constraints (512 MB RAM, 1 GB storage).
- **Testing Requirements**: Documentation consistency, schema validation, and diagram syntax verification.
- **Acceptance Criteria**: All 11 architectural documents created, internally consistent, approved by stakeholders, with zero application code written.

---

### Phase 1: Backend Foundation, PostgreSQL, Authentication and Project Management
- **Objectives**: Initialize backend application repository, configure SQLAlchemy 2.0 async engine, implement Alembic migrations, build Argon2id authentication, JWT issuance, multi-tenant workspace isolation, and project management.
- **Key Deliverables**:
  - Modular FastAPI application skeleton (`evalforge.core`, `evalforge.modules.auth`, `evalforge.modules.projects`).
  - Database migrations for `workspaces`, `users`, `workspace_memberships`, `projects`, `provider_configs`, and `audit_logs`.
  - AES-256-GCM encryption vault for provider API keys.
  - REST endpoints for registration, login, token refresh, workspaces, projects, and encrypted provider configs.
- **Dependencies**: Phase 0 approval.
- **Technical Challenges**: Establishing seamless async SQLAlchemy 2.0 session handling, zero-leakage AES-256-GCM encryption with per-tenant HKDF keys, and multi-tenant scoping middleware.
- **Testing Requirements**: 100% unit test coverage on crypto primitives and JWT logic; integration tests for user registration, login, and workspace tenant isolation.
- **Acceptance Criteria**: Clean `pytest` pass with $\ge 90\%$ code coverage; 0 Ruff violations; 0 MyPy errors; migration up/down verified on Neon PostgreSQL.

---

### Phase 2: LLM Provider Abstraction and Integrations
- **Objectives**: Implement the unified `BaseProvider` abstract interface and build resilient adapters for Google Gemini, Groq, Ollama, Hugging Face, and OpenAI-compatible endpoints.
- **Key Deliverables**:
  - `BaseProvider` ABC with async `generate()` and `generate_stream()` methods.
  - `GeminiProvider`, `GroqProvider`, `OllamaProvider`, `HuggingFaceProvider`, and `GenericOpenAIProvider` implementations.
  - Provider Factory with dynamic configuration loading and key decryption.
  - Bounded concurrency rate limiter (token bucket per provider).
  - Dynamic token pricing lookup table for cost estimation.
- **Dependencies**: Phase 1 (Provider credentials model & AES-256 vault).
- **Technical Challenges**: Normalizing heterogeneous API response formats, token usage schemas, and error responses across Google, Groq, and Ollama into a unified `ProviderResponseDTO`.
- **Testing Requirements**: Comprehensive mock testing via `respx` covering successful completions, rate limits (HTTP 429), timeouts, and malformed JSON responses.
- **Acceptance Criteria**: All 5 provider adapters tested offline with 100% mock coverage; normalized latency and token counts verified.

---

### Phase 3: Evaluation Engine, Datasets and Scoring
- **Objectives**: Build the core evaluation engine capable of generating Cartesian matrices ($P \times T \times M$), executing test cases asynchronously via the PostgreSQL queue, and evaluating deterministic and model-assisted assertions.
- **Key Deliverables**:
  - Cartesian matrix generator expanding prompt templates, test variables, and model targets.
  - Variable substitution parser supporting `{{var}}` templating.
  - Dataset ingestion and batch test case importer (CSV and JSON).
  - Deterministic assertion engine: `exact-match`, `contains`, `regex`, `json-schema`, `levenshtein`, `rouge-l`.
  - Model-graded assertion engine: `llm-rubric` and `factuality`.
  - Background task worker running PostgreSQL `FOR UPDATE SKIP LOCKED`.
- **Dependencies**: Phase 1 (Database & Models) and Phase 2 (Provider Adapters).
- **Technical Challenges**: Maintaining bounded memory (< 512 MB) during large matrix runs; orchestrating reliable background tasks without Redis.
- **Testing Requirements**: Verification against golden dataset; worker recovery tests simulating container crash mid-run.
- **Acceptance Criteria**: 100 test cases evaluated across 2 models in < 60s; assertions score with 100% mathematical precision; zero memory leaks.

---

### Phase 4: Model Comparison and Benchmarking
- **Objectives**: Deliver a standardized benchmarking engine to compare models on identical datasets, measuring normalized latency (TTFT, total), throughput (TPS), token usage, and cost efficiency.
- **Key Deliverables**:
  - Side-by-side model comparison service and API endpoint (`/benchmarks/compare`).
  - Benchmarking telemetry aggregator (computing P50, P90, P99 latency and tokens/second).
  - Cost comparison matrix computing estimated spend per million tokens.
  - Exportable benchmark summaries in JSON and CSV.
- **Dependencies**: Phase 3 (Evaluation engine & dataset pipeline).
- **Technical Challenges**: High-precision latency measurement in async event loops; handling fluctuating provider speeds without skewing results.
- **Testing Requirements**: Benchmark calculation tests comparing synthetic model metrics; accuracy verification of cost calculation formulas.
- **Acceptance Criteria**: Benchmark API returns comparative ranking matrix with correct cost and latency calculations.

---

### Phase 5: RAG Evaluation
- **Objectives**: Implement end-to-end evaluation for Retrieval-Augmented Generation pipelines, measuring Context Recall, Context Precision, Answer Faithfulness, and Hallucination Index.
- **Key Deliverables**:
  - `rag_evaluations` and `rag_contexts` models with `pgvector` HNSW indexes.
  - Stage 1: Retrieval evaluator measuring Context Recall and Context Precision.
  - Stage 2: Generation evaluator measuring Answer Faithfulness and Hallucination Index.
  - Stage 3: Citation validator verifying source chunk attribution.
  - REST endpoint `POST /api/v1/rag/evaluate`.
- **Dependencies**: Phase 3 (Evaluation engine) and PostgreSQL `pgvector`.
- **Technical Challenges**: Sentence-level claim extraction and verification against multiple retrieved chunks without excessive judge LLM latency.
- **Testing Requirements**: RAG benchmark suite with known ground-truth hallucinations and faithful answers.
- **Acceptance Criteria**: Faithfulness metric scores 1.0 on fully grounded text and < 0.3 on deliberate hallucinations.

---

### Phase 6: AI Red Teaming and Vulnerability Detection
- **Objectives**: Build an automated adversarial security scanner implementing the OWASP Top 10 for LLMs and MITRE ATLAS taxonomies.
- **Key Deliverables**:
  - Red-team plugins: `prompt-injection`, `jailbreak`, `system-prompt-extraction`, `pii-leakage`, `harmful-content`.
  - Delivery strategies: `basic`, `encoding-base64`, `leetspeak`, `template-framing`.
  - Adversarial scanner orchestrator and finding classifier.
  - REST endpoints `POST /api/v1/redteam/scans` and `GET /api/v1/redteam/scans/{id}/findings`.
  - SARIF (Static Analysis Results Interchange Format) exporter for security tools.
- **Dependencies**: Phase 2 (Provider Adapters) and Phase 3 (Evaluation Pipeline).
- **Technical Challenges**: Balancing heuristic regex detection (canary tokens) with LLM safety judges while avoiding false positives.
- **Testing Requirements**: Synthetic red-team scans against intentionally vulnerable and guarded mock models.
- **Acceptance Criteria**: Successfully detects prompt injection and PII leakage in synthetic test targets; outputs compliant SARIF report.

---

### Phase 7: Evaluation History, Analytics and Regression Testing
- **Objectives**: Implement prompt version diffing, time-series evaluation analytics, and automated regression quality gates.
- **Key Deliverables**:
  - Prompt version diff engine with semantic visual diffs.
  - Time-series analytics service tracking quality scores, pass rates, latency, and costs across runs.
  - Automated regression detector: triggers failure alerts when quality drops beyond a threshold ($\Delta > 5\%$).
  - Regression report generator.
- **Dependencies**: Phase 3 (Evaluation Runs & History).
- **Technical Challenges**: Efficient time-series aggregation queries in PostgreSQL without exhausting free-tier compute hours.
- **Testing Requirements**: Regression detection tests asserting correct failure flags when score drops.
- **Acceptance Criteria**: Regression API detects score degradation across prompt versions and identifies specific regressed test cases.

---

### Phase 8: Frontend Dashboard and Visualization
- **Objectives**: Build a responsive web application in React 19, TypeScript, Vite, Tailwind CSS, and shadcn/ui, providing an intuitive interface for all EvalForge features.
- **Key Deliverables**:
  - Interactive evaluation matrix viewer with real-time SSE progress streaming.
  - Side-by-side model comparison dashboard with Recharts scatter plots and bar charts.
  - Prompt editor with version diffing and live testing playground.
  - RAG evaluation breakdown view with highlighted context attribution.
  - Red-team vulnerability explorer with severity badges and remediation guidance.
- **Dependencies**: Phase 1 through 7 backend APIs.
- **Technical Challenges**: Rendering large matrix grids (hundreds of cells) smoothly without DOM degradation; handling SSE reconnection on mobile/tab switch.
- **Testing Requirements**: Vitest component tests; MSW mock API tests; responsive UI testing.
- **Acceptance Criteria**: SPA builds with 0 TypeScript errors; bundles into optimized static assets under 500 KB gzip; passes accessibility audits.

---

### Phase 9: CLI and GitHub Actions Integration
- **Objectives**: Build a lightweight Python CLI tool and a reusable GitHub Action to run evaluations directly within CI/CD pipelines.
- **Key Deliverables**:
  - `evalforge` Python CLI built with Typer (`evalforge eval`, `evalforge redteam`, `evalforge view`, `evalforge init`).
  - Standalone HTML report generator.
  - GitHub Actions Action (`evalforge/evalforge-action`) with PR review comment automation.
  - JUnit XML and SARIF artifact generators.
- **Dependencies**: Phase 3 (Evaluation Engine) and Phase 6 (Red Teaming).
- **Technical Challenges**: Running CLI evaluations in air-gapped CI environments with minimal setup overhead (< 30 seconds).
- **Testing Requirements**: End-to-end CLI command tests; local GitHub Action simulation with `act`.
- **Acceptance Criteria**: CLI executes evaluation from YAML config and exits with non-zero code on failure; generates valid JUnit XML parsed by GitHub CI.

---

### Phase 10: Deployment, Security Hardening and Production Readiness
- **Objectives**: Deploy EvalForge to production on zero-cost infrastructure (Render, Neon, Vercel), perform comprehensive security audits, and verify production readiness.
- **Key Deliverables**:
  - Production deployment on Render Free Web Service + Neon Serverless PostgreSQL + Vercel SPA.
  - End-to-end SSL/TLS configuration and CORS lockdown.
  - Cold-start graceful waking screen and connection retry policies.
  - Production monitoring, health checks (`/health/live`, `/health/ready`), and security audit.
  - Complete user documentation and deployment guide.
- **Dependencies**: Phases 1 through 9.
- **Technical Challenges**: Verifying sustained execution within 512 MB RAM under load on Render; confirming zero monthly bill.
- **Testing Requirements**: Production smoke tests, memory profiling under 1,000-cell evaluation run, cold-start latency verification.
- **Acceptance Criteria**: Platform live on public URL; $0.00 infrastructure cost verified; all core workflows functional; zero critical security vulnerabilities.
