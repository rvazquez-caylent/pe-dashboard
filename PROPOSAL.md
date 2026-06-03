# Private Equity Intelligence Platform
## Solution Proposal

**Prepared by:** Rodrigo Vázquez, Caylent  
**Date:** June 2026  
**Version:** 1.0  
**Confidentiality:** For client review only

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Proposed Solution](#3-proposed-solution)
4. [Live Demo](#4-live-demo)
5. [Architecture](#5-architecture)
6. [Data Architecture — Medallion Layers](#6-data-architecture--medallion-layers)
7. [Dashboard Suite](#7-dashboard-suite)
8. [Technology Stack](#8-technology-stack)
9. [MVP Roadmap](#9-mvp-roadmap)
10. [Timeline Acceleration with Claude Code](#10-timeline-acceleration-with-claude-code)
11. [Investment & Cost](#11-investment--cost)
12. [Why Caylent](#12-why-caylent)
13. [Next Steps](#13-next-steps)

---

## 1. Executive Summary

Private equity firms manage complex, multi-layered data across fund performance, portfolio companies, capital accounts, deal pipelines, and risk exposure — yet most rely on Excel spreadsheets, quarterly PDF reports, and manual aggregation processes that are slow, error-prone, and opaque to LPs.

This proposal presents a **cloud-native Private Equity Intelligence Platform** built on AWS, featuring a medallion data architecture (Bronze → Silver → Gold) on Amazon S3 and Athena, a real-time API layer, and a suite of five interactive dashboards covering every dimension of fund management.

With AI-accelerated development using **Claude Code**, the full MVP can be delivered in **4–5 weeks** — roughly 60% faster than a traditional development cycle — at a production AWS infrastructure cost of **$70–100/month**.

The result: a single source of truth for GPs and LPs, updated automatically, accessible from any browser, secured with role-based access control.

---

## 2. Problem Statement

### What PE firms deal with today

| Pain Point | Impact |
|---|---|
| Fund data lives in Excel & email | Manual errors, version conflicts, hours of reconciliation |
| LP reports are quarterly PDFs | LPs have no real-time visibility, increasing inbound inquiries |
| IRR/TVPI computed by hand | Risk of calculation errors, inconsistency across reports |
| Portfolio company data is siloed | No cross-portfolio analytics or early warning system |
| Deal pipeline tracked in spreadsheets | No probability-weighted forecasting, no audit trail |
| No concentration or risk dashboard | Blind spots in portfolio construction health |

### The cost of the status quo
- A typical mid-market PE fund spends **8–12 hours per quarter per analyst** preparing LP reports manually.
- Errors in NAV or IRR calculations can trigger LP audit requests and damage credibility.
- LPs increasingly expect digital portals — firms without them are losing LP commitments to competitors that offer them.

---

## 3. Proposed Solution

A **three-layer platform** that transforms raw fund data into real-time, interactive intelligence:

```
Raw Data  →  Cloud Data Warehouse  →  API  →  Interactive Dashboards
(any source)   (S3 + Athena/Redshift)          (browser, any device)
```

### Core capabilities

- **Automated data ingestion** from fund admin systems (Allvue, Investran), portfolio company ERPs, and manual uploads
- **Medallion architecture** on AWS S3: Bronze (raw) → Silver (clean) → Gold (metrics computed)
- **Accurate financial metrics** — IRR, TVPI, DPI, RVPI, MOIC — calculated automatically from cash flow data
- **Five interactive dashboards** for Fund Overview, Portfolio Companies, Capital Account, Deal Pipeline, and Risk & Concentration
- **Role-based access control** — GPs see everything; LPs see their account and portfolio performance only
- **Export-ready** — PDF reports generated on demand from any dashboard view

---

## 4. Live Demo

A fully functional prototype with sample data is available at:

🔗 **https://rvazquez-caylent.github.io/pe-dashboard/**

The demo uses **Summit Capital Partners III, L.P.** — a fictional $500M 2019-vintage North America buyout fund — with realistic metrics and 8 portfolio companies across Technology, Healthcare, Aerospace, Industrials, and other sectors.

| Demo Dashboard | URL path |
|---|---|
| Fund Overview | `/index.html` |
| Portfolio Companies | `/portfolio.html` |
| Capital Account | `/capital.html` |
| Deal Pipeline | `/pipeline.html` |
| Risk & Concentration | `/risk.html` |

> The prototype uses static data. The production system connects to live AWS data via authenticated API calls — the dashboards themselves are identical.

---

## 5. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  DATA SOURCES                                                    │
│  Fund Admin (Allvue / Investran)  ·  Portfolio Company ERPs      │
│  Market Data (Bloomberg / PitchBook)  ·  Manual CSV/Excel        │
└───────────────────────┬──────────────────────────────────────────┘
                        │  AWS Glue ETL / Lambda Ingestion
┌───────────────────────▼──────────────────────────────────────────┐
│  STORAGE LAYER — Amazon S3 (Medallion Architecture)             │
│                                                                  │
│  🥉 Bronze   s3://fund-data/bronze/    Raw files, as-received    │
│  🥈 Silver   s3://fund-data/silver/    Cleaned & validated       │
│  🥇 Gold     s3://fund-data/gold/      Business metrics computed  │
└───────────────────────┬──────────────────────────────────────────┘
                        │
          ┌─────────────┴──────────────┐
          │                            │
┌─────────▼─────────┐      ┌──────────▼──────────┐
│  Amazon Athena    │      │  Redshift Serverless │
│  (ad-hoc queries, │  OR  │  (complex analytics, │
│   low cost)       │      │   high concurrency)  │
└─────────┬─────────┘      └──────────┬───────────┘
          └─────────────┬─────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────────┐
│  API LAYER                                                       │
│  FastAPI on AWS Lambda  ·  API Gateway  ·  Cognito Auth (JWT)   │
│                                                                  │
│  GET /api/fund/overview     GET /api/portfolio/companies         │
│  GET /api/cashflows         GET /api/pipeline                    │
│  GET /api/risk/concentration                                     │
└───────────────────────┬──────────────────────────────────────────┘
                        │  fetch() calls
┌───────────────────────▼──────────────────────────────────────────┐
│  PRESENTATION LAYER                                              │
│  5 HTML Dashboards  ·  Chart.js  ·  CloudFront CDN              │
│  Role-based views (GP / LP)  ·  PDF export                      │
└──────────────────────────────────────────────────────────────────┘
```

### Infrastructure principles
- **Serverless-first** — Lambda + Athena means zero idle cost; you pay only for queries run
- **Infrastructure as Code** — all AWS resources defined in Terraform, fully reproducible
- **Least-privilege IAM** — each service has only the permissions it needs
- **Encryption at rest and in transit** — S3 SSE-S3, HTTPS everywhere
- **Multi-environment** — staging mirrors production with anonymized data

---

## 6. Data Architecture — Medallion Layers

### Bronze — Raw Ingestion
Data lands exactly as received. No transformation. Full audit trail.

| Source | Format | Cadence |
|---|---|---|
| Fund admin export | CSV | Monthly |
| Portfolio company P&L | Excel / CSV | Quarterly |
| Deal pipeline tracker | JSON / CSV | Weekly |
| Market comparables | JSON (API) | Daily |
| LP capital accounts | PDF → structured | Quarterly |

### Silver — Cleaned & Validated
- Standardized field names and data types
- Null handling and deduplication
- Cross-source entity resolution (same company referenced differently in two systems)
- Schema validation with failure alerts

### Gold — Business Metrics
All financial metrics computed once, consistently, and stored for fast query:

| Metric | Definition | Computed by |
|---|---|---|
| **IRR** | Internal Rate of Return — annualized return accounting for cash flow timing | Python Lambda (XIRR) |
| **TVPI** | (Distributions + NAV) / Called Capital | Athena SQL |
| **DPI** | Cumulative Distributions / Called Capital | Athena SQL |
| **RVPI** | NAV / Called Capital | Athena SQL |
| **MOIC** | Current FMV / Cost Basis | Athena SQL |
| **J-Curve** | Cumulative net cash flow over fund life | Athena SQL |
| **HHI** | Σ(sector_share²) — concentration index | Athena SQL |
| **Debt/EBITDA** | Total debt / trailing 12-month EBITDA | Athena SQL |

---

## 7. Dashboard Suite

### Dashboard 1 — Fund Overview
The executive summary. Everything a GP needs to understand fund health at a glance.

- **KPI Cards:** NAV, Net IRR, TVPI, DPI, Called Capital, Distributions
- **J-Curve Chart:** Net cumulative cash flows from inception
- **NAV & IRR Progression:** Dual-axis time series showing fund maturation
- **Sector Allocation:** Portfolio FMV by sector (doughnut chart)
- **Capital Deployment Waterfall:** Committed → Called → Invested → Realized → Unrealized
- **Portfolio Summary Table:** All companies with status, MOIC, and IRR at a glance

### Dashboard 2 — Portfolio Companies
Deep-dive into each investment's operational and financial performance.

- **Company Cards:** Status, MOIC, IRR, FMV, ownership stake per company
- **Revenue & EBITDA Trends:** Time series for top performers
- **MOIC Ranking:** Visual comparison across all portfolio companies
- **Detailed Financials Table:** Entry vs. current EV, multiples, employee count

### Dashboard 3 — Capital Account
The LP-facing cash flow view. Answers: "Where is my money and when do I get it back?"

- **Capital Deployment Waterfall:** Progress bars from committed to distributed
- **Quarterly Cash Flows:** Capital calls and distributions by quarter
- **Cumulative J-Curve:** Running net cash flow from fund inception
- **Cash Flow Table:** Full quarterly history with cumulative totals

### Dashboard 4 — Deal Pipeline
The deal team view. Tracks active opportunities from screening to close.

- **Pipeline KPIs:** Active deals, deals in exclusivity, probability-weighted equity, avg EV/EBITDA
- **Stage Funnel:** Visual deal count and EV by pipeline stage
- **Deal Cards:** Each opportunity with description, metrics, probability bar, and deal owner
- **Full Pipeline Table:** All deals sorted by stage with source, probability, and team owner

### Dashboard 5 — Risk & Concentration
Portfolio construction health. Answers: "Are we too concentrated anywhere?"

- **HHI Index:** Herfindahl–Hirschman Index for sector concentration
- **Sector & Geographic Doughnuts:** FMV distribution across sectors and regions
- **Vintage Year Analysis:** Number of companies and FMV by entry year cohort
- **Concentration Heat Map:** Color-coded risk table by sector
- **Company Risk Table:** Leverage, MOIC, and risk flags (🔴🟡🟢) per company
- **Benchmark vs Strategy:** Current portfolio metrics vs. target construction parameters

---

## 8. Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| **Storage** | Amazon S3 | Unlimited scale, $0.023/GB, native Glue/Athena integration |
| **Catalog & ETL** | AWS Glue | Serverless Spark, auto-schema discovery, native S3 integration |
| **Query Engine** | Amazon Athena | Serverless SQL over S3, $5/TB scanned, no cluster to manage |
| **Optional upgrade** | Redshift Serverless | If query volume or concurrency increases above Athena's sweet spot |
| **Transformations** | dbt (Data Build Tool) | SQL-native transformations, lineage, testing, documentation |
| **IRR Calculation** | AWS Lambda (Python) | `numpy_financial.xirr()` — standard, auditable XIRR implementation |
| **API** | FastAPI + Lambda | Auto-generated OpenAPI docs, type-safe, serverless |
| **API Gateway** | AWS API Gateway | Throttling, caching, usage plans, CORS |
| **Authentication** | Amazon Cognito | Managed auth, MFA, JWT tokens, SAML/SSO ready |
| **CDN** | Amazon CloudFront | Global dashboard delivery, S3 origin, HTTPS |
| **Charts** | Chart.js 4.x | Best-in-class financial charts, CDN, no license cost |
| **IaC** | Terraform | Reproducible infrastructure, state management, all environments |
| **CI/CD** | GitHub Actions | Automated deploy on merge to main |
| **Monitoring** | CloudWatch | Alarms, dashboards, cost anomaly detection |

---

## 9. MVP Roadmap

### Phase 0 — Setup & Foundation · *2 days*
AWS account structure, IAM roles, Terraform state backend, S3 bucket hierarchy, GitHub Actions CI/CD skeleton.

### Phase 1 — Bronze Layer · *2 days*
Glue crawlers, ingestion Lambda (S3 event trigger), schema validation, raw Athena tables.

### Phase 2 — Silver & Gold Layers · *5 days*
Glue ETL (Bronze→Silver), dbt models (Silver→Gold), XIRR Lambda, Athena Gold views for all 5 dashboards, EventBridge refresh schedules.

### Phase 3 — API Layer · *3 days*
FastAPI application, all 8 REST endpoints, API Gateway config, Cognito JWT middleware, integration test suite.

### Phase 4 — Dashboard Integration · *2 days*
Replace static `data.js` with async `fetch()` calls, loading skeletons, error states, quarter selector filter, PDF export, CloudFront deployment.

### Phase 5 — Auth & Access Control · *2 days*
Cognito User Pool, GP vs LP roles, field-level filtering in API, login page.

### Phase 6 — Testing & Launch · *3 days*
pytest suites, load tests (Locust), staging environment, CloudWatch alarms, go-live.

**Total: 4–5 weeks to production MVP**

---

## 10. Timeline Acceleration with Claude Code

Claude Code (Anthropic's AI coding assistant) is used throughout development to generate boilerplate, infrastructure configs, test suites, and API scaffolding — tasks that are time-consuming but not where human judgment adds the most value.

### Time savings by phase

| Phase | Manual Estimate | With Claude Code | Savings |
|---|---|---|---|
| Infrastructure (Terraform, IAM) | 1.5 weeks | 2 days | **75%** |
| ETL pipeline (Glue, dbt) | 3 weeks | 5 days | **70%** |
| API layer (FastAPI, Gateway) | 2 weeks | 3 days | **70%** |
| Dashboard integration | 1.5 weeks | 2 days | **65%** |
| Auth & security | 1 week | 2 days | **70%** |
| Testing & launch | 1 week | 3 days | **55%** |
| **Total** | **10–12 weeks** | **4–5 weeks** | **~60%** |

### What Claude Code handles
- Terraform modules for every AWS resource
- Glue ETL PySpark scripts
- dbt SQL transformation models
- FastAPI route handlers and Pydantic models
- pytest and Locust test suites
- CloudWatch alarm configurations
- Operational runbooks

### What always requires human judgment
- Validating IRR/TVPI against fund admin audited numbers
- FMV methodology decisions (legal/accounting)
- LP data sharing permissions and compliance review
- Security architecture sign-off
- Stakeholder alignment on what GPs/LPs actually need to see

---

## 11. Investment & Cost

### AWS Monthly Infrastructure Cost (production)

| Service | Usage Assumption | Est. Monthly Cost |
|---|---|---|
| S3 | 100 GB data + transfers | ~$3 |
| AWS Glue | Monthly ETL runs | ~$15 |
| Amazon Athena | ~4 TB/month scanned | ~$20 |
| AWS Lambda | API + XIRR (low traffic) | ~$5 |
| API Gateway | ~100K requests/month | ~$10 |
| CloudFront | Dashboard CDN | ~$5 |
| Amazon Cognito | < 50K MAU | **Free** |
| CloudWatch | Logs + alarms | ~$10 |
| **Total** | | **~$70–100/month** |

> Upgrading to Redshift Serverless adds ~$300–600/month depending on query volume — recommended only when concurrent LP users exceed ~20 or cross-fund analytics are needed.

### Total Cost of Ownership vs. Alternatives

| Option | Setup | Monthly | LP Portal | Customizable |
|---|---|---|---|---|
| **This solution** | 4–5 weeks | $70–100 | ✅ | ✅ Full |
| Dynamo (fund software) | 3–6 months | $2,000–5,000 | ✅ | ⚠️ Limited |
| Cobalt (LP platform) | 2–4 months | $1,500–3,000 | ✅ | ⚠️ Limited |
| Excel + PDF reports | Ongoing | $0 | ❌ | ✅ Full |
| PowerBI / Tableau | 4–8 weeks | $500–1,500 | ⚠️ | ⚠️ Limited |

---

## 12. Why Caylent

- **AWS expertise** — Caylent is an AWS Premier Partner with deep expertise in data, analytics, and cloud-native architecture
- **PE domain knowledge** — proven experience with financial data pipelines, IRR calculations, and LP reporting workflows
- **AI-accelerated delivery** — use of Claude Code and other AI tooling reduces delivery timelines without sacrificing quality
- **Outcome-focused** — we build for your business outcome (LP confidence, GP efficiency, audit readiness), not just for technical completeness
- **Post-launch support** — optional managed service to handle quarterly data loads, schema changes, and dashboard enhancements

---

## 13. Next Steps

| Step | Owner | Timeline |
|---|---|---|
| Review this proposal and demo | Client | This week |
| Technical discovery call (data sources, fund admin system) | Caylent + Client | Week 1 |
| Confirm AWS account access and data sharing agreements | Client | Week 1 |
| Kick off Phase 0 | Caylent | Week 2 |
| Bronze layer live with first data load | Caylent | Week 2–3 |
| Gold layer + API demo with real data | Caylent | Week 3–4 |
| Full MVP go-live | Caylent | Week 5 |

---

### Questions or to schedule a demo

**Rodrigo Vázquez**  
rodrigo.vazquez@caylent.com  
Caylent — AWS Premier Partner

---

*This proposal and the associated demo are confidential and intended solely for the recipient. All financial data presented in the demo is fictional and for illustration purposes only.*
