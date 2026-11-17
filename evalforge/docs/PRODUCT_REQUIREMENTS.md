# EvalForge — Product Requirements Document (PRD)

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Architecture (Phase 0) |
| **Authors** | Lead Software Architect & AI Infrastructure Engineer |
| **Target Release** | Phase 1 (MVP) through Phase 10 (Full Production) |

---

## 1. Executive Summary & Problem Statement

Building production software with Large Language Models (LLMs) requires a radical shift in software quality assurance. Unlike traditional deterministic software where code behaves consistently given identical inputs, generative AI introduces non-determinism, stochastic drift, silent hallucinations, security vulnerabilities, and unpredictable latency and pricing dynamics.

Today, engineering teams frequently encounter critical blockers when moving LLM applications to production:
1. **Regression Blindness**: Modifying a system prompt to fix one edge case often silently breaks five previously working scenarios.
2. **Hallucination in RAG**: Retrieval-Augmented Generation (RAG) applications retrieve context, but models routinely invent facts, cite non-existent sources, or fail to adhere to grounding context.
3. **Model Selection Confusion**: Teams struggle to quantitatively evaluate whether upgrading from a cheaper model (e.g., Llama 3.3 70B on Groq) to a larger proprietary model (e.g., Gemini 1.5 Pro or GPT-4o) justifies the 10x-50x increase in API cost and latency.
4. **Vulnerability to Adversarial Attacks**: Generative applications are vulnerable to prompt injections, jailbreaks, system prompt exfiltration, and sensitive data leakage that traditional Web Application Firewalls (WAFs) cannot detect.
5. **Prohibitive Tooling Costs**: Many commercial evaluation platforms demand expensive enterprise subscriptions and require routing proprietary prompts and API keys through closed third-party SaaS backends.

**EvalForge** solves these challenges by providing an open-source, modular, cost-efficient, and developer-centric platform for LLM evaluation, benchmarking, RAG validation, and automated red teaming. Crucially, EvalForge is engineered to be self-hostable at **₹0 / $0 monthly infrastructure cost** using modern serverless free tiers and Bring-Your-Own-Key (BYOK) architecture.

---

## 2. Target Personas

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             Target Personas                                 │
├───────────────────────┬─────────────────────────┬───────────────────────────┤
│  AI Application Dev   │  Security / Red Teamer  │  QA / DevOps Engineer     │
│  - Prompt engineering │  - Adversarial testing  │  - CI/CD test gates       │
│  - RAG grounding      │  - Jailbreak prevention │  - Regression alerts      │
│  - Model selection    │  - PII leak detection   │  - Quality metrics        │
└───────────────────────┴─────────────────────────┴───────────────────────────┘
```

### Persona 1: The AI Application Engineer
- **Goal**: Rapidly iterate on prompts, RAG chunking parameters, and model selections with quantitative feedback.
- **Pain Point**: Spends hours manually testing prompt tweaks in web playgrounds; has no systematic way to verify if changes broke edge cases.
- **EvalForge Value**: Automated test suite execution across hundreds of test cases in minutes with deterministic and model-graded assertions.

### Persona 2: The Machine Learning & RAG Architect
- **Goal**: Optimize retrieval pipelines, verify context recall and answer faithfulness, and select optimal embedding models and LLMs.
- **Pain Point**: No unified way to separate retrieval failures (bad context) from generation failures (hallucination).
- **EvalForge Value**: Dedicated RAG evaluation module measuring Context Recall, Context Precision, and Faithfulness independently.

### Persona 3: The AI Security & Red-Team Engineer
- **Goal**: Audit LLM applications against prompt injection, jailbreaking, PII extraction, and malicious payload execution before production release.
- **Pain Point**: Manual adversarial prompting is slow, non-exhaustive, and difficult to reproduce.
- **EvalForge Value**: Automated adversarial scan generator implementing OWASP Top 10 for LLMs and MITRE ATLAS strategies (single-turn encodings, DAN templates, multi-turn adaptive attacks).

### Persona 4: The DevOps / QA Engineer
- **Goal**: Embed automated LLM quality gates directly into CI/CD pipelines (GitHub Actions) to prevent regressions from merging to main.
- **Pain Point**: Existing evaluation tools are heavy, expensive SaaS products with proprietary runners.
- **EvalForge Value**: Lightweight CLI and GitHub Actions integration that runs in CI, outputs SARIF and JUnit XML, and blocks failing pull requests.

---

## 3. Product Vision & Core Modules

EvalForge is structured into seven primary functional modules:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            EvalForge Core Modules                           │
├───────────────────────────────────┬─────────────────────────────────────────┤
│ A. Project & Workspace Management │ B. LLM Evaluation Engine                │
│ C. Model Benchmarking Suite       │ D. RAG Evaluation Engine                │
│ E. AI Red Teaming & Security      │ F. Regression Testing & Analytics       │
│ G. Developer Tooling (CLI/CI/API) │                                         │
└───────────────────────────────────┴─────────────────────────────────────────┘
```

---

### Module A: Project & Workspace Management

#### Objectives:
Provide multi-tenant organization, collaborative workspaces, access controls, and secure credential handling.

#### Functional Requirements:
1. **Workspaces & Organizations**:
   - Multi-tenant data isolation: every project, dataset, evaluation, and credential belongs strictly to an isolated workspace.
   - Role-Based Access Control (RBAC): `Owner`, `Admin`, `Member`, and `Viewer`.
2. **Project Organization**:
   - Projects act as containers for specific AI applications (e.g., "Customer Support Agent", "Internal Legal RAG").
   - Grouping of prompts, datasets, evaluation configurations, and red-team scans per project.
3. **Provider Configuration & BYOK Management**:
   - Support Bring-Your-Own-Key (BYOK) for LLM providers.
   - Encrypted at rest using AES-256-GCM.
   - Zero key leakage: keys are never transmitted to frontend client bundles or emitted in logs.
   - Configurable per-project or per-workspace defaults.

---

### Module B: LLM Evaluation Engine

#### Objectives:
Execute test matrices combining multiple prompts, provider models, and dataset test cases, scoring outputs with both deterministic and model-assisted assertions.

#### Functional Requirements:
1. **Matrix Generation**:
   - Compute Cartesian product: $\text{Prompts} \times \text{Providers} \times \text{Test Cases}$.
   - Variable substitution engine supporting templated variables (e.g., `{{query}}`, `{{customer_tier}}`, `{{context}}`).
2. **Deterministic Evaluation Metrics (Zero LLM Cost)**:
   - `exact-match`: Strict string equality.
   - `contains` / `not-contains`: Substring inclusion/exclusion (case-sensitive and insensitive).
   - `regex`: Regular expression pattern matching.
   - `json-valid`: Validates whether the response is valid JSON.
   - `json-schema`: Validates response structure against a provided JSON Schema.
   - `levenshtein-distance`: String similarity distance with configurable threshold.
   - `rouge-1`, `rouge-2`, `rouge-l`: Recall-Oriented Understudy for Gisting Evaluation.
   - `bleu-score`: Bilingual Evaluation Understudy for machine translation/generation fidelity.
   - `latency-threshold`: Assert response generated within maximum milliseconds.
   - `cost-threshold`: Assert response estimated cost does not exceed limit.
3. **Model-Graded Evaluation (LLM-as-a-Judge)**:
   - `llm-rubric`: Evaluates output against subjective criteria defined in natural language.
   - `factuality`: Verifies factual consistency between model output and ground truth reference.
   - Support for dedicated judge model overrides (e.g., test on local Ollama, evaluate with Gemini Flash).
   - Structured JSON reasoning extraction: judge outputs must include numeric score (0.0 to 1.0) and step-by-step reasoning.
4. **Scoring & Pass/Fail Thresholds**:
   - Individual assertions support custom weights ($w_i \in [0, 1]$).
   - Test case overall pass threshold (e.g., score $\ge 0.8$).
   - Run-level pass threshold (e.g., $95\%$ of all test cases pass).

---

### Module C: Model Benchmarking

#### Objectives:
Deliver quantitative side-by-side performance, cost, and latency comparisons across heterogeneous LLM models on identical datasets.

#### Functional Requirements:
1. **Cross-Model Execution**:
   - Run identical prompt templates and inputs across distinct model families (e.g., Google Gemini 1.5 Flash vs. Groq Llama 3.3 70B vs. Ollama Mistral 7B).
2. **Performance Telemetry**:
   - **Time to First Token (TTFT)**: For streaming responses.
   - **Total Latency**: Total elapsed time from request dispatch to final token receipt.
   - **Throughput**: Tokens per second (TPS).
3. **Token Usage & Cost Accounting**:
   - Detailed breakdown: Prompt tokens, completion tokens, cached tokens, and total tokens.
   - Dynamic pricing lookup table (cost per $1\text{M}$ input/output tokens).
   - Estimated cost calculated and persisted per model response.
4. **Comparative Visualization**:
   - Side-by-side response diff viewer.
   - Aggregated scatter plots: Quality Score vs. Latency vs. Cost.

---

### Module D: RAG Evaluation

#### Objectives:
Diagnose and evaluate the dual stages of Retrieval-Augmented Generation pipelines: Retrieval Quality and Generation Faithfulness.

#### Functional Requirements:
1. **RAG Pipeline Input Model**:
   - Accepts: User Query, Retrieved Context Chunks (with source metadata), Generated Answer, and optional Ground Truth Reference Answer.
2. **Retrieval Evaluation**:
   - **Context Recall**: Measures if all information needed to answer the query was successfully retrieved in the context chunks.
   - **Context Precision**: Measures the ratio of relevant chunks to total retrieved chunks (penalizing noisy context).
3. **Generation Evaluation**:
   - **Answer Faithfulness (Groundedness)**: Detects whether every statement in the generated answer can be mathematically or semantically derived from the provided context chunks.
   - **Hallucination Detection**: Flags specific sentences containing ungrounded claims or external knowledge leakage.
   - **Citation Quality**: Validates that in-text citations (e.g., `[Doc 1]`, `[Page 4]`) accurately map to the context chunk supporting that claim.
4. **Document Access Boundary Testing**:
   - Verifies that queries intended for specific tenant IDs or clearance levels do not leak context chunks belonging to unauthorized domains.

---

### Module E: AI Red Teaming

#### Objectives:
Automate adversarial vulnerability discovery and safety auditing before deployment.

#### Functional Requirements:
1. **Vulnerability Taxonomies**:
   - OWASP Top 10 for LLMs (Prompt Injection, Insecure Output Handling, Sensitive Info Disclosure, etc.).
   - MITRE ATLAS (Adversarial Threat Landscape for Artificial-Intelligence Systems).
2. **Attack Generators (Plugins)**:
   - `prompt-injection`: Direct injection attempts to override system prompt instructions.
   - `jailbreak`: Roleplay and hypothetical framing attacks (e.g., "Do Anything Now" / DAN).
   - `system-prompt-extraction`: Probing attempts to force the model to disclose confidential developer instructions.
   - `pii-leakage`: Probes designed to extract social security numbers, API keys, passwords, or emails.
   - `harmful-content`: Testing guardrails against hate speech, toxic instructions, and harassment.
3. **Delivery Strategies (Payload Mutators)**:
   - `basic`: Direct raw payload delivery.
   - `encoding-base64`: Base64 encoded payload with instructions to decode and execute.
   - `leetspeak`: Character substitution evasion.
   - `template-framing`: Suffix and prefix framing (e.g., fictional scenario, research disclaimer).
   - `multi-turn-adaptive`: Iterative conversation simulation where the attacker adapts probes based on model refusals.
4. **Automated Vulnerability Grading**:
   - Hybrid grading: Deterministic regex heuristics (detecting canary tokens or leaked patterns) combined with LLM-as-a-judge safety rubrics.
   - Finding classification: Severity (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`), CWE mapping, and remediation suggestions.

---

### Module F: Regression Testing & Analytics

#### Objectives:
Maintain immutable historical records of all evaluation runs, trace prompt performance across iterations, and detect quality degradation.

#### Functional Requirements:
1. **Prompt Versioning**:
   - Git-like version tracking for prompt templates (`v1`, `v2`, etc.).
   - Semantic diff viewer highlighting additions, deletions, and variable changes between versions.
2. **Historical Run Tracking**:
   - Time-series persistence of evaluation metrics, pass rates, latency, and costs per project.
3. **Automated Degradation Detection**:
   - Regression alerting: Trigger failure when a new prompt version causes quality score to drop by more than a configurable delta (e.g., $\Delta > 5\%$) compared to baseline.
   - Breakdown of specific test cases that regressed (previously passed, now failed).

---

### Module G: Developer Tooling

#### Objectives:
Empower developers to integrate EvalForge into everyday terminal workflows, scripts, and CI/CD automation.

#### Functional Requirements:
1. **REST API**:
   - Comprehensive OpenAPI 3.1 compliant endpoints for all platform capabilities.
2. **EvalForge CLI**:
   - Python-based CLI (`evalforge eval`, `evalforge redteam`, `evalforge view`, `evalforge init`).
   - Ability to execute evaluations against local config files or remote EvalForge server.
3. **GitHub Actions Integration**:
   - Dedicated action to run evaluations on Pull Requests.
   - Automated PR commenting with evaluation summary tables and diffs.
   - Non-zero exit code on failure to block merges.
4. **Exportable Reports**:
   - Formats: JSON, CSV, interactive standalone HTML, JUnit XML (for CI test viewers), and SARIF (for GitHub Security tab).

---

## 4. Scope Matrix: MVP (Phase 1-4) vs. Future Releases

To guarantee focused execution and rapid time-to-value, the platform scope is structured as follows:

| Feature / Capability | Phase 1-4 (MVP Scope) | Future Releases (Phase 5+) |
|---|:---:|:---:|
| **Project & Workspace Management** | Single-workspace, user auth, project CRUD, encrypted BYOK keys | Multi-tenant teams, granular RBAC, SSO/SAML |
| **Provider Support** | Google Gemini, Groq, Ollama, OpenAI-compatible | Anthropic, AWS Bedrock, Azure OpenAI, Custom HTTP |
| **Evaluation Engine** | Cartesian matrix, `{{var}}` substitution, deterministic assertions | Complex multi-step agent evaluation |
| **Deterministic Metrics** | Exact, contains, regex, JSON schema, Levenshtein, ROUGE | BLEU, embedding cosine similarity, custom Python hooks |
| **Model-Graded Evaluation** | `llm-rubric`, basic factuality check | Few-shot rubric learning, ensemble judging |
| **Model Benchmarking** | Side-by-side run, latency, token count, cost table | Statistical significance tests, latency distribution P99 |
| **RAG Evaluation** | Context Recall, Faithfulness (basic) | Citation validation, doc-level ACL security testing |
| **AI Red Teaming** | Prompt injection, PII leak, static strategies (Base64) | Adaptive multi-turn agentic jailbreaks, MITRE ATLAS |
| **Regression Testing** | Run history, basic pass rate comparison | Automated baseline diffing, regression webhooks |
| **Developer Tooling** | REST API, CLI (`eval`), JSON/HTML report | GitHub Actions Marketplace Action, SARIF exports |
| **Task Queue** | PostgreSQL `SKIP LOCKED` worker | Distributed Celery / Redis workers (if needed) |

---

## 5. Non-Functional Requirements (NFRs)

### NFR 1: Zero-Cost Deployment Feasibility
- The system must function in production on ₹0/month free-tier infrastructure (Vercel, Render, Neon).
- All core evaluation capabilities must support deterministic execution without requiring paid API tokens.

### NFR 2: Memory Footprint & Resource Bounding
- The backend application must operate within a **512 MB RAM** limit (Render Free Web Service constraint).
- Dataset processing must use generator-based streaming instead of loading entire multi-megabyte datasets into memory.
- Concurrency must be strictly bounded (e.g., maximum 3-5 concurrent provider requests).

### NFR 3: Security & Credential Isolation
- Provider API keys must never be stored in plaintext. Keys must be encrypted at rest using AES-256-GCM.
- Decrypted keys must exist only ephemerally in memory during provider execution and must never appear in logs or client responses.
- SSRF prevention must block requests to internal IP ranges (`127.0.0.1`, `10.0.0.0/8`, `169.254.169.254`).

### NFR 4: Extensibility & Modularity
- Adding a new LLM provider must require implementing a single abstract class (`BaseProvider`) without modifying core engine logic.
- Adding a new assertion metric must require implementing a single interface (`BaseAssertion`) without affecting other evaluators.

### NFR 5: Data Integrity & Durability
- All evaluation runs, configuration snapshots, test outputs, and scores must be stored transactionally in PostgreSQL with foreign key constraints.
- Evaluation runs must be resumable or gracefully marked as failed in the event of worker container restarts.
