# EvalForge — Promptfoo Reference Architecture Analysis & Comparative Study

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Architecture (Phase 0) |
| **Reference Subject** | Promptfoo (`promptfoo/promptfoo`, MIT License) |
| **Scope** | Architectural Analysis, Design Patterns, Strengths, Limitations, and EvalForge Evolution |

---

## 1. Executive Summary & Purpose

To engineer a resilient, modern LLM evaluation platform, we conducted an in-depth architectural study of **Promptfoo**, one of the pioneering open-source command-line evaluation and red-teaming tools. 

The purpose of this analysis is not to clone or rebrand Promptfoo, but to:
1. Deconstruct its battle-tested design patterns (provider abstractions, matrix generation, red-team plugin/strategy separation).
2. Identify architectural bottlenecks and limitations inherent in its design (file/SQLite storage model, Node.js runtime disconnection from the Python AI/ML ecosystem, single-user CLI focus).
3. Architect **EvalForge** with modern, modular, Python-native engineering abstractions, enterprise multi-tenancy, PostgreSQL durability, and zero-cost cloud deployment feasibility.

---

## 2. Deconstruction of Promptfoo's Core Architecture

Promptfoo is architected primarily as a **Node.js/TypeScript command-line tool** that parses declarative YAML configurations, executes tests concurrently across model providers, evaluates outputs using assertions, and outputs results to terminal, HTML, or local SQLite storage.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Promptfoo Architecture Deconstruction                    │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      Declarative YAML Config                        │   │
│   │   prompts: [...]   providers: [...]   tests: [...]   defaultTest:   │   │
│   └──────────────────────────────────┬──────────────────────────────────┘   │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │               Configuration Parser & Variable Resolver              │   │
│   │  - file:// references     - Nunjucks templates    - Env vars        │   │
│   └──────────────────────────────────┬──────────────────────────────────┘   │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     Matrix Orchestration Engine                     │   │
│   │  - Cartesian product generator (Prompts x Providers x Tests)        │   │
│   │  - Concurrency control (p-limit)   - Token/Response Cache (SQLite)  │   │
│   └───────────────┬──────────────────┬──────────────────┬───────────────┘   │
│                   │                  │                  │                   │
│                   ▼                  ▼                  ▼                   │
│   ┌───────────────────────┐ ┌──────────────────┐ ┌───────────────────────┐  │
│   │  Provider Adapters    │ │ Assertions Engine│ │ Red Team Orchestrator │  │
│   │  ApiProvider interface│ │ Deterministic &  │ │ Plugins (Generators)  │  │
│   │  callApi() method     │ │ LLM Rubrics      │ │ Strategies (Mutators) │  │
│   └───────────────────────┘ └──────────────────┘ └───────────────────────┘  │
│                   │                  │                  │                   │
│                   └──────────────────┼──────────────────┘                   │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      Output & Persistence Layer                     │   │
│   │  - Terminal Table   - JSON/CSV Export   - Local Web Viewer (Vite)   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Deep Dive into Subsystem Implementations

### 3.1. Provider Abstraction & Integrations
- **Core Interface**: Promptfoo enforces a minimal TypeScript interface `ApiProvider`:
  ```typescript
  interface ApiProvider {
    id: () => string;
    callApi: (prompt: string, context?: CallApiContextOptions) => Promise<ProviderResponse>;
  }
  ```
- **Response Normalization**: All providers normalize model output into a standard `ProviderResponse` object containing `output`, `tokenUsage` (prompt, completion, total), `cached`, `cost`, and optional error strings.
- **Provider Registry**: Uses a factory pattern matching URI prefixes (e.g., `openai:gpt-4o`, `anthropic:claude-3-5-sonnet`, `ollama:llama3`, `webhook:https://...`, `python:script.py`).
- **Caching**: File-based or SQLite-backed token and response cache (keyed by provider ID, prompt string, and serialized options) to avoid redundant API billing during rapid test iteration.

### 3.2. Evaluation Workflows & Matrix Generation
- **Cartesian Expansion**: Promptfoo generates the full combinatorial matrix:
  $$\text{Runs} = |\text{Prompts}| \times |\text{Providers}| \times |\text{Tests}|$$
- **Concurrency & Rate Limiting**: Employs `p-limit` in Node.js to restrict concurrent HTTP connections, preventing rate limit throttling (HTTP 429).
- **Variable Rendering**: Uses Nunjucks templating to inject test variables into prompt strings at runtime.

### 3.3. Test Case & Dataset Handling
- **Declarative Structure**: Tests are defined as arrays of objects with `vars`, `assert`, `description`, and `metadata`.
- **External Imports**: Supports referencing external datasets via `tests: file://tests.csv` or `tests: file://tests.json`.
- **Dynamic Context Transformation**: Uses `contextTransform` scripts (JavaScript snippets) to extract specific sub-fields from complex JSON outputs before applying assertions.

### 3.4. Assertion & Scoring Systems
- **Two-Tier Verification**:
  1. **Deterministic Assertions**: Execute locally in Node.js with zero token cost (`equals`, `contains`, `regex`, `is-json`, `javascript`, `python`).
  2. **Model-Graded Assertions**: Make secondary LLM calls (`llm-rubric`, `factuality`, `model-graded-closedqa`, `context-recall`, `context-faithfulness`).
- **Scoring Semantics**: Each assertion produces a boolean `pass` and a floating-point score $s \in [0.0, 1.0]$. Test cases combine assertions via weighted averages.

### 3.5. Red-Teaming Architecture (The Triad)
Promptfoo structures automated red-teaming into three decoupled layers:
1. **Plugins (Intent Generators)**: Probes targeting specific risk categories (e.g., `owasp:llm:01`, `pii:leak`, `harmful:hate`, `contracts`).
2. **Strategies (Delivery Wrappers & Mutators)**: Transform the raw probes to evade safety filters (e.g., `base64`, `jailbreak:meta`, `jailbreak:hydra` multi-turn agentic conversations).
3. **Assertions (Vulnerability Evaluators)**: Evaluate whether the target model successfully rejected the attack or succumbed to malicious fulfillment.

### 3.6. Configuration Management & CLI
- **File-Driven**: Centralized configuration in `promptfooconfig.yaml` with environment variable substitution (`${OPENAI_API_KEY}`).
- **CLI Commands**: Built with `commander.js` exposing `eval`, `view`, `init`, `redteam`, `share`, and `cache`.
- **CI/CD Action**: `promptfoo-action` wraps the CLI in GitHub Actions, parses the test summary, and posts markdown review comments to Pull Requests.

---

## 4. Strengths & Architectural Merits of Promptfoo

1. **Low Friction & Developer Ergonomics**: Single YAML configuration allows an engineer to run an evaluation within 60 seconds of installation.
2. **Comprehensive Assertion Library**: Rich spectrum of built-in deterministic and subjective assertions covering diverse QA needs.
3. **Decoupled Red-Teaming Triad**: Clean separation between vulnerability definition (Plugins) and evasion techniques (Strategies) enables exponential test generation.
4. **Agile Caching Layer**: Aggressive response caching drastically reduces token expenditure during prompt debugging.
5. **Standardized CI/CD Integration**: Native PR comments and JUnit/SARIF export formats enable automated merge gates.

---

## 5. Architectural Limitations & Bottlenecks in Promptfoo

Despite its strengths, Promptfoo's architecture exhibits key constraints that EvalForge resolves:

| Domain | Promptfoo Architecture | Limitation / Failure Mode | EvalForge Solution |
|---|---|---|---|
| **Ecosystem & Language** | Node.js / TypeScript | Disconnected from the Python-centric AI/ML ecosystem (PyTorch, Hugging Face, NumPy, pandas, LangChain, LlamaIndex). Custom Python evaluators require slow subprocess IPC calls. | **Python 3.12+ / FastAPI**: Native execution of Python ML evaluation libraries, NumPy calculations, and direct model tensors without IPC overhead. |
| **Data Persistence & Multi-Tenancy** | Local filesystem & SQLite (`promptfoo.db`) | Single-user desktop orientation. Impossible to share state, audit logs, or prompt histories across enterprise teams without closed SaaS sync. | **PostgreSQL 16 + pgvector**: Multi-tenant relational schema with workspaces, role-based access, and vector similarity search. |
| **Web Experience** | Ephemeral local viewer (`promptfoo view`) | The web viewer is a static dashboard launched locally from CLI to view previous runs; lacks real-time interactive evaluation configuration, prompt editing, and collaborative management. | **Dedicated Web Application**: Modern React 19 SPA with interactive prompt editors, real-time SSE progress streaming, and live benchmarking dashboards. |
| **Task Queue & Durability** | In-process Node.js memory (`p-limit`) | If the Node CLI crashes or is terminated, all in-flight evaluation state is lost; no durable background job tracking or worker distribution. | **Durable PostgreSQL Job Queue**: Robust `FOR UPDATE SKIP LOCKED` transactional worker pool with heartbeat recovery and zero-cost operation. |
| **Zero-Cost Feasibility** | Heavy reliance on paid LLMs for judging | Advanced features (faithfulness, recall, rubrics) mandate calling OpenAI/Anthropic APIs, quickly accumulating large bills. | **Deterministic-First & Free Providers**: Deep deterministic metric suite (ROUGE, Levenshtein, regex) plus native support for free Gemini, Groq, and local Ollama inference. |
| **Credential Security** | Plaintext `.env` / Shell environment variables | No centralized, encrypted credential store; keys are stored in plaintext on disk or CI runner memory. | **AES-256-GCM Encryption**: Secure key vault per workspace with envelope encryption; keys never leave backend memory. |

---

## 6. How EvalForge Adapts, Evolves, and Differentiates

EvalForge honors the architectural discoveries of Promptfoo while establishing an independent, production-grade foundation:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Architectural Evolution Matrix                        │
├──────────────────────────┬──────────────────────────┬───────────────────────┤
│ Dimension                │ Promptfoo Reference      │ EvalForge Platform    │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ Core Runtime             │ Node.js 18+              │ Python 3.12+ / FastAPI│
│ Primary Storage          │ SQLite / Local YAML      │ PostgreSQL 16 + vector│
│ Job Execution            │ In-memory event loop     │ DB-backed ACID queue  │
│ System Architecture      │ Standalone CLI tool      │ Modular Monolith+CLI  │
│ Multi-Tenancy            │ None (single-user/local) │ Workspaces & RBAC     │
│ API Standards            │ Internal Express / TRPC  │ OpenAPI 3.1 REST API  │
│ Target Infrastructure    │ Local developer machine  │ ₹0/mo Serverless Cloud│
│ Red Team Classification  │ Custom YAML taxonomy     │ OWASP Top 10 + ATLAS  │
└──────────────────────────┴──────────────────────────┴───────────────────────┘
```

### Specific Architectural Adaptations:
1. **The Adapter Pattern for AI Providers**:
   - EvalForge adopts Promptfoo's concept of a unified provider abstraction, but implements it as a clean Python Abstract Base Class (`BaseProvider`) utilizing `asyncio`, native typing via Pydantic v2, and streaming token counters.
2. **The Red Team Triad Evolution**:
   - EvalForge retains the elegant separation of **Plugins** (vulnerabilities), **Strategies** (mutators), and **Evaluators** (verifiers), but standardizes vulnerability metadata strictly against industry standards: **OWASP Top 10 for LLM Applications (2025)** and **MITRE ATLAS**.
3. **Hybrid Assertion Engine**:
   - EvalForge implements Promptfoo's dual deterministic/model-graded philosophy, but expands the deterministic tier with vectorized semantic similarity via `pgvector` and standard NLP metrics (ROUGE-L, Levenshtein, BLEU) so users can achieve 90% of evaluation accuracy without spending a single dollar on LLM judges.

---

## 7. Intellectual Property, Clean-Room Engineering & Licensing

EvalForge maintains strict adherence to open-source governance and intellectual property standards:

1. **Independent Implementation**:
   - EvalForge is built from the ground up in Python and TypeScript. No source code from Promptfoo is copied, transpiled, or redistributed.
2. **Clean-Room Design**:
   - Prompts, interfaces, schemas, database tables, and API contracts are newly designed to meet EvalForge's specific architectural requirements.
3. **MIT License Compliance**:
   - Where architectural concepts or benchmark taxonomies are informed by Promptfoo's prior work, formal attribution is recorded in the project documentation and repository notices, fully respecting Promptfoo's MIT license.
