# Database Schema — Annuity Advisor Platform

PostgreSQL 15+. All tables include `created_at` and `updated_at` timestamps. UUIDs used for primary keys.

---

## Entity Relationship Overview

```
agencies ──< users (agents)
users ──< clients
users ──< quotes
users ──< applications
clients ──< quotes
clients ──< applications
clients ──< client_documents
products ──< quotes
products ──< applications
applications ──< application_documents
applications ──< commissions
users ──< commissions
users ──< notifications
```

---

## Tables

### `agencies`
```sql
CREATE TABLE agencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    npn             TEXT UNIQUE,          -- National Producer Number (agency level)
    phone           TEXT,
    email           TEXT,
    address         JSONB,                -- {street, city, state, zip}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### `users`
```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agency_id       UUID REFERENCES agencies(id),
    email           TEXT NOT NULL UNIQUE,
    -- No password: authentication is passwordless via email PIN
    role            TEXT NOT NULL CHECK (role IN ('agent', 'agency_admin', 'super_admin')),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    npn             TEXT,                 -- National Producer Number (individual)
    phone           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_agency_id ON users(agency_id);
CREATE INDEX idx_users_email ON users(email);
```

---

### `login_pins`
Stores hashed one-time PINs for passwordless authentication.

```sql
CREATE TABLE login_pins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    pin_hash        TEXT NOT NULL,           -- bcrypt hash of the 6-digit PIN
    expires_at      TIMESTAMPTZ NOT NULL,    -- 10 minutes from creation
    used_at         TIMESTAMPTZ,             -- set when consumed; NULL = still valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_login_pins_user_id ON login_pins(user_id);
-- Partial index for fast lookup of unconsumed, unexpired PINs
CREATE INDEX idx_login_pins_active ON login_pins(user_id, expires_at)
    WHERE used_at IS NULL;
```

---

### `refresh_tokens`
```sql
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash      TEXT NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
```

---

### `clients`
```sql
CREATE TABLE clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES users(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    date_of_birth   DATE NOT NULL,
    ssn_encrypted   BYTEA,               -- AES-256 encrypted SSN
    email           TEXT,
    phone           TEXT,
    address         JSONB,               -- {street, city, state, zip}
    -- Financial profile
    annual_income   NUMERIC(15,2),
    net_worth       NUMERIC(15,2),
    liquid_net_worth NUMERIC(15,2),
    risk_tolerance  TEXT CHECK (risk_tolerance IN ('conservative', 'moderate', 'aggressive')),
    investment_objectives TEXT[],        -- ['income', 'growth', 'capital_preservation']
    tax_bracket     TEXT,
    -- Suitability
    suitability_completed_at TIMESTAMPTZ,
    -- Metadata
    notes           TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_clients_agent_id ON clients(agent_id);
CREATE INDEX idx_clients_last_name ON clients(last_name);
```

---

### `client_beneficiaries`
```sql
CREATE TABLE client_beneficiaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
    type            TEXT NOT NULL CHECK (type IN ('primary', 'contingent')),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    relationship    TEXT NOT NULL,
    date_of_birth   DATE,
    ssn_encrypted   BYTEA,
    percentage      NUMERIC(5,2) NOT NULL, -- must sum to 100 per type
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_beneficiaries_client_id ON client_beneficiaries(client_id);
```

---

### `carriers`
```sql
CREATE TABLE carriers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    am_best_rating  TEXT,
    phone           TEXT,
    website         TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### `products`
```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier_id      UUID NOT NULL REFERENCES carriers(id),
    name            TEXT NOT NULL,
    product_type    TEXT NOT NULL CHECK (product_type IN ('fixed', 'fixed_indexed', 'variable')),
    minimum_premium NUMERIC(15,2) NOT NULL,
    maximum_premium NUMERIC(15,2),
    -- Rate / crediting
    current_rate    NUMERIC(6,4),        -- for fixed: annual interest rate
    floor_rate      NUMERIC(6,4),        -- for FIA: minimum guaranteed rate
    cap_rate        NUMERIC(6,4),        -- for FIA: maximum credited rate
    -- Surrender
    surrender_charge_schedule JSONB,     -- [{year: 1, rate: 0.09}, ...]
    -- Riders available
    available_riders JSONB,             -- [{id, name, type, annual_cost_pct}]
    -- Availability
    available_states TEXT[],
    -- Commission
    base_commission_rate NUMERIC(6,4),
    -- Metadata
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_date  DATE,
    expiration_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_carrier_id ON products(carrier_id);
CREATE INDEX idx_products_product_type ON products(product_type);
```

---

### `quotes`
```sql
CREATE TABLE quotes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES users(id),
    client_id       UUID NOT NULL REFERENCES clients(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    -- Inputs
    premium_amount  NUMERIC(15,2) NOT NULL,
    purchase_type   TEXT NOT NULL CHECK (purchase_type IN ('single', 'flexible')),
    payout_option   TEXT NOT NULL CHECK (payout_option IN ('immediate', 'deferred')),
    deferral_years  INTEGER,
    selected_riders JSONB,               -- [{rider_id, name, annual_cost}]
    -- Outputs (stored snapshot at time of generation)
    projections     JSONB,               -- [{year, guaranteed_value, illustrated_value}]
    income_stream   JSONB,               -- {annual_guaranteed, annual_illustrated}
    estimated_commission NUMERIC(15,2),
    -- PDF
    pdf_s3_key      TEXT,
    -- Status
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'converted', 'expired')),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_quotes_agent_id ON quotes(agent_id);
CREATE INDEX idx_quotes_client_id ON quotes(client_id);
```

---

### `applications`
```sql
CREATE TABLE applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES users(id),
    client_id       UUID NOT NULL REFERENCES clients(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    quote_id        UUID REFERENCES quotes(id),
    -- Owner & annuitant
    owner           JSONB NOT NULL,      -- {first_name, last_name, dob, ssn_encrypted, ...}
    joint_owner     JSONB,
    annuitant       JSONB NOT NULL,
    -- Funding
    funding_source  TEXT NOT NULL CHECK (funding_source IN ('new_money', '1035_exchange', 'rollover', 'transfer')),
    premium_amount  NUMERIC(15,2) NOT NULL,
    funding_details JSONB,               -- bank account or transfer info
    -- Riders
    elected_riders  JSONB,
    -- Suitability
    suitability_responses JSONB,
    -- Signatures
    agent_signed_at    TIMESTAMPTZ,
    client_sign_token  TEXT UNIQUE,      -- secure token for client e-sign link
    client_sign_token_expires_at TIMESTAMPTZ,
    client_signed_at   TIMESTAMPTZ,
    -- Status
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (
                        status IN ('draft', 'pending_signatures', 'submitted',
                                   'in_review', 'approved', 'declined', 'returned')
                    ),
    status_reason   TEXT,
    submitted_at    TIMESTAMPTZ,
    -- PDF
    pdf_s3_key      TEXT,
    -- Audit
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_applications_agent_id ON applications(agent_id);
CREATE INDEX idx_applications_client_id ON applications(client_id);
CREATE INDEX idx_applications_status ON applications(status);
```

---

### `documents`
```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Polymorphic: belongs to client or application (one must be set)
    client_id       UUID REFERENCES clients(id) ON DELETE CASCADE,
    application_id  UUID REFERENCES applications(id) ON DELETE CASCADE,
    -- Document metadata
    document_type   TEXT NOT NULL,       -- 'government_id', 'suitability_form', '1035_form', etc.
    file_name       TEXT NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    mime_type       TEXT NOT NULL,
    s3_key          TEXT NOT NULL UNIQUE,
    -- Status
    status          TEXT NOT NULL DEFAULT 'uploaded' CHECK (
                        status IN ('uploaded', 'under_review', 'approved', 'rejected')
                    ),
    rejection_reason TEXT,
    expires_at      TIMESTAMPTZ,
    -- Audit
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Ensure at least one FK is set
    CONSTRAINT document_owner_check CHECK (
        (client_id IS NOT NULL)::int + (application_id IS NOT NULL)::int = 1
    )
);

CREATE INDEX idx_documents_client_id ON documents(client_id);
CREATE INDEX idx_documents_application_id ON documents(application_id);
```

---

### `commissions`
```sql
CREATE TABLE commissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES users(id),
    application_id  UUID NOT NULL REFERENCES applications(id),
    -- Amounts
    premium_amount  NUMERIC(15,2) NOT NULL,
    commission_rate NUMERIC(6,4) NOT NULL,
    expected_amount NUMERIC(15,2) NOT NULL,
    actual_amount   NUMERIC(15,2),
    -- Chargeback
    is_chargeback   BOOLEAN NOT NULL DEFAULT false,
    chargeback_reason TEXT,
    -- Status
    status          TEXT NOT NULL CHECK (status IN ('pending', 'approved', 'paid', 'charged_back')),
    paid_at         TIMESTAMPTZ,
    statement_period TEXT,               -- e.g. '2025-01'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commissions_agent_id ON commissions(agent_id);
CREATE INDEX idx_commissions_application_id ON commissions(application_id);
CREATE INDEX idx_commissions_statement_period ON commissions(statement_period);
```

---

### `notifications`
```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            TEXT NOT NULL,       -- 'application_status_change', 'document_missing', etc.
    title           TEXT NOT NULL,
    body            TEXT,
    related_id      UUID,               -- ID of related entity (application, document, etc.)
    related_type    TEXT,               -- 'application', 'document', 'commission'
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_read_at ON notifications(user_id, read_at) WHERE read_at IS NULL;
```

---

### `audit_logs`
```sql
CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,
    user_id         UUID REFERENCES users(id),
    action          TEXT NOT NULL,       -- 'create', 'update', 'delete', 'login', 'view'
    entity_type     TEXT NOT NULL,       -- 'client', 'application', 'document', etc.
    entity_id       UUID,
    changes         JSONB,               -- {field: {old, new}} for updates
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
```

---

### `ai_recommendations`
Stores AI product recommendations generated for a client.

```sql
CREATE TABLE ai_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
    agent_id        UUID NOT NULL REFERENCES users(id),
    recommended_product_ids UUID[],          -- ordered by recommendation rank
    rationale       JSONB,                   -- [{product_id, explanation}]
    input_snapshot  JSONB,                   -- client profile at time of generation
    model_version   TEXT NOT NULL,           -- e.g. 'claude-sonnet-4-6'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_recommendations_client_id ON ai_recommendations(client_id);
```

---

### `ai_suitability_flags`
Records AI-generated suitability compliance flags on an application.

```sql
CREATE TABLE ai_suitability_flags (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id      UUID NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    flags               JSONB NOT NULL,      -- [{field, issue, severity}]
    agent_acknowledgments JSONB,             -- [{flag_id, acknowledged_at, override_reason}]
    model_version       TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_suitability_flags_application_id ON ai_suitability_flags(application_id);
```

---

### `ai_chat_sessions`
Stores advisor chat assistant conversation history.

```sql
CREATE TABLE ai_chat_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    messages        JSONB NOT NULL DEFAULT '[]', -- [{role, content, created_at}]
    context         JSONB,                   -- {page, client_id, application_id}
    last_message_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_chat_sessions_user_id ON ai_chat_sessions(user_id);
```

---

## Migrations Strategy

- Alembic manages all schema changes
- Migration files committed to `app/migrations/versions/`
- Never alter existing migrations; always add new ones
- Staging runs migrations automatically on deploy; production requires manual approval
