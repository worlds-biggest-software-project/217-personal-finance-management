# Standards & API Reference

> Project: Personal Finance Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

No domain-specific ISO standard governs consumer personal finance management software directly. The following ISO standards are relevant to data security, privacy, and quality requirements:

- **ISO/IEC 27001:2022** — Information security management systems (ISMS). The primary international standard for managing information security. Relevant to any PFM product storing sensitive financial credentials and transaction data. https://www.iso.org/standard/27001
- **ISO/IEC 27018:2019** — Code of practice for protection of personally identifiable information (PII) in public clouds. Directly applicable to cloud-hosted PFM services processing financial transaction data. https://www.iso.org/standard/76559.html
- **ISO 20022** — Universal financial industry message scheme. Defines XML-based message formats for financial transactions and is increasingly adopted by payment rails (SWIFT, SEPA, FedNow). While a bank-to-bank standard rather than a consumer PFM standard, awareness is relevant when ingesting transaction metadata. https://www.iso20022.org/
- **ISO/IEC 29101:2018** — Privacy architecture framework. Provides guidance on building privacy-by-design systems; relevant to the data architecture of any PFM handling financial behaviour data. https://www.iso.org/standard/45124.html

---

### W3C & IETF Standards

- **OAuth 2.0 (RFC 6749)** — The authorisation framework used by all major PFM aggregation APIs (Plaid, MX, FDX, YNAB API). Mandatory for any integration with bank or financial data sources. https://datatracker.ietf.org/doc/html/rfc6749
- **OAuth 2.0 Bearer Tokens (RFC 6750)** — Defines how bearer tokens are transmitted in HTTP requests; used in all PFM API authentication flows. https://datatracker.ietf.org/doc/html/rfc6750
- **PKCE (RFC 7636)** — Proof Key for Code Exchange; the recommended OAuth 2.0 extension for mobile and single-page app flows where client secrets cannot be safely stored. Required for secure PFM mobile app OAuth flows. https://datatracker.ietf.org/doc/html/rfc7636
- **OpenID Connect 1.0** — Identity layer on top of OAuth 2.0; used for user authentication in conjunction with financial data authorisation. Built on top of RFC 6749. https://openid.net/connect/
- **FAPI 1.0 / FAPI 2.0 (Financial-grade API)** — W3C/OpenID Foundation profile of OAuth 2.0 designed for high-security financial data access. FDX API certification requires FAPI 1.0 compliance as of 2026, with ecosystem migration toward FAPI 2.0 underway. https://openid.net/wg/fapi/
- **JSON:API (Specification 1.1)** — Widely adopted convention for structuring REST API responses; used by several financial data APIs. https://jsonapi.org/
- **RFC 7231 — HTTP/1.1 Semantics** — Baseline HTTP protocol. Relevant for understanding REST API design requirements for financial data access. https://datatracker.ietf.org/doc/html/rfc7231
- **RFC 8288 — Web Linking** — Standard for expressing typed relationships between resources via Link headers; relevant for pagination in financial transaction feeds. https://datatracker.ietf.org/doc/html/rfc8288

---

### Data Model & API Specifications

- **Financial Data Exchange (FDX) API v6.5** — The primary US open banking API standard, currently at version 6.5 (released late 2025). Defines RESTful APIs for sharing deposit, loan, investment, insurance, tax, and reward account data. Replaces screen scraping as the data access method under the CFPB Section 1033 mandate. Large banks must provide FDX-compliant API access by 2026. Fee-free access to specifications available via IP agreement. https://financialdataexchange.org/
- **Open Financial Exchange (OFX) 2.3** — Legacy XML-based file format standard used by 7,000+ banks and brokerages for financial data export. Managed by FDX since 2019. Widely supported for QFX/OFX file import in desktop PFM tools (Quicken). https://financialdataexchange.org/about-fdx/ofx-work-group/
- **OpenAPI 3.1** — The standard specification format for documenting RESTful APIs. All major PFM aggregation APIs (Plaid, MX, Finicity/Mastercard Open Banking) provide OpenAPI specifications. Required if the project exposes its own API. https://www.openapis.org/
- **JSON Schema (Draft 2020-12)** — Standard for validating JSON data structures. Used by FDX API and OpenAPI 3.1 for data model validation. https://json-schema.org/

---

### Security & Authentication Standards

- **PCI DSS v4.0** — Payment Card Industry Data Security Standard. Applies to any system that stores, processes, or transmits cardholder data. Consumer PFM tools that display credit card transaction detail must understand their PCI DSS scope. https://www.pcisecuritystandards.org/
- **GDPR (EU) 2016/679** — General Data Protection Regulation. Applies to any PFM product with EU-based users. Imposes consent, data minimisation, right to erasure, and breach notification requirements. Financial transaction data constitutes personal data under GDPR. https://gdpr.eu/
- **CCPA / CPRA (California)** — California Consumer Privacy Act and California Privacy Rights Act. Grants California residents the right to know, delete, and opt out of sale of personal data. Enforceable for PFM apps serving California users. https://oag.ca.gov/privacy/ccpa
- **CFPB Section 1033 (Open Banking Rule, 2024)** — US Consumer Financial Protection Bureau rule requiring banks to provide machine-readable consumer financial data access. Transforms the data aggregation landscape from screen scraping to API-first access. Large banks must comply by 2026. https://www.consumerfinance.gov/rules-policy/final-rules/personal-financial-data-rights/
- **GLBA (Gramm-Leach-Bliley Act)** — US federal law requiring financial institutions to explain data-sharing practices and protect consumer data. Applies to fintech products handling non-public personal financial information. https://www.ftc.gov/business-guidance/privacy-security/gramm-leach-bliley-act
- **NIST Cybersecurity Framework 2.0** — US federal guidance for managing cybersecurity risk; widely adopted as a baseline by financial services companies. https://www.nist.gov/cyberframework
- **SOC 2 Type II** — Service Organisation Control audit standard assessing security, availability, and confidentiality controls. Expected by enterprise customers and financial institution partners. Not a specification but an industry de-facto requirement for SaaS financial products. https://www.aicpa.org/
- **EU AI Act (2024, Tier classification)** — The EU AI Act entered enforcement in 2026. AI-driven financial advice and credit scoring features must demonstrate auditability if classified as high-risk. AI-powered personal finance insights are likely moderate-risk and subject to transparency and accuracy requirements. https://artificialintelligenceact.eu/

---

### MCP Server Specifications

- **Model Context Protocol (MCP) 1.0** — Open protocol developed by Anthropic defining how AI models communicate with external tools and data sources. Monarch Money has a published MCP server (colvint/monarch-money-mcp) enabling AI agents to query personal financial data. A well-designed open-source PFM tool should expose an MCP server to enable integration with Claude, ChatGPT, and other LLM environments. https://modelcontextprotocol.io/
- **MCP Server Registry** — Community catalogue of published MCP servers at https://mcpservers.org/; the Monarch Money MCP server is listed here as a reference implementation for personal finance data access. https://mcpservers.org/servers/colvint/monarch-money-mcp

---

## Similar Products — Developer Documentation & APIs

### Plaid

- **Description:** The dominant US financial data aggregation middleware. Connects consumer applications to 12,000+ bank and financial institutions for account verification, transaction data, balance checks, income verification, and identity verification.
- **API Documentation:** https://plaid.com/docs/api/
- **SDKs/Libraries:** JavaScript, Python (official), Ruby, Go, Java, .NET — https://plaid.com/docs/libraries/
- **Developer Guide:** https://plaid.com/docs/quickstart/
- **Standards:** REST/JSON; POST-based; OAuth 2.0 and Personal Access Token authentication; OpenAPI specification available
- **Authentication:** OAuth 2.0 (Authorization Code Grant); Personal Access Token for development; PLAID-CLIENT-ID + PLAID-SECRET headers

---

### MX Technologies

- **Description:** Open finance data platform connecting to tens of thousands of financial institutions. Provides transaction data, data enhancement (cleansing and categorisation), account aggregation, and identity verification. FDX and OAuth-aligned.
- **API Documentation:** https://docs.mx.com/
- **SDKs/Libraries:** Ruby, Python, Java, PHP, JavaScript — documented at https://docs.mx.com/api-reference/
- **Developer Guide:** https://docs.mx.com/api-reference/platform-api/overview/
- **Standards:** REST/JSON; HTTPS (TLS 1.2+); FDX-aligned; OAuth 2.0
- **Authentication:** Basic authentication with Base64-encoded `client_id:api_key`; development and production environments available

---

### Mastercard Open Banking (formerly Finicity)

- **Description:** Mastercard's open banking and financial data access platform for the US. Provides permissioned access to consumer financial data from 16,000+ financial institutions, supporting open finance use cases including account aggregation, income verification, and cash flow analysis.
- **API Documentation:** https://developer.mastercard.com/open-banking-us/documentation/
- **API Reference:** https://developer.mastercard.com/open-banking-us/documentation/api-reference/
- **SDKs/Libraries:** Available via Mastercard Developers portal; OpenAPI specification downloadable
- **Developer Guide:** https://developer.mastercard.com/open-banking-us/documentation/ (onboarding, sandbox access)
- **Standards:** REST/JSON; FDX-aligned; OAuth 2.0; OpenAPI 3.x specification
- **Authentication:** OAuth 2.0; sandbox credentials available via Mastercard Developers registration

---

### YNAB API

- **Description:** Official REST API for YNAB (You Need A Budget). Provides read/write access to budgets, accounts, categories, transactions, and payees for a YNAB user. Intended for third-party integrations and personal automation tools.
- **API Documentation:** https://api.ynab.com/ and https://develop-api.ynab.com/
- **SDKs/Libraries:** Official Python SDK (ynab/ynab-sdk-python), official JavaScript starter kit (ynab/ynab-api-starter-kit); community PHP SDK (JPry/ynab-sdk-php)
- **Developer Guide:** https://api.ynab.com/v1 (OpenAPI specification)
- **Standards:** REST/JSON; OpenAPI 3.x; OAuth 2.0
- **Authentication:** OAuth 2.0 (Authorization Code Grant and Implicit Grant); Personal Access Tokens for individual developer use

---

### Financial Data Exchange (FDX) — Reference Implementation

- **Description:** The US open banking standards body. FDX API v6.5 (as of early 2026) defines the canonical API specification for consumer financial data sharing, covering deposit, loan, investment, insurance, tax data, and rewards. Used by JPMorgan Chase, Bank of America, Wells Fargo, Capital One, and major aggregators.
- **API Documentation:** https://financialdataexchange.org/ (member access)
- **Mastercard FDX Hub:** https://developer.mastercard.com/fdx-dev-hub/documentation
- **Akoya FDX Reference:** https://docs.akoya.com/docs/intro-to-fdx
- **Standards:** REST/JSON; FAPI 1.0 (migration to FAPI 2.0 in progress); OAuth 2.0; OpenAPI 3.x
- **Authentication:** FAPI 1.0-compliant OAuth 2.0 flows; mTLS for high-security endpoints

---

### Monarch Money (Unofficial API / MCP Server)

- **Description:** Monarch Money is a personal finance SaaS with no official public API, but the community has produced a Python API wrapper and an MCP server for AI agent access. These are the closest available developer resources for understanding Monarch's data model.
- **Python API wrapper:** https://github.com/hammem/monarchmoney
- **Unofficial API (community):** https://github.com/pbassham/monarch-money-api
- **MCP Server:** https://mcpservers.org/servers/colvint/monarch-money-mcp
- **Standards:** Unofficial; reverse-engineered from web app traffic
- **Authentication:** Session-based login (email/password); no OAuth support in unofficial wrappers
- **Note:** Use only for reference and design inspiration; production use carries legal risk

---

### Empower Retirement Developer Portal

- **Description:** Empower's official developer portal, focused on enterprise and advisor integrations with the Empower Retirement platform. Provides participant-level balance data (total balances, investments, loans, vesting). Not a consumer personal dashboard API.
- **API Documentation:** https://developer.empower-retirement.com/
- **API Catalog:** https://developer.empower-retirement.com/api-catalog
- **Standards:** REST/JSON; OAuth 2.0
- **Authentication:** OAuth 2.0; access by request through Empower representative
- **Note:** The consumer Personal Dashboard (formerly Personal Capital) does not have an official public API as of 2026; only unofficial community libraries exist for that product

---

## Notes

**Aggregation infrastructure dependency:** Any new PFM product must integrate with at least one of Plaid, MX, or Mastercard Open Banking (Finicity) for bank connectivity. Plaid is the default choice for US-market consumer apps. MX is strong for institutions wanting FDX-aligned open finance. Building direct FDX connections to individual banks is feasible long-term but not practical for an MVP.

**CFPB Section 1033 transition:** The 2024 rule mandates FDX-compliant API access from large US banks starting 2026. This means the dependency on aggregation middleware will reduce over time as banks expose their own FDX-compliant APIs. A forward-looking architecture should abstract the aggregation layer so it can shift from Plaid toward direct FDX connections as bank coverage grows.

**AI Act compliance gap:** No existing consumer PFM tool has published an AI Act compliance posture as of 2026. An open-source tool built with transparency-first design (explainable AI decisions, user-visible categorisation confidence scores, no opaque credit scoring) would be well-positioned ahead of enforcement.

**OFX legacy:** OFX file import remains relevant for users migrating from Quicken or legacy bank data exports. Supporting QFX/OFX import is low-cost and captures users with years of historical data in that format.
