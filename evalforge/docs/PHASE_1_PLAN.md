# EvalForge — Phase 1 Actionable Implementation Plan

| Metadata | Value |
|---|---|
| **Phase Title** | Backend Foundation, PostgreSQL, Authentication & Project Management |
| **Phase Target** | Sprint 1 Execution |
| **Prerequisites** | Phase 0 Architectural Approval |
| **Primary Outcome** | Production-Ready Core API, Auth Engine, Multi-Tenant Database, and Encrypted Key Vault |

---

## 1. Executive Summary & Scope

Phase 1 establishes the operational core of EvalForge. At the conclusion of this phase, the system will have a fully functioning asynchronous FastAPI backend, a normalized PostgreSQL 16 database managed via Alembic migrations, secure user authentication with Argon2id and JWT rotation, multi-tenant workspace isolation, project organization, and an AES-256-GCM encrypted vault for storing BYOK provider credentials.

No evaluation logic or external LLM API calls are executed in Phase 1. This phase is dedicated strictly to foundational infrastructure, data models, security, and project management APIs.

---

## 2. Target Directory Structure for Phase 1

```text
evalforge/
├── pyproject.toml
├── alembic.ini
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── .gitignore
│
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       └── 001_initial_core_schema.py
│
├── evalforge/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py              # Pydantic v2 Settings (.env loader)
│   │   ├── database.py            # Async engine & sessionmaker
│   │   ├── security.py            # Argon2id hashing & JWT encode/decode
│   │   ├── crypto.py              # AES-256-GCM encryption & HKDF key derivation
│   │   ├── exceptions.py          # Custom domain exceptions
│   │   └── middleware.py          # CORS, request logging, tenant resolution
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base.py                # Base SQLAlchemy declarative model with UUID & timestamps
│   │   ├── workspace.py           # Workspaces & WorkspaceMemberships
│   │   ├── user.py                # Users
│   │   ├── project.py             # Projects
│   │   ├── provider_config.py     # ProviderConfigs (with encrypted API key fields)
│   │   └── audit_log.py           # AuditLogs
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── common.py              # Standard response envelopes & pagination
│   │   ├── auth.py                # Register, Login, Token, UserProfile schemas
│   │   ├── workspace.py           # Workspace schemas & member roles
│   │   ├── project.py             # Project CRUD schemas
│   │   └── provider.py            # ProviderConfig input & masked output schemas
│   │
│   └── modules/
│       ├── __init__.py
│       ├── auth/
│       │   ├── __init__.py
│       │   ├── router.py          # /api/v1/auth routes
│       │   ├── service.py         # Business logic for auth & registration
│       │   └── dependencies.py    # get_current_user, get_active_workspace
│       ├── workspaces/
│       │   ├── __init__.py
│       │   ├── router.py          # /api/v1/workspaces routes
│       │   └── service.py         # Workspace management & invitation logic
│       ├── projects/
│       │   ├── __init__.py
│       │   ├── router.py          # /api/v1/projects routes
│       │   └── service.py         # Project lifecycle service
│       └── providers/
│           ├── __init__.py
│           ├── router.py          # /api/v1/providers routes
│           └── service.py         # Provider credential encryption & verification
│
└── tests/
    ├── conftest.py                # Pytest async database fixtures & mock settings
    ├── unit/
    │   ├── test_crypto.py         # AES-256-GCM & HKDF encryption verification
    │   ├── test_security.py       # Argon2id hashing & JWT token expiration tests
    │   └── test_schemas.py        # Pydantic schema validation tests
    └── integration/
        ├── test_auth_api.py       # Full registration & login flow
        ├── test_workspaces_api.py # Workspace creation & member isolation
        ├── test_projects_api.py   # Project CRUD within workspace
        └── test_providers_api.py  # Provider key encryption & masked return tests
```

---

## 3. Core Python Dependencies (`pyproject.toml`)

```toml
[project]
name = "evalforge"
version = "0.1.0"
description = "Production-Grade Open-Source LLM Evaluation & Security Platform"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.31.0",
    "pydantic>=2.9.0",
    "pydantic-settings>=2.5.0",
    "sqlalchemy[asyncio]>=2.0.35",
    "asyncpg>=0.29.0",
    "alembic>=1.13.3",
    "argon2-cffi>=23.1.0",
    "python-jose[cryptography]>=3.3.0",
    "cryptography>=43.0.1",
    "httpx>=0.27.2",
    "python-multipart>=0.0.12",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.3",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=5.0.0",
    "respx>=0.21.1",
    "ruff>=0.6.9",
    "mypy>=1.11.2",
]
```

---

## 4. Step-by-Step Task Breakdown

### Task 1.1: Project Skeleton & Configuration Engine
- Initialize repository structure according to the target layout.
- Implement `evalforge/core/config.py` using `pydantic-settings` to parse environment variables (`DATABASE_URL`, `SECRET_KEY`, `ENCRYPTION_MASTER_KEY`, `CORS_ORIGINS`).
- Configure `ruff` and `mypy` in `pyproject.toml`.
- **Definition of Done**: Backend initializes cleanly, loads `.env`, passes `ruff check` and `mypy --strict`.

### Task 1.2: Asynchronous Database Foundation & Base Models
- Implement `evalforge/core/database.py` configuring SQLAlchemy 2.0 `create_async_engine` and `async_sessionmaker`.
- Implement `evalforge/models/base.py` providing standard UUID primary keys, `created_at`, and `updated_at` timestamps.
- Configure Alembic in async mode (`alembic/env.py`) for automated migrations.
- **Definition of Done**: Alembic can connect to PostgreSQL, generate migrations, and execute `alembic upgrade head`.

### Task 1.3: Core Domain Models & Initial Migration
- Implement SQLAlchemy ORM models:
  - `Workspace` and `WorkspaceMembership` in `evalforge/models/workspace.py`.
  - `User` in `evalforge/models/user.py`.
  - `Project` in `evalforge/models/project.py`.
  - `ProviderConfig` in `evalforge/models/provider_config.py`.
  - `AuditLog` in `evalforge/models/audit_log.py`.
- Generate migration `001_initial_core_schema.py`.
- **Definition of Done**: All 6 tables created in PostgreSQL with foreign keys, uniqueness constraints, and indexes.

### Task 1.4: Cryptographic Engine & BYOK Key Vault
- Implement `evalforge/core/crypto.py`:
  - `derive_tenant_key(master_secret: str, workspace_id: UUID) -> bytes` using HKDF-SHA256.
  - `encrypt_api_key(plaintext_key: str, workspace_id: UUID) -> tuple[str, str, str]` returning Base64 ciphertext, IV, and auth tag.
  - `decrypt_api_key(ciphertext: str, iv: str, tag: str, workspace_id: UUID) -> str` validating AEAD tag before decryption.
- Implement `evalforge/core/security.py`:
  - `hash_password(password: str) -> str` using Argon2id.
  - `verify_password(password: str, hashed: str) -> bool`.
  - `create_access_token(data: dict) -> str` (15 min lifespan).
  - `create_refresh_token(data: dict) -> str` (7 days lifespan).
- **Definition of Done**: 100% test coverage on crypto operations; tampered tags trigger `DecryptionSecurityError`.

### Task 1.5: Authentication Module & Tenant Resolution Middleware
- Implement Pydantic schemas in `evalforge/schemas/auth.py`.
- Build `evalforge/modules/auth/service.py` managing registration, login, and token rotation.
- Build `evalforge/modules/auth/router.py` with endpoints:
  - `POST /api/v1/auth/register`
  - `POST /api/v1/auth/login`
  - `POST /api/v1/auth/refresh`
  - `GET /api/v1/auth/me`
- Build FastAPI dependency `get_current_user` and `get_active_workspace` in `evalforge/modules/auth/dependencies.py`.
- **Definition of Done**: User can register, receive JWT, authenticate, and query their profile.

### Task 1.6: Workspace & Project Management Endpoints
- Implement workspace and project services and routers:
  - `GET /api/v1/workspaces`
  - `POST /api/v1/projects`
  - `GET /api/v1/projects`
  - `GET /api/v1/projects/{id}`
  - `DELETE /api/v1/projects/{id}`
- Enforce strict workspace scoping: users can only see projects within their authenticated workspace.
- **Definition of Done**: Integration tests confirm cross-workspace data leakage is strictly blocked with `403 Forbidden` / `404 Not Found`.

### Task 1.7: Provider Configuration & Encrypted Key Ingestion
- Implement `evalforge/modules/providers/router.py`:
  - `POST /api/v1/providers` (ingests BYOK credentials, encrypts key via AES-256-GCM, stores IV/tag).
  - `GET /api/v1/providers` (returns list with masked keys, e.g. `gsk_..._4a9f`).
  - `DELETE /api/v1/providers/{id}` (deactivates / deletes configuration).
- **Definition of Done**: Database records contain only AES-256 ciphertext; API responses never expose plaintext keys.

### Task 1.8: Audit Logging & Test Automation Suite
- Implement `evalforge/models/audit_log.py` service hook logging key creations, role changes, and project deletions.
- Write complete test suite in `tests/`:
  - Unit tests for password hashing, JWT expiration, and AES-256-GCM encryption.
  - Integration tests for auth, workspaces, projects, and provider vault.
- Set up GitHub Actions workflow `.github/workflows/ci.yml` running `ruff`, `mypy`, and `pytest`.
- **Definition of Done**: 100% of tests pass offline; code coverage $\ge 90\%$; 0 linting or typing warnings.

---

## 5. Verification & Acceptance Criteria for Phase 1

Phase 1 is complete and approved for Phase 2 transition only when:
1. `alembic upgrade head` executes cleanly on a fresh PostgreSQL instance.
2. `pytest --cov=evalforge` passes with $\ge 90\%$ test coverage.
3. `ruff check .` and `ruff format --check .` return 0 errors.
4. `mypy --strict evalforge/` returns 0 errors.
5. Integration tests confirm:
   - Passwords are encrypted with Argon2id.
   - Provider API keys are encrypted at rest with AES-256-GCM.
   - Plaintext keys never appear in API responses or logs.
   - Workspace boundaries prevent cross-tenant access.
