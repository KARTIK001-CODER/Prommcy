# ADR 003: LLM Provider Integration Architecture (Unified Adapter Pattern & Factory)

| Metadata | Value |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-11-17 |
| **Deciders** | Lead Software Architect & AI Infrastructure Engineer |

---

## 1. Context & Problem Statement

EvalForge must interface with multiple heterogeneous AI model providers: Google Gemini, Groq, Ollama (local), Hugging Face Serverless, and generic OpenAI-compatible proxies.

Each provider has different authentication headers, request payload formats, response JSON structures, token counting conventions, streaming mechanisms, and rate limit errors.

We must decide how to architect provider integrations so that:
1. Core evaluation and red-teaming engines remain completely decoupled from specific provider SDKs.
2. Adding a new provider requires minimal code changes.
3. Telemetry (latency, TTFT, token counts, cost) is captured consistently across all models.

---

## 2. Decision Drivers

1. **Decoupling**: Core evaluation logic must never depend on vendor-specific SDK classes.
2. **Standardized Telemetry**: Universal extraction of prompt tokens, completion tokens, TTFT, latency, and estimated cost.
3. **Pluggability & Extensibility**: Open/Closed Principle — new providers added without modifying core matrix generators.
4. **Lightweight Dependencies**: Avoid bundling heavy vendor SDKs if standard async HTTP calls via `httpx` suffice.
5. **BYOK Security**: Clean separation between stored encrypted credentials and runtime decrypted client instances.

---

## 3. Considered Options

- **Option 1: Unified Abstract Adapter Pattern (`BaseProvider`) + Factory with Native `httpx` Async Calls**
- **Option 2: Direct Coupling to Vendor SDKs (`google-generativeai`, `groq`, `openai`, etc.)**
- **Option 3: Reliance on LangChain / LlamaIndex Provider Wrappers**

---

## 4. Decision & Justification

**Chosen Option: Option 1 — Unified Abstract Adapter Pattern (`BaseProvider`) + Factory with Native `httpx` Async Calls.**

### Key Justifications:
1. **Lightweight & Fast**: Direct async HTTP calls using `httpx` eliminate the dependency bloat, version conflicts, and overhead of installing dozens of proprietary vendor SDKs.
2. **Normalized Response DTO**: All providers return a standard `ProviderResponseDTO` containing:
   - `output_text: str`
   - `prompt_tokens: int`
   - `completion_tokens: int`
   - `total_tokens: int`
   - `latency_ms: float`
   - `time_to_first_token_ms: float | None`
   - `estimated_cost_usd: float`
3. **Open/Closed Extensibility**: To add a new provider (e.g., Mistral API or Anthropic Claude), an engineer simply implements one class inheriting from `BaseProvider` and registers it in `ProviderFactory`. No other file in the evaluation engine changes.
4. **Rejection of Heavy Wrappers**: Third-party frameworks like LangChain introduce frequent breaking changes, high abstractions, and complex internal state that make low-level latency and token profiling difficult.

---

## 5. Consequences

### Positive:
- Minimal bundle size and memory consumption (< 512 MB RAM footprint preserved).
- Deterministic control over timeouts, HTTP retries, and headers.
- Consistent telemetry and cost calculations across all LLM models.

### Negative / Trade-offs:
- When a provider updates its raw REST API, EvalForge must maintain the HTTP payload mapping directly rather than waiting for an official SDK update.
