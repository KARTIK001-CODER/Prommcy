# ADR 002: Primary Database Selection (PostgreSQL 16 + pgvector)

| Metadata | Value |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-11-17 |
| **Deciders** | Lead Software Architect & AI Infrastructure Engineer |

---

## 1. Context & Problem Statement

EvalForge requires a persistence engine to manage structured relational entities (users, workspaces, projects, test cases, prompt versions), analytical evaluation metrics (scores, latency, token usage), and vector embeddings for RAG retrieval evaluation.

The reference project, **Promptfoo**, uses local YAML files and an embedded SQLite database (`promptfoo.db`). We must choose the database engine for EvalForge.

---

## 2. Decision Drivers

1. **Multi-Tenancy & Data Isolation**: Strict relational separation between customer workspaces.
2. **Unified Relational & Vector Capabilities**: Ability to perform vector similarity search for RAG evaluation without introducing a separate vector database.
3. **Transactional Background Queueing**: Native support for row-level locking (`FOR UPDATE SKIP LOCKED`) to power background evaluation tasks.
4. **Zero-Cost Serverless Hosting**: Availability of a reliable free-tier cloud offering (Neon.tech).
5. **Storage Footprint**: Ability to operate within free-tier storage boundaries (1 GB).

---

## 3. Considered Options

- **Option 1: PostgreSQL 16+ with `pgvector` extension**
- **Option 2: SQLite / DuckDB (Local File-Based)**
- **Option 3: PostgreSQL + Dedicated Vector DB (Pinecone / Qdrant)**
- **Option 4: MongoDB / Document Store**

---

## 4. Decision & Justification

**Chosen Option: Option 1 — PostgreSQL 16+ with `pgvector` extension.**

### Key Justifications:
1. **Unification of Relational and Vector Data**: With `pgvector`, text chunk embeddings can be queried with standard SQL using HNSW indexes (`ORDER BY embedding <=> query_vector LIMIT 5`). This completely eliminates the complexity, latency, and financial cost of hosting a separate vector database like Pinecone or Weaviate.
2. **Multi-Tenant ACID Integrity**: Complex relations (prompts $\to$ versions, datasets $\to$ test cases, evaluation runs $\to$ results $\to$ metrics) require foreign key integrity, cascade deletes, and ACID transactions.
3. **Queueing Without External Infrastructure**: PostgreSQL's `FOR UPDATE SKIP LOCKED` allows the database to serve as a high-concurrency, durable job queue, eliminating the need for Redis in the MVP.
4. **Neon Serverless Free Tier**: Neon provides a generous free tier of 1 GB storage, 100 CU-hours/month, instant branching, and native `pgvector` support, meeting our ₹0/month requirement.

---

## 5. Consequences

### Positive:
- Single operational data store for relational models, JSONB documents, vector embeddings, and background job queues.
- Rock-solid ACID guarantees for evaluation runs and prompt versioning.
- Multi-tenant tenant scoping via standard SQL `WHERE workspace_id = :id` filters.

### Negative / Trade-offs:
- Serverless PostgreSQL has an auto-suspend sleep policy on Neon after 5 minutes of inactivity; mitigated by async connection retries and pool warmup in SQLAlchemy.
- Sizing discipline is required to remain within 1 GB storage; mitigated by raw model payload caps and compression.
