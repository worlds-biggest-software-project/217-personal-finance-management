# Personal Finance Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source personal finance platform that combines budget tracking, goal setting, net worth monitoring, and conversational financial coaching in a single tool.

Personal Finance Management is a candidate project to build an open-source alternative to YNAB, Monarch Money, Copilot Money, and Empower. It is aimed at individuals and households who want to track spending, plan cash flow, manage net worth, and receive AI-driven financial coaching without paying $10–$15 per month and without their data being locked into a closed proprietary platform.

---

## Why Personal Finance Management?

- The market leaders all charge $4–$15 per month, and the post-Mint shutdown migration validated that millions of users are paying for tools that remain closed-source and US-only.
- No incumbent combines all four pillars well: YNAB lacks investment and net worth tracking, Empower's budgeting is weak, Copilot is Apple-only with no public API, and Monarch's AI Assistant is reactive rather than proactive.
- Existing AI assistants in this category answer questions only when asked — none proactively surface forgotten subscriptions, fee creep, or upcoming cash-flow shortfalls.
- Split transactions, reimbursable expenses, multi-goal trade-offs, and irregular freelance income are poorly handled across every tool surveyed.
- Almost all incumbents are US-only because of their aggregation stack; an open-source project can integrate FDX-compliant and international aggregators on the user's terms.

---

## Key Features

### Accounts and Transactions

- Bank account sync via Plaid, MX, Finicity, or FDX-compliant aggregation
- AI-powered transaction categorisation that learns from user corrections and applies retroactively
- Manual entry and CSV import as alternatives to bank sync
- Account balance tracking across multiple institutions with 256-bit encryption

### Budgeting and Cash Flow

- Monthly budget creation with category-level tracking (zero-based, envelope, or 50/30/20 methods)
- Forward-looking cash flow projection up to 12 months
- Subscription and recurring charge detection with anomaly alerts for fee creep and duplicates
- Spending Plan view: income minus bills and savings equals available-to-spend

### Net Worth and Investments

- Unified dashboard showing bank, investment, and liability accounts
- Net worth tracking with investment account aggregation (401(k), IRA, brokerage)
- Investment fee impact modelling and asset-allocation analysis
- Multiple savings goal tracking with contribution modelling

### Collaboration and Coaching

- Collaborative household access with shared dashboards and tagging
- Natural language Q&A over personal transaction history
- LLM-powered financial coaching grounded in the user's actual data and goals
- Goal-to-action translation: converting stated goals into concrete monthly plans

### Extensibility

- OAuth 2.0 API for third-party integrations (modelled on YNAB's published API)
- MCP (Model Context Protocol) server for AI agent access to user-authorised data
- CSV export and full data portability

---

## AI-Native Advantage

Incumbent tools use AI primarily for transaction categorisation, and only Monarch offers a conversational assistant — and it does not proactively surface insights. This project is designed AI-first: proactive cash-flow forecasting that warns of upcoming shortfalls before the user asks, anomaly detection for forgotten subscriptions and merchant fees, simultaneous modelling of competing financial goals with dynamic trade-off recommendations, and natural-language coaching grounded in the user's real transaction history rather than generic advice.

---

## Tech Stack & Deployment

Expected deployment modes include self-hosted (for users who want full control of their financial data) and an optional managed cloud tier. Integration with bank data is via FDX-compliant connections where available, falling back to Plaid, MX, or Finicity aggregators. Authentication is OAuth 2.0 with at-rest encryption. Mobile apps target iOS and Android with real-time sync. Compliance scope includes PCI DSS where card data is involved, CFPB Section 1033 open-banking requirements, and GDPR/CCPA for personal data handling.

---

## Market Context

The personal finance management software market is estimated at USD 1.3–1.6 billion in 2025, projected to reach USD 2.0–2.4 billion by 2029–2033, with the AI-powered segment growing at 9.6–10.1% CAGR (Fortune Business Insights, 2025; Grand View Research, 2025; The Business Research Company, 2025). Incumbent pricing ranges from free ad-supported (Credit Karma) to $14.99/month (YNAB, Monarch). Primary buyer personas include millennials and Gen Z managing debt and first-time budgeting, dual-income households tracking shared finances, and mass-affluent individuals monitoring investments alongside spending.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
