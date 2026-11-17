# EvalForge — Zero-Cost Deployment Strategy (₹0 / $0 per Month)

| Metadata | Value |
|---|---|
| **Document Version** | 1.0.0 |
| **Status** | Approved for Architecture (Phase 0) |
| **Target Infrastructure Cost** | ₹0.00 / $0.00 per month (100% Free-Tier Legitimate) |
| **Hosting Stack** | Cloudflare Pages / Vercel + Render + Neon + GitHub Actions |

---

## 1. Executive Summary & Cost Philosophy

A core architectural tenet of EvalForge is **democratizing LLM evaluation and security testing**. Developers, open-source contributors, independent researchers, and bootstrapped startups should not be priced out of rigorous AI evaluation by proprietary SaaS subscriptions or expensive cloud hosting bills.

EvalForge is deliberately engineered to be deployed in production at **₹0 / $0 per month** utilizing legitimate, non-promotional free tiers of cloud platforms.

### Key Architectural Cost Principles:
1. **Zero Server Costs**: Running frontend, backend, database, and CI/CD without paying a single dollar.
2. **Bring Your Own Key (BYOK)**: The platform never bills for or marks up LLM API usage. Users supply their own API keys or run completely free local models.
3. **Deterministic-First Evaluation**: Over 80% of routine test assertions (regex, JSON Schema, Levenshtein, ROUGE-L, exact match) execute in local memory with zero token consumption.
4. **Optional LLM-as-a-Judge**: Model-graded judging is strictly opt-in and defaults to free-tier models (Google Gemini 1.5 Flash or Groq Llama 3.3 70B).

---

## 2. Free-Tier Infrastructure Evaluation & Limits

We evaluated all prominent cloud providers to construct a resilient zero-cost topology.

| Provider & Service | Free-Tier Limits & Quotas | Sleep / Inactivity Policy | Critical Constraints & Cold Starts | Suitability |
|---|---|---|---|---|
| **Vercel** *(Frontend)* | 100 GB bandwidth/mo, unlimited deployments, global edge CDN. | None (Static assets always hot). | Max 100 projects; Hobby license terms. | **Primary Frontend** |
| **Cloudflare Pages** *(Frontend Alt)* | Unlimited bandwidth, 500 builds/month, global anycast edge. | None (Always hot). | Zero cold starts; instant global edge propagation. | **Alternative Frontend** |
| **Render** *(Backend API)* | **512 MB RAM**, **0.1 shared CPU**, 750 free instance hours/month. | **Spins down after 15 min of inactivity**. | **~50 second cold start** on wake; ephemeral disk (resets on restart). | **Primary Backend** |
| **Neon** *(PostgreSQL)* | **1 GB storage**, **100 CU-hours/month**, up to 10 branches, 5 GB egress. | **Auto-suspends after 5 min of inactivity**. | Autoscales up to 2 CU; includes native `pgvector`. Wakes in < 1 second. | **Primary Database** |
| **GitHub Actions** *(CI/CD)* | **2,000 free runner minutes/month** for private repos (unlimited for public). | Ephemeral runners. | 6-hour max job limit; standard 2-core VMs. | **Primary CI/CD** |

---

## 3. Free-Tier AI Inference Providers

EvalForge provides native out-of-the-box adapters for the following free inference endpoints:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Free-Tier AI Provider Matrix                          │
├─────────────────────┬───────────────────┬───────────────────┬───────────────┤
│ Provider            │ Rate Limit (RPM)  │ Token Limit (TPM) │ Monthly Cost  │
├─────────────────────┼───────────────────┼───────────────────┼───────────────┤
│ Google Gemini Flash │ 15 RPM / 1,500 RPD│ 1,000,000 TPM     │ ₹0.00 (Free)  │
│ Groq Cloud          │ 30 RPM            │ 6,000-14,400 TPM  │ ₹0.00 (Free)  │
│ Ollama (Local)      │ Unlimited (Local) │ Unlimited (Local) │ ₹0.00 (Free)  │
│ Hugging Face Server │ Varies by model   │ Rate-limited      │ ₹0.00 (Free)  │
└─────────────────────┴───────────────────┴───────────────────┴───────────────┘
```

### 1. Google Gemini API (Free Tier)
- **Model**: `gemini-1.5-flash`, `gemini-2.0-flash`
- **Quotas**: 15 Requests Per Minute (RPM), 1,000,000 Tokens Per Minute (TPM), 1,500 Requests Per Day (RPD).
- **Application in EvalForge**: Ideal for high-context RAG evaluation and LLM-as-a-judge rubrics due to its 1M context window and generous free quotas.

### 2. Groq Cloud (Free Developer Tier)
- **Model**: `llama-3.3-70b-versatile`, `llama-3.1-8b-instant`
- **Quotas**: 30 RPM, 6,000 to 14,400 TPM.
- **Application in EvalForge**: Ultra-low latency generation (200-300 tokens/second), perfect for fast matrix generation and deterministic red-team probe execution.

### 3. Ollama (Self-Hosted Local Inference)
- **Models**: Any GGUF model (`llama3.2`, `mistral`, `deepseek-r1`, `phi-3`).
- **Quotas**: Completely unlimited; runs directly on the developer's laptop or workstation GPU/CPU.
- **Application in EvalForge**: Guaranteed 100% offline, private, zero-cost evaluation for air-gapped or sensitive proprietary datasets.

---

## 4. Engineering Solutions for Free-Tier Constraints

Deploying on free-tier infrastructure requires deliberate engineering to navigate resource boundaries:

### 4.1. Conquering the 512 MB RAM Budget (Render Free Service)
Render's free tier terminates containers that exceed 512 MB of resident memory (OOM Kill). EvalForge prevents this through:
1. **Single Uvicorn Worker**: Running a single asynchronous Uvicorn process (`--workers 1`), consuming only $\approx 65 \text{ MB}$ of baseline memory.
2. **Generator-Based Streaming**: Datasets are never loaded as massive in-memory arrays. They are ingested via SQLAlchemy streaming cursors (`yield_per(100)`).
3. **Payload Truncation & Capping**: Model responses are bounded to a maximum of 50 KB per record. Excessive text outputs are truncated before database insertion.
4. **Garbage Collection Optimization**: Explicit invocation of `gc.collect()` following completion of large evaluation matrices.

### 4.2. Handling Render Cold Starts (~50s Inactivity Sleep)
When a free Render web service receives no HTTP traffic for 15 minutes, it spins down. When a user subsequently visits the app, the request hangs for up to 50 seconds.
- **Client-Side Graceful Loader**: The React SPA detects cold starts via a fast health check timeout (2 seconds). If the backend is sleeping, the UI renders an animated "Waking up server..." status banner with an estimated 45-second progress countdown.
- **Optional Client Keep-Alive Heartbeat**: During active user evaluation sessions in the browser, the frontend sends a lightweight `/api/v1/health` ping every 10 minutes to prevent the backend from sleeping mid-workflow.

### 4.3. Handling Neon Database Auto-Suspend (<1s Reconnection)
Neon PostgreSQL serverless compute automatically suspends after 5 minutes of idle time.
- **Async Connection Retry with Exponential Backoff**: The SQLAlchemy async engine is configured with connection health recycling (`pool_pre_ping=True`) and exponential backoff retry logic. If the initial TCP handshake times out because Neon is waking, SQLAlchemy retries automatically after 500ms, seamlessly absorbing the wake-up latency.

### 4.4. Mitigating In-Flight Task Loss on Container Restart
Because Render free instances can restart during deployment or idle sleep:
- All task state is stored in the PostgreSQL `job_queue` table with atomic state transitions.
- If a worker crashes mid-task, the locked row's heartbeat (`locked_at`) expires after 5 minutes. A startup watchdog query automatically resets expired `RUNNING` tasks back to `PENDING` for clean re-execution.

---

## 5. Step-by-Step Zero-Cost Production Deployment Blueprint

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Zero-Cost Deployment Topology & URLs                     │
│                                                                             │
│  [User Browser] ──> https://evalforge.vercel.app (Vercel CDN)               │
│                            │                                                │
│                            ▼ (CORS API Requests)                            │
│                     https://evalforge-api.onrender.com (Render Free Tier)   │
│                            │                                                │
│                            ▼ (Encrypted PostgreSQL Connection)               │
│                     ep-cool-butterfly-1234.us-east-2.aws.neon.tech          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Step 1: Provision Free Database on Neon.tech
1. Create a free account on [Neon.tech](https://neon.tech).
2. Create project: `evalforge-db` (Region: `us-east-2` or closest to Render).
3. Enable extension: Run `CREATE EXTENSION IF NOT EXISTS vector;` in the Neon SQL Console.
4. Copy the connection string: `postgresql://user:pass@ep-xyz.us-east-2.aws.neon.tech/neondb?sslmode=require`.

### Step 2: Deploy Backend Web Service on Render.com
1. Create a free account on [Render.com](https://render.com).
2. Connect your GitHub repository and select `New Web Service`.
3. Configure environment settings:
   - **Environment**: `Python`
   - **Build Command**: `pip install -r requirements.txt && alembic upgrade head`
   - **Start Command**: `uvicorn evalforge.main:app --host 0.0.0.0 --port $PORT --workers 1`
   - **Instance Type**: `Free (512 MB RAM, 0.1 CPU)`
4. Set Environment Variables:
   - `DATABASE_URL`: *(Neon connection string with `postgresql+asyncpg://` prefix)*
   - `SECRET_KEY`: *(Generate via `openssl rand -hex 32`)*
   - `ENCRYPTION_MASTER_KEY`: *(Generate via `openssl rand -hex 32` for AES-256 vault)*
   - `CORS_ORIGINS`: `https://evalforge.vercel.app`

### Step 3: Deploy Frontend SPA on Vercel
1. Create a free account on [Vercel.com](https://vercel.com).
2. Import the frontend directory `evalforge/frontend`.
3. Configure build settings:
   - **Framework Preset**: `Vite`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`
4. Set Environment Variable:
   - `VITE_API_BASE_URL`: `https://evalforge-api.onrender.com/api/v1`
5. Click **Deploy**. Vercel will deploy globally across its edge network with free SSL.

---

## 6. Deployment Feasibility & Trade-Off Summary

Deploying at ₹0/month introduces specific trade-offs compared to a $500/month enterprise Kubernetes cluster:

| Capability | Zero-Cost Deployment (Free Tier) | Paid Dedicated Infrastructure |
|---|---|---|
| **Monthly Cost** | **₹0.00 / $0.00** | $150 – $800+ / month |
| **Cold Starts** | ~50s after 15 min idle | 0s (Always hot instances) |
| **Max Concurrent Tests** | 3 to 5 simultaneous requests | 50 to 500+ simultaneous requests |
| **Database Storage** | 1 GB (~500,000 evaluated records) | Hundreds of GBs / TBs |
| **Background Processing**| Embedded Async Worker coroutine | Distributed Celery / Redis cluster |
| **SLA & Uptime** | No formal SLA (Best effort) | 99.9% Uptime Guarantee |

**Verdict**: The zero-cost architecture is exceptionally well-suited for individual developers, small engineering teams, open-source projects, and pre-revenue startups. When a team scales, EvalForge's modular architecture enables a seamless upgrade to paid compute (Render Starter or AWS ECS) with zero code changes.
