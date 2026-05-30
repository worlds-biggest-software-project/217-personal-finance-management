# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Personal Finance Management · Created: 2026-05-20

## Philosophy

This model follows traditional third-normal-form (3NF) relational design where every concept gets its own table with strict foreign key constraints. Accounts, transactions, budgets, goals, categories, and payees are all first-class entities with explicit relationships. This is the pattern used by most mature financial software (Quicken, enterprise banking systems) and aligns closely with how the YNAB API exposes its data model.

The core principle is **data integrity through structure**. Every relationship is enforced at the database level. Category hierarchies, budget allocations, and goal contributions are all tracked through dedicated junction and allocation tables. There is no ambiguity about where data lives or how entities relate.

This approach is best suited for teams that value query predictability, strong referential integrity, and straightforward ORM mapping. It produces the most tables but the simplest individual queries.

**Best for:** Teams building a production-grade, compliance-oriented PFM with well-understood requirements and no need for schema flexibility per tenant.

**Trade-offs:**
- Pro: Maximum data integrity — the database enforces all business rules
- Pro: Standard SQL queries, easy to reason about, excellent tooling support
- Pro: Aligns with FDX and YNAB API entity structures for clean API mapping
- Pro: Simple to add indexes, run reports, and build aggregations
- Con: Schema migrations required for every new feature or field
- Con: Many tables and joins for common queries (e.g., budget summary requires 4-5 joins)
- Con: Multi-currency and jurisdiction-specific fields require schema changes or nullable columns
- Con: Less flexible for rapid prototyping and MVP iteration

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FDX API v6.5 | Account types, transaction categories, and institution identifiers align with FDX entity definitions |
| Plaid Transaction Schema | Transaction fields (amount, date, merchant_name, personal_finance_category) map directly to columns |
| YNAB API | Budget/category/payee entity model mirrors YNAB's OpenAPI spec structure |
| ISO 4217 | Currency codes stored as 3-character codes per the standard |
| ISO 3166-1 | Country codes for institution and user locale |
| ISO 8601 | All dates and timestamps stored in ISO 8601 format (TIMESTAMPTZ) |
| OAuth 2.0 (RFC 6749) | User authentication token storage follows OAuth 2.0 patterns |

---

## Core Identity & Authentication

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255), -- NULL if using OAuth-only
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    preferred_currency CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    locale VARCHAR(10) NOT NULL DEFAULT 'en-US',
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE households (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    owner_user_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE household_members (
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'member', -- 'owner', 'member', 'viewer'
    invited_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    accepted_at TIMESTAMPTZ,
    PRIMARY KEY (household_id, user_id)
);

CREATE TABLE oauth_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL, -- 'google', 'apple', 'github'
    provider_user_id VARCHAR(255) NOT NULL,
    access_token_encrypted BYTEA,
    refresh_token_encrypted BYTEA,
    token_expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (provider, provider_user_id)
);

CREATE INDEX idx_oauth_connections_user ON oauth_connections(user_id);
```

## Financial Institutions & Connections

```sql
CREATE TABLE institutions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    plaid_institution_id VARCHAR(100) UNIQUE,
    fdx_institution_id VARCHAR(100) UNIQUE,
    logo_url TEXT,
    country_code CHAR(2) NOT NULL DEFAULT 'US', -- ISO 3166-1
    primary_color VARCHAR(7), -- hex color
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE aggregator_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    institution_id UUID REFERENCES institutions(id),
    aggregator VARCHAR(20) NOT NULL, -- 'plaid', 'mx', 'finicity', 'fdx', 'manual'
    aggregator_item_id VARCHAR(255), -- Plaid item_id or equivalent
    access_token_encrypted BYTEA,
    consent_expires_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'active', -- 'active', 'error', 'revoked', 'pending'
    last_synced_at TIMESTAMPTZ,
    error_code VARCHAR(100),
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_aggregator_connections_household ON aggregator_connections(household_id);
```

## Accounts

```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    connection_id UUID REFERENCES aggregator_connections(id) ON DELETE SET NULL,
    institution_id UUID REFERENCES institutions(id),
    name VARCHAR(255) NOT NULL,
    official_name VARCHAR(255),
    account_type VARCHAR(30) NOT NULL, -- 'checking', 'savings', 'credit_card', 'loan', 'investment', 'mortgage', 'other'
    account_subtype VARCHAR(50), -- 'money_market', 'cd', '401k', 'ira', 'brokerage', etc.
    currency_code CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    current_balance NUMERIC(15,2),
    available_balance NUMERIC(15,2),
    credit_limit NUMERIC(15,2),
    mask VARCHAR(10), -- last 4 digits
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    is_manual BOOLEAN NOT NULL DEFAULT FALSE,
    display_order INTEGER NOT NULL DEFAULT 0,
    aggregator_account_id VARCHAR(255), -- external ID from Plaid/MX/FDX
    last_synced_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_accounts_household ON accounts(household_id);
CREATE INDEX idx_accounts_connection ON accounts(connection_id);
CREATE INDEX idx_accounts_type ON accounts(household_id, account_type);
```

## Categories

```sql
CREATE TABLE category_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_system BOOLEAN NOT NULL DEFAULT FALSE, -- TRUE for built-in groups (Income, Transfer)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (household_id, name)
);

CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_group_id UUID NOT NULL REFERENCES category_groups(id) ON DELETE CASCADE,
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    plaid_primary_category VARCHAR(100), -- maps to Plaid PFC primary
    plaid_detailed_category VARCHAR(100), -- maps to Plaid PFC detailed
    icon VARCHAR(50),
    color VARCHAR(7),
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (household_id, category_group_id, name)
);

CREATE INDEX idx_categories_household ON categories(household_id);
CREATE INDEX idx_categories_group ON categories(category_group_id);
```

## Payees

```sql
CREATE TABLE payees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(500) NOT NULL, -- YNAB allows up to 500 chars
    normalized_name VARCHAR(500), -- lowercase, stripped for matching
    default_category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    is_transfer_payee BOOLEAN NOT NULL DEFAULT FALSE,
    transfer_account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payees_household ON payees(household_id);
CREATE INDEX idx_payees_normalized ON payees(household_id, normalized_name);
```

## Transactions

```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    payee_id UUID REFERENCES payees(id) ON DELETE SET NULL,
    date DATE NOT NULL,
    amount NUMERIC(15,2) NOT NULL, -- negative = outflow, positive = inflow
    currency_code CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    merchant_name VARCHAR(255),
    description TEXT,
    memo TEXT,
    cleared_status VARCHAR(20) NOT NULL DEFAULT 'uncleared', -- 'uncleared', 'cleared', 'reconciled'
    is_pending BOOLEAN NOT NULL DEFAULT FALSE,
    is_split_parent BOOLEAN NOT NULL DEFAULT FALSE,
    parent_transaction_id UUID REFERENCES transactions(id) ON DELETE CASCADE,
    transfer_transaction_id UUID REFERENCES transactions(id) ON DELETE SET NULL,
    aggregator_transaction_id VARCHAR(255), -- external ID from Plaid/MX
    check_number VARCHAR(20),
    flag_color VARCHAR(20),
    is_reimbursable BOOLEAN NOT NULL DEFAULT FALSE,
    reimbursed_at TIMESTAMPTZ,
    ai_category_confidence NUMERIC(3,2), -- 0.00 to 1.00
    ai_categorized_at TIMESTAMPTZ,
    user_recategorized_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_household_date ON transactions(household_id, date DESC);
CREATE INDEX idx_transactions_account_date ON transactions(account_id, date DESC);
CREATE INDEX idx_transactions_category ON transactions(category_id);
CREATE INDEX idx_transactions_payee ON transactions(payee_id);
CREATE INDEX idx_transactions_pending ON transactions(account_id, is_pending) WHERE is_pending = TRUE;
CREATE INDEX idx_transactions_parent ON transactions(parent_transaction_id) WHERE parent_transaction_id IS NOT NULL;
```

## Tags

```sql
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    color VARCHAR(7),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (household_id, name)
);

CREATE TABLE transaction_tags (
    transaction_id UUID NOT NULL REFERENCES transactions(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (transaction_id, tag_id)
);
```

## Recurring Transactions

```sql
CREATE TABLE recurring_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    payee_id UUID REFERENCES payees(id) ON DELETE SET NULL,
    amount NUMERIC(15,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    frequency VARCHAR(20) NOT NULL, -- 'weekly', 'biweekly', 'monthly', 'quarterly', 'yearly'
    start_date DATE NOT NULL,
    end_date DATE,
    next_occurrence DATE NOT NULL,
    description TEXT,
    is_subscription BOOLEAN NOT NULL DEFAULT FALSE,
    is_income BOOLEAN NOT NULL DEFAULT FALSE,
    auto_detected BOOLEAN NOT NULL DEFAULT FALSE, -- TRUE if AI detected this pattern
    detection_confidence NUMERIC(3,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_recurring_household ON recurring_transactions(household_id);
CREATE INDEX idx_recurring_next ON recurring_transactions(next_occurrence);
```

## Budgets

```sql
CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL DEFAULT 'My Budget',
    method VARCHAR(20) NOT NULL DEFAULT 'zero_based', -- 'zero_based', 'envelope', '50_30_20'
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE budget_months (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id UUID NOT NULL REFERENCES budgets(id) ON DELETE CASCADE,
    month DATE NOT NULL, -- first day of month, e.g., '2026-05-01'
    income_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    ready_to_assign NUMERIC(15,2) NOT NULL DEFAULT 0,
    age_of_money_days INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (budget_id, month)
);

CREATE TABLE budget_allocations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_month_id UUID NOT NULL REFERENCES budget_months(id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    budgeted NUMERIC(15,2) NOT NULL DEFAULT 0,
    activity NUMERIC(15,2) NOT NULL DEFAULT 0, -- sum of transactions in this month
    available NUMERIC(15,2) NOT NULL DEFAULT 0, -- budgeted + carryover + activity
    carryover NUMERIC(15,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (budget_month_id, category_id)
);

CREATE INDEX idx_budget_months_budget ON budget_months(budget_id, month);
CREATE INDEX idx_budget_allocations_month ON budget_allocations(budget_month_id);
CREATE INDEX idx_budget_allocations_category ON budget_allocations(category_id);
```

## Goals

```sql
CREATE TABLE goals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    goal_type VARCHAR(30) NOT NULL, -- 'savings', 'debt_payoff', 'emergency_fund', 'custom'
    target_amount NUMERIC(15,2) NOT NULL,
    current_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    target_date DATE,
    linked_account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
    linked_category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    monthly_contribution_target NUMERIC(15,2),
    priority INTEGER NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'active', -- 'active', 'completed', 'paused', 'abandoned'
    completed_at TIMESTAMPTZ,
    icon VARCHAR(50),
    color VARCHAR(7),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE goal_contributions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_id UUID NOT NULL REFERENCES goals(id) ON DELETE CASCADE,
    amount NUMERIC(15,2) NOT NULL,
    date DATE NOT NULL,
    transaction_id UUID REFERENCES transactions(id) ON DELETE SET NULL,
    note TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_goals_household ON goals(household_id);
CREATE INDEX idx_goal_contributions_goal ON goal_contributions(goal_id, date DESC);
```

## Investment Holdings

```sql
CREATE TABLE investment_holdings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    security_name VARCHAR(255) NOT NULL,
    ticker_symbol VARCHAR(20),
    security_type VARCHAR(30), -- 'stock', 'etf', 'mutual_fund', 'bond', 'cash', 'crypto', 'other'
    quantity NUMERIC(20,8) NOT NULL,
    cost_basis_per_unit NUMERIC(15,4),
    current_price NUMERIC(15,4),
    current_value NUMERIC(15,2),
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    aggregator_security_id VARCHAR(255),
    last_price_update TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_holdings_account ON investment_holdings(account_id);
```

## Net Worth Snapshots

```sql
CREATE TABLE net_worth_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    snapshot_date DATE NOT NULL,
    total_assets NUMERIC(15,2) NOT NULL,
    total_liabilities NUMERIC(15,2) NOT NULL,
    net_worth NUMERIC(15,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (household_id, snapshot_date)
);

CREATE TABLE net_worth_snapshot_accounts (
    snapshot_id UUID NOT NULL REFERENCES net_worth_snapshots(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    balance NUMERIC(15,2) NOT NULL,
    PRIMARY KEY (snapshot_id, account_id)
);

CREATE INDEX idx_net_worth_household_date ON net_worth_snapshots(household_id, snapshot_date DESC);
```

## AI & Coaching

```sql
CREATE TABLE ai_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    title VARCHAR(255),
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_message_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ai_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES ai_conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL, -- 'user', 'assistant', 'system'
    content TEXT NOT NULL,
    tokens_used INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ai_insights (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    insight_type VARCHAR(50) NOT NULL, -- 'subscription_alert', 'fee_creep', 'cash_flow_warning', 'goal_progress', 'anomaly'
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    severity VARCHAR(20) NOT NULL DEFAULT 'info', -- 'info', 'warning', 'critical'
    related_transaction_id UUID REFERENCES transactions(id),
    related_account_id UUID REFERENCES accounts(id),
    related_recurring_id UUID REFERENCES recurring_transactions(id),
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_conversations_user ON ai_conversations(user_id);
CREATE INDEX idx_ai_messages_conversation ON ai_messages(conversation_id, created_at);
CREATE INDEX idx_ai_insights_household ON ai_insights(household_id, created_at DESC);
CREATE INDEX idx_ai_insights_unread ON ai_insights(household_id, is_read) WHERE is_read = FALSE;
```

## Categorisation Rules (AI Learning)

```sql
CREATE TABLE categorisation_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    rule_type VARCHAR(20) NOT NULL, -- 'user_correction', 'ai_learned', 'manual_rule'
    match_field VARCHAR(20) NOT NULL, -- 'merchant_name', 'description', 'amount_range'
    match_pattern VARCHAR(500) NOT NULL,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    payee_id UUID REFERENCES payees(id) ON DELETE SET NULL,
    priority INTEGER NOT NULL DEFAULT 0,
    match_count INTEGER NOT NULL DEFAULT 0,
    last_matched_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cat_rules_household ON categorisation_rules(household_id, priority DESC);
```

## Debt Payoff Plans

```sql
CREATE TABLE debt_payoff_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    strategy VARCHAR(20) NOT NULL, -- 'avalanche', 'snowball', 'custom'
    extra_monthly_payment NUMERIC(15,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE debt_payoff_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id UUID NOT NULL REFERENCES debt_payoff_plans(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    current_balance NUMERIC(15,2) NOT NULL,
    interest_rate NUMERIC(5,3) NOT NULL, -- e.g., 0.199 = 19.9% APR
    minimum_payment NUMERIC(15,2) NOT NULL,
    payoff_order INTEGER, -- used for 'custom' strategy
    UNIQUE (plan_id, account_id)
);
```

## API Tokens & Sessions

```sql
CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    token_hash VARCHAR(255) NOT NULL UNIQUE,
    scopes TEXT[] NOT NULL DEFAULT '{}', -- 'read:transactions', 'write:transactions', etc.
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_tokens_user ON api_tokens(user_id);
CREATE INDEX idx_api_tokens_hash ON api_tokens(token_hash);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Auth | 4 | users, households, household_members, oauth_connections |
| Institutions & Connections | 2 | institutions, aggregator_connections |
| Accounts | 1 | accounts (with type/subtype) |
| Categories & Payees | 4 | category_groups, categories, payees, categorisation_rules |
| Transactions | 4 | transactions, tags, transaction_tags, recurring_transactions |
| Budgets | 3 | budgets, budget_months, budget_allocations |
| Goals | 2 | goals, goal_contributions |
| Investments | 1 | investment_holdings |
| Net Worth | 2 | net_worth_snapshots, net_worth_snapshot_accounts |
| AI & Coaching | 3 | ai_conversations, ai_messages, ai_insights |
| Debt Payoff | 2 | debt_payoff_plans, debt_payoff_accounts |
| API & Tokens | 1 | api_tokens |
| **Total** | **29** | |

---

## Key Design Decisions

1. **Household as the primary tenant boundary** — all financial data belongs to a household, not a user. This enables collaborative finance from day one without a later migration. Users belong to households via `household_members`.

2. **Separate category_groups and categories tables** — mirrors YNAB's proven two-level category hierarchy. Categories belong to groups, groups belong to households.

3. **Split transactions via self-referencing foreign key** — `parent_transaction_id` on the transactions table enables split transactions without a separate table. The parent has `is_split_parent = TRUE` and child rows reference it.

4. **Money stored as NUMERIC(15,2)** — avoids floating-point precision errors. All monetary values are in the account's currency with 2 decimal places. Investment quantities use NUMERIC(20,8) for fractional shares.

5. **AI categorisation tracked per transaction** — `ai_category_confidence` and `ai_categorized_at` fields let the system show confidence scores and track when the AI vs. user made the decision, supporting EU AI Act transparency requirements.

6. **Aggregator abstraction layer** — the `aggregator_connections` table stores the aggregator type ('plaid', 'mx', 'fdx') so the same schema supports multiple data sources. This aligns with the CFPB Section 1033 migration path from Plaid to direct FDX connections.

7. **Net worth snapshots are point-in-time copies** — rather than computing net worth from current balances, daily snapshots are stored for historical charting. This avoids expensive recomputation and handles accounts that are later closed or disconnected.

8. **Recurring transactions are a separate entity** — detected subscriptions and bills are stored independently from individual transactions, enabling the subscription management and anomaly detection features without complex pattern matching on every query.
