# ADR 005: Zero-Cost Deployment Topology (Vercel + Render + Neon + BYOK)

| Metadata | Value |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-11-17 |
| **Deciders** | Lead Software Architect & AI Infrastructure Engineer |

---

## 1. Context & Problem Statement

A core requirement for EvalForge is to be deployable at **₹0 / $0 per month** using legitimate, sustainable free-tier cloud infrastructure, without relying on ephemeral trial credits.

We need to select the optimal topology for:
1. Frontend web application hosting.
2. Backend API and worker process execution.
3. Database and vector storage.
4. AI model inference costs.

---

## 2. Decision Drivers

1. **Zero Cost (₹0 / $0 monthly bill)**: No hidden fees, credit card requirements, or unexpected overage bills.
2. **Ease of Deployment**: Git-based continuous deployment workflows.
3. **Global Performance**: Fast global CDN delivery for frontend static assets.
4. **Relational & Vector Support**: Fully managed PostgreSQL supporting the `pgvector` extension.
5. **BYOK Privacy & Cost Shifting**: Eliminating platform-level LLM costs by having users bring their own API keys or run local models.

---

## 3. Considered Options

- **Option 1: Split Jamstack & Serverless PaaS (Vercel / Cloudflare Pages + Render + Neon)**
- **Option 2: Single All-in-One PaaS (Railway / Fly.io)**
- **Option 3: Self-Hosted Single VPS (Hetzner / DigitalOcean $4-$6/month)**
- **Option 4: AWS Free Tier (EC2 t2.micro + RDS PostgreSQL db.t3.micro)**

---

## 4. Decision & Justification

**Chosen Option: Option 1 — Split Jamstack & Serverless PaaS (Vercel / Cloudflare Pages + Render + Neon).**

### Breakdown by Layer:
1. **Frontend**: Hosted on **Vercel Hobby** or **Cloudflare Pages**.
   - **Cost**: ₹0.00 / $0.00.
   - **Advantage**: Edge CDN distribution, instant cache invalidation, free SSL, zero cold starts for UI assets.
2. **Backend**: Hosted on **Render Free Web Service**.
   - **Cost**: ₹0.00 / $0.00.
   - **Advantage**: 750 free instance hours/month (runs one service 24/7). 512 MB RAM is sufficient for our single-worker async FastAPI backend. Automatic Git-push redeployments.
3. **Database**: Hosted on **Neon Serverless PostgreSQL**.
   - **Cost**: ₹0.00 / $0.00.
   - **Advantage**: 1 GB storage (~500,000 evaluation results), 100 CU-hours/month, serverless auto-suspend, and native `pgvector` extension included.
4. **AI Inference**: **Bring Your Own Key (BYOK)**.
   - **Cost**: ₹0.00 / $0.00.
   - **Advantage**: Zero server-side AI API billing. Users connect free Google Gemini (15 RPM / 1M TPM), Groq (30 RPM), or local Ollama instances.

### Rejection of Alternatives:
- **Railway**: Ended its permanent free tier in 2023; now charges a minimum platform fee.
- **Fly.io**: Requires credit card on file and frequently changes free allowance limits.
- **AWS Free Tier**: Expires after 12 months, and accidental overages (e.g., NAT gateways, EBS snapshots) frequently result in surprise billing.
- **Self-Hosted VPS**: Incurs fixed monthly costs ($4 to $10/month) and introduces system administration burden (OS patching, SSL renewals, backups).

---

## 5. Consequences

### Positive:
- True ₹0 / $0 monthly infrastructure cost.
- High scalability: Frontend is globally cached on edge networks.
- Zero maintenance overhead: Database backups and OS patches are managed by Neon and Render.

### Negative / Trade-offs:
- **Render Cold Starts**: After 15 minutes of inactivity, Render spins down the backend. The next incoming request experiences a ~50-second wake-up delay.
  - *Mitigation*: Client-side animated waking loader and optional 10-minute client keep-alive pings during active testing sessions.
- **Neon Auto-Suspend**: Database suspends after 5 minutes of idle time.
  - *Mitigation*: SQLAlchemy connection pool pre-ping with automatic exponential backoff retry.
- **512 MB RAM Cap**: Backend container will be killed if memory exceeds 512 MB.
  - *Mitigation*: Generator-based dataset streaming, single Uvicorn worker, bounded concurrency (max 3-5 tasks), and payload truncation.
