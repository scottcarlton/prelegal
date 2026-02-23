# API Design — Annuity Advisor Platform

Base URL: `/api/v1`

All endpoints return JSON. Authentication required on all routes except `/auth/login` and `/auth/refresh`.

**Request headers:**
```
Authorization: Bearer <access_token>
Content-Type: application/json
```

**Standard error response:**
```json
{
  "detail": "Human-readable error message",
  "code": "MACHINE_READABLE_CODE"
}
```

---

## Authentication — `/auth`

Passwordless PIN-based login. No passwords stored or required.

### `POST /auth/request-pin`
Send a one-time PIN to the user's email address.

**Request:**
```json
{ "email": "agent@example.com" }
```

**Response `200`:**
```json
{ "message": "If this email is registered, a PIN has been sent." }
```

Always returns 200 regardless of whether the email exists (prevents enumeration).
PIN is valid for 10 minutes. Rate-limited: 3 requests per email per 10 minutes.

---

### `POST /auth/verify-pin`
Exchange a valid PIN for an access token.

**Request:**
```json
{
  "email": "agent@example.com",
  "pin": "482917"
}
```

**Response `200`:**
```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 900,
  "user": {
    "id": "uuid",
    "email": "agent@example.com",
    "first_name": "Jane",
    "last_name": "Smith",
    "role": "agent"
  }
}
```
Refresh token set as httpOnly cookie.

**Error `401`:** `{ "detail": "Invalid or expired PIN.", "code": "INVALID_PIN" }`
**Error `429`:** `{ "detail": "Too many attempts. Try again in 15 minutes.", "code": "RATE_LIMITED" }`

---

### `POST /auth/refresh`
Exchange refresh token (cookie) for a new access token.

**Response `200`:** Same shape as verify-pin response.

---

### `POST /auth/logout`
Revoke the refresh token.

**Response `204`:** No content.

---

## Users — `/users`

### `GET /users/me`
Get authenticated user's profile.

**Response `200`:**
```json
{
  "id": "uuid",
  "email": "agent@example.com",
  "first_name": "Jane",
  "last_name": "Smith",
  "role": "agent",
  "npn": "12345678",
  "agency_id": "uuid",
  "is_active": true
}
```

### `PATCH /users/me`
Update own profile (name, phone, password).

### `GET /users` _(agency_admin, super_admin only)_
List users in the agency.

**Query params:** `role`, `is_active`, `search`, `page`, `per_page`

### `POST /users` _(super_admin only)_
Create a new user.

### `GET /users/{id}` _(agency_admin, super_admin)_

### `PATCH /users/{id}` _(super_admin)_

### `DELETE /users/{id}` _(super_admin)_
Soft-deactivate (sets `is_active = false`).

---

## Clients — `/clients`

### `GET /clients`
List agent's clients (agents see only their own; agency_admin sees all in agency).

**Query params:**
- `search` — name or email substring
- `status` — `active` | `archived`
- `page`, `per_page`

**Response `200`:**
```json
{
  "items": [
    {
      "id": "uuid",
      "first_name": "John",
      "last_name": "Doe",
      "date_of_birth": "1960-05-15",
      "email": "john@example.com",
      "phone": "555-0100",
      "status": "active",
      "open_applications": 1,
      "updated_at": "2025-01-15T10:00:00Z"
    }
  ],
  "total": 42,
  "page": 1,
  "per_page": 20
}
```

### `POST /clients`
Create a new client.

**Request:** Full client profile (see schema).

### `GET /clients/{id}`
Get full client detail including beneficiaries.

### `PATCH /clients/{id}`
Update client information.

### `DELETE /clients/{id}`
Archive client (`status = 'archived'`).

---

### Beneficiaries

### `GET /clients/{id}/beneficiaries`

### `PUT /clients/{id}/beneficiaries`
Replace all beneficiaries (primary + contingent). Validates percentages sum to 100 per type.

---

## Products — `/products`

### `GET /products`
Browse available products.

**Query params:**
- `product_type` — `fixed` | `fixed_indexed` | `variable`
- `state` — 2-letter state code (filters by `available_states`)
- `min_premium` — max value for minimum_premium filter
- `carrier_id`
- `page`, `per_page`

**Response `200`:**
```json
{
  "items": [
    {
      "id": "uuid",
      "carrier": { "id": "uuid", "name": "Acme Life", "am_best_rating": "A+" },
      "name": "SecureIncome 7",
      "product_type": "fixed_indexed",
      "minimum_premium": 25000.00,
      "current_rate": null,
      "cap_rate": 0.09,
      "floor_rate": 0.00,
      "surrender_charge_schedule": [
        {"year": 1, "rate": 0.08},
        {"year": 2, "rate": 0.07}
      ],
      "base_commission_rate": 0.065,
      "available_riders": [
        {"id": "uuid", "name": "Guaranteed Income Rider", "type": "income", "annual_cost_pct": 0.0095}
      ]
    }
  ],
  "total": 18,
  "page": 1,
  "per_page": 20
}
```

### `GET /products/{id}`
Full product detail.

### `POST /products/compare`
Compare up to 3 products side-by-side.

**Request:** `{ "product_ids": ["uuid1", "uuid2", "uuid3"] }`

**Response:** Array of full product objects.

---

## Quotes — `/quotes`

### `POST /quotes`
Generate and save a quote.

**Request:**
```json
{
  "client_id": "uuid",
  "product_id": "uuid",
  "premium_amount": 100000.00,
  "purchase_type": "single",
  "payout_option": "deferred",
  "deferral_years": 10,
  "selected_rider_ids": ["uuid"]
}
```

**Response `201`:** Full quote object including projections.

### `GET /quotes`
List quotes for the authenticated agent.

**Query params:** `client_id`, `status`, `page`, `per_page`

### `GET /quotes/{id}`

### `GET /quotes/{id}/pdf`
Returns a pre-signed S3 URL for the quote PDF.

### `PATCH /quotes/{id}`
Update quote inputs and regenerate projections.

### `DELETE /quotes/{id}`
Expire a quote.

---

## Applications — `/applications`

### `POST /applications`
Create a new application (starts as `draft`).

**Request:**
```json
{
  "client_id": "uuid",
  "product_id": "uuid",
  "quote_id": "uuid",
  "owner": { "first_name": "...", "last_name": "...", "date_of_birth": "...", "ssn": "..." },
  "annuitant": { "first_name": "...", "last_name": "...", "date_of_birth": "..." },
  "funding_source": "new_money",
  "premium_amount": 100000.00,
  "elected_rider_ids": ["uuid"],
  "suitability_responses": { "question_1": "answer", ... }
}
```

### `GET /applications`
List applications.

**Query params:** `client_id`, `status`, `agent_id` (admin only), `page`, `per_page`

### `GET /applications/{id}`
Full application detail.

### `PATCH /applications/{id}`
Update a draft application.

### `POST /applications/{id}/sign`
Record agent e-signature and send client signature request email.

**Request:** `{ "agent_attestation": true }`

**Response `200`:** Application moves to `pending_signatures`.

### `GET /applications/sign/{token}`
Public endpoint. Validates client signature token.

**Response `200`:** `{ "application_summary": {...}, "client_name": "..." }`

### `POST /applications/sign/{token}`
Record client e-signature.

**Request:** `{ "agreed": true }`

**Response `200`:** Application moves to `submitted` after validation.

### `POST /applications/{id}/submit` _(admin)_
Manually advance application status.

### `GET /applications/{id}/pdf`
Pre-signed URL for application PDF.

---

## Documents — `/documents`

### `POST /documents/upload-url`
Request a pre-signed S3 upload URL.

**Request:**
```json
{
  "client_id": "uuid",        // or application_id
  "document_type": "government_id",
  "file_name": "drivers_license.pdf",
  "mime_type": "application/pdf",
  "file_size_bytes": 204800
}
```

**Response `200`:**
```json
{
  "upload_url": "https://s3.../presigned...",
  "document_id": "uuid",
  "expires_in": 300
}
```

Frontend uploads directly to S3 using this URL, then calls confirm.

### `POST /documents/{id}/confirm`
Confirm upload is complete. Backend verifies file exists in S3.

### `GET /documents`
List documents. Query params: `client_id`, `application_id`, `document_type`, `status`

### `GET /documents/{id}/download-url`
Get a short-lived pre-signed download URL.

### `DELETE /documents/{id}`
Delete a document (soft-delete, mark as deleted).

---

## Commissions — `/commissions`

### `GET /commissions`
List commission records for authenticated agent.

**Query params:** `status`, `statement_period` (e.g. `2025-01`), `page`, `per_page`

**Response `200`:**
```json
{
  "items": [...],
  "summary": {
    "total_expected": 15000.00,
    "total_paid": 12000.00,
    "total_pending": 3000.00
  },
  "total": 8,
  "page": 1,
  "per_page": 20
}
```

### `GET /commissions/{id}`

### `GET /commissions/export`
Export to CSV. Query params same as list.

---

## Reports — `/reports` _(agency_admin, super_admin)_

### `GET /reports/production`
Agent production summary.

**Query params:** `start_date`, `end_date`, `agent_id`

### `GET /reports/pipeline`
Applications grouped by status.

### `GET /reports/commissions`
Commission totals by period and agent.

All report endpoints support `?format=csv` for CSV download.

---

## Notifications — `/notifications`

### `GET /notifications`
List notifications for authenticated user.

**Query params:** `unread_only` (bool), `page`, `per_page`

### `POST /notifications/{id}/read`
Mark a notification as read.

### `POST /notifications/read-all`
Mark all as read.

---

## AI — `/ai`

All AI endpoints require authentication. Responses may be slow (LLM latency); clients should show loading states.

---

### `POST /ai/recommend-products`
Generate AI-powered product recommendations for a client.

**Request:**
```json
{ "client_id": "uuid" }
```

**Response `200`:**
```json
{
  "recommendation_id": "uuid",
  "recommendations": [
    {
      "product_id": "uuid",
      "product_name": "SecureIncome 7",
      "carrier": "Acme Life",
      "rank": 1,
      "rationale": "SecureIncome 7's 0% floor and 9% cap aligns well with John's conservative risk tolerance while still providing upside potential. The guaranteed income rider supports his stated objective of supplemental retirement income."
    },
    {
      "product_id": "uuid",
      "product_name": "Fixed Shield Plus",
      "carrier": "Pacific Annuity",
      "rank": 2,
      "rationale": "..."
    }
  ],
  "disclaimer": "These recommendations are generated by AI to assist the advisor and do not constitute financial advice."
}
```

**Error `422`:** Client profile incomplete (missing risk tolerance or investment objectives).

---

### `POST /ai/check-suitability`
AI compliance review of a draft application before submission.

**Request:**
```json
{ "application_id": "uuid" }
```

**Response `200`:**
```json
{
  "flag_id": "uuid",
  "passed": false,
  "flags": [
    {
      "id": "flag-1",
      "field": "product_type",
      "severity": "warning",
      "issue": "Client indicated conservative risk tolerance, but selected product has no principal protection floor.",
      "suggestion": "Consider a fixed annuity or an FIA with a minimum guaranteed rate."
    }
  ]
}
```

If `passed: true`, `flags` is an empty array.

---

### `POST /ai/acknowledge-suitability-flag`
Record advisor acknowledgment of a suitability flag.

**Request:**
```json
{
  "flag_id": "uuid",
  "flag_item_id": "flag-1",
  "override_reason": "Client understands the risk and explicitly requested this product after full disclosure."
}
```

**Response `200`:** Updated flag record.

---

### `POST /ai/extract-document`
Extract structured data from an uploaded document.

**Request:**
```json
{ "document_id": "uuid" }
```

**Response `200`:**
```json
{
  "extracted_fields": {
    "policy_number": "ACM-1234567",
    "surrender_value": 87500.00,
    "existing_carrier": "Old Guard Life",
    "owner_name": "John Doe"
  },
  "confidence": "high",
  "requires_review": false
}
```

---

### `POST /ai/explain-quote`
Generate a plain-English quote summary for advisor use.

**Request:**
```json
{ "quote_id": "uuid" }
```

**Response `200`:**
```json
{
  "explanation": "With a $100,000 premium in the SecureIncome 7, John is guaranteed at least $148,000 after 10 years, with potential growth up to $193,000 depending on index performance. Starting in year 11, he'd receive a guaranteed income of $7,800 per year for life — and potentially up to $10,500 if the index performs well."
}
```

---

### `GET /ai/chat`
Open a Server-Sent Events (SSE) stream for the AI chat assistant.

**Query params:** `session_id` (optional; omit to start a new session)

**Request:** SSE connection; send messages via `POST /ai/chat/{session_id}/message`

### `POST /ai/chat`
Start a new chat session.

**Response `201`:** `{ "session_id": "uuid" }`

### `POST /ai/chat/{session_id}/message`
Send a message to the chat assistant.

**Request:**
```json
{
  "message": "What documents are required for a 1035 exchange?",
  "context": {
    "page": "application",
    "application_id": "uuid"
  }
}
```

**Response**: SSE stream. Each event is a partial response chunk:
```
data: {"delta": "For a 1035 "}
data: {"delta": "exchange, you'll need..."}
data: {"done": true, "full_response": "..."}
```

### `DELETE /ai/chat/{session_id}`
Clear chat history and end session.

---

## Admin — `/admin` _(super_admin only)_

### `GET /admin/carriers`
### `POST /admin/carriers`
### `PATCH /admin/carriers/{id}`

### `GET /admin/products`
### `POST /admin/products`
### `PATCH /admin/products/{id}`

### `GET /admin/agencies`
### `POST /admin/agencies`
### `PATCH /admin/agencies/{id}`

---

## Pagination

All list endpoints use cursor-based pagination via `page` and `per_page` (default 20, max 100). Responses include `total`, `page`, `per_page`.

## Rate Limiting

| Endpoint group | Limit |
|---|---|
| `POST /auth/login` | 10/min per IP |
| `POST /auth/forgot-password` | 5/min per IP |
| All other endpoints | 300/min per user |

## Versioning

Current version: `v1`. New versions introduced only for breaking changes. Deprecated versions announced 6 months in advance.
