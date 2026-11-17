# ADR 001: Backend Language & Framework Selection (Python 3.12+ / FastAPI)

| Metadata | Value |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-11-17 |
| **Deciders** | Lead Software Architect & AI Infrastructure Engineer |

---

## 1. Context & Problem Statement

EvalForge is an evaluation, benchmarking, RAG testing, and red-teaming platform for Large Language Models. A foundational architectural decision is selecting the primary backend programming language and web framework.

The reference project, **Promptfoo**, is implemented in **Node.js / TypeScript**. We must decide whether to adopt TypeScript/Node.js or build the platform in **Python 3.12+ / FastAPI**.

---

## 2. Decision Drivers

1. **AI/ML Ecosystem Synergy**: Direct compatibility with existing NLP tokenizers, evaluation metrics (ROUGE, BLEU, Levenshtein), vector embeddings, and LLM libraries.
2. **Asynchronous I/O Throughput**: Ability to handle hundreds of concurrent outbound HTTP calls to LLM providers with minimal overhead.
3. **Developer Ergonomics & Typing**: Strict compile-time and runtime type safety.
4. **Memory Efficiency**: Ability to run within Render's free-tier memory constraint (512 MB RAM).
5. **Team Velocity**: Rapid development of APIs and background workers.

---

## 3. Considered Options

- **Option 1: Python 3.12+ with FastAPI & Pydantic v2**
- **Option 2: TypeScript / Node.js with Fastify / Express (Promptfoo model)**
- **Option 3: Go with Gin / Echo**

---

## 4. Decision & Justification

**Chosen Option: Option 1 — Python 3.12+ with FastAPI & Pydantic v2.**

### Key Justifications:
1. **Dominance in the AI/ML Domain**: Python is the lingua franca of AI engineering. Evaluating LLMs requires computing NLP metrics (ROUGE-L, BLEU, Levenshtein distance, embedding cosine similarity, tokenizers). In Python, these libraries (`rouge-score`, `nltk`, `rapidfuzz`, `tiktoken`, `numpy`) run natively in process. In Node.js, running complex evaluation rubrics frequently requires spawning slow child Python processes via IPC or bridging via external microservices.
2. **Modern AsyncIO Performance in Python 3.12**: Python 3.12 features major improvements in the `asyncio` event loop and garbage collector. Combined with `uvloop` and FastAPI's ASGI architecture, it handles concurrent I/O-bound LLM API calls with negligible latency differences compared to Node.js.
3. **Pydantic v2 Rust Core**: Pydantic v2 executes data serialization, parsing, and schema validation in native compiled Rust. This provides 5x–20x performance speedups over traditional Python serialization and auto-generates OpenAPI 3.1 specifications.
4. **Memory Bounding**: A minimal FastAPI application running under Uvicorn consumes $\approx 65 \text{ MB}$ of resident memory, well within the 512 MB Render free-tier budget.

---

## 5. Consequences

### Positive:
- Direct access to the rich Python AI/ML ecosystem.
- Clean separation of concerns with modern asynchronous Python syntax (`async`/`await`).
- Automatic generation of OpenAPI 3.1 interactive Swagger/ReDoc documentation.
- Streamlined integration with data science tools and Jupyter workflows.

### Negative / Trade-offs:
- Frontend is TypeScript/React, creating a dual-language repository (Python backend, TypeScript frontend).
- Python packaging historically had nuances; mitigated by using modern `pyproject.toml` and `uv` / `pip`.
