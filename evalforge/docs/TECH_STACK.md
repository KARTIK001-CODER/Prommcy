# EvalForge — Technology Stack Selection & Architectural Justifications

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Architecture (Phase 0) |
| **Architecture Pattern** | Modular Monolith |
| **Target Deployment** | Zero-Cost Infrastructure (Render / Neon / Vercel) |

---

## 1. Executive Summary & Selection Principles

The technology stack for EvalForge is engineered to balance **rapid developer iteration**, **deep alignment with the modern AI/ML ecosystem**, **exceptional runtime performance**, and **strict zero-cost deployment feasibility**.

Every technology choice is evaluated against three non-negotiable criteria:
1. **AI Ecosystem Synergy**: Does this language/tool allow seamless integration with state-of-the-art evaluation algorithms, tokenizers, vector math, and LLM SDKs?
2. **Resource Efficiency**: Can the backend operate smoothly within Render's free tier of **512 MB RAM** and **0.1 shared vCPU**?
3. **Simplicity & Operational Footprint**: Does this technology minimize operational overhead, eliminating mandatory external dependencies (like Redis, RabbitMQ, or complex Kubernetes clusters) for the MVP?

---

## 2. Comprehensive Technology Matrix

| Layer | Selected Technology | Version | Purpose & Rationale |
|---|---|---|---|
| **Backend Runtime** | **Python** | `3.12+` | The native lingua franca of AI/ML; exceptional asyncio performance, native typing, and direct compatibility with evaluation libraries. |
| **Backend Framework** | **FastAPI** | `0.115+` | High-throughput asynchronous ASGI framework with native OpenAPI 3.1 generation, dependency injection, and high concurrency. |
| **Data Validation** | **Pydantic** | `v2.9+` | Rust-backed blazing-fast data serialization and validation; strict runtime type enforcement. |
| **ORM / Persistence** | **SQLAlchemy** | `2.0+` | Modern asynchronous ORM providing compile-time type safety, raw SQL control, and database-agnostic query building. |
| **Database Migrations**| **Alembic** | `1.13+` | Version-controlled, reproducible schema migrations with autogenerate capabilities. |
| **Database Engine** | **PostgreSQL** | `16+` | Battle-tested ACID RDBMS with enterprise JSONB support, transactional locking, and serverless compatibility. |
| **Vector Search** | **pgvector** | `0.7+` | PostgreSQL native extension for HNSW vector indexing; eliminates the need for a separate vector database (e.g., Pinecone/Milvus). |
| **Frontend Runtime** | **TypeScript** | `5.6+` | Type-safe web client development, preventing runtime type errors. |
| **Frontend Framework**| **React** | `19.0` | Leading UI ecosystem with rich component libraries, server components, and active community. |
| **Build Tool** | **Vite** | `6.0+` | Ultra-fast ESM-based local development server and optimized rollup production bundling. |
| **Styling** | **Tailwind CSS** | `3.4+` | Utility-first CSS framework ensuring zero bundle bloat and consistent design tokens. |
| **UI Components** | **shadcn/ui** | Latest | Accessible, customizable Radix UI primitives; code is owned directly within the repository. |
| **Data Visualization**| **Recharts** | `2.13+` | Composable SVG chart library optimized for React; renders model comparison and regression curves. |
| **State & Data Fetch** | **TanStack Query** | `v5+` | Robust client-side cache management, optimistic updates, and background refetching. |
| **Testing (Backend)** | **Pytest + HTTPX** | `8.3+` | Async testing harness with `respx` for offline zero-cost LLM provider mocking. |
| **Testing (Frontend)**| **Vitest** | `2.1+` | Blazing-fast Vite-native unit and component test runner. |
| **Linting & Quality** | **Ruff & MyPy** | Latest | Extremely fast Rust-based Python linter/formatter (Ruff) + strict static type checker (MyPy). |
| **Deployment** | **Render + Neon + Vercel**| Free | 100% legitimate zero-cost hosting stack satisfying all compute, storage, and egress needs. |

---

## 3. Deep Architectural Justifications

### 3.1. Backend: Python 3.12+ & FastAPI vs. Node.js / TypeScript
While Promptfoo is built on Node.js/TypeScript, EvalForge deliberately chooses **Python 3.12+**:

- **Native AI/ML Ecosystem**: Evaluation often requires NLP tokenization, Levenshtein distance, ROUGE-L calculations, BLEU scoring, and vector embeddings. In Python, these are single library imports (`nltk`, `rouge-score`, `rapidfuzz`, `numpy`). In Node.js, developers must either rely on substandard JS ports or make expensive subprocess calls to Python scripts.
- **AsyncIO Concurrency**: Python 3.12 provides substantial performance improvements in asyncio event loop execution. Combined with FastAPI, it effortlessly manages hundreds of concurrent I/O-bound HTTP calls to LLM APIs.
- **Pydantic v2 Performance**: Pydantic's core is written in Rust, yielding 5x-20x faster validation and serialization compared to Pydantic v1 or standard JavaScript runtime validation libraries.

### 3.2. Primary Database: PostgreSQL 16 + pgvector vs. Alternatives
Instead of an embedded SQLite database (like Promptfoo) or a specialized vector database:

- **Unified Relational & Vector Storage**: `pgvector` adds high-dimensional vector similarity search directly into standard SQL queries (`ORDER BY embedding <=> query_vector LIMIT 5`). This eliminates the architectural overhead, cost, and synchronization issues of maintaining a separate vector database like Pinecone, Qdrant, or Weaviate.
- **ACID Multi-Tenancy**: Relational foreign keys and workspace boundaries guarantee that tenant data cannot leak. SQLite cannot provide reliable multi-user concurrent write scaling in cloud serverless environments.
- **Serverless Free Tier**: Neon.tech provides a generous free tier (1 GB storage, 100 CU-hours) with instant branching and auto-suspend capabilities.

### 3.3. Job Execution: PostgreSQL `FOR UPDATE SKIP LOCKED` vs. Celery & Redis
A common pitfall in modern web architecture is prematurely introducing Redis and Celery:

- **Zero Operational Cost**: Running a dedicated Redis instance on cloud platforms costs between $5 and $15/month or consumes an entire free-tier container.
- **ACID Transactional Guarantees**: Enqueuing a task is a standard SQL `INSERT INTO job_queue`. If the user transaction aborts, the job is rolled back. In contrast, dual-writing to PostgreSQL and Redis introduces split-brain race conditions.
- **Simplicity**: PostgreSQL's `FOR UPDATE SKIP LOCKED` allows multiple asynchronous Python worker coroutines to lock independent tasks concurrently with zero lock contention:
  ```sql
  SELECT id FROM job_queue WHERE status = 'PENDING' LIMIT 1 FOR UPDATE SKIP LOCKED;
  ```
- **Future Migration Path**: If EvalForge reaches scale requiring 10,000+ jobs/second, the queue abstraction can be swapped for Redis/Celery without altering the evaluation domain logic.

### 3.4. System Structure: Modular Monolith vs. Microservices
EvalForge is intentionally implemented as a **Modular Monolith**:
- Microservices introduce network serialization overhead, distributed tracing requirements, and multi-container deployment hurdles that violate the zero-cost requirement.
- A modular monolith keeps domain logic cleanly separated in packages (`evalforge.modules.*`), allowing internal method calls with zero network latency, while preserving the freedom to extract services later if needed.

---

## 4. Technology Trade-Off Analysis

| Category | Option A (Chosen) | Option B (Alternative) | Why Option A Won | Why Option B Was Rejected |
|---|---|---|---|---|
| **Backend Language** | Python 3.12 | TypeScript (Node.js) | Dominant language for AI/ML; native NLP metrics and tokenizers. | Node.js requires subprocess IPC or HTTP bridges to run Python evaluation scripts. |
| **Web Framework** | FastAPI | Django Ninja / Flask | Built-in async, native OpenAPI generation, automatic Pydantic validation. | Flask is synchronous by default; Django is too heavy and opinionated for an evaluation engine. |
| **Vector Engine** | PostgreSQL (pgvector) | Pinecone / Qdrant | Single database for relational data and vectors; free in Neon. | Pinecone free tier has strict pod limits; adds separate API keys and network latency. |
| **Task Queue** | PostgreSQL SKIP LOCKED | Celery + Redis | Zero extra infrastructure; runs directly inside existing PostgreSQL. | Redis requires an additional hosting instance or paid subscription; overkill for MVP. |
| **Frontend Styling** | Tailwind CSS + shadcn | Material UI / Ant Design | Copy-paste components; zero runtime CSS overhead; easily customized. | MUI and AntD have heavy runtime CSS-in-JS footprints and rigid opinionated aesthetics. |
| **State Management** | TanStack Query | Redux Toolkit | Server-state caching, automatic refetching, and zero boilerplate. | Redux is designed for complex local state, whereas evaluation platforms primarily handle server cache. |
| **Linter / Formatter**| Ruff | Black + Flake8 + isort | 10x-50x faster execution; replaces 5 tools with a single binary. | Running separate Flake8, Black, and isort slows down local development and CI pipelines. |

---

## 5. Development Tooling & Code Quality Standards

To maintain high code quality across the codebase, the following automated tooling is configured:

- **Formatting & Linting**: `ruff format` and `ruff check --fix` run on every pre-commit hook and CI run.
- **Static Type Checking**: `mypy --strict` enforces 100% type annotations across all backend modules.
- **Frontend Quality**: `eslint` with `@typescript-eslint/recommended` and `prettier`.
- **Pre-Commit Automation**: Fast pre-commit hooks ensure broken code or malformed typing cannot be committed to Git.
