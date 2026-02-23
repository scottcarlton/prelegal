# Product Specification — Annuity Advisor Platform

## Purpose

Enable licensed annuity advisors and agents to manage the full sales lifecycle — from client onboarding through quote generation, application submission, and commission tracking — in a single, compliant platform.

## Users

| Role | Description |
|---|---|
| **Agent** | Front-line advisor who manages clients and submits applications |
| **Agency Admin** | Manages agents within their agency, views aggregate reports |
| **Super Admin** | Platform administrator; manages carriers, products, and user accounts |

---

## Feature Modules

### 1. Authentication & Access Control

- **Passwordless PIN login**: Agent enters email → receives a 6-digit one-time PIN via email → enters PIN to authenticate
- PINs are single-use, expire after 10 minutes, and are rate-limited (5 attempts before lockout)
- Role-based access control (Agent, Agency Admin, Super Admin)
- Session timeout after inactivity (30 minutes)
- Audit log of all login activity including failed PIN attempts

### 2. Dashboard

The default landing page after login. Provides an at-a-glance summary:

- **Pipeline summary**: Active quotes, pending applications, recently completed
- **Recent activity feed**: Last 10 actions (new client, quote generated, application submitted, etc.)
- **Commission snapshot**: Month-to-date and year-to-date pending/paid commissions
- **Alerts**: Applications requiring attention (missing documents, awaiting signature, declined)
- **Quick actions**: New Client, New Quote, New Application buttons

### 3. Client Management

#### Client Profile
- Full name, date of birth, SSN (masked), contact info
- Beneficiary information
- Financial profile: income, assets, risk tolerance, investment objectives
- Suitability questionnaire responses
- Notes and activity history

#### Client List
- Searchable, filterable table of all clients
- Sort by name, last activity, open applications
- Status badges (Active, Pending, Archived)

#### Operations
- Create, view, edit, archive clients
- Link a client to one or more quotes/applications

### 4. Product Catalog

Annuity product types supported:
- **Fixed Annuity (FA)**: Guaranteed interest rate
- **Fixed Indexed Annuity (FIA)**: Returns linked to a market index with floor protection
- **Variable Annuity (VA)**: Sub-account based, market exposure

#### Product Card includes:
- Carrier name and product name
- Product type
- Minimum premium
- Surrender charge schedule
- Available riders (income, death benefit, LTC)
- State availability
- Commission rate

#### Operations
- Browse and filter products by type, carrier, minimum premium, state
- Compare up to 3 products side-by-side

### 5. Quote Engine

- Select a client and a product to begin
- Input fields:
  - Premium amount
  - Purchase type (single premium / flexible premium)
  - Payout option (immediate / deferred)
  - Deferral period (if deferred)
  - Riders to include
- Output:
  - Projected accumulation value at 1, 5, 10, 20 years
  - Guaranteed vs illustrated income stream
  - Rider costs breakdown
  - Commission estimate
- Save quote to client profile
- Print/export quote as PDF

### 6. Application Workflow

Multi-step form wizard for submitting an annuity application:

**Step 1 — Select client and product** (pre-filled from quote if applicable)

**Step 2 — Owner & annuitant info**
- Owner(s): primary and optional joint owner
- Annuitant (may differ from owner)
- Beneficiary designations (primary + contingent)

**Step 3 — Funding details**
- Funding source: new money, 1035 exchange, rollover
- Bank account / transfer instructions (for exchanges)
- Initial premium amount

**Step 4 — Rider elections**
- Select optional income, death benefit, or LTC riders
- Acknowledge rider costs and terms

**Step 5 — Suitability & compliance**
- Suitability questionnaire (auto-filled from client profile, editable)
- Agent attestation and e-signature
- Client e-signature request (via email link)

**Step 6 — Document upload**
- Required: government-issued ID, suitability form
- Conditional: 1035 exchange form, transfer paperwork, trust documents
- Drag-and-drop upload, file type and size validation

**Step 7 — Review & submit**
- Full application summary
- Submit to carrier (via carrier API or manual review queue)

#### Application Statuses
`Draft` → `Pending Signatures` → `Submitted` → `In Review` → `Approved` / `Declined` / `Returned`

### 7. Document Management

- Per-client document vault
- Per-application document checklist with status (uploaded, pending, approved)
- Document types: ID, suitability form, 1035 exchange, trust docs, death certificates, etc.
- Secure download with access logging
- Document expiration tracking (e.g., suitability forms expire after 12 months)

### 8. Commission Tracking

- Commission ledger per agent
- Per-application: expected vs actual commission amount, status (pending, approved, paid)
- Chargeback tracking for policies that lapse within charge-back period
- Monthly statement view
- Export to CSV

### 9. Notifications

In-app notification center + email notifications for:
- Application status changes
- Documents missing or expiring
- Client e-signature completed or overdue
- Commission paid or chargebacked

### 10. AI Assistant

The platform integrates an AI layer powered by a large language model (Claude API) to assist advisors at key points in the workflow.

#### Product Recommendation Engine
- After a client profile and financial goals are entered, AI analyzes the profile and recommends the top 3 most suitable annuity products with a plain-language rationale
- Rationale explains the match (e.g., "SecureIncome 7's 0% floor protects against loss, aligning with John's conservative risk tolerance")

#### Suitability Compliance Checker
- Before an advisor submits an application, AI reviews the suitability questionnaire responses against the selected product
- Flags potential mismatches (e.g., aggressive product selected for a conservative client) and suggests corrective action
- Advisor must acknowledge or override each flag before submission

#### Document Data Extraction
- When a document (e.g., 1035 exchange form, existing policy statement) is uploaded, AI extracts key fields (policy number, surrender value, carrier name) and pre-populates the corresponding application fields
- Advisor reviews and confirms extracted data before saving

#### Quote Explainer
- On the quote results screen, AI generates a 2–3 sentence plain-English summary of the quote for the advisor to read to or share with the client
- Adjusts reading level to be jargon-free

#### Advisor Chat Assistant
- Persistent chat widget available on all pages
- Advisor can ask questions like: "What's the difference between a fixed and fixed-indexed annuity?", "What documents are required for a 1035 exchange?", "How do I explain surrender charges to a client?"
- Context-aware: knows which client/application the advisor is currently viewing and can answer accordingly
- Does not provide regulated financial advice; responses include appropriate disclaimers

### 12. Reporting (Agency Admin / Super Admin)

- Agent production report: applications submitted, approved, total premium by agent
- Pipeline report: applications by status
- Product report: premium volume by product/carrier
- Commission report: payout totals by period
- Date range filters, export to CSV/PDF

---

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Availability | 99.9% uptime |
| Page load | < 2s for all primary views |
| File upload | Up to 25 MB per file |
| Data retention | 7 years (regulatory) |
| Encryption | TLS in transit, AES-256 at rest |
| Compliance | SOC 2 Type II readiness; FINRA/state annuity regulations |

---

## Out of Scope (v1)

- Carrier direct integration (applications submitted to manual review queue initially)
- Mobile native app
- CRM integrations (Salesforce, Redtail)
- Policy servicing (post-issue changes, surrenders)
