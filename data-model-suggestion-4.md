# Data Model Suggestion 4: Time-Series / Analytics-First with Relational Core

> Project: Personal Finance Management · Created: 2026-05-20

## Philosophy

This model optimises for the analytical and forecasting workloads that differentiate an AI-native personal finance tool from a simple transaction ledger. The core insight is that a PFM product's most valuable features -- cash flow forecasting, spending trend analysis, net worth tracking, anomaly detection, and AI coaching -- are all time-series problems. This architecture pairs a conventional relational layer for CRUD operations with time-partitioned tables for analytical data.

The design principle is **analytics as a first-class workload, not an afterthought**. Transactions are stored in a time-partitioned table for fast range scans. Daily balance snapshots, spending aggregates, and forecast projections are pre-computed and stored in dedicated time-series tables. The AI layer reads from these materialised analytics tables rather than computing aggregations on the fly from raw transactions.

This pattern is used by fintech analytics platforms, portfolio tracking tools, and any system where historical trend queries dominate the read workload. PostgreSQL's declarative partitioning (available since v10, mature since v14) provides the partitioning infrastructure without requiring TimescaleDB, though TimescaleDB is a natural extension.

**Best for:** Teams building a product where AI-driven insights, trend analysis, cash flow forecasting, and historical net worth charting are the primary differentiators -- not just budgeting CRUD.

**Trade-offs:**
- Pro: Range queries on date-bounded transaction data are extremely fast due to partition pruning
- Pro: Pre-computed aggregates eliminate expensive real-time GROUP BY queries for dashboards
- Pro: AI models read from clean, indexed time-series tables rather than joining across normalised entities
- Pro: Natural fit for PostgreSQL partitioning -- no additional infrastructure required
- Pro: Historical data (older than 2 years) can be moved to cheaper storage or compressed partitions
- Con: More complex insert path -- transactions must land in the correct partition
- Con: Pre-computed aggregates require background jobs to keep materialised tables fresh
- Con: Cross-partition queries (e.g., "all transactions for payee X across 3 years") can be slower than a single-table scan
- Con: Partition management adds operational overhead (creating new monthly partitions, dropping old ones)
- Con: More total tables due to materialised aggregate tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FDX API v6.5 | Transaction import aligns with FDX transaction date and amount fields; account types follow FDX classification |
| Plaid PFC v2 Taxonomy | Category codes used in spending aggregation breakdowns |
| ISO 4217 | Currency codes on all monetary columns and aggregate tables |
| ISO 8601 | All dates and timestamps; partition boundaries align to calendar months |
| ISO 3166-1 | Country codes for multi-currency conversion baselines |
| CFPB Section 1033 | Data retention periods inform partition lifecycle and archival strategy |

---

## Identity & Households (Standard Relational)

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    preferred_currency CHAR(3) NOT NULL DEFAULT 'USD',
    locale VARCHAR(10) NOT NULL DEFAULT 'en-US',
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE households (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    owner_user_id UUID NOT NULL REFERENCES users(id),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    country_code CHAR(2) NOT NULL DEFAULT 'US',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE household_members (
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'member',
    PRIMARY KEY (household_id, user_id)
);
```

## Accounts (Standard Relational)

```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    official_name VARCHAR(255),
    account_type VARCHAR(30) NOT NULL,
    account_subtype VARCHAR(50),
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    current_balance NUMERIC(15,2),
    available_balance NUMERIC(15,2),
    credit_limit NUMERIC(15,2),
    mask VARCHAR(10),
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    is_manual BOOLEAN NOT NULL DEFAULT FALSE,
    aggregator VARCHAR(20),
    aggregator_account_id VARCHAR(255),
    interest_rate NUMERIC(7,4),
    last_synced_at TIMESTAMPTZ,
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_accounts_household ON accounts(household_id);
```

## Categories (Standard Relational)

```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    is_income BOOLEAN NOT NULL DEFAULT FALSE,
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    plaid_pfc_primary VARCHAR(100),
    plaid_pfc_detailed VARCHAR(100),
    icon VARCHAR(50),
    color VARCHAR(7),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_categories_household ON categories(household_id);
```

## Transactions (Time-Partitioned)

```sql
-- Parent table with declarative range partitioning by date
CREATE TABLE transactions (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    account_id UUID NOT NULL,
    category_id UUID,
    date DATE NOT NULL,
    amount NUMERIC(15,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    payee_name VARCHAR(500),
    merchant_name VARCHAR(255),
    description TEXT,
    memo TEXT,
    cleared_status VARCHAR(20) NOT NULL DEFAULT 'uncleared',
    is_pending BOOLEAN NOT NULL DEFAULT FALSE,
    is_split_parent BOOLEAN NOT NULL DEFAULT FALSE,
    parent_transaction_id UUID,
    transfer_pair_id UUID,
    flag_color VARCHAR(20),
    tags TEXT[] NOT NULL DEFAULT '{}',
    ai_category_confidence NUMERIC(3,2),
    aggregator_transaction_id VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, date) -- date must be in PK for partitioning
) PARTITION BY RANGE (date);

-- Create monthly partitions (automate via cron or pg_partman)
CREATE TABLE transactions_2026_01 PARTITION OF transactions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE transactions_2026_02 PARTITION OF transactions
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE transactions_2026_03 PARTITION OF transactions
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE transactions_2026_04 PARTITION OF transactions
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE transactions_2026_05 PARTITION OF transactions
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE transactions_2026_06 PARTITION OF transactions
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... continue for future months; pg_partman can automate this

-- Default partition for out-of-range dates (imports, corrections)
CREATE TABLE transactions_default PARTITION OF transactions DEFAULT;

-- Indexes are created on each partition automatically
CREATE INDEX idx_tx_household_date ON transactions(household_id, date DESC);
CREATE INDEX idx_tx_account_date ON transactions(account_id, date DESC);
CREATE INDEX idx_tx_category ON transactions(category_id);
CREATE INDEX idx_tx_tags ON transactions USING GIN(tags);
CREATE INDEX idx_tx_payee ON transactions(household_id, payee_name);
```

## Daily Balance Snapshots (Time-Series)

```sql
-- One row per account per day. Updated by a nightly job or on each sync.
CREATE TABLE daily_balances (
    account_id UUID NOT NULL,
    date DATE NOT NULL,
    opening_balance NUMERIC(15,2) NOT NULL,
    closing_balance NUMERIC(15,2) NOT NULL,
    total_inflows NUMERIC(15,2) NOT NULL DEFAULT 0,
    total_outflows NUMERIC(15,2) NOT NULL DEFAULT 0,
    transaction_count INTEGER NOT NULL DEFAULT 0,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    PRIMARY KEY (account_id, date)
) PARTITION BY RANGE (date);

CREATE TABLE daily_balances_2026_q1 PARTITION OF daily_balances
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE daily_balances_2026_q2 PARTITION OF daily_balances
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE daily_balances_2026_q3 PARTITION OF daily_balances
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE daily_balances_2026_q4 PARTITION OF daily_balances
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');
CREATE TABLE daily_balances_default PARTITION OF daily_balances DEFAULT;

CREATE INDEX idx_daily_bal_date ON daily_balances(date);
```

## Daily Net Worth (Time-Series)

```sql
-- One row per household per day.
CREATE TABLE daily_net_worth (
    household_id UUID NOT NULL,
    date DATE NOT NULL,
    total_assets NUMERIC(15,2) NOT NULL,
    total_liabilities NUMERIC(15,2) NOT NULL,
    net_worth NUMERIC(15,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    -- Breakdown columns for fast dashboard rendering without joins
    checking_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    savings_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    investment_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    credit_card_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    loan_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    mortgage_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    other_assets NUMERIC(15,2) NOT NULL DEFAULT 0,
    other_liabilities NUMERIC(15,2) NOT NULL DEFAULT 0,
    PRIMARY KEY (household_id, date)
) PARTITION BY RANGE (date);

CREATE TABLE daily_net_worth_2026_h1 PARTITION OF daily_net_worth
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE daily_net_worth_2026_h2 PARTITION OF daily_net_worth
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');
CREATE TABLE daily_net_worth_default PARTITION OF daily_net_worth DEFAULT;
```

## Monthly Spending Aggregates (Pre-Computed Analytics)

```sql
-- Pre-computed monthly spending by category. Updated by background job after each sync.
CREATE TABLE monthly_spending (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    month DATE NOT NULL, -- first of month
    category_id UUID,
    category_name VARCHAR(100),
    category_group_name VARCHAR(100),
    total_amount NUMERIC(15,2) NOT NULL DEFAULT 0, -- sum of transactions
    transaction_count INTEGER NOT NULL DEFAULT 0,
    avg_transaction NUMERIC(15,2),
    min_transaction NUMERIC(15,2),
    max_transaction NUMERIC(15,2),
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    is_income BOOLEAN NOT NULL DEFAULT FALSE,
    computed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (household_id, month, category_id)
);

CREATE INDEX idx_monthly_spending_household ON monthly_spending(household_id, month);
CREATE INDEX idx_monthly_spending_category ON monthly_spending(category_id, month);

-- Year-over-year comparison materialised view
CREATE MATERIALIZED VIEW mv_spending_yoy AS
SELECT
    ms.household_id,
    ms.category_id,
    ms.category_name,
    EXTRACT(MONTH FROM ms.month)::INTEGER AS month_number,
    EXTRACT(YEAR FROM ms.month)::INTEGER AS year,
    ms.total_amount,
    ms.transaction_count,
    LAG(ms.total_amount) OVER (
        PARTITION BY ms.household_id, ms.category_id, EXTRACT(MONTH FROM ms.month)
        ORDER BY ms.month
    ) AS same_month_prior_year
FROM monthly_spending ms
WHERE ms.is_income = FALSE;

CREATE UNIQUE INDEX idx_mv_spending_yoy ON mv_spending_yoy(household_id, category_id, year, month_number);
```

## Recurring Pattern Analysis (Analytics)

```sql
CREATE TABLE recurring_patterns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    account_id UUID NOT NULL,
    category_id UUID,
    payee_name VARCHAR(500),
    expected_amount NUMERIC(15,2) NOT NULL,
    amount_variance NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    frequency VARCHAR(20) NOT NULL,
    next_expected_date DATE NOT NULL,
    is_income BOOLEAN NOT NULL DEFAULT FALSE,
    is_subscription BOOLEAN NOT NULL DEFAULT FALSE,
    auto_detected BOOLEAN NOT NULL DEFAULT FALSE,
    detection_confidence NUMERIC(3,2),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Tracks every occurrence of a recurring pattern for trend analysis
CREATE TABLE recurring_occurrences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pattern_id UUID NOT NULL REFERENCES recurring_patterns(id) ON DELETE CASCADE,
    transaction_id UUID NOT NULL,
    transaction_date DATE NOT NULL,
    actual_amount NUMERIC(15,2) NOT NULL,
    expected_amount NUMERIC(15,2) NOT NULL,
    amount_delta NUMERIC(15,2) NOT NULL, -- actual - expected
    days_from_expected INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_recurring_household ON recurring_patterns(household_id);
CREATE INDEX idx_recurring_occ_pattern ON recurring_occurrences(pattern_id, transaction_date DESC);
```

## Cash Flow Forecasts (Time-Series)

```sql
-- Daily granularity forecast, regenerated by AI on each sync or weekly
CREATE TABLE cash_flow_forecasts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    forecast_date DATE NOT NULL,
    forecast_generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    projected_balance NUMERIC(15,2) NOT NULL,
    projected_inflows NUMERIC(15,2) NOT NULL DEFAULT 0,
    projected_outflows NUMERIC(15,2) NOT NULL DEFAULT 0,
    confidence_lower NUMERIC(15,2), -- 10th percentile
    confidence_upper NUMERIC(15,2), -- 90th percentile
    confidence_score NUMERIC(3,2), -- overall confidence 0-1
    has_shortfall_risk BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (household_id, forecast_date)
);

CREATE INDEX idx_forecast_household ON cash_flow_forecasts(household_id, forecast_date);
CREATE INDEX idx_forecast_shortfall ON cash_flow_forecasts(household_id)
    WHERE has_shortfall_risk = TRUE;

-- Individual items contributing to the forecast
CREATE TABLE forecast_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    forecast_date DATE NOT NULL,
    source_type VARCHAR(30) NOT NULL, -- 'recurring_income', 'recurring_expense', 'budget_allocation', 'ai_predicted'
    source_id UUID, -- recurring_pattern_id, budget_allocation_id, or NULL for AI predictions
    description VARCHAR(255),
    amount NUMERIC(15,2) NOT NULL,
    confidence NUMERIC(3,2),
    UNIQUE (household_id, forecast_date, source_type, source_id)
);

CREATE INDEX idx_forecast_items ON forecast_items(household_id, forecast_date);
```

## Budgets (Standard Relational)

```sql
CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL REFERENCES households(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL DEFAULT 'My Budget',
    method VARCHAR(20) NOT NULL DEFAULT 'zero_based',
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE budget_periods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id UUID NOT NULL REFERENCES budgets(id) ON DELETE CASCADE,
    month DATE NOT NULL,
    income_total NUMERIC(15,2) NOT NULL DEFAULT 0,
    ready_to_assign NUMERIC(15,2) NOT NULL DEFAULT 0,
    age_of_money_days INTEGER,
    UNIQUE (budget_id, month)
);

CREATE TABLE budget_allocations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_period_id UUID NOT NULL REFERENCES budget_periods(id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    budgeted NUMERIC(15,2) NOT NULL DEFAULT 0,
    activity NUMERIC(15,2) NOT NULL DEFAULT 0, -- linked to monthly_spending for consistency
    available NUMERIC(15,2) NOT NULL DEFAULT 0,
    carryover NUMERIC(15,2) NOT NULL DEFAULT 0,
    UNIQUE (budget_period_id, category_id)
);

CREATE INDEX idx_budget_periods ON budget_periods(budget_id, month);
CREATE INDEX idx_budget_alloc ON budget_allocations(budget_period_id);
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Goal progress over time for trend charting
CREATE TABLE goal_progress_daily (
    goal_id UUID NOT NULL REFERENCES goals(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    amount NUMERIC(15,2) NOT NULL,
    percent_complete NUMERIC(5,2) NOT NULL,
    on_track BOOLEAN NOT NULL DEFAULT TRUE,
    projected_completion_date DATE,
    PRIMARY KEY (goal_id, date)
);

CREATE INDEX idx_goals_household ON goals(household_id);
```

## Investment Snapshots (Time-Series)

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
    expense_ratio NUMERIC(7,5),
    last_price_update TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Daily portfolio value snapshots for performance charting
CREATE TABLE portfolio_daily (
    account_id UUID NOT NULL,
    date DATE NOT NULL,
    total_value NUMERIC(15,2) NOT NULL,
    total_cost_basis NUMERIC(15,2),
    total_gain_loss NUMERIC(15,2),
    total_gain_loss_pct NUMERIC(7,4),
    holdings_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (account_id, date)
);

CREATE INDEX idx_holdings_account ON investment_holdings(account_id);
```

## AI Insights & Anomalies (Analytics)

```sql
CREATE TABLE ai_insights (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    insight_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    severity VARCHAR(20) NOT NULL DEFAULT 'info',
    category VARCHAR(30), -- 'spending', 'subscription', 'cash_flow', 'goal', 'anomaly', 'tax'
    related_entity_type VARCHAR(30),
    related_entity_id UUID,
    annual_impact NUMERIC(15,2), -- estimated annual dollar impact
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed BOOLEAN NOT NULL DEFAULT FALSE,
    is_actionable BOOLEAN NOT NULL DEFAULT TRUE,
    model_version VARCHAR(50),
    confidence NUMERIC(3,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ
);

CREATE INDEX idx_insights_household ON ai_insights(household_id, created_at DESC);
CREATE INDEX idx_insights_unread ON ai_insights(household_id) WHERE is_read = FALSE AND is_dismissed = FALSE;
CREATE INDEX idx_insights_type ON ai_insights(household_id, insight_type);
CREATE INDEX idx_insights_impact ON ai_insights(household_id, annual_impact DESC NULLS LAST)
    WHERE is_dismissed = FALSE;

-- Spending anomaly detection log
CREATE TABLE spending_anomalies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    transaction_id UUID NOT NULL,
    category_id UUID,
    anomaly_type VARCHAR(30) NOT NULL, -- 'unusual_amount', 'unusual_merchant', 'unusual_time', 'duplicate', 'fee_increase'
    expected_value NUMERIC(15,2),
    actual_value NUMERIC(15,2),
    deviation_score NUMERIC(5,2), -- standard deviations from mean
    baseline_data JSONB NOT NULL DEFAULT '{}',
    -- baseline_data example:
    -- {
    --   "category_mean_30d": 45.00,
    --   "category_stddev_30d": 12.50,
    --   "merchant_mean": 42.00,
    --   "merchant_frequency": "monthly",
    --   "last_occurrence": "2026-04-15"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_anomalies_household ON spending_anomalies(household_id, created_at DESC);
CREATE INDEX idx_anomalies_transaction ON spending_anomalies(transaction_id);
```

## AI Conversations

```sql
CREATE TABLE ai_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    household_id UUID NOT NULL,
    title VARCHAR(255),
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_message_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ai_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES ai_conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    tokens_used INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_conv_user ON ai_conversations(user_id);
CREATE INDEX idx_ai_msg_conv ON ai_messages(conversation_id, created_at);
```

## Categorisation Rules

```sql
CREATE TABLE categorisation_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    rule_type VARCHAR(20) NOT NULL, -- 'user_correction', 'ai_learned', 'manual'
    match_field VARCHAR(20) NOT NULL,
    match_pattern VARCHAR(500) NOT NULL,
    payee_name_override VARCHAR(500),
    priority INTEGER NOT NULL DEFAULT 0,
    match_count INTEGER NOT NULL DEFAULT 0,
    last_matched_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cat_rules ON categorisation_rules(household_id, priority DESC);
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
```

---

## Example: Fast Dashboard Queries Using Pre-Computed Tables

```sql
-- Net worth chart for the last 12 months (reads from pre-computed table, no joins)
SELECT date, net_worth, total_assets, total_liabilities
FROM daily_net_worth
WHERE household_id = 'household-uuid'
  AND date >= CURRENT_DATE - INTERVAL '12 months'
ORDER BY date;

-- Monthly spending trend by category (reads from aggregate table)
SELECT month, category_name, total_amount, transaction_count,
       total_amount - LAG(total_amount) OVER (
           PARTITION BY category_id ORDER BY month
       ) AS month_over_month_change
FROM monthly_spending
WHERE household_id = 'household-uuid'
  AND month >= '2025-06-01'
  AND is_income = FALSE
ORDER BY month, total_amount DESC;

-- Cash flow forecast with confidence bands
SELECT forecast_date, projected_balance,
       confidence_lower, confidence_upper,
       has_shortfall_risk
FROM cash_flow_forecasts
WHERE household_id = 'household-uuid'
  AND forecast_date BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '90 days'
ORDER BY forecast_date;

-- Goal progress over time
SELECT g.name, gpd.date, gpd.amount, gpd.percent_complete,
       gpd.projected_completion_date
FROM goals g
JOIN goal_progress_daily gpd ON gpd.goal_id = g.id
WHERE g.household_id = 'household-uuid'
  AND gpd.date >= CURRENT_DATE - INTERVAL '6 months'
ORDER BY g.name, gpd.date;

-- Partition pruning example: this query only scans the May 2026 partition
EXPLAIN SELECT * FROM transactions
WHERE household_id = 'household-uuid'
  AND date BETWEEN '2026-05-01' AND '2026-05-31';
-- -> Index Scan on transactions_2026_05 ...
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Auth | 3 | users, households, household_members |
| Accounts | 1 | accounts |
| Categories & Rules | 2 | categories, categorisation_rules |
| Transactions | 1 | transactions (partitioned -- ~12 physical partition tables per year) |
| Time-Series: Balances | 1 | daily_balances (partitioned quarterly) |
| Time-Series: Net Worth | 1 | daily_net_worth (partitioned semi-annually) |
| Time-Series: Spending | 1 | monthly_spending + 1 materialised view (mv_spending_yoy) |
| Recurring Analysis | 2 | recurring_patterns, recurring_occurrences |
| Cash Flow Forecast | 2 | cash_flow_forecasts, forecast_items |
| Budgets | 3 | budgets, budget_periods, budget_allocations |
| Goals | 2 | goals, goal_progress_daily |
| Investments | 2 | investment_holdings, portfolio_daily |
| AI & Insights | 4 | ai_insights, spending_anomalies, ai_conversations, ai_messages |
| API | 1 | api_tokens |
| **Total** | **26** | Plus ~12 partition tables/year for transactions, ~4/year for daily_balances |

---

## Key Design Decisions

1. **Monthly partitioning for transactions** -- a household with 500 transactions/month accumulates 6,000/year. Monthly partitions keep each partition small enough for fast scans while being large enough to avoid excessive partition overhead. PostgreSQL's declarative partitioning handles this natively.

2. **Quarterly partitioning for daily_balances** -- one row per account per day means ~365 rows per account per year. With 10 accounts, that is 3,650 rows/year, which is small. Quarterly partitions group data logically and support archival.

3. **Pre-computed monthly_spending aggregate table** -- the most common dashboard query ("how much did I spend on groceries this month vs. last month?") reads from this table directly rather than aggregating across thousands of transaction rows. A background job updates it after each bank sync.

4. **Materialised view for year-over-year comparison** -- `mv_spending_yoy` pre-computes the LAG window function so the dashboard can show "you spent 15% more on dining out this May vs. last May" without real-time window computation.

5. **Cash flow forecast as first-class time-series** -- rather than computing forecasts on the fly, the AI generates daily projected balances with confidence bands and writes them to `cash_flow_forecasts`. The dashboard reads pre-computed rows. Forecasts are regenerated on each sync or weekly.

6. **Goal progress tracked daily** -- `goal_progress_daily` stores the running total and projected completion date each day, enabling trend charts and "on track" indicators without recomputation.

7. **Spending anomalies as a separate table** -- anomaly detection results are stored with statistical baselines (mean, standard deviation) so the AI coaching layer can explain why something was flagged. This separates the detection output from the insight presented to the user.

8. **Budget `activity` linked to monthly_spending** -- the `activity` column in `budget_allocations` is computed from `monthly_spending` for the same category and month, ensuring consistency between the budget view and the spending analytics view.
