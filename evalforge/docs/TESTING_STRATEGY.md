# EvalForge — Comprehensive Testing & Quality Assurance Strategy

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Architecture (Phase 0) |
| **Harnesses** | Pytest, HTTPX, Respx, Vitest, Playwright |
| **Zero-Cost Requirement** | 100% of automated test suites must run without paid API keys |

---

## 1. Executive Summary & Testing Pyramid

In an AI evaluation platform, testing rigor must exceed standard web applications. If the evaluation engine itself has bugs or non-deterministic behavior, developers cannot trust its judgments of their LLMs.

EvalForge implements a comprehensive, multi-layered testing pyramid designed to ensure mathematical accuracy, API stability, security isolation, and reliable offline execution.

```
                   ┌───────────────────────────┐
                   │   End-to-End & CI/CD      │  10% (Playwright / CLI)
                   │       Acceptance Tests    │
                   ├───────────────────────────┤
                   │   Integration & API Tests │  30% (Pytest + HTTPX + DB)
                   │   (Workspaces, Matrix, DB)│
                   ├───────────────────────────┤
                   │     Unit & Domain Tests   │  60% (Assertions, Parsers,
                   │   (Mocked Providers/Math) │       Crypto, Pure Functions)
                   └───────────────────────────┘
```

---

## 2. Multi-Tier Testing Architecture

### 2.1. Tier 1: Unit & Domain Tests (Pytest)
- **Scope**: Individual business logic functions, assertion metrics, prompt template rendering, cryptographic utilities, and Pydantic validation schemas.
- **Speed**: Sub-second execution for hundreds of tests.
- **Dependencies**: Completely isolated from external databases and network sockets.

### 2.2. Tier 2: Database & Repository Tests
- **Scope**: SQLAlchemy 2.0 asynchronous repositories, transaction rollbacks, composite index efficiency, and `pgvector` HNSW index similarity queries.
- **Fixture Strategy**: Uses an ephemeral test database running within Docker or an isolated Neon database branch. Every test executes inside a nested transaction that is automatically rolled back upon completion, ensuring zero test pollution.

### 2.3. Tier 3: API Contract & Integration Tests
- **Scope**: FastAPI endpoints, authentication middleware, Argon2id verification, JWT rotation, and tenant boundary enforcement.
- **Harness**: `httpx.AsyncClient` invoking the FastAPI ASGI app directly in memory without binding network ports.

### 2.4. Tier 4: Provider Adapter Tests (Zero-Cost Mocking)
- **Scope**: Validating adapter payload serialization, header formulation, token extraction, and error parsing for Gemini, Groq, Ollama, and Hugging Face.
- **Zero-Cost Strategy**: Zero real external API calls during routine CI. All HTTP interactions are mocked using `respx` and pre-recorded JSON cassettes.

### 2.5. Tier 5: Evaluation Engine & Metric Verification Tests
- **Scope**: Cartesian matrix expansion, concurrency limits, weighted assertion aggregations, and edge-case handling (empty responses, malformed JSON, timeouts).

### 2.6. Tier 6: Frontend Component & State Tests (Vitest)
- **Scope**: React 19 UI components, form validations, TanStack Query caching hooks, Recharts rendering, and error boundary handling.

---

## 3. Mock Provider Framework & Offline CI Strategy

To guarantee that continuous integration (CI) runs cost **₹0 / $0** and never exhaust rate limits, EvalForge uses a standardized provider mocking framework with `respx`.

```python
# tests/fixtures/mock_providers.py
import pytest
import respx
import httpx

@pytest.fixture
def mock_groq_api():
    """Intercept outbound calls to Groq API with deterministic synthetic responses."""
    with respx.mock(base_url="https://api.groq.com/openai/v1") as respx_mock:
        respx_mock.post("/chat/completions").mock(
            return_value=httpx.Response(
                status_code=200,
                json={
                    "id": "chatcmpl-test-123",
                    "object": "chat.completion",
                    "created": 1700000000,
                    "model": "llama-3.3-70b-versatile",
                    "choices": [
                        {
                            "index": 0,
                            "message": {
                                "role": "assistant",
                                "content": "Gold tier members qualify for 60-day refunds."
                            },
                            "finish_reason": "stop"
                        }
                    ],
                    "usage": {
                        "prompt_tokens": 25,
                        "completion_tokens": 12,
                        "total_tokens": 37
                    }
                }
            )
        )
        yield respx_mock
```

---

## 4. Evaluation Metric Verification Suites

Every evaluation metric must be verified against an immutable "Golden Dataset" of test cases:

| Metric | Golden Test Scenarios | Verification Target |
|---|---|---|
| `exact-match` | Case differences, trailing spaces, unicode normalization | Must pass only when normalized strings are identical |
| `contains` | Substring found, substring missing, negative contains | Validates inclusion and exclusion |
| `regex` | Valid regex match, invalid regex pattern, catastrophic backtracking | Validates regex matching within bounded execution time |
| `json-schema` | Valid payload, missing required key, wrong data type | Validates schema conformity via `jsonschema` |
| `levenshtein` | Exact match (1.0), 1-char difference, complete mismatch (0.0) | Correct distance calculation |
| `rouge-l` | Partial overlap, identical text, empty string | Matches reference `rouge-score` library results |
| `context-recall` | All claims present in chunks, 1 missing claim, 0 claims | Computes exact fraction: $\frac{\text{matched}}{\text{total}}$ |
| `faithfulness` | Fully grounded response, ungrounded hallucination | Flags ungrounded claim sentences accurately |

---

## 5. Security & Red-Team Testing Strategy

To verify the AI Red-Teaming engine:
1. **Probe Generation Tests**: Verify that plugins generate valid syntactic attack vectors matching OWASP Top 10 categories.
2. **Strategy Mutation Tests**: Verify that `encoding-base64` correctly encodes the payload and prefixes the execution instruction, and `leetspeak` correctly replaces characters without corrupting intent.
3. **SSRF Boundary Tests**: Verify that any provider configured with `http://127.0.0.1:8000` or `http://169.254.169.254` is blocked with an `SSRFSecurityException` before a network socket is created.
4. **Key Decryption Isolation Tests**: Verify that decrypting an API key for Tenant A using Tenant B's salt fails cryptographically.

---

## 6. Measurable Quality Acceptance Criteria

A phase or pull request is accepted only when all of the following criteria are satisfied:

| Metric | Threshold | Verification Method |
|---|---|---|
| **Core Engine Line Coverage** | $\ge 85\%$ | `pytest --cov=evalforge` |
| **Security & Auth Coverage** | $100\%$ | Strict branch coverage on `evalforge.core.crypto` & `evalforge.modules.auth` |
| **Static Type Checking** | 0 errors | `mypy --strict evalforge/` |
| **Linting & Code Style** | 0 violations | `ruff check` & `ruff format --check` |
| **Offline CI Execution** | 100% | Full test suite completes in GitHub Actions with zero external internet access |
| **Test Suite Execution Time** | $\le 60\text{ seconds}$ | Fast feedback loop for developers |
| **Zero Memory Leaks** | Verified | Valgrind / Python tracemalloc verification under sustained 1,000-task matrix |
