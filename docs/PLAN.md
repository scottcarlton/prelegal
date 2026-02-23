# Implementation Plan — Annuity Advisor Platform

Each step is a discrete, shippable unit of work. Steps are ordered so that every phase builds on the previous one. Relevant planning documents are called out per step.

---

## Step 1 — Project Scaffold & Local Environment

**Goal**: Both the frontend and backend repos boot locally with a single command.

- Initialize monorepo structure (`/frontend`, `/backend`)
- Create `docker-compose.yml` with services: `frontend`, `backend`, `db` (PostgreSQL 15), `redis`, `minio` (S3-compatible), `worker` (Celery)
- Add `.env.example` files for both apps
- Configure GitHub repository, branch protection on `main`, and PR-required workflow
- Set up GitHub Actions CI skeleton (lint, test, build jobs — initially passing with no-ops)

**References**:
- `architecture.md` → Docker Compose (Local), Infrastructure, CI/CD sections

---

## Step 2 — Backend Foundation

**Goal**: FastAPI app runs, connects to the database, and is structured for the full build.

- Initialize FastAPI app with `app/` directory structure (`api/`, `core/`, `models/`, `schemas/`, `services/`, `tasks/`, `tests/`)
- Configure `pydantic-settings` for environment-based config (`core/config.py`)
- Set up SQLAlchemy async engine + session factory (`core/database.py`)
- Set up Alembic for migrations
- Add health check endpoint `GET /health`
- Wire pytest + `pytest-asyncio` with a test database session fixture (transactions rolled back per test)
- Configure `pip-audit` in CI

**References**:
- `architecture.md` → Backend — Python FastAPI, Directory Structure
- `testing-plan.md` → Testing layers overview, CI/CD Integration

---

## Step 3 — Database Schema: Core Tables

**Goal**: All database tables exist and are managed via Alembic migrations.

- Create and run migrations for all tables:
  - `agencies`, `users`, `login_pins`, `refresh_tokens`
  - `carriers`, `products`
  - `clients`, `client_beneficiaries`
  - `quotes`
  - `applications`
  - `documents`
  - `commissions`
  - `notifications`
  - `audit_logs`
  - AI tables: `ai_recommendations`, `ai_suitability_flags`, `ai_chat_sessions`
- Write SQLAlchemy ORM models for every table
- Write pytest unit tests for model constraints (e.g., beneficiary percentages, document owner check)

**References**:
- `database-schema.md` → All tables

---

## Step 4 — Authentication (Passwordless PIN)

**Goal**: An agent can log in via email PIN, receive JWT tokens, and refresh or revoke them.

**Backend**:
- `POST /api/v1/auth/request-pin` — generate PIN, hash it, store in `login_pins`, send via email (SendGrid)
- `POST /api/v1/auth/verify-pin` — validate PIN hash, issue access token (JWT) + refresh token (httpOnly cookie)
- `POST /api/v1/auth/refresh` — exchange refresh token for new access token
- `POST /api/v1/auth/logout` — revoke refresh token
- Implement rate limiting (Redis): 3 PIN requests/email/10 min; 5 failed verify attempts triggers 15-min lockout
- `GET /api/v1/users/me` — authenticated user profile
- Role-based `require_role()` FastAPI dependency

**Backend tests**:
- PIN request success, unknown email (always 200), rate limit enforcement
- PIN verify: success, expired, already used, wrong PIN, lockout after 5 attempts
- Token refresh: success, revoked token rejected
- `require_role()` enforcement per role

**References**:
- `product-spec.md` → Authentication & Access Control
- `architecture.md` → Passwordless PIN Flow
- `database-schema.md` → `login_pins`, `refresh_tokens`, `users` tables
- `api-design.md` → Authentication — `/auth`
- `testing-plan.md` → Unit Tests (Auth/security), Integration Tests (Auth module)

---

## Step 5 — Users & Agency Management

**Goal**: Super admin can manage agencies and users; agents can view and update their own profile.

**Backend**:
- `GET /api/v1/users/me`, `PATCH /api/v1/users/me`
- `GET /api/v1/users`, `POST /api/v1/users`, `GET /api/v1/users/{id}`, `PATCH /api/v1/users/{id}`, `DELETE /api/v1/users/{id}` (soft-deactivate)
- Admin endpoints: `GET/POST/PATCH /api/v1/admin/agencies`
- Audit log writes on all mutations

**References**:
- `product-spec.md` → Users section
- `api-design.md` → Users `/users`, Admin `/admin`
- `database-schema.md` → `users`, `agencies`, `audit_logs`

---

## Step 6 — Carrier & Product Catalog

**Goal**: Products exist in the database and are browsable via the API.

**Backend**:
- `GET /api/v1/products` with filtering (type, state, carrier, min premium)
- `GET /api/v1/products/{id}`
- `POST /api/v1/products/compare` (up to 3 products)
- Admin endpoints: `GET/POST/PATCH /api/v1/admin/carriers`, `GET/POST/PATCH /api/v1/admin/products`
- Seed script for initial carrier and product data

**Backend tests**:
- Product list filtering (by type, state, min premium)
- Compare endpoint validates max 3 products

**References**:
- `product-spec.md` → Product Catalog
- `api-design.md` → Products `/products`, Admin `/admin`
- `database-schema.md` → `carriers`, `products`

---

## Step 7 — Client Management

**Goal**: Agents can create and manage clients and their beneficiaries.

**Backend**:
- `GET /api/v1/clients`, `POST /api/v1/clients`
- `GET /api/v1/clients/{id}`, `PATCH /api/v1/clients/{id}`, `DELETE /api/v1/clients/{id}` (archive)
- `GET /api/v1/clients/{id}/beneficiaries`, `PUT /api/v1/clients/{id}/beneficiaries`
- Agent-scoping: agents only see their own clients; agency_admin sees all in agency
- SSN encrypted at application layer before storage

**Backend tests**:
- Agent A cannot view Agent B's client (returns 404)
- Beneficiary percentages must sum to 100 per type
- SSN is masked in response schema

**References**:
- `product-spec.md` → Client Management
- `api-design.md` → Clients `/clients`
- `database-schema.md` → `clients`, `client_beneficiaries`
- `testing-plan.md` → Integration Tests (Clients module)

---

## Step 8 — Quote Engine

**Goal**: Agents can generate, save, and export annuity quotes.

**Backend**:
- Quote calculation service: projections (guaranteed vs illustrated) for fixed, FIA, and variable product types
- `POST /api/v1/quotes` — calculate + save
- `GET /api/v1/quotes`, `GET /api/v1/quotes/{id}`
- `PATCH /api/v1/quotes/{id}`, `DELETE /api/v1/quotes/{id}` (expire)
- `GET /api/v1/quotes/{id}/pdf` — trigger PDF generation via Celery + return pre-signed S3 URL
- Quote auto-expires after 30 days

**Backend tests**:
- Projection math verified against known expected values per product type
- Commission estimate applies correct base rate
- Quote expiry logic

**References**:
- `product-spec.md` → Quote Engine
- `api-design.md` → Quotes `/quotes`
- `database-schema.md` → `quotes`
- `testing-plan.md` → Unit Tests (Service layer — quote tests)

---

## Step 9 — Application Workflow

**Goal**: Agents can create and submit applications; clients can sign via email link.

**Backend**:
- `POST /api/v1/applications`, `GET /api/v1/applications`, `GET /api/v1/applications/{id}`, `PATCH /api/v1/applications/{id}`
- `POST /api/v1/applications/{id}/sign` — agent e-signature + generate client sign token, send email
- `GET /api/v1/applications/sign/{token}` (public) — validate token + return summary
- `POST /api/v1/applications/sign/{token}` (public) — record client signature, advance status to `submitted`
- `POST /api/v1/applications/{id}/submit` (admin) — manual status advancement
- `GET /api/v1/applications/{id}/pdf`
- Status machine: `draft → pending_signatures → submitted → in_review → approved/declined/returned`
- Celery task: generate application PDF on submission

**Backend tests**:
- Full status transition sequence
- Expired/invalid client sign token rejected
- Agent cannot access another agent's application

**References**:
- `product-spec.md` → Application Workflow
- `api-design.md` → Applications `/applications`
- `database-schema.md` → `applications`
- `testing-plan.md` → Integration Tests (Applications module)

---

## Step 10 — Document Management

**Goal**: Agents can upload, view, and download compliance documents linked to clients or applications.

**Backend**:
- `POST /api/v1/documents/upload-url` — generate pre-signed S3 PUT URL + create pending document record
- `POST /api/v1/documents/{id}/confirm` — verify S3 upload completed, update record
- `GET /api/v1/documents` (filterable), `GET /api/v1/documents/{id}/download-url`, `DELETE /api/v1/documents/{id}`
- Document expiration tracking (suitability forms expire after 12 months)
- Celery daily cron: flag expiring documents + send notifications

**References**:
- `product-spec.md` → Document Management
- `api-design.md` → Documents `/documents`
- `database-schema.md` → `documents`
- `architecture.md` → File Storage — S3

---

## Step 11 — Commission Tracking & Notifications

**Goal**: Commission records are created on application approval and are viewable by agents.

**Backend**:
- Commission records auto-created when application moves to `approved`
- `GET /api/v1/commissions`, `GET /api/v1/commissions/{id}`, `GET /api/v1/commissions/export` (CSV)
- Notification system: create records in `notifications` table on key events (application status change, document missing, commission paid)
- `GET /api/v1/notifications`, `POST /api/v1/notifications/{id}/read`, `POST /api/v1/notifications/read-all`
- Celery tasks: send email notifications via SendGrid

**References**:
- `product-spec.md` → Commission Tracking, Notifications
- `api-design.md` → Commissions `/commissions`, Notifications `/notifications`
- `database-schema.md` → `commissions`, `notifications`

---

## Step 12 — Reporting (Admin)

**Goal**: Agency admins and super admins can view production, pipeline, and commission reports.

**Backend**:
- `GET /api/v1/reports/production` — agent production summary with date range filter
- `GET /api/v1/reports/pipeline` — applications grouped by status
- `GET /api/v1/reports/commissions` — payout totals by period and agent
- All reports support `?format=csv` download
- Read replica used for reporting queries

**References**:
- `product-spec.md` → Reporting
- `api-design.md` → Reports `/reports`

---

## Step 13 — AI Integration

**Goal**: All five AI features are live and integrated end-to-end with the backend.

**Backend**:
- Set up `app/services/ai_service.py` with Anthropic SDK client
- Prompt templates (Jinja2) for each feature stored in `app/prompts/`
- Per-user daily token budget tracked in Redis
- Result caching in Redis (1-hour TTL, keyed by input hash)
- `POST /api/v1/ai/recommend-products` — product recommendation (claude-sonnet-4-6)
- `POST /api/v1/ai/check-suitability` — compliance flag check (claude-sonnet-4-6)
- `POST /api/v1/ai/acknowledge-suitability-flag` — record advisor override
- `POST /api/v1/ai/extract-document` — extract fields from uploaded document (claude-sonnet-4-6)
- `POST /api/v1/ai/explain-quote` — plain-English quote summary (claude-haiku-4-5)
- `POST /api/v1/ai/chat` — start chat session
- `POST /api/v1/ai/chat/{session_id}/message` — SSE streaming response (claude-sonnet-4-6)
- `DELETE /api/v1/ai/chat/{session_id}` — clear session

**Backend tests** (Claude API mocked):
- Recommendation returns ranked list
- Suitability check flags product mismatch
- Budget exceeded blocks request
- Cache hit skips API call
- SSE stream emits correct event format

**References**:
- `product-spec.md` → AI Assistant (Module 10)
- `architecture.md` → AI Integration — Claude API
- `database-schema.md` → `ai_recommendations`, `ai_suitability_flags`, `ai_chat_sessions`
- `api-design.md` → AI `/ai`
- `testing-plan.md` → AI Feature Tests (Section 4)

---

## Step 14 — Frontend Foundation

**Goal**: Next.js app boots, authenticates via PIN, and shows the authenticated shell with navigation.

**Frontend**:
- Initialize Next.js 14 App Router project with TypeScript, Tailwind CSS, shadcn/ui
- Set up Axios instance with JWT interceptor (attach token, silent refresh on 401)
- Zustand store for auth state (user, access token in memory)
- React Query client setup
- Two-step PIN login page:
  - Step 1: email input → `POST /auth/request-pin`
  - Step 2: 6-box PIN input with countdown timer → `POST /auth/verify-pin`
  - Lockout and error states
- Authenticated shell: top nav (role-aware links), notification bell, avatar dropdown
- Route protection: redirect unauthenticated users to login with return URL
- Profile page (`/profile`) wired to `GET/PATCH /users/me`

**Frontend tests**:
- PIN form auto-advances box focus on input
- Shows countdown timer that decrements
- Expired/wrong PIN shows correct error states
- Unauthenticated route redirects to login

**References**:
- `architecture.md` → Frontend — Next.js, Directory Structure, Authentication Flow
- `ui-ux.md` → Layout, Login (Passwordless PIN)
- `api-design.md` → Authentication `/auth`, Users `/users`
- `testing-plan.md` → Frontend unit tests, E2E Authentication scenarios

---

## Step 15 — Frontend: Clients

**Goal**: Agents can browse, create, and manage clients in the UI.

**Frontend**:
- `/clients` — searchable, filterable client list table
- `/clients/new` — new client form (all fields including financial profile + suitability)
- `/clients/[id]` — client detail: two-column layout with profile, beneficiaries, linked quotes/applications/documents
- `/clients/[id]/edit` — edit client form
- Beneficiary editor: add/remove/edit with live percentage validation

**Frontend tests**:
- Client list search filters results
- Beneficiary percentages must sum to 100 before save is enabled

**References**:
- `product-spec.md` → Client Management
- `ui-ux.md` → Client List, Client Detail
- `api-design.md` → Clients `/clients`

---

## Step 16 — Frontend: Products & Quote Builder

**Goal**: Agents can browse products, compare them, and generate quotes.

**Frontend**:
- `/products` — product catalog with filter panel (type, state, carrier, min premium); product cards
- `/products/compare` — side-by-side comparison table for up to 3 products
- `/quotes/new` — quote builder: left panel inputs + right panel live preview (updates on change)
- Quote PDF download button
- Quote saved state visible on client detail page

**Frontend tests**:
- Quote preview updates when premium amount changes
- Compare button disabled when fewer than 2 products selected

**References**:
- `product-spec.md` → Product Catalog, Quote Engine
- `ui-ux.md` → Product Catalog, Quote Builder
- `api-design.md` → Products `/products`, Quotes `/quotes`

---

## Step 17 — Frontend: Application Wizard

**Goal**: Agents can complete and submit the full 7-step application in the UI.

**Frontend**:
- `/applications/new` — 7-step wizard with progress stepper:
  1. Client & Product selection
  2. Owner & Annuitant info
  3. Funding details
  4. Rider elections
  5. Suitability questionnaire
  6. Document upload (drag-and-drop, S3 pre-signed URL flow)
  7. Review & Submit (with confirmation modal)
- `/applications/[id]` — application detail: status badge, all sections read-only, document checklist, signature status, status timeline
- `/applications/sign/[token]` — public client signature page (no auth required)
- Application list at `/applications` with status filter

**Frontend tests**:
- Cannot advance wizard step with invalid/incomplete data
- Confirmation modal appears before final submit
- Document upload completes the `upload-url` → direct S3 PUT → `confirm` flow

**References**:
- `product-spec.md` → Application Workflow
- `ui-ux.md` → Application Wizard, Application Detail
- `api-design.md` → Applications `/applications`
- `testing-plan.md` → E2E Application Workflow scenarios

---

## Step 18 — Frontend: Documents, Commissions & Notifications

**Goal**: Document vault, commission ledger, and in-app notifications are fully functional.

**Frontend**:
- Document vault accessible from client detail and application detail pages
- Commission ledger at `/commissions`: period selector, summary totals, per-application table, CSV export
- Notification bell: unread count badge, dropdown list, mark as read, navigate to related entity on click

**References**:
- `product-spec.md` → Document Management, Commission Tracking, Notifications
- `ui-ux.md` → Commission Ledger
- `api-design.md` → Documents `/documents`, Commissions `/commissions`, Notifications `/notifications`

---

## Step 19 — Frontend: AI Features

**Goal**: All AI-powered panels and the chat assistant are live in the UI.

**Frontend**:
- Client detail page: AI recommendation panel with ranked products + rationale + "Start Quote" shortcut
- Quote results page: plain-English quote explainer below projections table
- Application wizard Step 7: suitability flag panel — display flags, require override reason to proceed
- Application wizard Step 6: document upload triggers AI extraction; pre-fills application fields on complete
- Floating `✨ Ask AI` button on all pages → slide-up chat panel with SSE streaming responses and session persistence

**Frontend tests**:
- Recommendation panel does not render when client profile is incomplete
- Suitability flag blocks submit until acknowledged
- Chat panel streams tokens as they arrive and appends to conversation

**References**:
- `product-spec.md` → AI Assistant (Module 10)
- `ui-ux.md` → AI Features section
- `api-design.md` → AI `/ai`
- `testing-plan.md` → AI Feature Tests E2E scenarios

---

## Step 20 — Reporting (Admin Frontend)

**Goal**: Agency admins and super admins can view and export production, pipeline, and commission reports.

**Frontend**:
- `/reports/production` — agent production table with date range filter
- `/reports/pipeline` — applications by status (chart + table)
- `/reports/commissions` — commission totals by period and agent
- CSV export on all report pages
- Only accessible to `agency_admin` and `super_admin` roles (redirect others)

**References**:
- `product-spec.md` → Reporting
- `api-design.md` → Reports `/reports`
- `ui-ux.md` → Layout (role-aware navigation)

---

## Step 21 — Full Test Suite

**Goal**: All testing layers pass in CI with coverage targets met.

- Backend: `pytest --cov=app --cov-fail-under=80` passes
- Frontend: `jest --coverage` passes
- Security: `pip-audit` and `npm audit --audit-level=high` pass clean
- E2E: all Playwright scenarios in `testing-plan.md` implemented and passing against staging
- AI feature tests: all mocked unit + integration tests passing
- Load test baselines established with k6 (p95 < 500ms under 50 concurrent users)
- OWASP checks documented and passing (IDOR, auth bypass, mass assignment, sensitive data exposure)

**References**:
- `testing-plan.md` → All sections (Unit, Integration, E2E, AI, Security, Performance)

---

## Step 22 — Security Hardening & Non-Functional Requirements

**Goal**: Platform meets security and compliance readiness targets before production.

- Enforce HTTPS + HSTS headers
- CORS locked to known frontend origins
- Rate limiting verified on all auth endpoints
- SSN field encryption tested end-to-end (write encrypted, read decrypted, never plain in logs)
- Audit log coverage verified for all data mutations
- Input validation confirmed: `extra='forbid'` on all Pydantic schemas
- `sqlmap` scan run against staging — no injectable endpoints
- Session timeout (30 min inactivity) enforced on frontend
- Document access logging via S3 server access logs confirmed active
- SOC 2 readiness checklist reviewed

**References**:
- `architecture.md` → Security Considerations
- `product-spec.md` → Non-Functional Requirements
- `testing-plan.md` → Security Tests (Section 5)

---

## Step 23 — Staging Deployment

**Goal**: Full application running on AWS staging environment accessible to internal testers.

- Provision AWS ECS Fargate clusters for frontend (or Vercel) and backend
- Provision RDS PostgreSQL 15 with automated backups
- Provision ElastiCache Redis
- Provision S3 bucket with server access logging
- Set up SendGrid account and verify sending domain
- Configure Anthropic API key in secrets manager
- Run Alembic migrations against staging database
- Deploy via GitHub Actions on merge to `main`
- Smoke test all critical paths against staging

**References**:
- `architecture.md` → Infrastructure, Environments, CI/CD

---

## Step 24 — Production Launch

**Goal**: Platform is live and available to the first cohort of advisors.

- Provision production AWS infrastructure (RDS Multi-AZ, ECS, ElastiCache)
- Run migrations against production database
- Configure production environment variables and secrets
- Enable Dependabot for automated security patch PRs
- Set up CloudWatch alarms for error rate, latency, and CPU
- Onboard first agency: create agency record, create admin user, send PIN to verify login
- Monitor logs and error rates for first 48 hours post-launch

**References**:
- `architecture.md` → Infrastructure, Environments
- `product-spec.md` → Non-Functional Requirements (availability, data retention)

---

## Summary

| Step | Focus Area | Key Docs |
|---|---|---|
| 1 | Project scaffold + Docker + CI | `architecture.md` |
| 2 | FastAPI foundation + test setup | `architecture.md`, `testing-plan.md` |
| 3 | All database tables + migrations | `database-schema.md` |
| 4 | PIN auth (backend) | `product-spec.md`, `architecture.md`, `api-design.md`, `testing-plan.md` |
| 5 | User & agency management | `api-design.md` |
| 6 | Carrier & product catalog | `product-spec.md`, `api-design.md` |
| 7 | Client management (backend) | `product-spec.md`, `api-design.md`, `testing-plan.md` |
| 8 | Quote engine (backend) | `product-spec.md`, `api-design.md`, `testing-plan.md` |
| 9 | Application workflow (backend) | `product-spec.md`, `api-design.md`, `testing-plan.md` |
| 10 | Document management (backend) | `product-spec.md`, `api-design.md`, `architecture.md` |
| 11 | Commissions + notifications | `product-spec.md`, `api-design.md` |
| 12 | Reporting (backend) | `product-spec.md`, `api-design.md` |
| 13 | AI integration (backend) | `product-spec.md`, `architecture.md`, `api-design.md`, `testing-plan.md` |
| 14 | Frontend foundation + PIN login | `architecture.md`, `ui-ux.md`, `api-design.md`, `testing-plan.md` |
| 15 | Frontend: Clients | `product-spec.md`, `ui-ux.md` |
| 16 | Frontend: Products + Quotes | `product-spec.md`, `ui-ux.md` |
| 17 | Frontend: Application wizard | `product-spec.md`, `ui-ux.md`, `testing-plan.md` |
| 18 | Frontend: Documents + Commissions | `product-spec.md`, `ui-ux.md` |
| 19 | Frontend: AI features | `product-spec.md`, `ui-ux.md`, `testing-plan.md` |
| 20 | Frontend: Reporting (admin) | `product-spec.md`, `ui-ux.md` |
| 21 | Full test suite | `testing-plan.md` |
| 22 | Security hardening | `architecture.md`, `testing-plan.md` |
| 23 | Staging deployment | `architecture.md` |
| 24 | Production launch | `architecture.md`, `product-spec.md` |
