# UI/UX Design — Annuity Advisor Platform

## Design Principles

- **Clarity over density**: Advisors are often in client meetings; screens must be scannable at a glance
- **Progressive disclosure**: Show the right information at the right step — don't overwhelm with data upfront
- **Safe defaults**: Sensible pre-fills from client/quote data reduce re-entry and errors
- **Compliance-forward**: Suitability and attestation steps are prominent, never skippable

## Design System

- **Component library**: shadcn/ui (Radix UI primitives + Tailwind)
- **Typography**: Inter (system font fallback)
- **Color palette**:
  - Primary: Deep blue `#1E3A5F`
  - Accent: Teal `#0EA5A0`
  - Success: `#16A34A`
  - Warning: `#CA8A04`
  - Danger: `#DC2626`
  - Background: `#F8FAFC`
  - Surface: `#FFFFFF`
- **Border radius**: `0.5rem` (8px) for cards, `0.375rem` (6px) for inputs
- **Breakpoints**: Mobile (sm 640px), Tablet (md 768px), Desktop (lg 1024px, xl 1280px)

---

## Layout

### Authenticated Shell

```
┌─────────────────────────────────────────────────────────┐
│  LOGO       Dashboard  Clients  Products  Applications   │  ← Top nav (desktop)
│             Reports  Commissions          [Bell] [Avatar] │
├──────────────────────────────────────────────────────────┤
│                                                          │
│                     Page Content                         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

On mobile: hamburger menu reveals a slide-in sidebar drawer.

**Top nav items (Agent role):**
- Dashboard
- Clients
- Products
- Applications
- Commissions

**Additional items (Agency Admin):**
- Reports

**Additional items (Super Admin):**
- Admin (carriers, products, agencies, users)

**Right side of nav:**
- Notification bell with unread badge count
- Avatar dropdown: Profile, Settings, Logout

---

## Pages

### 1. Login (Passwordless PIN)

Two-step flow on the same full-page card:

**Step 1 — Enter email:**
```
┌─────────────────────────────────────────┐
│                                         │
│              [LOGO]                     │
│                                         │
│   ┌─────────────────────────────────┐   │
│   │  Sign in to your account        │   │
│   │                                 │   │
│   │  Email address                  │   │
│   │  [agent@example.com           ] │   │
│   │                                 │   │
│   │  [       Send me a PIN        ] │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Step 2 — Enter PIN (after email is sent):**
```
┌─────────────────────────────────────────┐
│              [LOGO]                     │
│                                         │
│   ┌─────────────────────────────────┐   │
│   │  Check your email               │   │
│   │  We sent a 6-digit PIN to       │   │
│   │  agent@example.com              │   │
│   │                                 │   │
│   │  [_][_][_][_][_][_]             │   │
│   │                                 │   │
│   │  PIN expires in 9:45            │   │
│   │                                 │   │
│   │  [       Verify PIN           ] │   │
│   │                                 │   │
│   │  Didn't receive it?             │   │
│   │  [Resend PIN]  ·  [Change email]│   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

- PIN input is 6 individual single-digit boxes that auto-advance focus on input
- Countdown timer shows remaining PIN validity
- Error displayed inline: "Invalid PIN. X attempts remaining."
- After 5 failed attempts: "Account temporarily locked. Try again in 15 minutes."

---

### 2. Dashboard

```
┌─────────────────────────────────────────────────────────┐
│  Good morning, Jane                  [+ New Client]      │
│                                      [+ New Application] │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │
│  │  Active      │ │  Pending     │ │  This Month  │     │
│  │  Clients     │ │  Apps        │ │  Commission  │     │
│  │     24       │ │      3       │ │  $4,200      │     │
│  └──────────────┘ └──────────────┘ └──────────────┘     │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  Applications Needing Attention          (3 items)        │
│  ┌────────────────────────────────────────────────────┐  │
│  │ ⚠ Smith, John — Missing ID document    [View]     │  │
│  │ ⚠ Lee, Mary — Client signature pending [View]     │  │
│  │ ✕ Garcia, Bob — Application returned   [View]     │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Recent Activity                                         │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Quote generated for Williams, Pat    2 hrs ago     │  │
│  │ New client added: Torres, Ana        Yesterday     │  │
│  │ Application submitted: Chen, David   Jan 14        │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

### 3. Client List

```
┌──────────────────────────────────────────────────────────┐
│  Clients                              [+ New Client]      │
│  ─────────────────────────────────────────────────────── │
│  [🔍 Search clients...      ]  [Status ▾]  [Sort ▾]      │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ NAME ↑          EMAIL              STATUS   APPS   │  │
│  │────────────────────────────────────────────────────│  │
│  │ Doe, John       john@email.com     Active    2     │  │
│  │ Garcia, Maria   maria@email.com    Active    0     │  │
│  │ Smith, Robert   —                  Active    1     │  │
│  │ ...                                                │  │
│  └────────────────────────────────────────────────────┘  │
│  Showing 1–20 of 42              [← Prev]  [Next →]      │
└──────────────────────────────────────────────────────────┘
```

Row click → Client Detail.

---

### 4. Client Detail

Two-column layout on desktop, stacked on mobile:

```
┌──────────────────────────────────────────────────────────┐
│  ← Clients  /  Doe, John                                 │
│  ─────────────────────────────────────────────────────── │
│  ┌─────────────────────────┐  ┌────────────────────────┐ │
│  │ Profile                 │  │ Applications (2)       │ │
│  │ ─────────────────────── │  │ ──────────────────────│ │
│  │ DOB: May 15, 1960       │  │ • FIA — In Review  [→]│ │
│  │ Email: john@example.com │  │ • FA  — Approved   [→]│ │
│  │ Phone: 555-0100         │  │                       │ │
│  │ Address: ...            │  │ [+ New Application]   │ │
│  │ Risk: Moderate          │  └────────────────────────┘ │
│  │ [Edit Profile]          │                            │ │
│  └─────────────────────────┘  ┌────────────────────────┐ │
│                               │ Quotes (1)             │ │
│  ┌─────────────────────────┐  │ ──────────────────────│ │
│  │ Beneficiaries           │  │ • SecureIncome 7 [→]  │ │
│  │ ─────────────────────── │  │ [+ New Quote]         │ │
│  │ Primary: Jane Doe 100%  │  └────────────────────────┘ │
│  │ [Edit Beneficiaries]    │                            │ │
│  └─────────────────────────┘  ┌────────────────────────┐ │
│                               │ Documents (3)          │ │
│  ┌─────────────────────────┐  │ ──────────────────────│ │
│  │ Financial Profile       │  │ • Driver's License ✓  │ │
│  │ Income: $80,000/yr      │  │ • Suitability Form ✓  │ │
│  │ Net Worth: $500,000     │  │ [Upload Document]     │ │
│  │ Suitability: ✓ Current  │  └────────────────────────┘ │
│  └─────────────────────────┘                            │ │
└──────────────────────────────────────────────────────────┘
```

---

### 5. Product Catalog

```
┌──────────────────────────────────────────────────────────┐
│  Products                                                 │
│  ─────────────────────────────────────────────────────── │
│  [All Types ▾]  [State ▾]  [Min Premium ▾]  [Carrier ▾] │
│                                                          │
│  ┌───────────────────┐ ┌───────────────────┐             │
│  │ Acme Life         │ │ Pacific Annuity   │             │
│  │ SecureIncome 7    │ │ Growth Plus FIA   │             │
│  │ Fixed Indexed     │ │ Fixed Indexed     │             │
│  │ Min: $25,000      │ │ Min: $10,000      │             │
│  │ Cap: 9% Floor: 0% │ │ Cap: 7% Floor: 0% │             │
│  │ Commission: 6.5%  │ │ Commission: 7.0%  │             │
│  │ Rating: A+        │ │ Rating: A         │             │
│  │ [View] [Compare]  │ │ [View] [Compare]  │             │
│  └───────────────────┘ └───────────────────┘             │
│                                                          │
│  Selected for compare: SecureIncome 7  [Compare Now →]   │
└──────────────────────────────────────────────────────────┘
```

Compare view: 3-column table with product attributes as rows.

---

### 6. Quote Builder

Full-page form, two sections:

**Left panel — Inputs:**
- Client selector (searchable dropdown, pre-fill if arrived from client)
- Product selector (searchable, shows type + carrier)
- Premium amount
- Purchase type (radio: Single / Flexible)
- Payout option (radio: Immediate / Deferred)
- Deferral period (conditional on Deferred)
- Rider checkboxes with annual cost shown per rider

**Right panel — Live Preview (updates on input change):**
```
┌────────────────────────────────┐
│  Quote Summary                 │
│  ──────────────────────────── │
│  Client: John Doe              │
│  Product: SecureIncome 7       │
│  Premium: $100,000             │
│  Deferral: 10 years            │
│                                │
│  Projections                   │
│  Year  Guaranteed  Illustrated │
│  5     $121,899    $139,050    │
│  10    $148,595    $193,201    │
│  20    $220,832    $374,506    │
│                                │
│  Annual Income (after 10 yrs)  │
│  Guaranteed:  $7,800           │
│  Illustrated: $10,500          │
│                                │
│  Est. Commission:  $6,500      │
│                                │
│  [Save Quote]  [Print PDF]     │
└────────────────────────────────┘
```

---

### 7. Application Wizard

Progress stepper at top showing all steps:

```
Step 1 ──── Step 2 ──── Step 3 ──── Step 4 ──── Step 5 ──── Step 6 ──── Step 7
Client &    Owner &    Funding    Riders     Suitability  Documents  Review &
Product     Annuitant                                                Submit
```

Current step highlighted. Completed steps show a checkmark.

**Navigation:** [← Back] and [Continue →] buttons at bottom of each step.

**Step 7 — Review & Submit** shows a full read-only summary of all entered data with a [Submit Application] button. Clicking submit shows a confirmation modal.

---

### 8. Application Detail

Status badge prominently displayed at top (color-coded):
- `Draft` — Gray
- `Pending Signatures` — Amber
- `Submitted` — Blue
- `In Review` — Purple
- `Approved` — Green
- `Declined` — Red
- `Returned` — Orange

Sections:
- Owner & Annuitant info (read-only)
- Funding details
- Elected riders
- Documents checklist (with upload button for missing items)
- Signature status (agent ✓, client pending / ✓)
- Timeline of status changes

---

### 9. Commission Ledger

```
┌──────────────────────────────────────────────────────────┐
│  Commissions        [Period: January 2025 ▾]  [Export CSV]│
│  ─────────────────────────────────────────────────────── │
│  Expected: $15,000  |  Paid: $12,000  |  Pending: $3,000  │
│  ─────────────────────────────────────────────────────── │
│  CLIENT          PRODUCT        PREMIUM    COMM.  STATUS  │
│  Doe, John       SecureIncome   $100,000   $6,500 Paid    │
│  Garcia, Maria   Growth Plus    $50,000    $3,500 Pending │
│  Smith, Robert   Fixed Shield   $75,000    $5,250 Paid    │
│  ...                                                      │
└──────────────────────────────────────────────────────────┘
```

---

## Responsive Behavior

| Element | Desktop | Mobile |
|---|---|---|
| Navigation | Top horizontal nav | Hamburger → drawer |
| Dashboard stats | 3-column row | Single column |
| Client detail | 2-column | Stacked single column |
| Quote builder | Side-by-side form + preview | Form above, preview below |
| Application wizard | Horizontal stepper | Compact step indicator (1 of 7) |
| Tables | Full columns | Prioritized columns, tap row to expand |

---

### 10. AI Features

#### Product Recommendation Panel (Client Detail page)
```
┌────────────────────────────────────────────────────────────┐
│  ✨ AI Product Recommendations                             │
│  Based on John's profile: conservative, income-focused     │
│  ─────────────────────────────────────────────────────── │
│  #1  SecureIncome 7 · Acme Life · FIA                      │
│      "Aligns with John's conservative risk tolerance        │
│      while providing upside potential via the 9% cap.      │
│      The income rider supports his retirement income goal." │
│      [View Product]  [Start Quote]                         │
│                                                            │
│  #2  Fixed Shield Plus · Pacific Annuity · Fixed           │
│      "Provides a guaranteed 4.2% rate with no market       │
│      risk — ideal for capital preservation goals."         │
│      [View Product]  [Start Quote]                         │
│                                                            │
│  [Refresh Recommendations]                                 │
└────────────────────────────────────────────────────────────┘
```

#### Suitability Check (Application Wizard, Step 7)
Before the Submit button, if AI flags are present:
```
┌────────────────────────────────────────────────────────────┐
│  ⚠ AI Suitability Review — 1 issue found                   │
│  ─────────────────────────────────────────────────────── │
│  Conservative client selected a product with no floor      │
│  protection. Consider a fixed annuity or an FIA with a    │
│  guaranteed minimum rate.                                  │
│                                                            │
│  Override reason (required to proceed):                    │
│  [Client understands the risk and...                     ] │
│                                                            │
│  [Acknowledge & Proceed]                                   │
└────────────────────────────────────────────────────────────┘
```

#### Quote Explainer (Quote Results)
Below the projections table:
```
┌────────────────────────────────────────────────────────────┐
│  ✨ Plain-English Summary                                  │
│  "With a $100,000 premium, John is guaranteed at least     │
│   $148,000 after 10 years, with potential growth up to     │
│   $193,000. Starting in year 11, he'd receive a           │
│   guaranteed $7,800/year for life."                        │
│                                [Copy to clipboard]         │
└────────────────────────────────────────────────────────────┘
```

#### AI Chat Assistant (all pages)
Collapsed state: floating button in bottom-right corner — `✨ Ask AI`

Expanded state (slide-up panel, 400px wide):
```
┌───────────────────────────────────┐
│  AI Assistant              [✕]   │
│  ─────────────────────────────── │
│                                   │
│  [AI] Hi Jane! You're viewing     │
│  John Doe's application. How can  │
│  I help?                          │
│                                   │
│  [You] What docs are needed for   │
│  a 1035 exchange?                 │
│                                   │
│  [AI] For a 1035 exchange, you'll │
│  need: (1) the completed 1035     │
│  exchange form signed by the      │
│  owner, (2) a current policy      │
│  statement from the existing      │
│  carrier, and (3) ...             │
│                                   │
│  ─────────────────────────────── │
│  [Ask a question...          ][→] │
│                                   │
│  AI responses are for advisor     │
│  reference only and do not        │
│  constitute financial advice.     │
└───────────────────────────────────┘
```

---

## Accessibility

- WCAG 2.1 AA compliance target
- All form inputs have associated `<label>`
- Color is never the sole indicator of state (icons + text accompany badge colors)
- Keyboard navigation tested for all interactive elements
- Focus ring visible on all focusable elements
- Screen reader announcements for async state changes (loading, success, error)
