# EvalForge

> **Production-Grade Open-Source LLM Evaluation, Benchmarking, RAG Testing & AI Red-Teaming Platform**  
> *Inspired by Promptfoo, engineered for the Python AI ecosystem, deployable at ₹0 / $0 monthly infrastructure cost.*

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python: 3.12+](https://img.shields.io/badge/Python-3.12+-3776AB.svg?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/Backend-FastAPI-009688.svg?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![React: 19](https://img.shields.io/badge/Frontend-React%2019-61DAFB.svg?logo=react&logoColor=black)](https://react.dev)
[![PostgreSQL: 16+pgvector](https://img.shields.io/badge/Database-PostgreSQL%20%2B%20pgvector-336791.svg?logo=postgresql&logoColor=white)](https://github.com/pgvector/pgvector)
[![Architecture: Modular Monolith](https://img.shields.io/badge/Architecture-Modular%20Monolith-orange.svg)]()
[![Deployment: Zero-Cost Ready](https://img.shields.io/badge/Deployment-Zero--Cost%20%E2%82%B90%2Fmo-brightgreen.svg)]()

---

## 1. Executive Summary

**EvalForge** is an enterprise-ready, developer-first platform designed to test, evaluate, benchmark, secure, and continuously monitor Large Language Model (LLM) applications and Retrieval-Augmented Generation (RAG) pipelines.

As generative AI transitions from experimental prototypes to mission-critical software, engineering teams face critical questions:
- *Is my prompt modification degrading model accuracy on edge cases?*
- *Is my RAG pipeline retrieving relevant context, or is the model hallucinating unsupported claims?*
- *Which LLM delivers the optimal quality-to-cost ratio for my specific task?*
- *Is my AI application vulnerable to prompt injection, jailbreaks, or PII leakage?*
- *Is my AI application truly ready for production deployment?*

EvalForge provides an automated, rigorous evaluation engine to answer these questions systematically throughout the software development lifecycle (SDLC) and CI/CD pipelines.

---

## 2. Core Capabilities

| Capability | Description | Key Capabilities |
|---|---|---|
| **LLM Evaluation Engine** | Matrix evaluation across prompts, models, and test cases | Deterministic assertions, regex, JSON schema validation, semantic similarity, Levenshtein, ROUGE, BLEU, and optional LLM-as-a-judge rubrics. |
| **Model Benchmarking** | Side-by-side comparative analysis of multiple models | Standardized test datasets, normalized latency tracking (TTFT, total), token usage analytics, and estimated API cost comparisons. |
| **RAG Evaluation** | End-to-end evaluation of Retrieval-Augmented Generation | Retrieval precision, context recall, answer faithfulness, hallucination index, citation validation, and document-level tenant boundary checks. |
| **AI Red Teaming** | Automated adversarial vulnerability and safety scanning | OWASP Top 10 for LLMs, MITRE ATLAS taxonomy, prompt injection, jailbreak templates, meta-agent attacks, system prompt extraction, and PII leakage probes. |
| **Regression Testing** | Continuous quality tracking across prompt iterations | Git-like prompt versioning, historical evaluation runs, automated degradation alerts, configurable quality gates, and pass/fail thresholds. |
| **Developer Tooling** | Seamless integration with existing developer workflows | OpenAPI 3.1 REST API, native Python CLI, GitHub Actions CI/CD integration, and exportable reports in JSON, CSV, HTML, JUnit XML, and SARIF. |

---

## 3. High-Level Architecture

EvalForge is designed as a **Modular Monolith** optimized for developer ergonomics, zero-cost cloud hosting, and straightforward maintenance.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Client & CI/CD Interfaces                         │
│   ┌──────────────────────────┐   ┌───────────────┐   ┌──────────────────┐   │
│   │   Vite + React 19 SPA    │   │  EvalForge    │   │  GitHub Actions  │   │
│   │  (shadcn/ui + Recharts)  │   │   Python CLI  │   │    CI Runner     │   │
│   └────────────┬─────────────┘   └───────┬───────┘   └────────┬─────────┘   │
└────────────────┼─────────────────────────┼────────────────────┼─────────────┘
                 │                         │                    │
                 ▼                         ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│               EvalForge Backend Application (Python 3.12+ / FastAPI)        │
│                                                                             │
│  ┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────┐  │
│  │   Auth & Workspaces   │ │  Dataset & Prompts    │ │ Evaluation Engine │  │
│  │  (Argon2id + JWT)     │ │  (Version Manager)    │ │ (Matrix Generator)│  │
│  └───────────────────────┘ └───────────────────────┘ └─────────┬─────────┘  │
│  ┌───────────────────────┐ ┌───────────────────────┐           │            │
│  │   RAG Evaluation      │ │    AI Red Teaming     │           │            │
│  │  (pgvector / Chunking)│ │  (OWASP / Strategies) │           ▼            │
│  └───────────────────────┘ └───────────────────────┘ ┌───────────────────┐  │
│  ┌─────────────────────────────────────────────────┐ │ Bounded Async Pool│  │
│  │  PostgreSQL Job Queue (SKIP LOCKED)             │ │ (Rate Limiter)    │  │
│  └─────────────────────────────────────────────────┘ └─────────┬─────────┘  │
└──────────────────────────────────────┬─────────────────────────┼────────────┘
                                       │                         │
                                       ▼                         ▼
┌──────────────────────────────────────────────┐ ┌────────────────────────────┐
│         Neon Serverless PostgreSQL           │ │  Provider Adapters (BYOK)  │
│  ┌────────────────────┬────────────────────┐ │ │  ┌──────────────┐         │
│  │ Normalized Data    │ pgvector Embeddings│ │ │  │Google Gemini │ (Free)   │
│  │ (3NF Relational)   │ (HNSW Indexes)     │ │ │  ├──────────────┤         │
│  ├────────────────────┼────────────────────┤ │ │  │Groq API      │ (Free)   │
│  │ Encrypted Keys     │ Job Queue State    │ │ │  ├──────────────┤         │
│  │ (AES-256-GCM)      │ (ACID Durability)  │ │ │  │Ollama        │ (Local)  │
│  └────────────────────┴────────────────────┘ │ │  ├──────────────┤         │
│                                              │ │  │Hugging Face  │ (Free)   │
│                                              │ │  ├──────────────┤         │
│                                              │ │  │OpenAI-compat │ (Custom) │
│                                              │ │  └──────────────┘         │
└──────────────────────────────────────────────┘ └────────────────────────────┘
```

---

## 4. Technology Stack Summary

- **Backend**: Python 3.12+, FastAPI, Pydantic v2, SQLAlchemy 2.0 (asyncio), Alembic.
- **Frontend**: React 19, TypeScript, Vite, Tailwind CSS, shadcn/ui, Recharts, TanStack Query.
- **Database**: PostgreSQL 16+ with `pgvector` extension.
- **Asynchronous Task Queue**: PostgreSQL `FOR UPDATE SKIP LOCKED` (no Redis/Celery required in MVP).
- **Cryptography**: AES-256-GCM for API keys at rest; Argon2id for password hashing.
- **Supported AI Providers**: Google Gemini API, Groq Cloud, Ollama (local), Hugging Face Serverless, and OpenAI-compatible endpoints.
- **Testing & Quality**: Pytest, Vitest, HTTPX, Respx, Ruff, MyPy, ESLint, Prettier.

For full architectural justifications and trade-off matrices, see [TECH_STACK.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/TECH_STACK.md).

---

## 5. Zero-Cost Deployment Blueprint (₹0 / $0 per month)

EvalForge is engineered to run in production completely within legitimate free tiers:

1. **Frontend**: Hosted on **Vercel Hobby** or **Cloudflare Pages** (100 GB/month egress, global CDN, free SSL).
2. **Backend**: Hosted on **Render Free Web Service** (512 MB RAM, 0.1 CPU, 750 free instance hours/month).
3. **Database**: Hosted on **Neon Free Tier** (1 GB storage, 100 CU-hours/month, serverless auto-suspend, `pgvector` included).
4. **CI/CD**: **GitHub Actions** (2,000 free runner minutes/month).
5. **AI Inference**: **Bring Your Own Key (BYOK)** leveraging free tiers of Google Gemini (15 RPM, 1M TPM) and Groq (30 RPM), or completely free local inference via **Ollama**.

For detailed configuration, memory profiling, and cold-start mitigations, see [DEPLOYMENT_STRATEGY.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/DEPLOYMENT_STRATEGY.md).

---

## 6. Promptfoo Reference Lineage & Relationship

EvalForge is inspired by [Promptfoo](https://github.com/promptfoo/promptfoo) (`../promptfoo-reference`), adopting its battle-tested evaluation concepts while creating an independent, Python-native platform.

### How EvalForge Differs:
- **Language Ecosystem**: Promptfoo is TypeScript/Node.js; EvalForge is Python 3.12+ / FastAPI, natively integrating with Python's data science and AI libraries (PyTorch, Hugging Face, NumPy, pandas, LangChain, LlamaIndex).
- **Data Persistence**: Promptfoo is primarily a CLI with file-based configs and local SQLite caching; EvalForge is a multi-tenant, database-driven platform with collaborative workspaces, versioned history, and audit logging.
- **Database-Backed Task Queue**: Durable job orchestration directly in PostgreSQL with `FOR UPDATE SKIP LOCKED` instead of ephemeral in-process concurrency.
- **Deterministic-First Scoring**: Deep library of zero-cost mathematical and semantic assertions, minimizing dependence on costly LLM-as-judge calls.
- **License Compliance**: Promptfoo is licensed under the MIT License. EvalForge respects all applicable intellectual property rights and maintains clean-room engineering abstractions.

For in-depth architectural comparison, see [REFERENCE_ANALYSIS.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/REFERENCE_ANALYSIS.md).

---

## 7. Documentation Index

The complete architectural and product specification of EvalForge is organized as follows:

| Document | Description |
|---|---|
| [PRODUCT_REQUIREMENTS.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/PRODUCT_REQUIREMENTS.md) | Comprehensive Product Requirements Document (PRD), personas, use cases, MVP vs future scope. |
| [REFERENCE_ANALYSIS.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/REFERENCE_ANALYSIS.md) | Exhaustive analysis of Promptfoo's architecture, patterns, lessons, and architectural divergences. |
| [SYSTEM_ARCHITECTURE.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/SYSTEM_ARCHITECTURE.md) | Mermaid diagrams and technical specifications for all subsystems, data flows, and worker lifecycles. |
| [DATABASE_DESIGN.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/DATABASE_DESIGN.md) | Normalized PostgreSQL schema, ER diagram, pgvector integration, indexing, and migration plan. |
| [API_DESIGN.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/API_DESIGN.md) | OpenAPI 3.1 REST API specification, standard envelopes, error handling, and endpoint contracts. |
| [TECH_STACK.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/TECH_STACK.md) | In-depth technology justification, architectural trade-offs, and dependency rationales. |
| [DEPLOYMENT_STRATEGY.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/DEPLOYMENT_STRATEGY.md) | ₹0/month zero-cost cloud deployment blueprint, resource constraints, and cold-start mitigations. |
| [SECURITY_STRATEGY.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/SECURITY_STRATEGY.md) | Threat modeling, AES-256-GCM key encryption, JWT rotation, SSRF guardrails, and audit logging. |
| [TESTING_STRATEGY.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/TESTING_STRATEGY.md) | Test pyramid, provider mocking with Respx, synthetic fixtures, coverage metrics, and CI gates. |
| [PHASE_ROADMAP.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/PHASE_ROADMAP.md) | Comprehensive 10-phase development roadmap from inception to enterprise production readiness. |
| [PHASE_1_PLAN.md](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/PHASE_1_PLAN.md) | Actionable sprint plan for Phase 1: backend foundation, auth, database, and project management. |

### Architectural Decision Records (ADRs)

- [ADR 001: Backend Language Selection (Python 3.12+ vs TypeScript)](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/adr/001-backend-language.md)
- [ADR 002: Primary Database Selection (PostgreSQL + pgvector vs SQLite / MongoDB)](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/adr/002-database-selection.md)
- [ADR 003: Provider Adapter Architecture (Unified Adapter Interface & Factory)](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/adr/003-provider-architecture.md)
- [ADR 004: Asynchronous Job Execution (PostgreSQL SKIP LOCKED vs Celery / Redis)](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/adr/004-job-execution.md)
- [ADR 005: Zero-Cost Deployment Architecture (Render + Neon + Vercel)](file:///c:/Users/karti/OneDrive/Desktop/temp/AiEvals/evalforge/docs/adr/005-deployment-strategy.md)

---

## 8. Development Roadmap Overview

The project is structured into 10 cohesive engineering phases:

```text
Phase 0: Product Definition & System Architecture (Current)
Phase 1: Backend Foundation, PostgreSQL, Auth & Workspace Management
Phase 2: LLM Provider Abstraction & Adapters (Gemini, Groq, Ollama, HuggingFace)
Phase 3: Core Evaluation Engine, Dataset Management & Assertions
Phase 4: Model Comparison & Benchmarking Suite
Phase 5: RAG Evaluation Engine (Context Recall, Precision & Faithfulness)
Phase 6: AI Red Teaming & Automated Adversarial Vulnerability Scanner
Phase 7: Evaluation History, Analytics & Regression Testing Engine
Phase 8: Frontend Web Dashboard & Interactive Visualization
Phase 9: Python CLI & GitHub Actions CI/CD Integration
Phase 10: Zero-Cost Cloud Deployment, Security Hardening & Production Launch
```

---

## 9. License & Attribution

EvalForge is released under the [MIT License](LICENSE).  
Copyright (c) 2026 EvalForge Contributors.

This software draws conceptual inspiration from [Promptfoo](https://github.com/promptfoo/promptfoo) (Copyright (c) Promptfoo, licensed under the MIT License).
