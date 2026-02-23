# Technical Architecture — Annuity Advisor Platform

## System Overview

```
┌────────────────────────────────────────────────────────────┐
│                        Client Browser                       │
└───────────────────────────┬────────────────────────────────┘
                            │ HTTPS
┌───────────────────────────▼────────────────────────────────┐
│              Next.js Frontend (Vercel / ECS)                │
│         App Router · TypeScript · Tailwind CSS              │
└───────────────────────────┬────────────────────────────────┘
                            │ REST / JSON (HTTPS)
┌───────────────────────────▼────────────────────────────────┐
│                Python FastAPI Backend (ECS)                  │
│           SQLAlchemy · Pydantic · Alembic                   │
└──────┬──────────────┬──────────────┬───────────────────────┘
       │              │              │
┌──────▼──────┐ ┌─────▼──────┐ ┌────▼────────────────────────┐
│  PostgreSQL  │ │   Redis    │ │  S3 Object Storage           │
│  (primary   │ │ (sessions/ │ │  (documents/files)           │
│   data)     │ │  caching)  │ │                              │
└─────────────┘ └────────────┘ └─────────────────────────────┘
                                      │
┌─────────────────────────────────────▼──────────────────────┐
│                  External Services                          │
│  Claude API (AI features) · SendGrid (email PIN + alerts)  │
└─────────────────────────────────────────────────────────────┘
```

---

## Frontend — Next.js

### Stack
- **Framework**: Next.js 14+ with App Router
- **Language**: TypeScript (strict mode)
- **Styling**: Tailwind CSS + shadcn/ui component library
- **State Management**: React Query (server state) + Zustand (UI state)
- **Forms**: React Hook Form + Zod validation
- **HTTP Client**: Axios with interceptors for auth token injection
- **PDF Generation**: react-pdf for quote/application PDFs

### Directory Structure
```
src/
  app/                    # Next.js App Router pages
    (auth)/               # Login, reset password (unauthenticated layout)
    (dashboard)/          # Authenticated layout with nav
      dashboard/
      clients/
      products/
      quotes/
      applications/
      documents/
      commissions/
      reports/
      admin/
  components/
    ui/                   # shadcn/ui primitives
    shared/               # reusable app components
    forms/                # form field components
  lib/
    api/                  # API client functions
    auth/                 # Auth utilities, token management
    utils/
  types/                  # Shared TypeScript types
  hooks/                  # Custom React hooks
```

### Rendering Strategy
- **Static**: Marketing/public pages (none in v1)
- **Server-side (SSR)**: Initial authenticated page loads (avoids flash of unauthenticated content)
- **Client-side**: Data-heavy interactive views (quote engine, application wizard)

---

## Backend — Python FastAPI

### Stack
- **Framework**: FastAPI
- **ORM**: SQLAlchemy 2.0 (async)
- **Validation**: Pydantic v2
- **Migrations**: Alembic
- **Auth**: python-jose (JWT), passlib (bcrypt password hashing)
- **File storage**: boto3 (S3 compatible)
- **Email**: SendGrid (or SMTP via FastMail)
- **Task queue**: Celery + Redis (async jobs: email, PDF generation, status sync)
- **Testing**: pytest + httpx (async test client)

### Directory Structure
```
app/
  api/
    v1/
      routers/
        auth.py
        clients.py
        products.py
        quotes.py
        applications.py
        documents.py
        commissions.py
        reports.py
        admin.py
  core/
    config.py             # Settings via pydantic-settings
    security.py           # JWT helpers, password hashing
    database.py           # SQLAlchemy engine + session factory
  models/                 # SQLAlchemy ORM models
  schemas/                # Pydantic request/response schemas
  services/               # Business logic layer
  tasks/                  # Celery background tasks
  migrations/             # Alembic migration files
  tests/
main.py
```

### API Versioning
All routes namespaced under `/api/v1/`. Breaking changes will introduce `/api/v2/`.

---

## Database — PostgreSQL

- Version: PostgreSQL 15+
- Connection pooling via PgBouncer (production)
- Read replica for reporting queries
- Automated backups: daily snapshots with 30-day retention
- Encryption at rest enabled

See [database-schema.md](./database-schema.md) for full schema.

---

## Authentication & Authorization

### Passwordless PIN Flow
1. Agent POSTs their email to `POST /api/v1/auth/request-pin`
2. Backend generates a cryptographically random 6-digit PIN, stores its bcrypt hash with a 10-minute TTL, and sends it via SendGrid email
3. Agent POSTs email + PIN to `POST /api/v1/auth/verify-pin`
4. Backend verifies the PIN hash, returns `access_token` (15-min JWT) + `refresh_token` (7-day, stored in httpOnly cookie)
5. Frontend stores `access_token` in memory (not localStorage)
6. Axios interceptor attaches `Authorization: Bearer <token>` to every request
7. On 401, interceptor uses refresh token to obtain new access token silently
8. Logout invalidates refresh token server-side

**Security details:**
- PIN is single-use; consumed immediately on verification
- 5 failed PIN attempts locks the account for 15 minutes (tracked in Redis)
- PIN request rate-limited: 3 requests per email per 10-minute window

### Role Enforcement
- FastAPI dependency `require_role(roles: list[Role])` applied at router level
- Row-level: agents can only access their own clients; agency admins scoped to their agency

---

## AI Integration — Claude API

### Overview
AI features are powered by the Anthropic Claude API, called server-side (never exposing the API key to the browser).

### Architecture
- All AI calls are made from `app/services/ai_service.py`
- Prompts are versioned and stored in `app/prompts/` as Jinja2 templates
- Responses are streamed to the frontend via Server-Sent Events (SSE) for the chat assistant
- Non-streaming calls (recommendation, suitability check, extraction) are standard async requests

### Features and Prompt Strategy

| Feature | Model | Context injected |
|---|---|---|
| Product recommendation | claude-sonnet-4-6 | Client financial profile, all available products |
| Suitability compliance check | claude-sonnet-4-6 | Application data, product details, suitability rules |
| Document data extraction | claude-sonnet-4-6 | Document text (extracted via PDF parser) |
| Quote explainer | claude-haiku-4-5 | Quote inputs and output projections |
| Advisor chat assistant | claude-sonnet-4-6 | Current page context, conversation history |

### Cost Controls
- Per-user daily token budget tracked in Redis; requests exceeding budget return a graceful degradation message
- Chat assistant conversation history truncated to last 10 messages
- Extraction and recommendation calls cached by input hash for 1 hour (Redis)

---

## File Storage — S3

- Bucket structure: `/{env}/clients/{client_id}/documents/{doc_id}`
- Uploads: backend generates pre-signed PUT URL → frontend uploads directly to S3 → backend records metadata
- Downloads: backend generates short-lived pre-signed GET URL (15 min TTL)
- All access logged via S3 server access logging

---

## Background Jobs — Celery + Redis

| Task | Trigger |
|---|---|
| Send email notification | Application status change, signature request |
| Generate application PDF | Application submitted |
| Document expiration check | Daily cron |
| Commission sync | Nightly (future: carrier webhook) |

---

## Infrastructure

### Environments
- `development`: Local Docker Compose (PostgreSQL, Redis, MinIO for S3)
- `staging`: AWS ECS Fargate + RDS + ElastiCache
- `production`: AWS ECS Fargate + RDS Multi-AZ + ElastiCache

### Docker Compose (Local)
```yaml
services:
  frontend:    # Next.js dev server
  backend:     # FastAPI with uvicorn --reload
  db:          # PostgreSQL 15
  redis:       # Redis 7
  minio:       # S3-compatible local storage
  worker:      # Celery worker
```

### CI/CD
- GitHub Actions: lint → test → build → deploy
- Branch protection on `main`; all changes via PR
- Staging auto-deploy on merge to `main`
- Production deploy via manual approval gate

---

## Security Considerations

- All API endpoints require authentication except `/auth/login` and `/auth/refresh`
- CORS restricted to known frontend origins
- Rate limiting on auth endpoints (10 req/min per IP)
- Input validation via Pydantic on all API inputs
- Parameterized queries via SQLAlchemy (no raw SQL string interpolation)
- HTTPS enforced everywhere; HSTS headers set
- SSN and sensitive fields encrypted at the application layer before database storage (AES-256)
- Audit log table records all data mutations with user ID and timestamp
