# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Personal Finance Management · Created: 2026-05-20

## Philosophy

This model keeps core, heavily-queried fields as typed relational columns while using JSONB columns for variable, extensible, and tenant-specific data. It is a pragmatic middle ground: the relational backbone provides data integrity and performant joins, while JSONB columns absorb the flexibility needed for multi-currency support, jurisdiction-specific fields, custom user preferences, aggregator-specific metadata, and AI model outputs that evolve rapidly.

The core principle is **structured where it matters, flexible where it changes**. Transaction date, amount, and account_id are always relational columns because every query touches them. But the raw data from Plaid vs. MX vs. FDX has different shapes, AI confidence metadata evolves as models improve, and users in different countries need different fields for tax reporting. JSONB handles all of this without schema migrations.

This pattern is widely used by modern SaaS products (Stripe, Shopify, Linear) that need to iterate quickly while maintaining relational integrity for core operations. It is particularly well-suited for an MVP that will evolve rapidly during its first year.

**Best for:** Teams building an MVP that needs to iterate quickly, support multiple aggregators with different data shapes, and accommodate international users with jurisdiction-specific requirements.

**Trade-offs:**
- Pro: Fast to iterate — new fields can be added to JSONB without migrations
- Pro: Fewer tables than fully normalized (JSONB replaces many small lookup tables)
- Pro: Aggregator-specific metadata stored without schema changes per aggregator
- Pro: Supports multi-currency and international variations without nullable column sprawl
- Pro: JSONB GIN indexes provide good query performance on flexible fields
- Con: JSONB fields lack database-enforced constraints (validation must happen in application code)
- Con: Risk of schema drift if JSONB structure is not documented and validated
- Con: Harder to write complex aggregation queries that span both relational and JSONB fields
- Con: ORM support for JSONB varies — some frameworks handle it poorly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FDX API v6.5 | Raw FDX response stored in `aggregator_data` JSONB; mapped fields extracted to relational columns |
| Plaid Transaction Schema | Plaid-specific fields (personal_finance_category, location, payment_channel) stored in `aggregator_data` JSONB |
| ISO 4217 | Currency code as relational CHAR(3) column on all monetary tables |
| ISO 3166-1 | Country/jurisdiction codes in relational columns; jurisdiction-specific tax fields in JSONB |
| ISO 8601 | All timestamps as TIMESTAMPTZ; dates as DATE |
| JSON Schema (Draft 2020-12) | Application-layer validation schemas defined for each JSONB column structure |
| Plaid PFC v2 Taxonomy | Category mapping stored in `category_mapping` JSONB on categories table |

---

## Users & Households

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    preferred_currency CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    locale VARCHAR(10) NOT NULL DEFAULT 'en-US',
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    preferences JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "date_format": "MM/DD/YYYY",
    --   "number_format": "1,234.56",
    --   "default_budget_method": "zero_based",
    --   "notification_settings": {
    --     "email_insights": true,
    --     "push_anomalies": true,
    --     "weekly_summary": true
    --   },
    --   "dashboard_layout": ["net_worth", "budget", "recent_transactions", "goals"],
    --   "onboarding_completed": true
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE households (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    owner_user_id UUID NOT NULL REFERENCES users(id),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    country_code CHAR(2) NOT NULL DEFAULT 'US', -- ISO 3166-1
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "tax_jurisdiction": "US-CA",
    --   "fiscal_year_start_month": 1,
    --   "tax_filing_status": "married_jointly",
    --   "business_mode_enabled": false,
    --   "data_retention_months": 84
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE household_members (
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'member',
    permissions JSONB NOT NULL DEFAULT '{}',
    -- permissions example:
    -- {
    --   "can_edit_budget": true,
    --   "can_add_accounts": false,
    --   "can_view_investments": true,
    --   "visible_account_ids": null  -- null = all accounts
    -- }
    invited_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    accepted_at TIMESTAMPTZ,
    PRIMARY KEY (household_id, user_id)
);
```

## Accounts & Connections

```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    official_name VARCHAR(255),
    account_type VARCHAR(30) NOT NULL, -- 'checking', 'savings', 'credit_card', 'loan', 'investment', 'mortgage', 'other'
    account_subtype VARCHAR(50),
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    current_balance NUMERIC(15,2),
    available_balance NUMERIC(15,2),
    credit_limit NUMERIC(15,2),
    mask VARCHAR(10),
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    is_manual BOOLEAN NOT NULL DEFAULT FALSE,
    display_order INTEGER NOT NULL DEFAULT 0,
    aggregator VARCHAR(20), -- 'plaid', 'mx', 'finicity', 'fdx', 'manual'
    aggregator_data JSONB NOT NULL DEFAULT '{}',
    -- Plaid example:
    -- {
    --   "item_id": "plaid_item_abc",
    --   "account_id": "plaid_acc_xyz",
    --   "access_token_ref": "vault:plaid:item_abc",
    --   "institution_id": "ins_3",
    --   "institution_name": "Chase",
    --   "logo_base64": null,
    --   "consent_expiration": "2027-05-20T00:00:00Z",
    --   "last_cursor": "CAoQAQ==",
    --   "error": null
    -- }
    --
    -- FDX example:
    -- {
    --   "fdx_account_id": "fdx_acc_123",
    --   "fdx_provider_id": "chase",
    --   "consent_id": "consent_456",
    --   "consent_granted_at": "2026-01-15T10:00:00Z",
    --   "supported_containers": ["deposit", "investment"],
    --   "last_sync_cursor": "2026-05-20T08:00:00Z"
    -- }
    --
    -- Manual example:
    -- {
    --   "notes": "Cash on hand",
    --   "interest_rate": 0.045,
    --   "original_loan_amount": 250000.00
    -- }
    interest_rate NUMERIC(7,4), -- APR for loans/savings, extracted for queries
    last_synced_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_accounts_household ON accounts(household_id);
CREATE INDEX idx_accounts_type ON accounts(household_id, account_type);
CREATE INDEX idx_accounts_aggregator ON accounts USING GIN(aggregator_data);
```

## Categories

```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES categories(id) ON DELETE CASCADE, -- NULL = top-level group
    name VARCHAR(100) NOT NULL,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    is_income BOOLEAN NOT NULL DEFAULT FALSE,
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    icon VARCHAR(50),
    color VARCHAR(7),
    category_mapping JSONB NOT NULL DEFAULT '{}',
    -- category_mapping example:
    -- {
    --   "plaid_pfc_v2": {
    --     "primary": "FOOD_AND_DRINK",
    --     "detailed": ["FOOD_AND_DRINK_GROCERIES", "FOOD_AND_DRINK_SUPERMARKETS"]
    --   },
    --   "mx_category_code": 107,
    --   "fdx_category": "food",
    --   "tax_category": "meals_entertainment",
    --   "ynab_import_name": "Groceries"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_categories_household ON categories(household_id);
CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_categories_mapping ON categories USING GIN(category_mapping);
```

## Transactions

```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    date DATE NOT NULL,
    amount NUMERIC(15,2) NOT NULL, -- negative = outflow, positive = inflow
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    payee_name VARCHAR(500),
    merchant_name VARCHAR(255),
    description TEXT,
    memo TEXT,
    cleared_status VARCHAR(20) NOT NULL DEFAULT 'uncleared',
    is_pending BOOLEAN NOT NULL DEFAULT FALSE,
    is_split_parent BOOLEAN NOT NULL DEFAULT FALSE,
    parent_transaction_id UUID REFERENCES transactions(id) ON DELETE CASCADE,
    transfer_pair_id UUID REFERENCES transactions(id) ON DELETE SET NULL,
    flag_color VARCHAR(20),
    tags TEXT[] NOT NULL DEFAULT '{}',
    -- AI categorisation fields (relational for filtering)
    ai_category_confidence NUMERIC(3,2),
    ai_categorized_at TIMESTAMPTZ,
    user_recategorized_at TIMESTAMPTZ,
    -- Flexible metadata from aggregator and AI
    extra JSONB NOT NULL DEFAULT '{}',
    -- extra example (Plaid-sourced):
    -- {
    --   "aggregator_id": "plaid_tx_789",
    --   "personal_finance_category": {
    --     "primary": "FOOD_AND_DRINK",
    --     "detailed": "FOOD_AND_DRINK_GROCERIES",
    --     "confidence_level": "VERY_HIGH"
    --   },
    --   "location": {
    --     "address": "123 Main St",
    --     "city": "San Francisco",
    --     "region": "CA",
    --     "postal_code": "94102",
    --     "country": "US",
    --     "lat": 37.7749,
    --     "lon": -122.4194
    --   },
    --   "payment_channel": "in store",
    --   "check_number": null,
    --   "is_reimbursable": true,
    --   "reimbursed_at": null,
    --   "reimbursed_by": null,
    --   "receipt_url": null,
    --   "tax_deductible": false,
    --   "business_expense": false
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tx_household_date ON transactions(household_id, date DESC);
CREATE INDEX idx_tx_account_date ON transactions(account_id, date DESC);
CREATE INDEX idx_tx_category ON transactions(category_id);
CREATE INDEX idx_tx_pending ON transactions(account_id, is_pending) WHERE is_pending = TRUE;
CREATE INDEX idx_tx_parent ON transactions(parent_transaction_id) WHERE parent_transaction_id IS NOT NULL;
CREATE INDEX idx_tx_tags ON transactions USING GIN(tags);
CREATE INDEX idx_tx_extra ON transactions USING GIN(extra);
CREATE INDEX idx_tx_payee ON transactions(household_id, payee_name);
```

## Recurring Transactions

```sql
CREATE TABLE recurring_patterns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    payee_name VARCHAR(500),
    expected_amount NUMERIC(15,2) NOT NULL,
    amount_variance NUMERIC(15,2) NOT NULL DEFAULT 0, -- acceptable deviation
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    frequency VARCHAR(20) NOT NULL, -- 'weekly', 'biweekly', 'monthly', 'quarterly', 'yearly'
    next_expected_date DATE NOT NULL,
    is_income BOOLEAN NOT NULL DEFAULT FALSE,
    is_subscription BOOLEAN NOT NULL DEFAULT FALSE,
    auto_detected BOOLEAN NOT NULL DEFAULT FALSE,
    status VARCHAR(20) NOT NULL DEFAULT 'active', -- 'active', 'paused', 'cancelled'
    detection_data JSONB NOT NULL DEFAULT '{}',
    -- detection_data example:
    -- {
    --   "detected_at": "2026-05-01T10:00:00Z",
    --   "confidence": 0.92,
    --   "sample_transaction_ids": ["uuid1", "uuid2", "uuid3"],
    --   "amount_history": [15.99, 15.99, 17.99],
    --   "date_history": ["2026-02-15", "2026-03-15", "2026-04-15"],
    --   "fee_creep_detected": true,
    --   "fee_creep_amount": 2.00,
    --   "merchant_category": "streaming_services"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_recurring_household ON recurring_patterns(household_id);
CREATE INDEX idx_recurring_next ON recurring_patterns(next_expected_date) WHERE status = 'active';
```

## Budgets

```sql
CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL DEFAULT 'My Budget',
    method VARCHAR(20) NOT NULL DEFAULT 'zero_based',
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "rollover_enabled": true,
    --   "rollover_overspent": "deduct_next_month",
    --   "envelope_groups": ["Needs", "Wants", "Savings"],
    --   "50_30_20_custom_split": {"needs": 50, "wants": 30, "savings": 20}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE budget_periods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id UUID NOT NULL REFERENCES budgets(id) ON DELETE CASCADE,
    month DATE NOT NULL, -- first of month
    income_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    ready_to_assign NUMERIC(15,2) NOT NULL DEFAULT 0,
    age_of_money_days INTEGER,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (budget_id, month)
);

CREATE TABLE budget_allocations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_period_id UUID NOT NULL REFERENCES budget_periods(id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    budgeted NUMERIC(15,2) NOT NULL DEFAULT 0,
    activity NUMERIC(15,2) NOT NULL DEFAULT 0,
    available NUMERIC(15,2) NOT NULL DEFAULT 0,
    carryover NUMERIC(15,2) NOT NULL DEFAULT 0,
    goal_target JSONB,
    -- goal_target example:
    -- {
    --   "type": "monthly_savings",
    --   "target_amount": 500.00,
    --   "target_date": "2026-12-31",
    --   "on_track": true,
    --   "needed_this_month": 500.00
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (budget_period_id, category_id)
);

CREATE INDEX idx_budget_periods ON budget_periods(budget_id, month);
CREATE INDEX idx_budget_alloc_period ON budget_allocations(budget_period_id);
CREATE INDEX idx_budget_alloc_category ON budget_allocations(category_id);
```

## Goals

```sql
CREATE TABLE goals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    goal_type VARCHAR(30) NOT NULL,
    target_amount NUMERIC(15,2) NOT NULL,
    current_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    target_date DATE,
    linked_account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
    linked_category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    monthly_contribution_target NUMERIC(15,2),
    priority INTEGER NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    completed_at TIMESTAMPTZ,
    icon VARCHAR(50),
    color VARCHAR(7),
    modeling_data JSONB NOT NULL DEFAULT '{}',
    -- modeling_data example:
    -- {
    --   "strategy": "avalanche",
    --   "projected_completion_date": "2027-08-15",
    --   "monthly_scenarios": [
    --     {"contribution": 300, "months_to_goal": 24},
    --     {"contribution": 500, "months_to_goal": 14},
    --     {"contribution": 750, "months_to_goal": 9}
    --   ],
    --   "competing_goals": ["uuid1", "uuid2"],
    --   "trade_off_recommendation": "Reducing dining out by $150/mo accelerates this by 3 months",
    --   "debt_details": {
    --     "interest_rate": 0.199,
    --     "minimum_payment": 85.00,
    --     "total_interest_saved": 2340.00
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_goals_household ON goals(household_id);
```

## Investments

```sql
CREATE TABLE investment_holdings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    ticker_symbol VARCHAR(20),
    security_name VARCHAR(255) NOT NULL,
    security_type VARCHAR(30),
    quantity NUMERIC(20,8) NOT NULL,
    cost_basis_per_unit NUMERIC(15,4),
    current_price NUMERIC(15,4),
    current_value NUMERIC(15,2),
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    security_data JSONB NOT NULL DEFAULT '{}',
    -- security_data example:
    -- {
    --   "aggregator_security_id": "plaid_sec_abc",
    --   "isin": "US0378331005",
    --   "cusip": "037833100",
    --   "asset_class": "equity",
    --   "sector": "technology",
    --   "expense_ratio": 0.0003,
    --   "dividend_yield": 0.0055,
    --   "52_week_high": 198.23,
    --   "52_week_low": 164.08,
    --   "last_price_source": "plaid"
    -- }
    last_price_update TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_holdings_account ON investment_holdings(account_id);
```

## Net Worth & Cash Flow

```sql
CREATE TABLE net_worth_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    snapshot_date DATE NOT NULL,
    total_assets NUMERIC(15,2) NOT NULL,
    total_liabilities NUMERIC(15,2) NOT NULL,
    net_worth NUMERIC(15,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    breakdown JSONB NOT NULL DEFAULT '{}',
    -- breakdown example:
    -- {
    --   "by_type": {
    --     "checking": 12500.00,
    --     "savings": 35000.00,
    --     "investment": 185000.00,
    --     "credit_card": -4200.00,
    --     "mortgage": -320000.00
    --   },
    --   "by_account": [
    --     {"id": "uuid", "name": "Chase Checking", "balance": 5432.10, "type": "checking"},
    --     {"id": "uuid", "name": "Ally Savings", "balance": 35000.00, "type": "savings"}
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (household_id, snapshot_date)
);

CREATE INDEX idx_nw_household_date ON net_worth_snapshots(household_id, snapshot_date DESC);
```

## AI Conversations & Insights

```sql
CREATE TABLE ai_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    title VARCHAR(255),
    context JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "initial_query_type": "spending_analysis",
    --   "date_range": {"start": "2026-01-01", "end": "2026-05-20"},
    --   "account_ids": ["uuid1", "uuid2"],
    --   "model_version": "claude-sonnet-4-20250514"
    -- }
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_message_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ai_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES ai_conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    tool_calls JSONB, -- MCP tool invocations
    tokens_used INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ai_insights (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    insight_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    severity VARCHAR(20) NOT NULL DEFAULT 'info',
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed BOOLEAN NOT NULL DEFAULT FALSE,
    action_taken BOOLEAN NOT NULL DEFAULT FALSE,
    insight_data JSONB NOT NULL DEFAULT '{}',
    -- insight_data example:
    -- {
    --   "related_entity_type": "recurring_pattern",
    --   "related_entity_id": "uuid",
    --   "related_transactions": ["uuid1", "uuid2"],
    --   "old_amount": 15.99,
    --   "new_amount": 17.99,
    --   "annual_impact": 24.00,
    --   "recommendation": "Consider cancelling — you haven't used this service in 45 days",
    --   "confidence": 0.87,
    --   "model_version": "v2.3"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ
);

CREATE TABLE categorisation_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    priority INTEGER NOT NULL DEFAULT 0,
    match_count INTEGER NOT NULL DEFAULT 0,
    rule_definition JSONB NOT NULL,
    -- rule_definition example:
    -- {
    --   "type": "merchant_pattern",
    --   "conditions": [
    --     {"field": "merchant_name", "op": "contains", "value": "whole foods"},
    --     {"field": "amount", "op": "between", "value": [-500, -5]}
    --   ],
    --   "source": "user_correction",
    --   "learned_from_transaction_id": "uuid",
    --   "payee_name_override": "Whole Foods Market"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_conv_user ON ai_conversations(user_id);
CREATE INDEX idx_ai_msg_conv ON ai_messages(conversation_id, created_at);
CREATE INDEX idx_insights_household ON ai_insights(household_id, created_at DESC);
CREATE INDEX idx_insights_unread ON ai_insights(household_id) WHERE is_read = FALSE AND is_dismissed = FALSE;
CREATE INDEX idx_cat_rules_household ON categorisation_rules(household_id, priority DESC);
```

## Data Import/Export

```sql
CREATE TABLE import_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    source_type VARCHAR(20) NOT NULL, -- 'csv', 'ofx', 'qfx', 'ynab', 'mint', 'monarch'
    file_name VARCHAR(255),
    status VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'processing', 'completed', 'failed'
    target_account_id UUID REFERENCES accounts(id),
    results JSONB NOT NULL DEFAULT '{}',
    -- results example:
    -- {
    --   "total_rows": 1250,
    --   "imported": 1230,
    --   "duplicates_skipped": 15,
    --   "errors": 5,
    --   "error_details": [
    --     {"row": 42, "error": "Invalid date format", "raw": "15/32/2026"}
    --   ],
    --   "date_range": {"start": "2020-01-01", "end": "2026-05-15"}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE INDEX idx_import_household ON import_jobs(household_id, created_at DESC);
```

## API Tokens

```sql
CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_tokens_hash ON api_tokens(token_hash);
```

---

## Example: Querying JSONB Fields

```sql
-- Find all transactions at a specific location (Plaid location data in JSONB)
SELECT id, date, amount, payee_name, extra->'location'->>'city' AS city
FROM transactions
WHERE household_id = 'household-uuid'
  AND extra->'location'->>'city' = 'San Francisco'
ORDER BY date DESC;

-- Find all subscriptions where fee creep was detected
SELECT id, payee_name, expected_amount,
       detection_data->>'fee_creep_amount' AS price_increase
FROM recurring_patterns
WHERE household_id = 'household-uuid'
  AND (detection_data->>'fee_creep_detected')::BOOLEAN = TRUE
ORDER BY (detection_data->>'fee_creep_amount')::NUMERIC DESC;

-- Find all categories mapped to a specific Plaid PFC primary category
SELECT id, name
FROM categories
WHERE household_id = 'household-uuid'
  AND category_mapping->'plaid_pfc_v2'->>'primary' = 'FOOD_AND_DRINK';

-- Find tax-deductible business expenses
SELECT id, date, amount, payee_name, category_id
FROM transactions
WHERE household_id = 'household-uuid'
  AND (extra->>'tax_deductible')::BOOLEAN = TRUE
  AND date BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY date;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Auth | 3 | users, households, household_members |
| Accounts | 1 | accounts (aggregator details in JSONB) |
| Categories & Rules | 2 | categories (self-referencing hierarchy), categorisation_rules |
| Transactions | 2 | transactions, recurring_patterns |
| Budgets | 3 | budgets, budget_periods, budget_allocations |
| Goals | 1 | goals (modeling scenarios in JSONB) |
| Investments | 1 | investment_holdings (security metadata in JSONB) |
| Net Worth | 1 | net_worth_snapshots (account breakdown in JSONB) |
| AI & Coaching | 3 | ai_conversations, ai_messages, ai_insights |
| Data Import | 1 | import_jobs |
| API & Tokens | 1 | api_tokens |
| **Total** | **19** | 10 fewer than normalized model |

---

## Key Design Decisions

1. **Single categories table with self-referencing parent_id** — replaces the separate `category_groups` and `categories` tables of the normalized model. Top-level categories have `parent_id = NULL`. This uses a simple adjacency list, which works well for 2-level hierarchies.

2. **Payees are not a separate table** — `payee_name` is a VARCHAR column on transactions. The normalized model's payee table adds complexity for a feature (payee management) that most users never interact with directly. Categorisation rules handle the payee-to-category mapping instead.

3. **`extra` JSONB on transactions absorbs aggregator differences** — Plaid sends location data, MX sends enhanced merchant data, FDX sends a different field set. Rather than lowest-common-denominator relational columns or aggregator-specific tables, the raw enrichment data lives in JSONB.

4. **`category_mapping` JSONB on categories** — maps each user category to Plaid PFC codes, MX category codes, and FDX categories simultaneously. Adding a new aggregator does not require a schema migration.

5. **`tags` as TEXT[] with GIN index** — tags are stored directly on the transaction row rather than in a junction table. PostgreSQL array operators (`@>`, `&&`) provide fast tag filtering.

6. **Goal modeling scenarios in JSONB** — debt payoff strategies, multi-goal trade-off recommendations, and scenario modeling outputs are stored in `modeling_data` JSONB. These structures evolve rapidly as the AI improves and do not benefit from relational constraints.

7. **`detection_data` on recurring_patterns** — the AI detection metadata (confidence, sample transactions, fee creep analysis) is stored in JSONB because it changes with every model iteration. The key operational fields (amount, frequency, next_date) remain relational for indexing and querying.

8. **Import job results in JSONB** — import results vary by source type (CSV has row numbers, OFX has different error types). JSONB handles this naturally without separate result tables per import format.
