# EvalForge — System Architecture & Technical Specifications

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Architecture (Phase 0) |
| **Architectural Style** | Modular Monolith with Asynchronous Task Worker |
| **Primary Runtime** | Python 3.12+ (FastAPI) & React 19 (Vite) |

---

## 1. Architectural Philosophy & Principles

EvalForge is architected according to five foundational engineering principles:

1. **Modular Monolith First**: Avoid premature distributed systems complexity. Group cohesive domain logic into isolated Python packages with strict internal boundaries.
2. **Deterministic-First Evaluation**: Maximize zero-cost mathematical and syntactic evaluation (regex, JSON Schema, ROUGE, Levenshtein, embeddings) before falling back to expensive LLM judges.
3. **Provider-Agnostic Extensibility**: Isolate all LLM provider interactions behind an abstract adapter interface (`BaseProvider`), making the addition of new providers a single-file implementation.
4. **PostgreSQL-Driven Durability**: Leverage PostgreSQL 16 for relational data, `pgvector` for semantic similarity, and transactional row-locking (`FOR UPDATE SKIP LOCKED`) for background job queueing, eliminating the need for Redis in the MVP.
5. **Zero-Cost Deployment Feasibility**: Ensure every layer of the architecture operates reliably within free-tier resource boundaries (specifically Render's 512 MB RAM and Neon's 1 GB storage limits).

---

## 2. High-Level System Architecture

The following diagram illustrates the complete high-level system context, user interfaces, backend services, persistence layer, and external AI providers.

```mermaid
flowchart TD
    subgraph Clients["Client & Automation Layer"]
        SPA["React 19 SPA (Vite / shadcn / Recharts)"]
        CLI["EvalForge CLI (Python / Typer)"]
        CI["GitHub Actions CI Runner"]
    end

    subgraph Edge["Edge & Ingress"]
        CDN["Vercel / Cloudflare Edge CDN"]
        LB["Render Web Service / Reverse Proxy"]
    end

    subgraph BackendApp["EvalForge Backend Application (Modular Monolith)"]
        API["FastAPI HTTP & SSE Gateway"]
        AuthM["Auth & Security Middleware"]
        
        subgraph DomainModules["Domain Modules"]
            ModProj["Project & Workspace Manager"]
            ModEval["Evaluation Orchestration Engine"]
            ModBench["Model Benchmarking Engine"]
            ModRAG["RAG Evaluation Engine"]
            ModRed["AI Red Teaming Engine"]
            ModReg["Regression & Analytics Engine"]
        end

        subgraph CoreInfra["Core Infrastructure Services"]
            JobWorker["Background Async Job Worker"]
            ProvFact["Provider Adapter Factory"]
            AssertPipe["Assertion & Metric Pipeline"]
            EncSvc["AES-256-GCM Vault Service"]
        end
    end

    subgraph StorageLayer["Data & Persistence Layer"]
        NeonDB[("Neon Serverless PostgreSQL 16")]
        PGV["pgvector Extension (HNSW)"]
        JobTbl["Job Queue (SKIP LOCKED)"]
    end

    subgraph Providers["External AI Provider Layer (BYOK)"]
        Gemini["Google Gemini API (Free Tier)"]
        Groq["Groq Cloud API (Free Tier)"]
        Ollama["Ollama Local Instance (HTTP)"]
        HF["Hugging Face Serverless API"]
        GenericOpenAI["OpenAI-Compatible Endpoints"]
    end

    SPA --> CDN
    CDN --> LB
    CLI --> LB
    CI --> LB
    LB --> API
    API --> AuthM
    AuthM --> DomainModules

    DomainModules --> JobWorker
    DomainModules --> AssertPipe
    JobWorker --> ProvFact
    ProvFact --> EncSvc

    EncSvc --> NeonDB
    JobWorker --> JobTbl
    ModRAG --> PGV
    DomainModules --> NeonDB

    ProvFact --> Gemini
    ProvFact --> Groq
    ProvFact --> Ollama
    ProvFact --> HF
    ProvFact --> GenericOpenAI
```

---

## 3. Backend Module Architecture

The backend adheres to a clean layered architecture within a modular monolith structure. Each module encapsulates its domain entities, service logic, repository queries, and Pydantic validation schemas.

```mermaid
flowchart TD
    subgraph Presentation["1. Presentation & API Layer"]
        R_Auth["/api/v1/auth"]
        R_Work["/api/v1/workspaces"]
        R_Proj["/api/v1/projects"]
        R_Prov["/api/v1/providers"]
        R_Eval["/api/v1/evaluations"]
        R_Bench["/api/v1/benchmarks"]
        R_RAG["/api/v1/rag"]
        R_Red["/api/v1/redteam"]
        R_Rep["/api/v1/reports"]
    end

    subgraph Domain["2. Core Domain & Business Logic"]
        S_Auth["Authentication & RBAC Service"]
        S_Eval["Evaluation Matrix Generator"]
        S_RAG["RAG Faithfulness & Recall Evaluator"]
        S_Red["Red Team Attack & Strategy Planner"]
        S_Bench["Benchmark Telemetry & Cost Aggregator"]
        S_Assert["Assertion Engine (Deterministic + Model)"]
    end

    subgraph Infrastructure["3. Infrastructure & Integration Layer"]
        Repo["SQLAlchemy 2.0 Async Repositories"]
        WorkerPool["PostgreSQL Async Worker (SKIP LOCKED)"]
        Crypto["AES-256-GCM Key Encryptor"]
        RateLimit["Token Bucket Rate Limiter"]
        Adapters["LLM Provider Adapters (HTTPX Async)"]
    end

    subgraph Persistence["4. Database & Storage Layer"]
        Postgres[("PostgreSQL 16 Engine")]
        VectorStore["pgvector HNSW Vector Store"]
    end

    Presentation --> Domain
    Domain --> Infrastructure
    Infrastructure --> Persistence

    Repo --> Postgres
    WorkerPool --> Postgres
    S_RAG --> VectorStore
```

### Module Responsibilities:
- **`evalforge.core`**: Cross-cutting concerns including database session management, base models, configuration settings, exceptions, and security cryptographic primitives.
- **`evalforge.modules.auth`**: User registration, Argon2id password hashing, JWT creation/refresh, and workspace tenant context resolution.
- **`evalforge.modules.projects`**: Workspace and project lifecycle, role assignments, and encrypted provider credential storage.
- **`evalforge.modules.evaluations`**: Evaluation matrix generation, configuration validation, progress tracking, and score aggregation.
- **`evalforge.modules.providers`**: The adapter implementations for Gemini, Groq, Ollama, Hugging Face, and generic OpenAI APIs.
- **`evalforge.modules.assertions`**: Deterministic evaluators (exact, regex, Levenshtein, ROUGE) and LLM-as-a-judge rubrics.
- **`evalforge.modules.rag`**: Chunk processing, context relevance scoring, faithfulness verification, and citation analysis.
- **`evalforge.modules.redteam`**: Adversarial prompt generation, delivery strategy mutation (Base64, jailbreaks), and vulnerability classification.
- **`evalforge.modules.jobs`**: The database-backed async job queue consumer running bounded concurrent workers.

---

## 4. LLM Provider Integration Flow

The Provider Subsystem uses an **Abstract Factory and Adapter Pattern**. All model calls pass through standardized rate-limiting, error handling, key decryption, and telemetry capture.

```mermaid
sequenceDiagram
    autonumber
    participant Worker as Evaluation Worker
    participant Factory as ProviderFactory
    participant Vault as AES-256 Vault
    participant Adapter as BaseProvider (e.g., GroqProvider)
    participant Limiter as Bounded Concurrency Limiter
    participant API as External LLM API (Groq/Gemini)

    Worker->>Factory: get_provider(provider_config_id)
    Factory->>Vault: decrypt_credentials(encrypted_blob)
    Vault-->>Factory: plaintext_api_key
    Factory->>Adapter: instantiate(config, decrypted_key)
    Factory-->>Worker: provider_instance

    Worker->>Limiter: acquire_slot(provider_id)
    Limiter-->>Worker: slot_granted

    Worker->>Adapter: generate(prompt, parameters)
    activate Adapter
    Note over Adapter: Record start_time (latency tracking)
    Adapter->>API: POST /chat/completions (HTTPX Async)
    API-->>Adapter: 200 OK (Response Payload)
    Note over Adapter: Record end_time, calculate TTFT & total_latency
    Note over Adapter: Extract token_usage (prompt, completion, cached)
    Note over Adapter: Lookup cost table -> compute estimated_cost_usd
    Adapter-->>Worker: ProviderResponseDTO
    deactivate Adapter

    Worker->>Limiter: release_slot(provider_id)
```

---

## 5. Evaluation Execution Lifecycle

The evaluation engine takes a declarative test configuration, compiles a Cartesian product of execution tasks, dispatches them through the durable PostgreSQL queue, executes assertions, and streams real-time progress to clients.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Web SPA / CLI / CI
    participant API as FastAPI Evaluation Router
    participant Engine as Matrix Generator
    participant DB as PostgreSQL 16
    participant Worker as Background Task Worker
    participant Provider as LLM Provider Adapter
    participant Assertions as Assertion Engine

    Client->>API: POST /api/v1/evaluations (config payload)
    API->>Engine: compile_matrix(prompts, providers, datasets)
    Engine-->>API: List[TaskItem] (N test executions)
    API->>DB: INSERT INTO evaluation_runs (status='PENDING')
    API->>DB: INSERT INTO job_queue (payload, run_id, status='PENDING')
    API-->>Client: 202 Accepted {run_id: "eval_123"}

    par Progress Streaming
        Client->>API: GET /api/v1/evaluations/eval_123/progress (SSE)
        API-->>Client: EventStream: {status: 'RUNNING', completed: 0, total: N}
    and Background Worker Execution
        Worker->>DB: SELECT * FROM job_queue FOR UPDATE SKIP LOCKED LIMIT 1
        DB-->>Worker: JobRow(task_item)
        Worker->>Provider: generate(prompt_with_vars)
        Provider-->>Worker: ProviderResponse(output, tokens, latency, cost)
        
        loop For each assertion in test_case
            Worker->>Assertions: evaluate(assertion_config, output, context)
            Assertions-->>Worker: MetricResult(passed, score, reason)
        end

        Worker->>DB: INSERT INTO evaluation_results & evaluation_metrics
        Worker->>DB: UPDATE evaluation_runs SET completed_tasks = completed_tasks + 1
        Worker->>DB: DELETE FROM job_queue WHERE id = job_id
        API-->>Client: EventStream: {status: 'RUNNING', completed: k, total: N}
    end

    Note over Worker,DB: When completed_tasks == total_tasks:
    Worker->>DB: UPDATE evaluation_runs SET status='COMPLETED', finished_at=NOW()
    API-->>Client: EventStream: {status: 'COMPLETED', summary: {...}}
```

---

## 6. AI Red-Teaming Execution Lifecycle

The Red-Teaming subsystem generates adversarial probes using plugins, applies transformation strategies to bypass model guardrails, tests target LLMs, and maps vulnerabilities to standard security frameworks.

```mermaid
flowchart TD
    Start([Initiate Red Team Scan]) --> LoadConfig["Load Target Config & Threat Scope"]
    
    subgraph Phase1["1. Probe Generation (Plugins)"]
        P_Inj["OWASP LLM01: Prompt Injection"]
        P_Jail["OWASP LLM01: Jailbreak & DAN"]
        P_PII["OWASP LLM06: Sensitive Info Disclosure"]
        P_Extract["OWASP LLM07: System Prompt Extraction"]
        P_Hate["Trust & Safety: Harmful Content"]
    end

    subgraph Phase2["2. Strategy Transformation (Mutators)"]
        S_Raw["Basic (Unmodified)"]
        S_B64["Base64 Obfuscation"]
        S_Leet["Leetspeak / Character Substitution"]
        S_Frame["Hypothetical / Fictional Framing"]
        S_Multi["Multi-Turn Adaptive Attack Tree"]
    end

    LoadConfig --> Phase1
    Phase1 --> Phase2

    subgraph Phase3["3. Attack Dispatch & Target Execution"]
        MutatedProbe["Mutated Adversarial Payload"]
        TargetApp["Target LLM Application"]
        RawOutput["Target Model Response"]
    end

    Phase2 --> MutatedProbe
    MutatedProbe --> TargetApp
    TargetApp --> RawOutput

    subgraph Phase4["4. Vulnerability Assessment & Scoring"]
        H_Heuristic["Deterministic Heuristics (Canary Tokens / Regex)"]
        J_Judge["LLM Security Judge (Safety Rubric)"]
        ScoreEval{"Vulnerability Triggered?"}
    end

    RawOutput --> H_Heuristic
    RawOutput --> J_Judge
    H_Heuristic --> ScoreEval
    J_Judge --> ScoreEval

    subgraph Phase5["5. Classification & Reporting"]
        Finding["Create Vulnerability Finding Record"]
        Taxonomy["Map to OWASP Top 10 & MITRE ATLAS"]
        Severity["Assign Severity (CRITICAL, HIGH, MED, LOW)"]
        Remediation["Generate Remediation Guidance"]
    end

    ScoreEval -- Yes --> Finding
    Finding --> Taxonomy --> Severity --> Remediation
    ScoreEval -- No --> Safe["Log Safe Defense"]
    
    Remediation --> PersistScan[("Persist to Database & Generate SARIF")]
    Safe --> PersistScan
    PersistScan --> EndScan([Scan Complete])
```

---

## 7. RAG Evaluation Workflow

RAG evaluation evaluates both the **Retrieval** stage and the **Generation** stage independently to isolate the root cause of failures.

```mermaid
sequenceDiagram
    autonumber
    participant App as RAG Pipeline Ingestion
    participant RAGEval as EvalForge RAG Engine
    participant PGV as pgvector Engine
    participant NLP as Deterministic NLP Scorer
    participant Judge as Evaluation Judge (Optional)
    participant DB as Evaluation Database

    App->>RAGEval: evaluate_rag(query, retrieved_chunks, generated_answer, ground_truth)
    
    rect rgb(240, 248, 255)
        Note over RAGEval,NLP: Stage 1: Retrieval Evaluation (Context Recall & Precision)
        RAGEval->>PGV: compute_cosine_similarity(query_emb, chunk_embs)
        PGV-->>RAGEval: chunk_relevance_scores
        RAGEval->>NLP: measure_context_recall(ground_truth, retrieved_chunks)
        NLP-->>RAGEval: context_recall_score (0.0 - 1.0)
        RAGEval->>NLP: measure_context_precision(query, retrieved_chunks)
        NLP-->>RAGEval: context_precision_score (0.0 - 1.0)
    end

    rect rgb(255, 245, 245)
        Note over RAGEval,Judge: Stage 2: Generation Evaluation (Faithfulness & Hallucination)
        RAGEval->>NLP: sentence_segmentation(generated_answer)
        NLP-->>RAGEval: List[ClaimSentence]
        
        loop For each claim sentence
            RAGEval->>Judge: verify_claim_grounding(claim, retrieved_chunks)
            Judge-->>RAGEval: {is_grounded: bool, supporting_chunk_id: str, reasoning: str}
        end

        Note over RAGEval: Faithfulness = Grounded Claims / Total Claims
        Note over RAGEval: Hallucination Index = 1.0 - Faithfulness
    end

    rect rgb(245, 255, 245)
        Note over RAGEval,NLP: Stage 3: Citation Verification
        RAGEval->>NLP: extract_citations(generated_answer)
        NLP-->>RAGEval: List[CitationRef(DocID, Claim)]
        RAGEval->>NLP: verify_citation_accuracy(CitationRef, retrieved_chunks)
        NLP-->>RAGEval: citation_accuracy_score (0.0 - 1.0)
    end

    RAGEval->>DB: INSERT INTO rag_evaluations (metrics, chunks, claims)
    RAGEval-->>App: RAGEvaluationReport(recall, precision, faithfulness, hallucination_index)
```

---

## 8. Background Job Execution Subsystem

To achieve zero-cost hosting without spinning up a separate Redis or RabbitMQ cluster, EvalForge implements a durable **PostgreSQL `FOR UPDATE SKIP LOCKED` worker queue**.

```mermaid
stateDiagram-v2
    [*] --> PENDING: Job Enqueued by API
    
    PENDING --> RUNNING: Worker locks row (FOR UPDATE SKIP LOCKED)
    
    state RUNNING {
        [*] --> FetchProvider: Decrypt API Key & Prepare Prompt
        FetchProvider --> InvokeLLM: HTTPX Async Request
        InvokeLLM --> RunAssertions: Parse Output & Run Assertions
        RunAssertions --> SaveResults: Persist Metric Rows
    }

    RUNNING --> COMPLETED: All assertions passed / processed
    RUNNING --> RETRYING: Transient Network Error (429, 503) & attempts < max_attempts
    RUNNING --> FAILED: Hard Exception / Timeout OR attempts >= max_attempts

    RETRYING --> PENDING: Exponential Backoff Elapsed
    
    COMPLETED --> [*]: Row Deleted or Archived
    FAILED --> [*]: Logged to Failure Table & Alerted
```

### PostgreSQL Worker Locking Semantics:
```sql
-- Atomically claim the next highest-priority pending task without blocking other workers:
WITH next_task AS (
    SELECT id 
    FROM job_queue 
    WHERE status = 'PENDING' 
      AND scheduled_at <= NOW()
    ORDER BY priority DESC, created_at ASC 
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue
SET status = 'RUNNING',
    locked_at = NOW(),
    locked_by = :worker_id,
    attempts = attempts + 1
FROM next_task
WHERE job_queue.id = next_task.id
RETURNING job_queue.*;
```

---

## 9. Zero-Cost Cloud Deployment Architecture

The following diagram maps EvalForge's components to verified free-tier cloud platforms, demonstrating zero monthly infrastructure expenditure.

```mermaid
flowchart LR
    subgraph Internet["Public Internet"]
        UserBrowser["Developer Browser"]
        GitHubRunner["GitHub Actions Runner"]
    end

    subgraph Vercel["Cloudflare Pages / Vercel (Hobby Free Tier)"]
        SPA_Build["React 19 Static SPA Assets"]
        EdgeRouting["Global Edge CDN & SSL"]
    end

    subgraph Render["Render.com (Free Web Service)"]
        subgraph Container["512 MB RAM / 0.1 CPU Shared Container"]
            Uvicorn["Uvicorn ASGI Server (1 Worker)"]
            FastAPIApp["FastAPI REST & SSE Server"]
            BackgroundPoll["Async Task Consumer Task"]
        end
    end

    subgraph Neon["Neon.tech (Free Serverless PostgreSQL)"]
        NeonCompute["PostgreSQL 16 Serverless Engine"]
        NeonStorage["1 GB Relational Storage + pgvector"]
        AutoSleep["Auto-Suspend (after 5 min idle)"]
    end

    subgraph ExternalAPIs["Free-Tier & Local AI Inference"]
        GeminiAPI["Google Gemini 1.5 Flash (15 RPM / 1M TPM)"]
        GroqAPI["Groq Llama 3.3 70B (30 RPM / Free)"]
        OllamaLocal["Local Ollama (0 Cost / Local Hardware)"]
    end

    UserBrowser --> EdgeRouting
    EdgeRouting --> SPA_Build
    UserBrowser -- "/api/* Requests" --> Uvicorn
    GitHubRunner -- "CLI / CI Eval" --> Uvicorn

    Uvicorn --> FastAPIApp
    FastAPIApp --> BackgroundPoll
    FastAPIApp --> NeonCompute
    BackgroundPoll --> NeonCompute
    NeonCompute --- NeonStorage
    NeonCompute --- AutoSleep

    BackgroundPoll -- "BYOK Invocations" --> GeminiAPI
    BackgroundPoll -- "BYOK Invocations" --> GroqAPI
    BackgroundPoll -- "Localhost Tunnel" --> OllamaLocal
```

---

## 10. Module Boundaries, Interfaces & Extensibility Contracts

To ensure that new providers, metrics, and red-team attacks can be integrated without breaking existing code, EvalForge mandates strict Python ABC interfaces.

### 10.1. Provider Interface (`BaseProvider`)
```python
from abc import ABC, abstractmethod
from typing import AsyncGenerator
from pydantic import BaseModel

class ProviderResponseDTO(BaseModel):
    output_text: str
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    cached_tokens: int = 0
    latency_ms: float
    time_to_first_token_ms: float | None = None
    estimated_cost_usd: float = 0.0
    raw_response: dict | None = None

class BaseProvider(ABC):
    def __init__(self, model_name: str, api_key: str | None = None, options: dict | None = None):
        self.model_name = model_name
        self.api_key = api_key
        self.options = options or {}

    @abstractmethod
    async def generate(self, prompt: str, temperature: float = 0.7, max_tokens: int = 1024) -> ProviderResponseDTO:
        """Execute a non-streaming completion call against the provider."""
        pass

    @abstractmethod
    async def generate_stream(self, prompt: str, **kwargs) -> AsyncGenerator[str, None]:
        """Stream response tokens for interactive debugging."""
        pass
```

### 10.2. Assertion Interface (`BaseAssertion`)
```python
from abc import ABC, abstractmethod
from pydantic import BaseModel

class AssertionResultDTO(BaseModel):
    passed: bool
    score: float  # Normalized between 0.0 and 1.0
    reason: str
    assertion_type: str

class BaseAssertion(ABC):
    @abstractmethod
    async def evaluate(self, actual_output: str, expected_value: any, context: dict | None = None) -> AssertionResultDTO:
        """Evaluate actual LLM response against expected constraints."""
        pass
```
