# PE Dashboard — MVP Roadmap
## S3 + Athena + Medallion Architecture · Accelerated with Claude Code

---

## Timeline Comparison

| | Without Claude Code | With Claude Code | Reduction |
|---|---|---|---|
| **Total MVP** | 10–12 weeks | **4–5 weeks** | ~60% |
| Infrastructure | 1.5 weeks | 2 days | 75% |
| ETL / Data Pipeline | 3 weeks | 5 days | 70% |
| API Layer | 2 weeks | 3 days | 70% |
| Dashboard Integration | 1.5 weeks | 2 days | 65% |
| Auth & Security | 1 week | 2 days | 70% |
| Testing & Launch | 1 week | 3 days | 55% |

**Why ~60% faster?** Claude Code eliminates the "plumbing" work:
Terraform modules, IAM policies, Glue schemas, Lambda boilerplate,
FastAPI scaffolding, test suites — all generated in minutes, not days.
Human effort concentrates on architecture decisions, data validation,
and business logic review.

---

## Phases

---

### Phase 0 — Setup & Foundation
**Duration:** 2 days (Claude Code) · 1 week (manual)

**What gets built:**
- AWS account structure, IAM roles & policies
- Terraform state backend (S3 + DynamoDB lock)
- S3 bucket structure with lifecycle policies
- CI/CD skeleton (GitHub Actions → AWS)
- Local dev environment

**S3 Structure:**
```
s3://scp3-fund-data/
├── bronze/
│   ├── portfolio-companies/year=2026/quarter=Q2/
│   ├── cashflows/
│   └── pipeline/
├── silver/
│   ├── portfolio-companies/
│   └── cashflows/
└── gold/
    ├── fund-metrics/
    ├── company-performance/
    └── risk-concentration/
```

**Claude Code does:**
- Writes all Terraform (VPC, S3, IAM, Glue, Athena)
- Generates S3 bucket policies and lifecycle rules
- Scaffolds GitHub Actions CI/CD workflow

**Human judgment required:**
- AWS account structure decisions
- Security & compliance review
- Tagging / cost allocation strategy

---

### Phase 1 — Bronze Layer (Raw Ingestion)
**Duration:** 2 days (Claude Code) · 2 weeks (manual)

**What gets built:**
- Glue Data Catalog databases & crawlers
- Ingestion Lambda for CSV/JSON uploads
- Schema validation at landing
- Raw Athena tables over S3

**Data sources handled:**
| Source | Format | Frequency |
|---|---|---|
| Fund admin export (Allvue/Investran) | CSV | Monthly |
| Portfolio company P&L | Excel/CSV | Quarterly |
| Deal pipeline tracker | JSON/CSV | Weekly |
| Market comparables | JSON (API) | Daily |
| Manual LP reports | PDF → structured | Quarterly |

**Claude Code does:**
- Writes Glue crawler configs for each source
- Generates ingestion Lambda with S3 event triggers
- Writes schema validation logic
- Creates Athena DDL for all bronze tables

**Human judgment required:**
- Mapping source system field names to canonical schema
- Handling edge cases in fund admin exports (they are always messy)

---

### Phase 2 — Silver & Gold Layers (Transformation)
**Duration:** 5 days (Claude Code) · 3 weeks (manual)

**What gets built:**
- Bronze → Silver: cleaning, deduplication, type casting, null handling
- Silver → Gold: business metric computation
- Scheduled refresh (EventBridge daily/quarterly triggers)

**Key computations in Gold layer:**
```
IRR        → Custom Lambda (Newton-Raphson XIRR, Python numpy_financial)
TVPI       → (SUM(distributions) + current_nav) / called_capital
DPI        → SUM(distributions) / called_capital
RVPI       → current_nav / called_capital
MOIC       → current_fmv / cost_basis
HHI        → SUM(POWER(sector_pct, 2))  -- concentration index
Debt/EBITDA → total_debt / trailing_ebitda
```

**Gold Athena views (one per dashboard endpoint):**
- `vw_fund_overview` — all FUND-level KPIs
- `vw_portfolio_companies` — per-company metrics + financials
- `vw_quarterly_cashflows` — J-curve data
- `vw_deal_pipeline` — pipeline with probability-weighted equity
- `vw_risk_concentration` — HHI, sector/geo breakdown

**Claude Code does:**
- Writes all Glue ETL PySpark scripts (Bronze → Silver)
- Writes dbt models for Silver → Gold
- Implements XIRR Lambda in Python
- Writes all Athena view DDL
- Generates EventBridge scheduler configs

**Human judgment required:**
- Validating IRR calculation against fund admin's own numbers
- Deciding FMV methodology (cost, mark-to-market, third-party)
- Handling restatements and prior-period corrections

---

### Phase 3 — API Layer
**Duration:** 3 days (Claude Code) · 2 weeks (manual)

**What gets built:**
- FastAPI application on AWS Lambda (via Mangum adapter)
- API Gateway with usage plans & throttling
- One endpoint per dashboard, one for each chart dataset
- Response caching (API Gateway cache or ElastiCache)

**Endpoints:**
```
GET  /api/fund/overview          → KPIs, NAV, IRR, TVPI, DPI
GET  /api/fund/history           → Quarterly NAV & IRR time series
GET  /api/portfolio/companies    → All 8 companies + metrics
GET  /api/portfolio/{id}         → Single company detail
GET  /api/cashflows              → Quarterly contributions & distributions
GET  /api/pipeline               → All deals + stage + probability
GET  /api/risk/concentration     → Sector, geo, HHI breakdowns
GET  /api/risk/companies         → Company-level risk flags
```

**Claude Code does:**
- Scaffolds entire FastAPI app structure
- Writes all route handlers (Athena query → JSON response)
- Writes Pydantic response models
- Generates API Gateway Terraform config
- Writes integration test suite

**Human judgment required:**
- Deciding which fields are LP-visible vs GP-only
- Cache TTL strategy (NAV can change daily, IRR less often)

---

### Phase 4 — Dashboard Integration
**Duration:** 2 days (Claude Code) · 1.5 weeks (manual)

**What changes:** `data.js` → `fetch()` calls. Everything else stays.

**Before (today):**
```js
const FUND = { nav: 528.75, irr: 18.4, tvpi: 1.636 … };
```

**After:**
```js
const FUND = await fetch(`${API_BASE}/api/fund/overview`,
  { headers: { Authorization: `Bearer ${token}` } }
).then(r => r.json());
```

**New features added in this phase:**
- Loading skeleton states for all charts
- Error banners with retry logic
- Quarter selector (filter all dashboards to a specific quarter)
- "Last refreshed" timestamp
- Export to PDF button (html2canvas + jsPDF)
- CloudFront distribution for the static dashboard files

**Claude Code does:**
- Rewrites all 5 dashboards to use async data fetching
- Implements loading skeletons and error states
- Writes PDF export logic
- Creates CloudFront Terraform config

**Human judgment required:**
- UX decisions on loading behavior
- Which filters matter most to LPs vs GPs

---

### Phase 5 — Auth & Access Control
**Duration:** 2 days (Claude Code) · 1 week (manual)

**What gets built:**
- Cognito User Pool (email + MFA)
- Two roles: GP (full access) and LP (read-only, no pipeline/cost data)
- JWT token validation in API Gateway
- Login page integrated with dashboards

**Access matrix:**
| Dashboard | GP | LP |
|---|---|---|
| Fund Overview | ✅ Full | ✅ Full |
| Portfolio Companies | ✅ Full | ✅ No cost basis |
| Capital Account | ✅ Full | ✅ Own account only |
| Deal Pipeline | ✅ Full | ❌ Hidden |
| Risk & Concentration | ✅ Full | ✅ Full |

**Claude Code does:**
- Writes Cognito Terraform config
- Writes JWT middleware for FastAPI
- Implements role-based field filtering in API responses
- Writes login page HTML

**Human judgment required:**
- Confirming LP data sharing permissions with legal/compliance
- MFA policy decisions

---

### Phase 6 — Testing, Staging & Launch
**Duration:** 3 days (Claude Code) · 1 week (manual)

**What gets built:**
- Unit tests for all Lambda/API functions
- Integration tests (real Athena queries against staging data)
- Load test (simulate 50 concurrent LP users)
- Staging environment (mirrors production with anonymized data)
- Runbook & monitoring (CloudWatch dashboards + alarms)

**Claude Code does:**
- Writes pytest test suites for all API endpoints
- Writes Locust load test scripts
- Generates CloudWatch alarm Terraform configs
- Writes operational runbook (Markdown)

**Human judgment required:**
- Sign-off on test results
- Go/no-go decision for production launch
- LP onboarding communication

---

## Full Timeline (with Claude Code)

```
Week 1     Week 2     Week 3     Week 4     Week 5
│          │          │          │          │
├─ Ph.0 ──┤
│  Setup   │
          ├─ Ph.1 ──┤
          │  Bronze  │
                    ├──── Ph.2 ─────┤
                    │  Silver/Gold  │
                                   ├─ Ph.3 ┤
                                   │  API  │
                                           ├─ Ph.4+5+6 ─┤
                                           │  Dashboards │
                                           │  Auth       │
                                           │  Launch     │
```

**Go-live: end of Week 5**

---

## Team & Cost

### Recommended Team (with Claude Code)
| Role | Time | Responsibility |
|---|---|---|
| Data Architect | 50% (you) | Architecture decisions, data model, validation |
| Claude Code | ~20 hrs/week | All code generation, boilerplate, tests |
| AWS review (optional) | 1 day | Security & cost review before launch |

### AWS Monthly Cost (production)
| Service | Est. Cost/month |
|---|---|
| S3 (100GB data) | ~$3 |
| Glue ETL (monthly runs) | ~$15 |
| Athena (query volume) | ~$20 |
| Lambda (API, low traffic) | ~$5 |
| API Gateway | ~$10 |
| CloudFront | ~$5 |
| Cognito (< 50K MAU) | Free |
| CloudWatch | ~$10 |
| **Total** | **~$70–100/month** |

---

## What Claude Code Cannot Replace

| Task | Why human judgment is irreplaceable |
|---|---|
| IRR validation | Must match fund admin's audited numbers exactly |
| FMV methodology | Legal/accounting decision, not an engineering one |
| LP data permissions | Compliance and legal review required |
| Security sign-off | Human accountable for access control decisions |
| Stakeholder alignment | Architecture must match what GPs actually need |

---

*Roadmap version 1.0 · Summit Capital Partners III · June 2026*
