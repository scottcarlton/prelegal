# Annuity Advisor Application — Planning Documents

This directory contains all planning and design documentation for the annuity advisor/agent platform.

## Documents

| Document | Description |
|---|---|
| [PLAN.md](./PLAN.md) | Step-by-step implementation plan |
| [product-spec.md](./product-spec.md) | Product requirements and feature specification |
| [architecture.md](./architecture.md) | Technical architecture and system design |
| [database-schema.md](./database-schema.md) | Database schema and data model |
| [api-design.md](./api-design.md) | REST API design and endpoint reference |
| [ui-ux.md](./ui-ux.md) | UI/UX layout and wireframe descriptions |
| [testing-plan.md](./testing-plan.md) | Testing strategy and coverage plan |

## Overview

A web-based platform that enables annuity advisors and agents to:
- Manage client profiles and relationships
- Browse and compare annuity products
- Generate accurate quotes
- Submit and track applications
- Manage required compliance documents
- View commission and pipeline reports

## Stack

- **Frontend**: Next.js (App Router), TypeScript, Tailwind CSS
- **Backend**: Python (FastAPI), SQLAlchemy
- **Database**: PostgreSQL
- **Storage**: S3-compatible object storage
- **Auth**: JWT-based authentication
