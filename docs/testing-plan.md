# Testing Plan — Annuity Advisor Platform

## Testing Philosophy

- **Test behavior, not implementation**: Tests should verify what the system does, not how it's internally structured
- **Pyramid model**: Many unit tests → fewer integration tests → few E2E tests
- **CI-enforced**: All tests run on every PR; merge blocked on failure
- **Coverage target**: 80% line coverage minimum for backend; critical paths at 100%

---

## Testing Layers

```
          /\
         /  \   E2E Tests (Playwright)
        /────\  ~30 scenarios, full user flows
       /      \
      /────────\ Integration Tests (pytest + httpx)
     /          \ ~150 tests, API layer + DB
    /────────────\
   /              \ Unit Tests (pytest / Jest)
  /────────────────\ ~500+ tests, business logic
```

---

## 1. Unit Tests

### Backend (pytest)

**Target**: Business logic in `services/` and utility functions.

**Tools**: `pytest`, `pytest-asyncio`, `pytest-cov`

#### Service layer tests

```python
# tests/services/test_quote_service.py

def test_calculate_fixed_annuity_projection():
    """10% rate over 10 years compounds correctly."""
    result = calculate_projection(
        premium=100_000,
        annual_rate=0.10,
        years=10,
        product_type="fixed"
    )
    assert result[10]["guaranteed_value"] == pytest.approx(259_374, rel=0.01)

def test_quote_expires_after_30_days():
    quote = QuoteFactory.build()
    assert quote.expires_at == quote.created_at + timedelta(days=30)

def test_commission_estimate_applies_base_rate():
    product = ProductFactory.build(base_commission_rate=0.065)
    commission = estimate_commission(premium=100_000, product=product)
    assert commission == pytest.approx(6_500)
```

#### Model / schema tests

```python
# tests/schemas/test_application_schema.py

def test_beneficiary_percentages_must_sum_to_100():
    with pytest.raises(ValidationError):
        ApplicationSchema(beneficiaries=[
            {"type": "primary", "name": "Jane", "percentage": 60},
            {"type": "primary", "name": "Bob", "percentage": 30},  # only 90%
        ])

def test_ssn_is_masked_in_responses():
    client = ClientResponseSchema(ssn="123-45-6789")
    assert client.ssn == "***-**-6789"
```

#### Auth / security tests

```python
def test_expired_access_token_is_rejected():
    token = create_token(user_id="uuid", expires_delta=timedelta(seconds=-1))
    with pytest.raises(TokenExpiredError):
        decode_token(token)

def test_pin_is_hashed_not_stored_plaintext():
    pin = "482917"
    hashed = hash_pin(pin)
    assert hashed != pin
    assert verify_pin(pin, hashed)

def test_pin_expires_after_10_minutes():
    pin_record = LoginPinFactory.build(
        expires_at=datetime.utcnow() - timedelta(minutes=1)
    )
    assert pin_record.is_expired()

def test_pin_cannot_be_used_twice():
    pin_record = LoginPinFactory.build(used_at=datetime.utcnow())
    assert not pin_record.is_valid()
```

---

### Frontend (Jest + React Testing Library)

**Target**: Component logic, form validation, state management, utility functions.

**Tools**: `jest`, `@testing-library/react`, `@testing-library/user-event`, `msw` (API mocking)

```typescript
// __tests__/components/QuoteForm.test.tsx

test("shows income projection after form submission", async () => {
  server.use(
    rest.post("/api/v1/quotes", (req, res, ctx) =>
      res(ctx.json(mockQuoteResponse))
    )
  );
  render(<QuoteForm />);
  await userEvent.type(screen.getByLabelText("Premium Amount"), "100000");
  await userEvent.click(screen.getByText("Generate Quote"));
  expect(await screen.findByText("$7,800")).toBeInTheDocument();
});

test("disables submit when required fields are missing", () => {
  render(<QuoteForm />);
  expect(screen.getByRole("button", { name: "Generate Quote" })).toBeDisabled();
});

// __tests__/lib/formatters.test.ts

test("formats currency with commas and 2 decimal places", () => {
  expect(formatCurrency(100000)).toBe("$100,000.00");
});

test("masks SSN showing only last 4 digits", () => {
  expect(maskSSN("123456789")).toBe("***-**-6789");
});
```

---

## 2. Integration Tests

### Backend API Tests (pytest + httpx)

Tests exercise the full request/response cycle against a real test database (PostgreSQL, not SQLite).

**Setup**: Each test module uses a transaction that is rolled back after each test (no state leakage).

```python
# tests/api/test_clients.py

async def test_agent_can_create_client(client: AsyncClient, agent_token: str):
    response = await client.post(
        "/api/v1/clients",
        json={
            "first_name": "John",
            "last_name": "Doe",
            "date_of_birth": "1960-05-15",
            "email": "john@example.com",
        },
        headers={"Authorization": f"Bearer {agent_token}"},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["last_name"] == "Doe"

async def test_agent_cannot_view_another_agents_client(
    client: AsyncClient,
    agent_a_token: str,
    agent_b_client_id: str,
):
    response = await client.get(
        f"/api/v1/clients/{agent_b_client_id}",
        headers={"Authorization": f"Bearer {agent_a_token}"},
    )
    assert response.status_code == 404   # not 403 — don't leak existence

async def test_unauthenticated_request_is_rejected(client: AsyncClient):
    response = await client.get("/api/v1/clients")
    assert response.status_code == 401
```

#### Coverage per module

| Module | Key scenarios |
|---|---|
| Auth | PIN request, PIN verify (success/expired/used/wrong), rate limiting, refresh, logout |
| Clients | CRUD, agent scoping, archive, search |
| Products | List + filter, compare |
| Quotes | Create, projections accuracy, PDF generation trigger |
| Applications | Full workflow (draft → submitted), e-sign token, status transitions |
| Documents | Pre-signed URL generation, confirm upload, download URL, access control |
| Commissions | List, export CSV, summary totals |
| Reports | Production, pipeline, commission reports (admin only) |

---

## 3. End-to-End Tests (Playwright)

Tests run against a staging environment with seeded test data. A dedicated test user account is used.

**Tools**: `@playwright/test`, TypeScript

### Scenarios

#### Authentication (PIN-based)
- [ ] Enter email → PIN email is sent → enter correct PIN → land on dashboard
- [ ] Expired PIN (wait >10 min) shows "PIN expired, request a new one"
- [ ] Wrong PIN 5 times triggers lockout message
- [ ] Resend PIN button sends a new PIN and invalidates the previous one
- [ ] Expired session redirects to login with return URL preserved
- [ ] Logout clears session and redirects to login

#### Client Management
- [ ] Create a new client with all required fields
- [ ] Search for a client by name
- [ ] Edit a client's financial profile
- [ ] Add and update beneficiaries

#### Quote Workflow
- [ ] Generate a quote for a client from the product catalog
- [ ] Quote projections update live as inputs change
- [ ] Download quote PDF
- [ ] Quote is saved and visible on the client's profile

#### Application Workflow
- [ ] Start an application from a quote
- [ ] Complete all 7 steps of the application wizard
- [ ] Upload required documents within the wizard
- [ ] Agent signs application; client signature email is triggered
- [ ] Client signs via email link; application moves to Submitted
- [ ] Application with missing documents shows alert on dashboard

#### Commission & Reporting
- [ ] Commission ledger displays correct totals for current period
- [ ] Export commission CSV downloads a valid file
- [ ] Agency admin can view production report for all agents

### Example test

```typescript
// e2e/application-wizard.spec.ts

test("agent can submit a complete application", async ({ page }) => {
  await loginAs(page, "agent@test.com");
  await page.goto("/clients/test-client-id");
  await page.click("text=+ New Application");

  // Step 1: product selection
  await page.selectOption('[name="product_id"]', { label: "SecureIncome 7" });
  await page.click("text=Continue");

  // Step 2: owner info
  await page.fill('[name="owner.first_name"]', "John");
  // ... remaining fields
  await page.click("text=Continue");

  // ... steps 3-6

  // Step 7: review and submit
  await expect(page.locator("text=John Doe")).toBeVisible();
  await expect(page.locator("text=$100,000")).toBeVisible();
  await page.click("text=Submit Application");
  await page.click("text=Confirm");

  await expect(page.locator('[data-testid="app-status"]')).toHaveText("Pending Signatures");
});
```

---

## 4. AI Feature Tests

### Unit Tests

AI service layer is tested with a mocked Claude API client to avoid LLM costs and non-determinism in CI.

```python
# tests/services/test_ai_service.py

@patch("app.services.ai_service.anthropic_client")
async def test_product_recommendation_returns_ranked_list(mock_client):
    mock_client.messages.create.return_value = mock_claude_response(
        content='[{"product_id": "uuid1", "rank": 1, "rationale": "..."}]'
    )
    result = await recommend_products(client_id="uuid", db=mock_db)
    assert len(result.recommendations) >= 1
    assert result.recommendations[0].rank == 1

@patch("app.services.ai_service.anthropic_client")
async def test_suitability_check_flags_product_mismatch(mock_client):
    mock_client.messages.create.return_value = mock_claude_response(
        content='{"passed": false, "flags": [{"severity": "warning", "issue": "..."}]}'
    )
    result = await check_suitability(application_id="uuid", db=mock_db)
    assert result.passed is False
    assert len(result.flags) == 1

async def test_ai_request_blocked_when_daily_budget_exceeded(redis_mock):
    redis_mock.get.return_value = str(AI_DAILY_TOKEN_BUDGET + 1)
    with pytest.raises(BudgetExceededError):
        await recommend_products(client_id="uuid", db=mock_db)

async def test_extraction_result_cached_on_second_call(redis_mock):
    redis_mock.get.return_value = json.dumps(mock_extraction_result)
    result = await extract_document(document_id="uuid", db=mock_db)
    # Should not call Claude API when cache hit
    assert anthropic_client.messages.create.call_count == 0
```

### Integration Tests

```python
async def test_post_recommend_products_returns_200(client, agent_token, seeded_client):
    response = await client.post(
        "/api/v1/ai/recommend-products",
        json={"client_id": str(seeded_client.id)},
        headers={"Authorization": f"Bearer {agent_token}"},
    )
    assert response.status_code == 200
    data = response.json()
    assert "recommendations" in data
    assert "disclaimer" in data

async def test_chat_message_streams_response(client, agent_token):
    # Create session first
    session_resp = await client.post(
        "/api/v1/ai/chat",
        headers={"Authorization": f"Bearer {agent_token}"},
    )
    session_id = session_resp.json()["session_id"]

    # Send a message and verify SSE stream
    async with client.stream(
        "POST",
        f"/api/v1/ai/chat/{session_id}/message",
        json={"message": "What is a 1035 exchange?"},
        headers={"Authorization": f"Bearer {agent_token}"},
    ) as response:
        assert response.status_code == 200
        assert response.headers["content-type"].startswith("text/event-stream")
```

### E2E Tests

- [ ] AI recommendation panel appears on client detail after profile is complete
- [ ] Clicking "Start Quote" from a recommendation pre-selects that product in the quote builder
- [ ] Suitability warning appears when a mismatched product is selected; submit is blocked until acknowledged
- [ ] Quote explainer text is displayed below projections table after quote is generated
- [ ] Chat assistant opens, accepts a message, and streams a response
- [ ] Chat history is preserved when navigating between pages during the same session

---

## 5. Security Tests

### OWASP-focused checks (automated in CI)

- **SQL injection**: All inputs pass through SQLAlchemy parameterization. `sqlmap` scan on staging before each release.
- **XSS**: Next.js escapes by default; dangerouslySetInnerHTML is prohibited via ESLint rule.
- **IDOR**: Integration tests verify cross-user resource access returns 404 (not 403 — avoids leaking existence).
- **Auth bypass**: Every protected endpoint tested without a token and with an invalid/expired token.
- **Mass assignment**: Pydantic schemas explicitly define allowed input fields; `model_config = ConfigDict(extra='forbid')`.
- **Sensitive data exposure**: Response schemas tested to ensure SSN and PIN hashes are never serialized in responses.
- **PIN security**: Tests verify PIN requests don't reveal whether an email is registered; verify PINs are single-use; verify brute-force lockout triggers correctly.

### Dependency scanning

- `pip-audit` (Python dependencies) — run in CI on every push
- `npm audit` (Node dependencies) — run in CI on every push
- Dependabot configured for automated dependency update PRs

---

## 5. Performance Tests (k6)

Run before each production release against staging.

**Scenarios:**

```javascript
// k6/quote-generation.js

export const options = {
  stages: [
    { duration: "1m", target: 50 },   // ramp up
    { duration: "3m", target: 50 },   // steady state
    { duration: "1m", target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<500"],  // 95th percentile < 500ms
    http_req_failed: ["rate<0.01"],    // < 1% error rate
  },
};
```

**Key flows to load test:**
- Login (token issuance throughput)
- Client list (database query performance)
- Quote generation (calculation throughput)
- Document upload URL generation

---

## CI/CD Integration

```yaml
# .github/workflows/test.yml (abbreviated)

jobs:
  backend-tests:
    steps:
      - run: pip install -r requirements-dev.txt
      - run: pytest --cov=app --cov-report=xml --cov-fail-under=80
      - run: pip-audit

  frontend-tests:
    steps:
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage --passWithNoTests
      - run: npm audit --audit-level=high

  e2e-tests:
    needs: [backend-tests, frontend-tests]
    environment: staging
    steps:
      - run: npx playwright install
      - run: npx playwright test
```

---

## Test Data Management

- **Unit/integration tests**: Factories (`factory_boy` for Python, custom builders for TypeScript) generate realistic test data
- **E2E tests**: Seeded staging database with a fixed set of test users, clients, and products; reset nightly
- **Sensitive data**: Test SSNs use dedicated test ranges (e.g., 900-xx-xxxx); never use real SSNs in tests
- **Fixtures**: Shared pytest fixtures for database session, authenticated clients per role, and common entities
