# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Personal Finance Management · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event stored in an append-only event store. The current state of accounts, balances, budgets, and goals is derived by replaying events, or more practically, maintained in materialised read models that are updated asynchronously. This is the Command Query Responsibility Segregation (CQRS) pattern combined with event sourcing.

The core principle is **auditability as a first-class concern**. In a personal finance application, users need to answer questions like "What was my net worth on March 15th?", "When did I change that transaction's category?", and "Show me every change to my budget this month." Event sourcing makes these queries natural rather than bolted on. Every modification, correction, and import is recorded permanently.

This approach is used by modern banking platforms, payment processors, and regulated financial systems where regulatory compliance demands full traceability. The trade-off is increased complexity in the write path and eventual consistency in read models.

**Best for:** Teams prioritising full audit trails, temporal queries, AI analytics on change patterns, and compliance with financial regulations requiring non-repudiation.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every change is recorded with timestamp and actor
- Pro: Temporal queries are trivial — reconstruct state at any point in time by replaying events up to that timestamp
- Pro: AI analytics benefit from rich event streams — pattern detection, anomaly identification, and behavioural insights
- Pro: Easy to add new read models without modifying the event store
- Pro: Natural fit for undo/redo and change history features
- Con: More complex architecture — requires event store, command handlers, projectors, and read models
- Con: Eventual consistency between write and read — UI may briefly show stale data
- Con: Event schema evolution requires careful versioning (upcasting)
- Con: Higher storage requirements — events are never deleted
- Con: Debugging requires understanding event replay, not just table inspection

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FDX API v6.5 | Inbound bank sync events carry FDX-aligned account and transaction schemas |
| Plaid Webhooks | Plaid webhook events map directly to domain events (TRANSACTIONS_SYNC, ITEM_ERROR) |
| ISO 8601 | All event timestamps use TIMESTAMPTZ; event dates in ISO 8601 |
| ISO 4217 | Currency codes embedded in monetary event payloads |
| OCSF (Open Cybersecurity Schema Framework) | Event metadata structure (actor, action, timestamp, outcome) aligns with OCSF audit event patterns |
| EU AI Act | AI decision events (categorisation, insight generation) stored with confidence scores for auditability |
| CloudEvents 1.0 | Event envelope structure follows CloudEvents specification for interoperability |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth. All state is derived from this table.
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,            -- aggregate root ID (e.g., household_id, account_id)
    stream_type VARCHAR(50) NOT NULL,   -- 'household', 'account', 'budget', 'goal'
    event_type VARCHAR(100) NOT NULL,   -- e.g., 'TransactionCreated', 'CategoryChanged'
    event_version INTEGER NOT NULL,     -- per-stream sequence number for ordering
    payload JSONB NOT NULL,             -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}', -- actor_id, ip_address, source, correlation_id
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, event_version)   -- ensures ordering within a stream
);

-- Optimised for stream replay
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
-- Optimised for type-based projections
CREATE INDEX idx_events_type ON events(event_type, created_at);
-- Optimised for time-range queries
CREATE INDEX idx_events_created ON events(created_at);
-- GIN index for JSONB payload queries
CREATE INDEX idx_events_payload ON events USING GIN(payload);

-- Example events and their payloads:
--
-- event_type: 'AccountCreated'
-- payload: {
--   "account_id": "uuid",
--   "household_id": "uuid",
--   "name": "Chase Checking",
--   "account_type": "checking",
--   "currency_code": "USD",
--   "institution_name": "Chase",
--   "aggregator": "plaid",
--   "initial_balance": 5432.10
-- }
--
-- event_type: 'TransactionImported'
-- payload: {
--   "transaction_id": "uuid",
--   "account_id": "uuid",
--   "date": "2026-05-15",
--   "amount": -42.50,
--   "merchant_name": "Whole Foods",
--   "category_id": "uuid",
--   "ai_confidence": 0.94,
--   "aggregator_transaction_id": "plaid_tx_123"
-- }
--
-- event_type: 'TransactionRecategorised'
-- payload: {
--   "transaction_id": "uuid",
--   "old_category_id": "uuid",
--   "new_category_id": "uuid",
--   "recategorised_by": "user"  -- or "ai_rule"
-- }
--
-- event_type: 'BudgetAllocated'
-- payload: {
--   "budget_id": "uuid",
--   "month": "2026-05",
--   "category_id": "uuid",
--   "amount": 500.00,
--   "previous_amount": 400.00
-- }
--
-- event_type: 'GoalContributed'
-- payload: {
--   "goal_id": "uuid",
--   "amount": 200.00,
--   "new_total": 3400.00,
--   "transaction_id": "uuid"
-- }
--
-- event_type: 'InsightGenerated'
-- payload: {
--   "insight_type": "subscription_alert",
--   "title": "Netflix price increased",
--   "description": "Your Netflix charge increased from $15.99 to $17.99",
--   "related_recurring_id": "uuid",
--   "severity": "warning",
--   "ai_model_version": "v2.3"
-- }
```

## Event Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying entire event history
CREATE TABLE event_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,      -- event_version at time of snapshot
    state JSONB NOT NULL,                   -- full aggregate state at this point
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON event_snapshots(stream_id, snapshot_version DESC);
```

## Read Models (Materialised Projections)

These tables are derived from the event store. They can be rebuilt at any time by replaying events.

### Users & Households Read Model

```sql
CREATE TABLE rm_users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    preferred_currency CHAR(3) NOT NULL DEFAULT 'USD',
    locale VARCHAR(10) NOT NULL DEFAULT 'en-US',
    timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_households (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    owner_user_id UUID NOT NULL REFERENCES rm_users(id),
    member_count INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_household_members (
    household_id UUID NOT NULL REFERENCES rm_households(id),
    user_id UUID NOT NULL REFERENCES rm_users(id),
    role VARCHAR(20) NOT NULL,
    PRIMARY KEY (household_id, user_id)
);
```

### Accounts Read Model

```sql
CREATE TABLE rm_accounts (
    id UUID PRIMARY KEY,
    household_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    official_name VARCHAR(255),
    account_type VARCHAR(30) NOT NULL,
    account_subtype VARCHAR(50),
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    current_balance NUMERIC(15,2),
    available_balance NUMERIC(15,2),
    credit_limit NUMERIC(15,2),
    institution_name VARCHAR(255),
    aggregator VARCHAR(20),
    mask VARCHAR(10),
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    is_manual BOOLEAN NOT NULL DEFAULT FALSE,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    last_synced_at TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_accounts_household ON rm_accounts(household_id);
```

### Transactions Read Model

```sql
CREATE TABLE rm_transactions (
    id UUID PRIMARY KEY,
    household_id UUID NOT NULL,
    account_id UUID NOT NULL,
    category_id UUID,
    category_name VARCHAR(100),
    category_group_name VARCHAR(100),
    payee_name VARCHAR(500),
    date DATE NOT NULL,
    amount NUMERIC(15,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    merchant_name VARCHAR(255),
    description TEXT,
    memo TEXT,
    cleared_status VARCHAR(20) NOT NULL DEFAULT 'uncleared',
    is_pending BOOLEAN NOT NULL DEFAULT FALSE,
    is_split_parent BOOLEAN NOT NULL DEFAULT FALSE,
    parent_transaction_id UUID,
    flag_color VARCHAR(20),
    is_reimbursable BOOLEAN NOT NULL DEFAULT FALSE,
    reimbursed_at TIMESTAMPTZ,
    ai_category_confidence NUMERIC(3,2),
    tags TEXT[], -- denormalized for fast read
    last_event_version INTEGER NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_tx_household_date ON rm_transactions(household_id, date DESC);
CREATE INDEX idx_rm_tx_account_date ON rm_transactions(account_id, date DESC);
CREATE INDEX idx_rm_tx_category ON rm_transactions(category_id);
CREATE INDEX idx_rm_tx_tags ON rm_transactions USING GIN(tags);
```

### Budget Read Model

```sql
CREATE TABLE rm_budget_summary (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    budget_id UUID NOT NULL,
    month DATE NOT NULL,
    category_id UUID NOT NULL,
    category_name VARCHAR(100) NOT NULL,
    category_group_name VARCHAR(100) NOT NULL,
    budgeted NUMERIC(15,2) NOT NULL DEFAULT 0,
    activity NUMERIC(15,2) NOT NULL DEFAULT 0,
    available NUMERIC(15,2) NOT NULL DEFAULT 0,
    carryover NUMERIC(15,2) NOT NULL DEFAULT 0,
    last_event_version INTEGER NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL,
    UNIQUE (budget_id, month, category_id)
);

CREATE INDEX idx_rm_budget_household ON rm_budget_summary(household_id, month);
```

### Goals Read Model

```sql
CREATE TABLE rm_goals (
    id UUID PRIMARY KEY,
    household_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    goal_type VARCHAR(30) NOT NULL,
    target_amount NUMERIC(15,2) NOT NULL,
    current_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    target_date DATE,
    monthly_contribution_target NUMERIC(15,2),
    percent_complete NUMERIC(5,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    last_event_version INTEGER NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_goals_household ON rm_goals(household_id);
```

### Net Worth Read Model

```sql
CREATE TABLE rm_net_worth_daily (
    household_id UUID NOT NULL,
    date DATE NOT NULL,
    total_assets NUMERIC(15,2) NOT NULL,
    total_liabilities NUMERIC(15,2) NOT NULL,
    net_worth NUMERIC(15,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    account_breakdown JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"account_id":"uuid","name":"Chase Checking","balance":5432.10,"type":"asset"}]
    last_event_version INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (household_id, date)
);
```

### Cash Flow Forecast Read Model

```sql
CREATE TABLE rm_cash_flow_forecast (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id UUID NOT NULL,
    forecast_date DATE NOT NULL,
    projected_balance NUMERIC(15,2) NOT NULL,
    projected_inflows NUMERIC(15,2) NOT NULL DEFAULT 0,
    projected_outflows NUMERIC(15,2) NOT NULL DEFAULT 0,
    confidence NUMERIC(3,2), -- 0.00 to 1.00
    contributing_recurring JSONB DEFAULT '[]',
    -- Example: [{"recurring_id":"uuid","payee":"Rent","amount":-2100.00}]
    generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (household_id, forecast_date)
);

CREATE INDEX idx_rm_forecast_household ON rm_cash_flow_forecast(household_id, forecast_date);
```

### AI Insights Read Model

```sql
CREATE TABLE rm_insights (
    id UUID PRIMARY KEY,
    household_id UUID NOT NULL,
    insight_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    severity VARCHAR(20) NOT NULL DEFAULT 'info',
    related_entity_type VARCHAR(30), -- 'transaction', 'account', 'recurring'
    related_entity_id UUID,
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL,
    expires_at TIMESTAMPTZ
);

CREATE INDEX idx_rm_insights_household ON rm_insights(household_id, created_at DESC);
CREATE INDEX idx_rm_insights_unread ON rm_insights(household_id) WHERE is_read = FALSE AND is_dismissed = FALSE;
```

## Projection Tracking

```sql
-- Tracks which events each projector has processed
CREATE TABLE projection_checkpoints (
    projector_name VARCHAR(100) PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_created_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Command Log (Write Side Audit)

```sql
-- Optional: log all commands for debugging and analytics
CREATE TABLE command_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type VARCHAR(100) NOT NULL, -- 'CreateTransaction', 'AllocateBudget', etc.
    actor_id UUID NOT NULL,
    household_id UUID NOT NULL,
    payload JSONB NOT NULL,
    result VARCHAR(20) NOT NULL, -- 'success', 'rejected', 'error'
    error_message TEXT,
    events_produced UUID[] DEFAULT '{}', -- IDs of events generated by this command
    duration_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_command_log_actor ON command_log(actor_id, created_at DESC);
CREATE INDEX idx_command_log_household ON command_log(household_id, created_at DESC);
```

---

## Example: Temporal Query (State at a Point in Time)

```sql
-- What was the balance of account X on March 15, 2026?
-- Replay all balance-affecting events up to that date
SELECT
    e.payload->>'account_id' AS account_id,
    SUM(
        CASE
            WHEN e.event_type = 'AccountCreated' THEN (e.payload->>'initial_balance')::NUMERIC
            WHEN e.event_type = 'TransactionImported' THEN (e.payload->>'amount')::NUMERIC
            WHEN e.event_type = 'TransactionDeleted' THEN -(e.payload->>'amount')::NUMERIC
            WHEN e.event_type = 'BalanceAdjusted' THEN (e.payload->>'adjustment')::NUMERIC
            ELSE 0
        END
    ) AS balance_at_date
FROM events e
WHERE e.stream_id = 'account-uuid-here'
  AND e.stream_type = 'account'
  AND e.created_at <= '2026-03-15T23:59:59Z'
  AND e.event_type IN ('AccountCreated', 'TransactionImported', 'TransactionDeleted', 'BalanceAdjusted')
GROUP BY e.payload->>'account_id';
```

## Example: Change History for a Transaction

```sql
-- Show all changes made to a specific transaction
SELECT
    e.event_type,
    e.payload,
    e.metadata->>'actor_id' AS changed_by,
    e.created_at AS changed_at
FROM events e
WHERE e.payload->>'transaction_id' = 'transaction-uuid-here'
ORDER BY e.created_at;

-- Returns rows like:
-- TransactionImported  | {amount: -42.50, category: "Groceries", ...} | system    | 2026-05-15 10:00
-- TransactionRecategorised | {old: "Groceries", new: "Dining Out"}    | user-uuid | 2026-05-15 14:30
-- TransactionMemoUpdated   | {memo: "Team lunch - reimbursable"}      | user-uuid | 2026-05-15 14:31
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, event_snapshots |
| Read Models — Identity | 3 | rm_users, rm_households, rm_household_members |
| Read Models — Accounts | 1 | rm_accounts |
| Read Models — Transactions | 1 | rm_transactions |
| Read Models — Budget | 1 | rm_budget_summary |
| Read Models — Goals | 1 | rm_goals |
| Read Models — Net Worth | 1 | rm_net_worth_daily |
| Read Models — Cash Flow | 1 | rm_cash_flow_forecast |
| Read Models — AI | 1 | rm_insights |
| Infrastructure | 2 | projection_checkpoints, command_log |
| **Total** | **14** | Plus any additional read models added later |

---

## Key Design Decisions

1. **Single events table rather than per-aggregate tables** — simpler schema, easier to query across aggregate boundaries, and stream_id + stream_type provide sufficient partitioning. For very high scale, this table could be partitioned by `created_at`.

2. **JSONB payloads rather than typed event tables** — each event type has a different shape. Using JSONB avoids hundreds of event-specific tables while still allowing GIN-indexed queries on payload fields.

3. **Read models are denormalized and disposable** — every `rm_*` table can be dropped and rebuilt from the event store. This enables fearless schema evolution on the read side. Add a column, rebuild the projection, done.

4. **Snapshot table for performance** — without snapshots, replaying thousands of events per account on every read is expensive. Periodic snapshots (e.g., every 100 events) provide a starting point, reducing replay to a small number of recent events.

5. **`last_event_version` on read models** — each read model row tracks which event it was last updated from. This enables projectors to detect and recover from gaps, and enables optimistic concurrency on the read side.

6. **Tags stored as TEXT[] in the read model** — denormalized from events for fast GIN-indexed tag queries. The events themselves record `TagAdded` and `TagRemoved` events.

7. **AI events are first-class citizens** — `InsightGenerated`, `TransactionRecategorised` (by AI), and `ForecastUpdated` events carry model version and confidence metadata, satisfying EU AI Act transparency requirements.

8. **Command log is separate from events** — commands may be rejected (validation failure, authorization denial) and should still be logged for security audit. The command log records intent; the event store records outcome.
