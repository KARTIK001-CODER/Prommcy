# EvalForge — Security Architecture & Threat Mitigation Strategy

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Architecture (Phase 0) |
| **Security Standard** | OWASP Top 10 for LLMs (2025), NIST AI RMF, CWE / CAPEC |
| **Cryptographic Standards** | AES-256-GCM, Argon2id, SHA-256, HMAC-SHA256 |

---

## 1. Executive Summary & Threat Model

EvalForge operates at the intersection of application security and generative AI. By its nature, the platform handles **sensitive provider API credentials**, **proprietary evaluation prompts**, and **adversarial attack payloads designed to subvert LLM guardrails**.

Security is treated as a foundational architectural constraint, not an afterthought. This document outlines the comprehensive security engineering principles, cryptographic standards, defense-in-depth layers, and STRIDE threat model that govern EvalForge.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STRIDE Threat Analysis                            │
├─────────────────────┬───────────────────────────┬───────────────────────────┤
│ Threat Category     │ Specific Risk in EvalForge│ Architectural Mitigation  │
├─────────────────────┼───────────────────────────┼───────────────────────────┤
│ **Spoofing**        │ Impersonating users/tenants│ Argon2id + signed JWT     │
│ **Tampering**       │ Modifying evaluation logs │ Strict RDBMS constraints  │
│ **Repudiation**     │ Denying security scans    │ Immutable audit_logs table│
│ **Info Disclosure** │ Leaking BYOK API keys     │ AES-256-GCM envelope crypt│
│ **Denial of Service**│ Flooding LLM API limits  │ Bounded concurrency pool  │
│ **Elevation of Priv**| Cross-workspace data leak │ Tenant ID query scoping   │
└─────────────────────┴───────────────────────────┴───────────────────────────┘
```

---

## 2. Cryptographic Architecture & BYOK Key Vault

### 2.1. AES-256-GCM Credential Encryption
Provider API keys (Google Gemini, Groq, OpenAI) are **never stored in plaintext**. 

EvalForge uses **Authenticated Encryption with Associated Data (AEAD)** via AES-256-GCM:
- **Key Derivation (HKDF)**: The encryption key is derived using HMAC-based Extract-and-Expand Key Derivation Function (HKDF) combining the application master secret with the unique `workspace_id` as the cryptographic salt. This ensures that even if one tenant's key is somehow compromised, no other tenant's keys are vulnerable.
- **Initialization Vector (IV)**: A unique, cryptographically secure 96-bit (12-byte) random nonce is generated per encryption operation using `os.urandom(12)`.
- **Authentication Tag**: A 128-bit (16-byte) authentication tag is generated and validated upon decryption. Any ciphertext tampering immediately triggers an authentication failure.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AES-256-GCM Encryption Pipeline                        │
│                                                                             │
│   [Plaintext API Key] + [Master Secret + Workspace Salt (HKDF)]             │
│                               │                                             │
│                               ▼                                             │
│                      AES-256-GCM Cipher                                     │
│                               │                                             │
│       ┌───────────────────────┼──────────────────────────┐                  │
│       ▼                       ▼                          ▼                  │
│  [Ciphertext]     [12-byte IV (Nonce)]     [16-byte Auth Tag]               │
│       │                       │                          │                  │
│       └───────────────────────┼──────────────────────────┘                  │
│                               ▼                                             │
│              Stored in PostgreSQL `provider_configs`                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2. Zero-Leakage Credential Lifecycle
1. **Frontend Isolation**: Provider API keys are ingested over HTTPS via `POST /api/v1/providers`. Keys are immediately encrypted in memory before database persistence.
2. **Response Redaction**: Plaintext keys are stripped from all API response serializations. The API returns only a masked indicator (e.g., `gsk_..._4a9f`).
3. **Transient Memory Lifespan**: Decrypted keys exist only inside the execution scope of the specific HTTP request coroutine and are immediately dereferenced for garbage collection.
4. **Log Masking**: Custom logging middleware inspects all outbound log records and scrubs any string matching API key patterns (`AIza...`, `gsk_...`, `sk-...`, `Bearer ...`).

---

## 3. Authentication & Multi-Tenant Authorization

### 3.1. Password Hashing with Argon2id
User passwords are exclusively hashed using **Argon2id** (the winner of the Password Hashing Competition):
- Memory cost: $65,536 \text{ KiB}$ ($64 \text{ MB}$)
- Time iterations: $3$
- Parallelism: $4 \text{ threads}$
- Salt: 16 cryptographically random bytes.

### 3.2. Stateless JWT Tokens with Rotation
- **Access Tokens**: Short-lived (15 minutes), signed using HMAC-SHA256 (`HS256`). Contain `user_id`, `email`, and active `workspace_id`.
- **Refresh Tokens**: Long-lived (7 days), stored hashed in database. Employ **single-use token rotation**: whenever a refresh token is used to issue a new access token, the old refresh token is immediately invalidated.

### 3.3. Tenant Isolation & Row-Level Scoping
EvalForge enforces strict multi-tenant boundaries at the software layer:
- Every query executed by the application must include an explicit `WHERE workspace_id = :active_workspace_id` predicate.
- Dependency injection in FastAPI automatically resolves the authenticated user's workspace membership and rejects any request attempting to access foreign workspace entities with `403 Forbidden`.

---

## 4. Server-Side Request Forgery (SSRF) Prevention

EvalForge allows users to configure custom HTTP webhook endpoints for proprietary model APIs and RAG retrieval pipelines. This introduces the risk of **Server-Side Request Forgery (SSRF)**, where an attacker configures an endpoint pointing to internal infrastructure or cloud metadata services.

### SSRF Guardrails Implemented:
1. **IP Range Blacklisting**: The HTTP client resolves target hostnames before socket connection and strictly blocks:
   - Loopback addresses: `127.0.0.0/8`, `::1`
   - RFC 1918 Private networks: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
   - Link-local addresses: `169.254.0.0/16`
   - Cloud metadata services: `http://169.254.169.254` (AWS, GCP, Azure metadata endpoints).
2. **DNS Rebinding Protection**: The pre-resolved IP address is pinned to the HTTP transport connection, ensuring that malicious DNS records switching from public to private IP post-validation are rejected.
3. **Protocol Restriction**: Only `https://` and `http://` protocols are permitted; `file://`, `gopher://`, `ftp://` are rejected at the schema level.

---

## 5. Input Validation & SQL Injection Prevention

1. **Pydantic v2 Strict Typing**: All incoming request payloads are strictly validated against Pydantic schemas. Excess or unvalidated fields are discarded (`extra = "forbid"`).
2. **SQLAlchemy 2.0 Parameterized Queries**: All database queries are compiled using parameterized SQL statements. Raw string concatenation in queries is forbidden and prevented by static code analysis rules.

---

## 6. Secure AI Red Teaming & Guardrails

Automated red-teaming involves dispatching adversarial payloads (prompt injections, jailbreaks, PII probes). To ensure ethical safety:

1. **Target Authorization Disclaimer**: Users must explicitly acknowledge that they own or have written authorization to audit the target LLM application before launching red-team scans.
2. **Heuristic Finding Validation**: LLM-generated security verdicts are treated as **probabilistic evidence**, not absolute proof. Findings are surfaced with confidence scores, raw attack prompts, and target responses, allowing human engineers to verify exploitability.
3. **Payload Containment**: Red-team test cases are stored in isolated database records and are marked with metadata flags preventing them from being accidentally included in standard regression benchmarking suites.

---

## 7. Rate Limiting, Audit Trails & Data Retention

1. **API Rate Limiting**:
   - Authentication routes (`/auth/login`, `/auth/register`): Bounded to 5 attempts per minute per IP using a sliding-window algorithm to prevent brute-force attacks.
   - Evaluation triggers (`/evaluations`): Bounded to 10 triggers per minute per workspace.
2. **Immutable Audit Logging**:
   - The `audit_logs` table records every security-sensitive action: credential creation, key rotation, dataset export, user invitation, and role change.
   - Logs include timestamp, actor user ID, workspace ID, originating IP address, action code, and state diff.
3. **Data Retention & Purging**:
   - Users can permanently delete projects, datasets, and provider configs. Deletion triggers cascading SQL deletes removing all associated responses, metrics, and vector embeddings.
