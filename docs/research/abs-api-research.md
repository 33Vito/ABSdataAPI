# ABS API Ecosystem — Research Notes (Ver. 3 — Corrected 2026-11-23)

> Date: 2026-11-23  
> Corrections applied per Orca review (orchestration msg_55257aa55d90)  
> All endpoint tests run live on 2026-11-23 UTC  
> Model: claude-sonnet-5-5 (pinned via Claude Code, authenticated via Pro account)  
> Project: ABSdataAPI — PoC for RBA automated data ingestion

> **Related:** this note is the implementation evidence for the pipeline. The design rationale is in the
> [central bank ingestion report](central-bank-ingestion/report.md): how RBA, Fed, Bundesbank and BoK pull
> national statistics, and why the PoC uses a bounded, auditable pull. The Australian source notes are in
> [australia-rba.md](central-bank-ingestion/notes/australia-rba.md).

---

## 1. The Three ABS APIs

| API | Status | Auth | Base URL | Data Scope | Custom Queries? |
|---|---|---|---|---|---|
| **Data API** | Beta (since Nov 2024 update) | None — key-free | `https://data.api.abs.gov.au/rest/` | All ABS data — economic, social, Census (~1,200+ dataflows) | Yes — SDMX 2.1 dimensional queries |
| **Indicator API** | Live/Production | API key required (email form → api.data@abs.gov.au) | `https://indicator.api.abs.gov.au/v1/` | 22 curated headline dataflows only | No — returns whole dataset per dataflow |
| **Time Series Directory API** | Live | None | `https://abs.gov.au/servlet/TSSearchServlet` | Metadata only — catalogue numbers, table titles, series IDs | Yes — search params (catno, ttitle, sid, ptitle) |

### Key difference

- **Data API** = free, open, detailed, customisable, but Beta quality/no SLA. Use for PoC.
- **Indicator API** = production-grade, but requires bureaucratic key request (DOCX form → email `api.data@abs.gov.au`, ~week turnaround). Limited to 22 topics. No SLA documented.
- **Time Series Directory API** = use for *discovery*: find series IDs to feed into the Data API. XML output only.

---

## 2. Data API — Technical Details

### Protocol
SDMX 2.1 REST — the international standard for statistical data exchange.

### URL structure
```
https://data.api.abs.gov.au/rest/data/{agencyId},{dataflowId},{version}/{dataKey}?{params}
```

### Format options
| Parameter | Notes |
|---|---|
| `format=csv` | SDMX-CSV. Required for >100K obs (unverified hard limit, community-reported). |
| `format=jsondata` | SDMX-JSON. May be rejected for large responses. |
| `format=structurespecificdata` | SDMX-XML (default). Verbose. |

### Query parameters — all verified live
| Parameter | Verified? | Example |
|---|---|---|
| `startPeriod` / `endPeriod` | ✅ YES | `startPeriod=2026-01&endPeriod=2026-03` |
| `lastNObservations=N` | ✅ YES | Returns N observations per series |
| `firstNObservations=N` | ✅ YES | Returns first N observations per series |
| `updatedAfter={ISO datetime}` | ⚠️ **Experimental — unreliable** | See subsection below |
| `dimensionAtObservation` | ✅ YES | Controls pivot orientation |

### updatedAfter — experimental, not production-ready
- **Initial test:** empty result for `updatedAfter=2026-11-01T00:00:00Z` on Oct 2026 LF data (which was published earlier). This suggested it works but was not a rigorous test.
- **Independent probe findings (2026-11-23 UTC):**
  - Single-series query with `updatedAfter=2025-01-01T00:00:00Z` terminated mid-transfer after HTTP 200.
  - All-series version with date range + `updatedAfter` returned a June row despite a January range before transfer failure.
  - Far-future cutoff gave 404 NoRecordsFound.
- **Conclusion: Do not rely on `updatedAfter` for production.** Behaviour is unreliable under load. Prefer:
  - **Bounded time-slice snapshots:** `startPeriod=X&endPeriod=Y` to cut the data into windows
  - **Value comparison:** store every (key, OBS_VALUE) and compare on each poll
  - **Complete-response validation:** verify total row count matches expected count per slice
- **ABS gateway limits:** The ABS troubleshooting page documents **30-second timeout** and **10 MB response size** limits on the API gateway. Queries approaching either limit may terminate mid-transfer — split large queries into smaller time windows.

### Metadata API — verified live
```
GET /rest/dataflow?detail=allstubs
  → Lists ALL dataflows (~1,200+ entries, 106K chars of output)

GET /rest/dataflow/ABS/{ID}?format=json-structure-2.0.0
  → Single dataflow structure with version, dimensions, description, methodology link
```

### Known limitations (all claims qualified — community-reported, not ABS-documented)

| Claim | Evidence | Status |
|---|---|---|
| 1M observation cap per request | From community repos | **Unverified** — no ABS documentation confirms this |
| 100K+ obs requires CSV | From community repos | **Unverified** — may be server-enforced for JSON only |
| Rate limiting (429) | OpenAPI spec lists 429 response | **Reasonable assumption** — threshold undocumented |
| Data may lag website by hours | Community report | **Unverified** — update timing not documented by ABS |

---

## 3. Indicator API — Technical Details

### URL structure
```
https://indicator.api.abs.gov.au/v1/data/{dataflowIdentifier}/{format}
```

- No custom filters — returns the full dataset for the dataflow
- Format: `json`, `csv`, `xml`
- Dataflow identifiers: `CPI_H`, `CPI_Q_H`, `LF_H`, `GDPE_H`, `WPI_H`, etc. (22 total)

### Authentication
Header: `x-api-key: <your_key>`
Request form: download DOCX from ABS → fill → email `api.data@abs.gov.au`

### Confirmed dataflows
| ID | Topic |
|---|---|
| `AWE_H` | Average Weekly Earnings |
| `BA_GCCSA_H` | Building Approvals by GCCSA |
| `BOP_H` | Balance of Payments |
| `CAPEX_H` | Private Capital Expenditure |
| `CPI_H` | Consumer Price Index (headline) |
| `CPI_Q_H` | CPI Quarterly |
| `GDPE_H` | GDP (Expenditure) |
| `HSI_M_H` | Monthly Household Spending |
| `ITGS_H` | International Trade in Goods |
| `ITPI_EXP_H` | Export Price Index |
| `ITPI_IMP_H` | Import Price Index |
| `JV_H` | Job Vacancies |
| `LEND_H` | Lending Indicators |
| `LF_H` | Labour Force |
| `PPI_FD_H` | Producer Price Index — Final Demand |
| `QBIS_H` | Quarterly Business Indicators |
| `RT_H` | Retail Trade — **Note: RT ceased in Data API (historical only); verify Indicator API status** |
| `WPI_H` | Wage Price Index |

**Note:** `MBTI_H` (Monthly Business Turnover) is an **Indicator API dataflow identifier**. Its Data API equivalent is `BUSINESS_TURNOVER`, which has live metadata labelled **CEASED** (confirmed 2026-11-23). Verify Indicator API status with a key before relying on MBTI_H.

---

## 4. CPI Dataflows — Corrected Status (Verified Live 2026-11-23)

Both `CPI` (2.0.0) and `CPI_Q` (1.0.0) are **active** — they serve different scopes. CPI_M is ceased.

| Dataflow | Ver | Status | Scope | Q3 2026 row count | Historical depth |
|---|---|---|---|---|---|
| **CPI** | 2.0.0 | **Active — primary monthly feed** | Monthly (FREQ=M) and quarterly (FREQ=Q) combined. 87 expenditure classes by state. **Base: September 2025 = 100** (per ABS Updates). | 24,165 M + 412 Q rows | 1972+ |
| **CPI_Q** | 1.0.0 | **Active — reinstated April 2026** (per ABS API Updates). Dedicated quarterly feed. Different scope from CPI(2.0.0): 96 Q rows vs 412 Q rows for same Q3. | Quarterly only. More selective dimension set than CPI(2.0.0) quarterly subset. | 96 Q rows | 1972+ |
| **CPI_M** | 1.2.0 | **CEASED** — description: "CEASED - Dataset: CPI_M". NonProductionDataflow flag. | Monthly CPI Indicator (Cat 6484.0). Superseded by CPI(2.0.0) which covers the same monthly data. | N/A | 2017-09 onward (historical) |

**Recommendation:** Use `CPI_Q(1.0.0)` for dedicated quarterly tables (more focused scope, fewer rows). Use `CPI(2.0.0)` for consolidated queries where M+Q data is needed together.

---

## 5. RBA-Core Data Coverage — Tested vs Candidate Dataflows

**Note:** CPI(2.0.0), CPI_Q(1.0.0), CPI_M(1.2.0), LF(1.0.0), RT(1.0.0), BOP(1.0.0), QBIS(1.0.0), CAPEX(1.0.0), LABOUR_ACCT_Q(1.1), ANA_AGG(1.0.0), WPI(1.2.0) and ITGS(1.2.0) have been verified — the first nine via metadata endpoints (HTTP 200), the last three via independent bounded data queries (HTTP 200). All other entries below are **candidates** — their Data API IDs and versions must be verified with a metadata or data query before use. The Indicator API `_H` suffix does not carry over to the Data API.

### Verified (metadata endpoint returned 200)
| Dataset | Dataflow ID | Ver | Cat No. | Freq | Notes |
|---|---|---|---|---|---|
| Consumer Price Index (monthly+quarterly combined) | `CPI` | 2.0.0 | 6401.0 | M+Q | Index, MoM%, YoY%, contribution pts by 87 expenditure classes, by state. **Base: September 2025 = 100** (per ABS Updates). |
| Consumer Price Index (quarterly only) | `CPI_Q` | 1.0.0 | 6401.0 | Q | Quarterly index, reinstated April 2026 (per ABS Updates), different scope from CPI(2.0.0). |
| Labour Force | `LF` | 1.0.0 | 6202.0 | M | Employed/Unemployed (000s), participation rate %, unemployment rate %, underemployment rate %, hours worked, by sex/state/age. |
| National Accounts Key Aggregates (GDP) | `ANA_AGG` | 1.0.0 | 5206.0 | Q | Independent HTTP 200 data observation: 2026-Q1 via `ANA_AGG/M1.GPM.20.AUS.Q`. |
| Wage Price Index | `WPI` | 1.2.0 | 6345.0 | Q | Independent HTTP 200 data observation: 2026-Q1 via `WPI/all` (bounded). |
| International Trade in Goods | `ITGS` | 1.2.0 | 5368.0 | M | Independent HTTP 200 data observation: Jan 2026 via `ITGS/all` (bounded). |
| Balance of Payments | `BOP` | 1.0.0 | 5302.0 | Q | Current account, capital/financial account. NonProductionDataflow flag present — verify currentness. |
| Business Indicators | `QBIS` | 1.0.0 | 5676.0 | Q | Profits, sales, wages, inventories by industry. NonProductionDataflow flag present — verify currentness. |
| Private New Capital Expenditure | `CAPEX` | 1.0.0 | 5625.0 | Q | Actual + expected capex by asset type, industry. NonProductionDataflow flag present — verify currentness. |
| Labour Account Australia | `LABOUR_ACCT_Q` | 1.1 | 6150.0.55.003 | Q | Jobs, Persons, Hours, Payments by industry. NonProductionDataflow flag present — verify currentness. |

### Candidates (Data API IDs to verify before use)
Entries below use IDs drawn from the Indicator API spec or community sources. **Provenance: unverified.** Query `/rest/dataflow/ABS/{ID}?format=json-structure-2.0.0` before relying on them.

| Dataset | Candidate ID | Cat No. (candidate) | Freq | Notes |
|---|---|---|---|---|
| Retail Trade (ceased) | `RT` | 8501.0 | M | CEASED — metadata confirmed on 2026-11-23, historical data reachable. |
| Monthly CPI Indicator (ceased) | `CPI_M` | 6484.0 | M | CEASED — metadata confirmed on 2026-11-23, 2017-2024 historical reachable. |
| Household Spending Indicator | `HSI_M` | 5682.0 | M | Candidate — verify ID. |
| Producer Price Index — Final Demand | `PPI_FD` | 6427.0 | Q | Candidate — verify ID. |
| Building Approvals (GCCSA) | `BA_GCCSA` | 8731.0 | M | Candidate — verify ID. |
| Average Weekly Earnings | `AWE` | 6302.0 | M | Candidate — verify ID. |
| Lending Indicators Housing | `LEND_HOUSING` | 5601.0 | M | Candidate — verify ID. |
| Job Vacancies | `JV` | 6354.0 | Q | Candidate — verify ID. |
| Estimated Resident Population | `ERP_Q` | 3101.0 | Q | Candidate — verify ID. |
| Residential Dwellings | `RES_DWELL_ST` | 6432.0 | Q | Candidate — verify ID. |
| Labour Force by Age | `LF_AGE` | 6291.0.55.001 | M | Candidate — verify ID. |
| Selected Living Cost Indexes | — | 6467.0 | Q | Dataflow ID unknown — search required. |
| International Trade Price Indexes | `ITPI_EXP_H` / `ITPI_IMP_H` | 6457.0 | Q | `_H` suffix indicates Indicator API; Data API ID unknown. |
| Modellers' Database | — | 1364.0.15.003 | Q | Dataflow ID unknown — search required. |

---

## 6. Release Schedule & Timing

| Dataset | Cadence | Notes |
|---|---|---|
| CPI (monthly) | Last Wednesday of following month | Embargo lifts and website releases generally at 11:30am Canberra; Indicator API publishes at embargo lift ([ABS pre-release policy](https://www.abs.gov.au/about/legislation-and-policy/policy-pre-release-access-abs-statistics)) |
| CPI (quarterly) | ~5 weeks after quarter end | Same embargo time |
| Labour Force | Monthly | Same |
| National Accounts | Quarterly (~9 weeks after quarter end) | Same |
| WPI | Quarterly | Same |
| International Trade | Monthly | Same |

**Release timing (official):** The [ABS Policy on Pre-Release Access](https://www.abs.gov.au/about/legislation-and-policy/policy-pre-release-access-abs-statistics) states that the ABS releases content to all users at the same time via its website, "in general" at 11.30am Canberra time. The **Indicator API** provides key economic data as soon as the embargo lifts at 11.30am. The policy doesn't mention the Data API.

**Data API availability timing:** The Beta **Data API**'s post-embargo arrival time is **variable and not ABS-documented**. The "~12:30pm" claim is a **community assumption**. The ABS API page specifies release date but not Data API update time. Data may appear on the ABS website (Excel) and Indicator API before the Data API. The PoC should poll after 12:00pm Canberra time on release days and accept variable latency. Retail Trade is ceased — no new releases expected.

> **Related:** pre-release and lock-up arrangements between the ABS and RBA are covered in
> [Australia pairs an embargo-lift API with opaque RBA plumbing](central-bank-ingestion/report.md#australia-pairs-an-embargo-lift-api-with-opaque-rba-plumbing)
> and in [australia-rba.md §3](central-bank-ingestion/notes/australia-rba.md#3-abs-pre-release-and-embargo-arrangements-with-the-rba-and-treasury).

---

## 7. Data Explorer (Visual/UI Layer)

URL: `https://dataexplorer.abs.gov.au/`

- Browser-based UI built on top of the Data API
- Browse, filter, chart, export (Excel/CSV), bookmark/share table views
- **Developer API tab** auto-generates full API calls for any table you build — useful for constructing Data API queries visually
- Beta quality — may lag latest releases

---

## 8. Known Limitations & Pitfalls

1. **Data API is Beta** — no SLA, performance may degrade under load. **30-second timeout and 10 MB response size** gateway limits documented on ABS troubleshooting page.
2. **Dimension codes are numeric** — `MEASURE=3`, `REGION=50`, `INDEX=20004` — must resolve via codelist metadata API for human readability. Codelist endpoint not yet verified.
3. **Version numbers differ by dataflow** — e.g. `ABS,CPI,2.0.0` vs `ABS,LF,1.0.0`; discover via `/rest/dataflow/ABS/{ID}?format=json-structure-2.0.0`.
4. **1M obs cap and CSV requirement** — community-reported but **not ABS-documented**. The 10MB gateway limit may be the actual constraint. Test empirically.
5. **`updatedAfter` is unreliable** — initial test passed but independent probe found mid-transfer failures and inconsistent results. Do not rely on it for production. Use bounded time-slice snapshots and value comparison instead.
6. **Indicator API needs bureaucratic key request** — ~week turnaround from email to provisioned key.
7. **Time Series Directory API returns only XML metadata** — not usable for bulk data retrieval.
8. **Data may lag the ABS website** — published Excel files arrive first, API updated within hours (community report, unverified).
9. **NonProductionDataflow flag** — present on CPI_M, BUSINESS_TURNOVER and RT. CPI_Q was **reinstated in April 2026** (per ABS API Updates); it is active and its scope differs from CPI(2.0.0). The NonProductionDataflow flag's exact semantics are undocumented.

---

## 9. Suggested PoC Architecture

> **Design rationale:** the report's [conclusion](central-bank-ingestion/report.md#conclusion) compares this
> design with how the five central banks ingest data. It also compares the jurisdictions in the
> [comparison matrix](central-bank-ingestion/report.md#master-comparison-matrix-public-dissemination-vs-documented-internal-ingestion).


### Phase 1 — Core Ingestion Pipeline (free, keyless)
```
[cron: M-F 12:00pm Canberra (poll after 12:00, accept variable latency)]
       │
       ▼
  Poll Data API using bounded time-slice snapshots (NOT updatedAfter)
    ├── CPI(2.0.0): startPeriod=YYYY-MM&endPeriod=YYYY-MM (monthly)
    ├── CPI_Q(1.0.0): startPeriod=YYYY&endPeriod=YYYY     (quarterly — prefer for quarterly tables)
    ├── LF(1.0.0):   startPeriod=YYYY-MM&endPeriod=YYYY-MM (monthly)
    ├── ANA_AGG(1.0.0): startPeriod=YYYY&endPeriod=YYYY    (quarterly)
    └── WPI(1.0.0):  startPeriod=YYYY&endPeriod=YYYY       (quarterly)
       │
       ▼
  Resolve codelists → human-readable labels
       │
       ▼
  Store every (key, OBS_VALUE) — compare on each poll to detect changes
       │
       ▼
  Write CSV/Parquet + provenance metadata (fetch_timestamp, dataflow_version, response_size)
       │
       ▼
  Validate: complete-response (check expected vs actual row count per slice)
       │
       ▼
  Log: new observations, revisions (value diff), gaps, gateway limit warnings
```

### Phase 2 — Expand
- Add more active dataflows (ITGS, PPI_FD, BUS_IND, HSI_M, RES_DWELL_ST, etc.)
- Build metadata registry (cache codelists, dataflow versions, dimension schemas)
- Dashboard showing last-fetched status per dataflow
- Test Indicator API after obtaining key (may add release-day latency improvement)

### Phase 3 — Economists' Consumption Layer
- SQL/Parquet database served to RStudio, Python notebook, or Tableau
- Scheduled refresh with delta detection and revision alerting

---

## 10. Useful Existing Repos & Tools

- **BillyBodean/abs-data-api** — Python scripts with 20 preset queries for common indicators (CPI, LF, GDP, WPI). Good reference for dimension key patterns.
- **Bigred97/abs-mcp** — MCP server wrapping ABS Data API with plain-English filters for 10 curated dataflows. Good for agent access.
- **ajsinha/sajhamcpserver** — Another MCP wrapper with CPI/GDP/Labour Force tools.
- **ausdata.io** — 23 curated ABS datasets as single-call API; nice documentation standard.
- **Starwell** — Indexes 1,235 ABS datasets with series IDs.

---

## 11. Primary Sources

- **ABS API Landing Page:** [https://www.abs.gov.au/statistics/application-programming-interfaces-apis](https://www.abs.gov.au/statistics/application-programming-interfaces-apis) — (last updated 23/03/2022)
- **ABS Data API Troubleshooting:** [https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/troubleshooting](https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/troubleshooting) — Documents 30s timeout and 10MB gateway size limits
- **ABS Data API User Guide:** [https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide](https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide)
- **ABS API Updates:** [https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/updates](https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/updates) — CPI_Q reinstatement, CPI base period changes
- **CPI Methodology:** [https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/consumer-price-index-australia/latest-release#methodology](https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/consumer-price-index-australia/latest-release#methodology)
- **Labour Force Methodology:** [https://www.abs.gov.au/statistics/labour/employment-and-unemployment/labour-force-australia/latest-release#methodology](https://www.abs.gov.au/statistics/labour/employment-and-unemployment/labour-force-australia/latest-release#methodology)

### Test Request Log (executed 2026-11-23 UTC)

All relative to `https://data.api.abs.gov.au/rest/`:

| Endpoint | Result |
|---|---|
| `GET /data/ABS,CPI,2.0.0/all?format=csv&lastNObservations=4` | ✅ ~1.8M chars, M+Q rows |
| `GET /data/ABS,CPI,2.0.0/all?format=csv&startPeriod=2026-01&endPeriod=2026-03` | ✅ CPI data for Jan-Mar 2026 |
| `GET /data/ABS,CPI,2.0.0/3.30002.10.50.M?format=csv&startPeriod=2026-01&endPeriod=2026-01` | ✅ Single-series tested by probe |
| `GET /data/ABS,LF,1.0.0/all?format=csv&lastNObservations=4` | ✅ ~46K chars |
| `GET /data/ABS,LF,1.0.0/all?format=csv&startPeriod=2026-10&endPeriod=2026-10` | ✅ Oct 2026 data returned |
| `GET /data/ABS,ANA_AGG,1.0.0/M1.GPM.20.AUS.Q?format=csv&startPeriod=2026-Q1&endPeriod=2026-Q1` | ✅ Independent HTTP 200 data observation |
| `GET /data/ABS,WPI,1.2.0/all?format=csv&startPeriod=2026-Q1&endPeriod=2026-Q1` | ✅ Independent HTTP 200 data observation |
| `GET /data/ABS,ITGS,1.2.0/all?format=csv&startPeriod=2026-01&endPeriod=2026-01` | ✅ Independent HTTP 200 data observation |
| `GET /data/ABS,LF,1.0.0/all?...&updatedAfter=2026-11-01T00:00:00Z` | ⚠️ Empty result (data older than cutoff) |
| `GET /data/ABS,CPI_Q,1.0.0/all?format=csv&firstNObservations=2` | ✅ 2 quarterly rows |
| `GET /data/ABS,CPI_M,1.2.0/all?format=csv&firstNObservations=2` | ✅ 2 monthly indicator rows (historical) |
| `GET /data/ABS,RT,1.0.0/all?format=csv&firstNObservations=1` | ✅ Historical RT data (1982-04 onward) |
| `GET /data/ABS,RT,1.0.0/all?format=csv&startPeriod=2026-09&endPeriod=2026-09` | ✅ NoRecordsFound (ceased) |
| `GET /rest/dataflow/ABS/CPI?format=json-structure-2.0.0` | ✅ Ver 2.0.0, active |
| `GET /rest/dataflow/ABS/CPI_Q?format=json-structure-2.0.0` | ✅ Ver 1.0.0, active |
| `GET /rest/dataflow/ABS/CPI_M?format=json-structure-2.0.0` | ✅ CEASED metadata |
| `GET /rest/dataflow/ABS/RT?format=json-structure-2.0.0` | ✅ CEASED metadata |
| `GET /rest/dataflow/ABS/BOP?format=json-structure-2.0.0` | ✅ Ver 1.0.0 |
| `GET /rest/dataflow/ABS/QBIS?format=json-structure-2.0.0` | ✅ Ver 1.0.0 |
| `GET /rest/dataflow/ABS/CAPEX?format=json-structure-2.0.0` | ✅ Ver 1.0.0 |
| `GET /rest/dataflow/ABS/LABOUR_ACCT_Q?format=json-structure-2.0.0` | ✅ Ver 1.1 |
| `GET /rest/dataflow/ABS/BOP_H?format=json-structure-2.0.0` | ❌ 404 |
| `GET /rest/dataflow/ABS/BUSINESS_INDICATORS?format=json-structure-2.0.0` | ❌ 404 |
| `GET /rest/dataflow/ABS/CAPEX_H?format=json-structure-2.0.0` | ❌ 404 |
| `GET /rest/dataflow/ABS/CPI_MONTHLY?format=json-structure-2.0.0` | ❌ 404 |
| `GET /rest/dataflow/ABS/MBTI_H?format=json-structure-2.0.0` | ❌ 404 |

---

## 12. Tooling Setup

| Tool | Status | Notes |
|---|---|---|
| Claude Code CLI | ✅ Installed v2.1.330, authenticated via Pro account | `claude-sonnet-5-5` model pinned |
| Python | Available | Standard library + requests for API calls |
| Git | Available | Track project under `./ABSdataAPI` |
| Cron/journald | TBD | For scheduled polling |
