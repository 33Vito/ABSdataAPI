# Reconciling ABS Data API Outputs with Published ABS and RBA Tables

> Date: 2026-11-23\
> All live ABS Data API, ABS Excel, and RBA CSV/Excel probes executed: 2026-11-23 UTC\
> Status: First draft by Gemini, strengthened by Claude’s cross-review. Code examples ported to R on 2026-11-23 UTC.\
> Project: ABSdataAPI — PoC for RBA automated data ingestion

> **Code examples are in R.** The verification scripts in [§7](#7-reproducible-verification-example-r) and [§7.1](#71-second-pass-supplementary-checks) were ported to R on 2026-11-23. The port reproduces every published count; the parity checks are in [§7.2](#72-conversion-parity-checks-for-the-r-port).

> **Second-pass note:** I re-ran the §7 script unchanged and it reproduced every printed line (see [§7](#7-reproducible-verification-example-r)). I then ran independent checks aimed at the claims that drive the Phase 1 recommendations; the script for them is in [§7.1](#71-second-pass-supplementary-checks). Findings that qualify or correct the first draft are marked **[2nd pass]** where they appear. The most consequential are:
> - the `GOPITT` → `BOP` attribution rests on a single discriminating quarter;
> - `DECIMALS` is populated only in `LF`, so it can't be a general rounding rule;
> - a naive base-period invariant would raise 53 false alarms;
> - Farm/Non-farm codes exist in an ANA codelist but have no data;
> - the documented WPI key was malformed;
> - several rounding knife-edge cases exist.

> **Related:** this note complements the endpoint inventory in [ABS API research notes](abs-api-research.md)
> ([§4 CPI dataflows](abs-api-research.md#4-cpi-dataflows--corrected-status-verified-live-2026-11-23),
> [§5 RBA-core data coverage](abs-api-research.md#5-rba-core-data-coverage--tested-vs-candidate-dataflows),
> [§9 PoC architecture](abs-api-research.md#9-suggested-poc-architecture)), the institutional context in
> [central bank ingestion report](central-bank-ingestion/report.md#design-rationale-for-the-absdataapi-phase-1-pipeline)
> and [australia-rba.md](central-bank-ingestion/notes/australia-rba.md#5-rba-handling-of-abs-revisions-metadata-and-re-dissemination-in-statistical-tables),
> and the repository [evidence rules](../../README.md#evidence-rules).

---

## 1. Executive Summary & Why Reconciliation Is Needed

For ABSdataAPI to serve Reserve Bank of Australia (RBA) economists safely, returning HTTP 200 and non-empty SDMX rows from `https://data.api.abs.gov.au/rest/` is not enough. Economists validate numbers against two published table surfaces:

1. **ABS published Excel time-series spreadsheets** on `abs.gov.au` (e.g., Cat. No. `6401.0` CPI workbooks `640101.xlsx`, `640106.xlsx`, `6401017.xlsx`, `64010Appendix1a.xlsx`; Cat. No. `5206.0` National Accounts `5206001_Key_Aggregates.xlsx` and `5206024_Selected_Analytical_Series.xlsx`; Cat. No. `6202.0` Labour Force `62020001.xlsx`), keyed by classic 9–11 character **ABS Series IDs** (such as `A2325846C` or `A2304402X`).
2. **RBA Statistical Tables** on `rba.gov.au/statistics/tables/` (CSV/Excel tables such as `G1`, `G4`, `H1`, `H4`, and `H5`), keyed by **RBA Series IDs** (such as `GCPIAG`, `GCPIAGM`, `GGDPCVGDP`, `GOPITT`, `GWPIYP`, and `GLFSURSA`), stamped with `Source` (`ABS`, `ABS / RBA`, or `RBA`) and `Publication date`.

### Core findings from live reconciliation (2026-11-23 UTC)

- **Exact numerical parity holds with ABS Excel once series scope, units, and float formatting are aligned:** Across **`10,777`** populated observations directly audited between the ABS Data API and ABS Excel time-series spreadsheets (`7,920` in `ANA_AGG`, `693` in monthly `CPI`, `625` in quarterly `CPI`, and `1,539` in `CPI_Q`), **numeric discrepancies = `0`**.
- **Naive string or series-name matching fails immediately:** The Data API omits ABS Series IDs altogether, emits IEEE-754 clean strings compared to raw float strings inside `.xlsx` XML (`427` string-only diffs in `ANA_AGG` alone), emits `OBS_STATUS=m` blank sentinel rows, and—in Labour Force (`LF 1.0.0`)—reports **8 decimal places** (`4.86483265`) where ABS Excel stores **7** (`4.8648326`) and RBA Table H5 rounds to **1** (`4.9`), yielding **0% raw equality (`0 / 1,755`)** but **100% rounded equality (`1,755 / 1,755`)**.
- **Real metadata bugs, stale table notes, and coverage gaps exist in production data today:**
  - **CPI split & three live reference bases:** `CPI(2.0.0)` `FREQ=Q` contains only `Original` (`TSEST=10`, `Sep 2025 = 100`) and omits Trimmed Mean and YoY%, whereas `CPI_Q(1.0.0)` contains only `Seasonally Adjusted` (`TSEST=20`, `2011-12 = 100`, matching `64010Appendix1a.xlsx`) and omits `Original`. Meanwhile, **`18` seasonally adjusted monthly aggregate/group/underlying series in `CPI(2.0.0)` are anchored at `2024-04 = 100.0` while mislabelled `BASE_PERIOD=25` (`Sep 2025 = 100.0`)**.
  - **Cross-publication ABS divergence in Terms of Trade (`2010-Q4`):** `ANA_AGG(1.0.0)` and `5206001_Key_Aggregates.xlsx` report `96.0` (matching unrounded `5206.0` IPDs `96.0499`), whereas RBA Table H1 (`GOPITT`) reports `96.1`—matching **`ABS:BOP(1.0.0)` `4.130.20.Q` across all 268 quarters (`1959-Q3..2026-Q2`)**, even though `h01hist.xlsx` Notes attribute Table H1 exclusively to Cat. No. `5206.0`.
  - **Stale RBA H1 reference-year note:** `h01hist.xlsx` Notes still cite `2022/2023` for chain volume measures and Terms of Trade, whereas live `5206.0`, `ANA_AGG`, and `GOPITT` have already rolled forward to **`2023-24`** (`Original` `sum(M1.GPM) == sum(M3.GPM) = 2,680,025`, `diff = 0`).
  - **Pre-2002/2003 RBA G1 vs ABS `CPI_Q` divergence:** RBA G1 Trimmed Mean (`GCPIOCPMTMYP`) and Weighted Median (`GCPIOCPMWMYP`) YoY% match `CPI_Q(1.0.0)` and `64010Appendix1a.xlsx` 100% from `2001-Q3` / `2003-Q1` onward, but diverge on **`34/74`** (`1983-Q1..2001-Q2`, up to `0.2 pp`) and **`33/80`** (`1983-Q1..2002-Q4`, `0.1 pp`) earlier quarters despite `100%` QoQ% parity (`178/178`).
  - **API gaps vs Excel:** `18` Non-market sector series in `5206001_Key_Aggregates.xlsx` are missing from `ANA_AGG(1.0.0)`, and Farm/Non-farm GDP (`5206024`, RBA H1) is absent from all `ANA_*` Data API flows. **[2nd pass]** I confirmed this for all five `ANA_*` flows, including `ANA_SFD`, which the first pass didn't check. There is a trap: `ANA_IND_GVA`'s `CL_INDUSTRY_ANA` codelist *does* define `97` Farm and `98` Non-farm, but no data exist for them (see [§3, Discrepancy 3](#followed-discrepancy-3-national-accounts-coverage-gaps-in-the-data-api)).
- **[2nd pass] Exact parity depends on rounding rules the API doesn't fully describe:**
  - `DECIMALS` is populated only in `LF`. It is blank in `CPI`/`CPI_Q` and the column is absent from `ANA_AGG`, `BOP` and `WPI`. Comparisons against rounded RBA tables therefore need a precision configured per series.
  - At least 4 `LF` observations and 6 derived YoY values sit within 0.001 of a 1-d.p. rounding boundary, the same knife-edge mechanism behind the 2010-Q4 Terms of Trade gap.
  - Today's "0 diffs" results are robust, but a revision as small as 1e-5 could flip a published digit.

---

## 2. Structural Mechanics of Cross-Channel Linking

### Absence of ABS Series IDs and Verified Codelist Resolution

Probes across `format=csv`, `format=csvfilewithlabels`, `format=jsondata` (SDMX-JSON), and `format=structurespecificdata` (SDMX-XML) for `ABS:CPI(2.0.0)` and `ABS:ANA_AGG(1.0.0)` confirm that **classic ABS Series IDs (`A2325846C`, `A2304402X`, etc.) are not exposed in ABS Data API payloads**. Reconciliation therefore requires an explicit mapping key `(dataflow_id, version, sdmx_key)` resolved via SDMX codelists.

Contrary to the open note in [abs-api-research.md §8](abs-api-research.md#8-known-limitations--pitfalls), **codelist resolution endpoints are live and verified (2026-11-23 UTC)**:

| Mechanism | Tested Request | Verified Result |
|---|---|---|
| Inline labels in CSV | `GET /rest/data/ABS,ANA_AGG,1.0.0/M1.GPM.20.AUS.Q?format=csvfilewithlabels&startPeriod=2025-Q1&endPeriod=2026-Q3` | HTTP 200; pairs every coded column with its label column (`MEASURE`, `Measure`, `DATA_ITEM`, `Data Item`, etc.) |
| All dataflow codelists in one call | `GET /rest/dataflow/ABS/ANA_AGG/1.0.0?references=all&detail=referencepartial&format=json-structure-2.0.0` | HTTP 200; returns all 8 referenced codelists (`CL_ANA_AGG_ITEM` [18], `CL_ANA_AGG_MEASURE` [7], `CL_FREQ` [9], `CL_OBS_STATUS` [16], `CL_STATE` [10], `CL_TSEST` [5], `CL_UNIT_MEASURE` [88], `CL_UNIT_MULT` [31]) |
| Direct codelist lookup | `GET /rest/codelist/ABS/CL_CPI_INDEX/1.0.0?format=json-structure-2.0.0` | HTTP 200; returns `CL_CPI_INDEX` (`1.0.0`, 166 codes) |

### Publication Cadence: ABS `T` vs RBA `T+1`

Comparing release timestamps on `abs.gov.au` against `Publication date` header rows inside RBA CSV tables shows a consistent **T+1 business-day update cadence** across all four tested releases (consistent with [australia-rba.md §5](central-bank-ingestion/notes/australia-rba.md#5-rba-handling-of-abs-revisions-metadata-and-re-dissemination-in-statistical-tables)):

| Indicator & ABS Cat. No. | Reference Period | ABS Website Release (`11:30 am`) | RBA Table (`CSV`) | RBA `Publication date` Stamp | Observed Lag |
|---|---|---|---|---|---|
| CPI (`6401.0`) | September 2026 | `28/10/2026` | `G1` (`g1-data.csv`), `G4` (`g4-data.csv`) | `29-Oct-2026` | `T+1` |
| National Accounts (`5206.0`) | June 2026 | `02/09/2026` | `H1` (`h1-data.csv`) | `03-Sep-2026` | `T+1` |
| Wage Price Index (`6345.0`) | September 2026 | `18/11/2026` | `H4` (`h4-data.csv`) | `19-Nov-2026` | `T+1` |
| Labour Force (`6202.0`) | October 2026 | `19/11/2026` | `H5` (`h5-data.csv`) | `20-Nov-2026` | `T+1` |

**[2nd pass] Independent timing evidence and limits.** I re-confirmed the ABS release dates from each `latest-release` page: 28/10/2026, 2/09/2026, 18/11/2026 and 19/11/2026. I then read the RBA server's HTTP `Last-Modified` headers, which give a time of day as well as a date:

| RBA file | `Last-Modified` (UTC) | Canberra time | After ABS release |
|---|---|---|---|
| `g1-data.csv` | Thu 29 Oct 2026 05:11 | 16:11 AEDT, 29 Oct | Next day, afternoon |
| `g4-data.csv` | Thu 29 Oct 2026 05:09 | 16:09 AEDT, 29 Oct | Next day, afternoon |
| `h1-data.csv` | Thu 03 Sep 2026 01:13 | 11:13 AEST, 3 Sep | Next day, late morning |
| `h4-data.csv` | Thu 19 Nov 2026 01:41 | 12:41 AEDT, 19 Nov | Next day, midday |
| `h5-data.csv` | Fri 20 Nov 2026 00:16 | 11:16 AEDT, 20 Nov | Next day, late morning |

This agrees with the `Publication date` stamps. There are two limits:

1. There is **one release per table** (n = 1), so "consistent T+1" is an observation, not an established pattern.
2. `Last-Modified` records the *last* write. A same-day first post followed by a later rewrite can't be ruled out.

The RBA publishes no SLA for this lag ([australia-rba.md §5](central-bank-ingestion/notes/australia-rba.md#5-rba-handling-of-abs-revisions-metadata-and-re-dissemination-in-statistical-tables) also leaves the intra-day lag open). Phase 1 should therefore trigger Tier 3 on a change in `Publication date`, not on a calendar offset.

### Date Serials, Spreadsheet Layout, and Trailing Placeholders

- **Period representations:** SDMX uses `YYYY-QN` and `YYYY-MM`; ABS `.xlsx` `Data1` sheets store Excel 1900 date serials (`base 1899-12-30`), using quarter-start/month-start dates (e.g., `2026-06-01` for `2026-Q2` in `5206001_Key_Aggregates.xlsx`, `2026-09-01` for `2026-Q3` in `6401017.xlsx`) with ABS Series IDs in Row 10 (0-based index `9`) and observations from Row 11 (`10`); RBA CSVs use period-end `DD/MM/YYYY` dates (`30/06/2026`, `30/09/2026`).
- **Placeholder rows:** RBA CSVs (`g1-data.csv`, `g4-data.csv`, `h1-data.csv`) are padded to `9,998` lines with blank rows and end with **three empty future-period date rows** (e.g., `31/12/2026`, `31/03/2027`, `30/06/2027` in `G1`). Similarly, `64010Appendix1a.xlsx` contains **four empty future-quarter rows** (`2026-Q4` to `2027-Q3`) where forward seasonal factors sit. Parsers must ignore rows with empty observation cells.

---

## 3. Case Study 1 — National Accounts / GDP (`ANA_AGG` vs `5206.0` Excel vs RBA Table H1)

### Reconciliation Summary

| Comparison | API Key / Transform | Published Target | Overlap | Exact / Numeric Match | Discrepancies & Cause |
|---|---|---|---|---|---|
| `ANA_AGG(1.0.0)` SA vs `5206001_Key_Aggregates.xlsx` SA (exhaustive) | `..20.AUS.Q` (36 of 42 SA series) | `5206001` `Data1` (`Seasonally Adjusted`) | `7,920` obs (`1959-Q3..2026-Q2`) | **`7,920 / 7,920` (`0` numeric diffs)** | `427` float-string XML diffs (e.g., `-1.1` vs `-1.1000000000000001`); `16` API `OBS_STATUS=m` blank rows; `6` SA non-market series absent from API |
| Real GDP CVM (`$m`) | `M1.GPM.20.AUS.Q` | `5206001` `A2304402X` / RBA H1 `GGDPCVGDP` | `268` qtrs (`1959-Q3..2026-Q2`) | **`268 / 268`** (`2026-Q2 = 699,461`) | None |
| Nominal GDP (`$m`) | `M3.GPM.20.AUS.Q` | `5206001` `A2304418T` / RBA H1 `GGDPECCPGDP` | `268` qtrs (`1959-Q3..2026-Q2`) | **`268 / 268`** (`2026-Q2 = 742,502`) | None |
| Real GDI (`$m`) | `M1.RDI.20.AUS.Q` | `5206001` `A2304410X` / RBA H1 `GGDPCVRGDI` | `268` qtrs (`1959-Q3..2026-Q2`) | **`268 / 268`** (`2026-Q2 = 690,389`) | None |
| Real GDP QoQ % | `M2.GPM.20.AUS.Q` | `5206001` `A2304370T` | `267` qtrs (`1959-Q4..2026-Q2`) | **`267 / 267`** (`0` numeric diffs) | API emits `1959-Q3` as blank (`OBS_STATUS=m`); `21` XML float-string diffs |
| Derived YoY % (GDP, GDP/cap, Real GDI, Nom GDP) | `round((v_t / v_{t-4} - 1) * 100, 1)` from `M1`/`M3` | RBA H1 `GGDPCVGDPY`, `GGDPCVGDPFY`, `GGDPCVRGDIY`, `GGDPECCPGDPY` | `1,000` qtrs total (`264+208+264+264`) | **`1,000 / 1,000`** (`0` diffs) | Justified deterministic transformation (`ANA_AGG` lacks YoY%) |
| Terms of Trade (Index) | `M5.TTR.20.AUS.Q` | `5206001` `A2304200A` / RBA H1 `GOPITT` | `268` qtrs (`1959-Q3..2026-Q2`) | `268/268` vs Excel; **`267/268` vs H1** | **`2010-Q4`:** `ANA_AGG`/`5206001` = `96.0` vs RBA H1 = `96.1` (**resolved via `BOP 1.0.0`**) |
| Terms of Trade (via BOP) | `BOP(1.0.0)` `4.130.20.Q` | RBA H1 `GOPITT` | `268` qtrs (`1959-Q3..2026-Q2`) | **`268 / 268` (`0` diffs)** | Inter-publication ABS difference (`5302.0` vs `5206.0`) |
| Non-farm & Farm GDP CVM | *Not in any `ANA_*` API flow* | `5206024` `A2302589X`, `A2302592L` vs RBA H1 `GGDPCVGDPNF`, `GGDPCVGDPF` | `268` / `164` qtrs | `268/268`, `164/164` (Excel vs H1) | Structural API coverage gap |

### Followed Discrepancy 1: `2010-Q4` Terms of Trade (`96.0` vs `96.1`) and Imprecise RBA Attribution

Comparing `ANA_AGG(1.0.0)` `M5.TTR.20.AUS.Q` against RBA Table H1 `GOPITT` across all 268 quarters (`1959-Q3..2026-Q2`) matches on 267 quarters and fails on exactly one observation: **`2010-Q4`**, where `ANA_AGG` and `5206001_Key_Aggregates.xlsx` (`A2304200A`) report **`96.0`**, while RBA H1 (`GOPITT`) reports **`96.1`**.

Tracing the calculation inside ABS Cat. No. `5206.0` explains why `2010-Q4` sits on a knife-edge:
- Dividing published `5206003` current-price exports/imports by `5206002` chain-volume exports/imports gives unrounded implicit price deflators (IPDs) of `ExpIPD = 66.5613` and `ImpIPD = 69.2987`, whose ratio is `66.5613 / 69.2987 * 100 = 96.0499`, rounding to **`96.0`** (just `0.0001` below `96.05`).
- Dividing the rounded 1-d.p. IPDs published in `5206005_Expenditure_Implicit_Price_Deflators.xlsx` (`66.6 / 69.3 * 100 = 96.1039`) yields `96.1`, but `5206.0` does **not** use rounded IPDs—in `2011-Q1`, rounded `70.8 / 69.7 * 100 = 101.578` (`101.6`), whereas both ABS `5206.0` and RBA H1 report **`101.7`** (unrounded `101.6874`).
- **Actual source of `96.1`:** Querying `ABS:BOP(1.0.0)` `4.130.20.Q` (Cat. No. `5302.0` *Balance of Payments*, Terms of Trade — Goods and services, Seasonally Adjusted) returns **`96.1`** in `2010-Q4` and matches RBA `GOPITT` on **all `268 / 268` quarters (`0` discrepancies)**, including being current through `2026-Q2` despite carrying the `NonProductionDataflow` flag noted in [abs-api-research.md §5](abs-api-research.md#5-rba-core-data-coverage--tested-vs-candidate-dataflows).

This exposes both an inter-publication `0.1`-point difference between ABS `5206.0` and `5302.0` and an imprecise source note in RBA `h01hist.xlsx`, whose `Notes` tab states *"The data for this table are from ABS Cat No 5206.0"* without noting that `GOPITT` reflects `5302.0` (`BOP`).

**[2nd pass] How strong is the `BOP` attribution?** Comparing the two ABS API series directly (`ANA_AGG` `M5.TTR.20.AUS.Q` vs `BOP` `4.130.20.Q`) shows they differ in **only one of 268 quarters, 2010-Q4** (`96.0` vs `96.1`). Both "`GOPITT` = `BOP`" (268/268) and "`GOPITT` = `ANA_AGG`" (267/268) therefore rest on that single discriminating observation, and that observation is a rounding knife-edge (unrounded `96.0499`).

The evidence is *consistent* with the RBA sourcing `GOPITT` from `5302.0`, but other explanations remain open. One is that the RBA holds an older `5206.0` vintage in which 2010-Q4 rounded to `96.1`; today's data can't test this. The RBA note, which names only `5206.0`, is direct evidence against the `BOP` attribution, while the value match is indirect evidence for it.

I've weakened the Phase 1 action accordingly (taxonomy B4, Tier 3). Compare `GOPITT` against **both** series and record which one matches. A future quarter where `ANA_AGG` and `BOP` differ would settle the question. Also note that `BOP` has no `TSEST=10` Terms of Trade series (`4.130.10.Q` returns HTTP 404), so only the SA series can be compared.

### Followed Discrepancy 2: Stale CVM Reference Year in RBA H1 Notes & Absent `BASE_PERIOD` in `ANA_AGG`

RBA `h01hist.xlsx` (`Notes` tab) currently states *"Chain volume measures are referenced to 2022/2023 values"* and *"referenced to 100.0 for the financial year 2022/2023"* for Terms of Trade. Both the live ABS `5206.0` `jun-2026` release page (*"The reference year for chain volume measures is 2023-24"*) and an empirical accounting check on `ANA_AGG(1.0.0)` prove **the live reference year has already moved to `2023-24`**:

| Financial Year | `Original` (`TSEST=10`) Sum of CVM GDP (`M1.GPM`) | `Original` (`TSEST=10`) Sum of Current-Price GDP (`M3.GPM`) | Difference (`M1 - M3`) | Terms of Trade (`M5.TTR`) 4-Qtr Mean (`Orig` / `SA`) |
|---|---|---|---|---|
| `2021-22` | `2,552,249` | `2,336,902` | `+215,347` | `106.925` / `106.925` |
| `2022-23` | `2,643,716` | `2,577,014` | `+66,702` | `106.825` / `106.725` |
| **`2023-24`** | **`2,680,025`** | **`2,680,025`** | **`0`** | **`100.025` / `100.025`** |
| `2024-25` | `2,715,042` | `2,777,012` | `-61,970` | `95.925` / `95.900` |
| `2025-26` | `2,779,155` | `2,927,146` | `-147,991` | `95.325` / `95.400` |

Note two nuances for automated checks: (1) this identity holds **strictly on `Original` (`TSEST=10`)** (`SA` has a residual `-174` discrepancy in `2023-24` due to independent seasonal adjustment), and (2) `ANA_AGG(1.0.0)` exposes **no `BASE_PERIOD` column or attribute**, so inferring the reference year dynamically from `abs(sum(M1) - sum(M3))` on `TSEST=10` is the only programmatic way to detect annual National Accounts reference-year roll-forwards (the next annual roll-forward to `2024-25` is expected with the September 2026 release scheduled for `2/12/2026`).

### Followed Discrepancy 3: National Accounts Coverage Gaps in the Data API

- **`18` series missing from `ANA_AGG(1.0.0)` vs `5206001_Key_Aggregates.xlsx`:** `ANA_AGG(1.0.0)` contains `108` series (`36` per `TSEST`: `10`, `20`, `30`), whereas `5206001_Key_Aggregates.xlsx` contains `126` (`42` per adjustment type). The 6 missing measures per `TSEST` are all **non-market sector** series (SA Series IDs: `A130272207A`, `A130272201L`, `A130272209F`, `A130272203T`, `A130272211T`, `A130272205W`).
- **Farm / Non-farm GDP absent from all `ANA_*` API flows:** RBA Table H1's `GGDPCVGDPNF` (`268/268`), `GGDPCVGDPNFY` (`264/264`), and `GGDPCVGDPF` (`164/164`, `1985-Q3..2026-Q2`) reconcile 100% against ABS `5206024_Selected_Analytical_Series.xlsx` (`A2302589X`, `A2302592L`), but live queries of `ANA_AGG`, `ANA_EXP`, `ANA_INC`, and `ANA_IND_GVA` confirm none of those dataflows publish Farm or Non-farm GDP levels (`ANA_EXP` has only Farm/Non-farm *Changes in inventories*).
- **[2nd pass] The codelist suggests the series exist, but no data do.** The full dataflow list contains five `ANA_*` flows; the fifth, `ANA_SFD`, has no farm codes. Its codelists are about state final demand.
  - In `ANA_IND_GVA(1.0.0)` (dimensions `MEASURE.DATA_ITEM.SECTOR.TSEST.INDUSTRY.REGION.FREQ`), `CL_INDUSTRY_ANA` defines **`97` Farm and `98` Non-farm**, and `CL_ANA_INDGVA_ITEM` defines `GPM` "Gross domestic product".
  - Even so, `GET /rest/data/ABS,ANA_IND_GVA,1.0.0/....97+98..Q` returns **`404 NoRecordsFound`**.
  - `GET /rest/availableconstraint/ABS,ANA_IND_GVA,1.0.0/all/all/INDUSTRY` lists only `A`–`S` and `TOTAL` as having data.
  - `GPM_AGR` ("Gross value added - Agriculture", industry `A`) is industry GVA, not the ABS Farm GDP concept, so it is not a substitute without economist sign-off.
  - **Implication:** series discovery must use `availableconstraint` (or a bounded data probe), not codelist contents. A configuration built from codelists alone would pass review and then return 404 in production.

---

## 4. Case Study 2 — Consumer Price Index (`CPI` 2.0.0 & `CPI_Q` 1.0.0 vs `6401.0` Excel vs RBA Tables G1 & G4)

### Reconciliation Summary

| Comparison | API Key / Transform | Published Target | Overlap | Exact / Numeric Match | Discrepancies & Cause |
|---|---|---|---|---|---|
| Monthly CPI `640101.xlsx` (all 27 series: Australia + 8 cities) | `CPI(2.0.0)` `1+2+3.10001.10..M` | `640101.xlsx` `Data1` | `693` obs (`2024-04..2026-09`) | **`693 / 693` (`0` num diffs)** | `38` XML float-string diffs only |
| Monthly CPI vs RBA Table G4 (`g4-data.csv`) | `1.10001.10.50.M`, `3.10001.10.50.M`, `2.10001.10.50.M`, `2.999901.20.50.M` | `GCPIAGM`, `GCPIAGYPM`, `GCPIAGMP`, `GCPIAGSAMP` | `106` obs (`30+18+29+29`) | **`106 / 106` (`0` diffs)** | None; recomputed YoY from `1.10001` also matches `3.10001` `18/18` |
| Quarterly Original CPI vs `6401017.xlsx` | `CPI(2.0.0)` `1.10001.10.50.Q`, `2.10001.10.50.Q` | `6401017` `A2325846C`, `A2325850V` | `625` obs (`1948-Q3..2026-Q3`) | **`625 / 625` (`0` num diffs)** | `38` XML float-string diffs on `A2325846C`; `6401017` lacks YoY% |
| Quarterly Original CPI vs RBA Table G1 (`g1-data.csv`) | `1.10001.10.50.Q`, `2.10001.10.50.Q`, derived YoY% | `GCPIAG`, `GCPIAGQP`, `GCPIAGYP` | `934` obs (`313+312+309`) | **`934 / 934` (`0` diffs)** | RBA G1 splices `105` pre-ABS C-Series RPI quarters (`1922-Q2..1948-Q2`) |
| Quarterly SA & Underlying CPI vs `64010Appendix1a.xlsx` | `CPI_Q(1.0.0)` `1+2+3.999901+999902+999903.20.50.Q` | `64010Appendix1a` (`A3604503X..A3604511X`) | `1,539` obs (`1982-Q1..2026-Q3`) | **`1,539 / 1,539` (`0` num diffs)** | `357` XML float-string diffs; `4` future empty rows in Excel |
| Quarterly SA & Underlying CPI vs RBA Table G1 (`g1-data.csv`) | `CPI_Q(1.0.0)` QoQ% (`2.*`) & YoY% (`3.*`) | `GCPIAGSAQP`, `GCPIOCPMTMQP`, `GCPIOCPMWMQP`, `GCPIOCPMTMYP`, `GCPIOCPMWMYP` | `865` obs (`159+178+178+175+175`) | All QoQ (`515/515`); YoY post-`2001Q3`/`2003Q1` (`196/196`) | **`34/74`** Trimmed Mean YoY diffs (`1983-Q1..2001-Q2`); **`33/80`** Weighted Median YoY diffs (`1983-Q1..2002-Q4`); `GCPIAGSAQP` has `19` pre-1987 RBA-computed qtrs (`1982-Q2..1986-Q4`) |

### Followed Discrepancy 4: Three Disjoint CPI Reference Bases Across `CPI(2.0.0)` and `CPI_Q(1.0.0)`

Following ABS's November 2025 transition to the complete monthly CPI and the April 2026 reinstatement of `CPI_Q` (per [ABS API Updates](https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/updates)), **`CPI(2.0.0)` `FREQ=Q` and `CPI_Q(1.0.0)` are disjoint in scope, seasonal adjustment, and reference base**—not interchangeable substitutes:

| API Slice | `TSEST` | `MEASURE` Available | Actual Index Reference Base | `BASE_PERIOD` Field | Sample Value (`2026-09` / `2026-Q3`) | Matching ABS Excel |
|---|---|---|---|---|---|---|
| `CPI(2.0.0)` `FREQ=M`, `TSEST=10` | `10` (Original) | `1`–`7` (incl. YoY% `3`, contributions `4`–`7`) | **`Sep 2025 = 100.0`** (`158/158` verified) | `25` (`Sep 2025 = 100.0`) | `104.34` (`10001`) | `640101.xlsx` (`A130393720C`), `640106.xlsx` |
| `CPI(2.0.0)` `FREQ=M`, `TSEST=20` (18 aggregate/group/underlying series) | `20` (SA) | `1`–`3` (for `999901/02/03`, etc.) | **`Apr 2024 = 100.0`** (`2025-09 = 104.00..105.05`) | **`25` (BUG — mislabelled!)** | `108.53` (`999901`), `108.3911` (`999902`), `108.6439` (`999903`) | `640106.xlsx`, `640107.xlsx` (`A130607789R`, `A130400381L`, `A130400612K`) |
| `CPI(2.0.0)` `FREQ=M`, `TSEST=20` (87 expenditure classes) | `20` (SA) | `1`–`3` | **`Sep 2025 = 100.0`** (`34` non-seasonal = `100.0` at `2025-09`) | `25` (`Sep 2025 = 100.0`) | e.g., `106.67` (`40085`) | `640107.xlsx` (history from `2017-09`) |
| `CPI(2.0.0)` `FREQ=Q` (`412` rows in `2026-Q3`) | **`10` ONLY** | `1`, `2`, `4` (**no YoY% `3`**) | **`Sep 2025 = 100.0`** | `25` (`Sep 2025 = 100.0`) | `103.64` (`10001`) | `6401017.xlsx` (`A2325846C`), `6401018.xlsx` |
| `CPI_Q(1.0.0)` (`96` rows in `2026-Q3`, `TSEST=10` → HTTP 404) | **`20` ONLY** | `1`, `2`, `3` (incl. `999901/02/03`) | **`2011-12 = 100.0`** (verified 4-qtr mean `= 100.0`) | *Column absent* | `149.2` (`999901`), `147.5875` (`999902`), `146.2586` (`999903`) | `64010Appendix1a.xlsx` (pre-Oct 2025 basis) |

### Followed Discrepancy 5: `BASE_PERIOD=25` Metadata Bug in `CPI(2.0.0)`

An exhaustive audit of all `105` seasonally adjusted monthly index series (`CPI(2.0.0)`, `1..20.50.M`) shows that **every single row carries `BASE_PERIOD=25` (`Sep 2025 = 100.0`)**, yet **`18` series—specifically All groups SA (`999901`), Trimmed Mean (`999902`), Weighted Median (`999903`), Tradables (`102675`), Non-tradables (`102676`), Goods (`104101`), Services (`104104`), and all 11 CPI Groups (`20001`–`20006`, `115486`, `115488`, `115489`, `115493`, `126670`)—actually equal `100.0` in `2024-04`** (matching `640106.xlsx` and `640107.xlsx`, where those series begin in April 2024). In `2025-09`, `All groups CPI, seasonally adjusted` (`999901`) is `104.0`, `Trimmed Mean` (`999902`) is `104.6361`, and `Weighted Median` (`999903`) is `105.0466`. Comparing `All groups Original` (`104.34` in `2026-09`) with `All groups SA` (`108.53` in `2026-09`) without correcting for the `2024-04 = 100` base produces a spurious `+4.19` index-point gap, even though their `2.999901` MoM% (`0.6%`) and `3.999901` YoY% (`4.4%`) rates remain internally consistent.

**[2nd pass] A complete classification, and why the obvious invariant over-fires.** I classified every `MEASURE=1` monthly index series in `CPI(2.0.0)` (`1...50.M`, 2024-04..2025-09) by the month in which it equals exactly `100`. Every row carries `BASE_PERIOD=25`.

| `TSEST` | Equals 100 at | Series | Interpretation |
|---|---|---|---|
| `10` Original | 2025-09 | 158 | Consistent with `BASE_PERIOD=25` |
| `20` SA | 2025-09 | 34 | Classes with no seasonality: SA equals original |
| `20` SA | **2024-04** | **18** | **Mislabelled.** Confirms the first-pass count |
| `20` SA | neither | **53** | **Legitimate.** The original equals 100 at 2025-09 and the SA value differs (e.g. `40034`: SA `100.79`, original `100.0`) |

A Tier 1 rule of the form "every `BASE_PERIOD=25` index must equal 100 at 2025-09" would raise **53 false alarms**. Seasonal adjustment is applied after referencing, so SA series don't have to equal 100 in the base month. The test that separates the cases is narrower:

- Flag any index series that equals exactly `100` in a period *other* than its declared base.
- Assert equality to 100 at the base only for `TSEST=10`.

The same mislabelling was found independently in the [series continuity draft](abs-series-continuity.md) (finding F6) for `999901`–`999903`.

### Followed Discrepancy 6: Pre-2002/2003 Underlying Inflation YoY% Divergence (`CPI_Q` vs RBA G1)

RBA Table G1 attributes `GCPIOCPMTMYP` (Trimmed Mean YoY%) and `GCPIOCPMWMYP` (Weighted Median YoY%) to `Source: ABS / RBA`. Reconciling `CPI_Q(1.0.0)` against `64010Appendix1a.xlsx` and RBA G1 reveals a sharp historical boundary:
- **Quarterly % changes (`QoQ%`):** `CPI_Q(1.0.0)` matches `64010Appendix1a.xlsx` and RBA G1 on **`178 / 178`** quarters (`1982-Q2..2026-Q3`) for both Trimmed Mean (`GCPIOCPMTMQP`) and Weighted Median (`GCPIOCPMWMQP`), **0 discrepancies**.
- **Year-ended % changes (`YoY%`):** `CPI_Q(1.0.0)` matches `64010Appendix1a.xlsx` on all **`175 / 175`** quarters (`1983-Q1..2026-Q3`), and matches RBA G1 on **`101 / 101`** quarters from **`2001-Q3..2026-Q3`** (Trimmed Mean) and **`95 / 95`** quarters from **`2003-Q1..2026-Q3`** (Weighted Median). However, before those cutoffs:
  - Trimmed Mean diverges on **`34 / 74`** quarters (`1983-Q1..2001-Q2`, up to **`0.2 pp`**, e.g. `1983-Q2`: ABS `9.8` vs RBA `10.0`; `1983-Q3`: ABS `8.3` vs RBA `8.5`).
  - Weighted Median diverges on **`33 / 80`** quarters (`1983-Q1..2002-Q4`, **`0.1 pp`**).
- **Verified algebraic diagnostic:** Across all `34` and `33` diverging quarters, `CPI_Q` YoY% equals `round((I_t / I_{t-4} - 1) * 100, 1)` computed from ABS's 4-decimal underlying index (`34/34`, `33/33`), whereas RBA G1 YoY% matches neither that index ratio (`0/34`, `0/33`) nor the compounded product of the four published 1-d.p. QoQ% changes (`0/34`, `0/33`). RBA `g01hist.xlsx` Notes document pre-September 1998 interest-charge exclusions, 1999–2000 tax adjustments, and Roberts (2005) / ABS (2011) methodology; the exact unrounded historical series or formula behind the pre-2002 RBA YoY% cells is not publicly exposed in `g01hist.xlsx`.

### Followed Discrepancy 7: `Source: RBA` Analytical Series vs ABS `CPI(2.0.0)`

RBA Table G1 tags 10 columns as `Source: RBA` (ex-interest/tax/volatile, tradables, and non-tradables). Two concrete live checks show why **pipelines must never try to reconcile `Source: RBA` analytical series against raw ABS API series**:
1. `GCPIEITCYP` (All groups ex-interest & 1999–2000 tax changes) diverges from headline `GCPIAGYP` across **`59` quarters (`1987-Q4..2005-Q3`)**.
2. `GCPITIYP` (Tradables YoY, quarterly-collection basis + RBA tax adjustments) differs dramatically from ABS `CPI(2.0.0)` monthly Tradables (`102675`, `3.102675.10.50.M`, end-of-quarter month; note `102675` and `102676` do not exist at `FREQ=Q` in `CPI(2.0.0)`): in `2026-Q1`, RBA reports `2.6%` vs ABS March `4.5%` (**`1.9 pp`** gap); in `2026-Q2`, `2.4%` vs `1.5%`; in `2026-Q3`, `3.1%` vs `4.0%` (**`0.9 pp`** gap).

---

## 5. Cross-Check on Remaining Phase 1 Dataflows (`LF` & `WPI`)

Extending the check to the remaining Phase 1 dataflows in [abs-api-research.md §9](abs-api-research.md#9-suggested-poc-architecture) confirms the same reconciliation patterns hold across ABS statistics:

- **`LF(1.0.0)` vs `62020001.xlsx` vs RBA Table H5 (`h5-data.csv`):**
  - On `2026-10` Seasonally Adjusted Unemployment Rate (`M13.3.1599.20.AUS.M`), **`LF(1.0.0)` emits 8 decimal places (`4.86483265`)** alongside `DECIMALS=1`, **`62020001.xlsx` (`A84423050A`) stores 7 decimal places (`4.8648326`, serialized as `4.8648325999999997`)**, and **RBA Table H5 (`GLFSURSA`) publishes 1 decimal place (`4.9`)**.
  - Across all **`585` months (`1978-02..2026-10`)** for Employment (`M3` vs `GLFSEPTSA`), Participation Rate (`M12` vs `GLFSPRSA`), and Unemployment Rate (`M13` vs `GLFSURSA`), unrounded float comparison matches **`0 / 1,755` (`0%`)**, whereas rounding API values to `DECIMALS=1` (`ROUND_HALF_UP` or round-half-to-even) matches **`1,755 / 1,755` (`100.0%`)**.
  - **[2nd pass] Rounding mode is irrelevant here; distance to the boundary is what matters.** Half-up and half-even disagree on 0 of 1,755 values, because no 8-d.p. value lies exactly on a `.x5` tie. Four values lie within 0.001 of one, though:

    | Measure | Month | API value |
    |---|---|---|
    | `M12` | 1982-04 | `60.85001439` |
    | `M12` | 2016-05 | `64.84999226` |
    | `M12` | 2024-10 | `67.04993198` |
    | `M13` | 1985-11 | `7.95005315` |

    A seasonal re-estimation that moved `2016-05` participation up by just 8e-6 would flip the RBA's published `64.8` to `64.9`. A canary mismatch of exactly one unit in the last published digit, where the unrounded value lies near a boundary, should be classified as **vintage/rounding noise, not a comparability failure**.
  - **[2nd pass] `DECIMALS` is an `LF` peculiarity, not a general rule.** Of the six Phase 1 flows probed, only `LF` populates `DECIMALS` (`1`):

    | Flow | `DECIMALS` |
    |---|---|
    | `LF(1.0.0)` | `1` |
    | `CPI(2.0.0)`, `CPI_Q(1.0.0)` | Column present, blank |
    | `ANA_AGG(1.0.0)`, `BOP(1.0.0)`, `WPI(1.2.0)` | Column absent |

    The `CPI_Q` trimmed mean index is delivered at 4 d.p. (`147.5875`), and `ANA_AGG`, `BOP` and `WPI` arrive already rounded to 1 d.p. The "honour `DECIMALS`" transform therefore applies to `LF` only. Other canaries need the published precision stated explicitly in config.
- **`WPI(1.2.0)` vs RBA Table H4 (`h4-data.csv`):**
  - **[2nd pass] Key corrected.** The first draft wrote `THRPEB.7.TOT.20.AUS.Q`, which has 6 positions; `WPI(1.2.0)` has 7 (`MEASURE.INDEX.SECTOR.INDUSTRY.TSEST.REGION.FREQ`), and the 6-position key returns HTTP 403 *"Not enough key values in query, expecting 7 got 6"*. The working keys are `2.THRPEB.7.TOT.20.AUS.Q` (QoQ%) and `3.THRPEB.7.TOT.20.AUS.Q` (YoY%). With them I re-ran the comparison and **confirmed the 229/229 result** below (116 + 113 quarters, 0 diffs). `WPI(1.2.0)` also carries an empty `BASE_PERIOD` column and no `DECIMALS` column.
  - `WPI(1.2.0)` `{2|3}.THRPEB.7.TOT.20.AUS.Q` matches RBA Table H4 cleanly with **zero rounding transforms required**: `GWPIQP` (`MEASURE=2`, `116/116`, `1997-Q4..2026-Q3`) and `GWPIYP` (`MEASURE=3`, `113/113`, `1998-Q3..2026-Q3`) match **`229 / 229` (`0` diffs)**. (Note: [abs-api-research.md §9](abs-api-research.md#9-suggested-poc-architecture) writes `WPI(1.0.0)` in the architecture diagram, whereas [§5](abs-api-research.md#5-rba-core-data-coverage--tested-vs-candidate-dataflows) and live probes verify `WPI(1.2.0)`.)

---

## 6. Decision Taxonomy: Justified Transformations vs Comparability Problems

| Category | Verified Mechanism / Example | Appropriate Pipeline Action |
|---|---|---|
| **A. Justified / Harmless Transformations (normalize before comparing)** | | |
| 1. Date & placeholder formatting | SDMX `YYYY-QN` / `YYYY-MM` vs Excel serials (`1899-12-30`) vs RBA `DD/MM/YYYY`; `9,998`-row padded RBA CSVs & `4` future rows in `64010Appendix1a` | Parse dates via explicit period normalizer; strip empty observation cells before aligning |
| 2. Excel XML IEEE-754 float serialization | `427` SA obs in `5206001` (`-1.1000000000000001` vs `-1.1`), `38` in `640101`, `357` in `64010Appendix1a`, `5e-8` in `62020001` | Never compare raw strings; compare numerics with `abs(a - b) <= 1e-6` or quantize to published precision |
| 3. SDMX `OBS_STATUS=m` blank rows | `16` SA rows in `ANA_AGG` (e.g., `1959-Q3` `M2.GPM`) have empty `OBS_VALUE` | Exclude `OBS_STATUS=m` empty rows before completeness and reconciliation checks |
| 4. Display precision rounding (`DECIMALS`) | `LF(1.0.0)` emits 8 d.p., `62020001.xlsx` stores 7 d.p., RBA H5 publishes 1 d.p. (`1,755/1,755` match after rounding) | Honor SDMX `DECIMALS` metadata when checking RBA tables: round the API value string with exact decimal half-up rounding to `DECIMALS` places (`round_half_up()` in [§7](#7-reproducible-verification-example-r)), not the parsed double. **[2nd pass]** `DECIMALS` is populated only in `LF`. For every other canary, set `published_decimals` explicitly in config and treat `DECIMALS` as a cross-check, not the source of truth |
| 5. Deterministic derived YoY% growth | `ANA_AGG` lacks YoY%; `CPI(2.0.0) Q` lacks YoY%; recomputing `round((v_t/v_{t-4}-1)*100, 1)` matches `1,000/1,000` in H1 & `309/309` in G1 | Apply explicit `yoy_pct` transform in canary config rather than treating absent YoY% as a mismatch |
| 6. `T+1` RBA publication lag | All 4 audited releases (`CPI`, `ANA`, `WPI`, `LF`) update RBA CSVs 1 business day after ABS 11:30 am release | Compare overlap `<= min(max_api, max_rba)` and alert only if RBA lag exceeds `T+2` business days |
| 7. Pre-ABS historical RBA splices | G1 pre-`1948-Q3` C-Series (`105` qtrs); G1 pre-`1987-Q1` SA (`19` qtrs); G1 pre-`2001Q3`/`2003Q1` underlying YoY% | Enforce `valid_from` bounds per series in reconciliation config |
| 8. **[2nd pass]** Rounding knife-edges | 4 `LF` values within 0.001 of a 1-d.p. boundary (e.g. `64.84999226`); 6 derived YoY values within 0.0005 pp (e.g. GDP `2008-Q1` `3.34985`); the 2010-Q4 ToT (`96.0499`) | If a diff is exactly one unit in the last published digit and the unrounded value is within ~1e-3 of a boundary, log it as `rounding_edge` rather than failing |
| 9. **[2nd pass]** Release-day vintage skew | On day `T`, ABS Excel holds the new vintage, including revised history, while the Data API may still serve the previous one. The 2026-11-23 audit saw no skew because it ran days after each release | If the Excel file's latest period is later than the API's, classify all diffs as `api_not_yet_updated` and re-run later. Only same-latest-period diffs count as mismatches |
| **B. Genuine Comparability Problems, Coverage Gaps & Metadata Defects (flag, fail, or exclude)** | | |
| 1. `BASE_PERIOD` metadata bug in API | `CPI(2.0.0)` labels `18` `TSEST=20` monthly series anchored at `2024-04 = 100` as `BASE_PERIOD=25` (`Sep 2025 = 100`) | Run base-period invariant test (`2025-09 == 100` vs `2024-04 == 100`); override/quarantine affected `BASE_PERIOD` metadata. **[2nd pass]** Assert "= 100 at base" only for `TSEST=10`, and flag series equal to exactly 100 at a *different* period. Otherwise 53 legitimate SA series are flagged |
| 2. Disjoint dataflows & incompatible bases | `CPI(2.0.0) Q` (`TSEST=10`, `Sep 2025=100`) vs `CPI_Q(1.0.0)` (`TSEST=20`, `2011-12=100`, no `BASE_PERIOD` column) | Never treat `CPI_Q` as a general quarterly replacement for `CPI(2.0.0) Q`; ingest both and keep bases tagged separately |
| 3. Conceptual / methodological divergence | `Source: RBA` columns in G1 (`GCPITIYP` `1.9 pp` off monthly ABS Tradables in `2026-Q1`; `GCPIEITCYP` `59` diffs) | Exclude `Source: RBA` columns from ABS API equality checks; ingest directly from RBA G1 if economists need them |
| 4. Cross-publication ABS divergence & imprecise RBA note | `2010-Q4` Terms of Trade = `96.0` in `5206.0`/`ANA_AGG` vs `96.1` in `5302.0`/`BOP` & RBA H1 `GOPITT` (note cites only `5206.0`) | Map `GOPITT` canary to `ABS:BOP(1.0.0)` `4.130.20.Q`, not `ANA_AGG`, and document `5206.0` vs `5302.0`. **[2nd pass]** The attribution rests on 1 of 268 quarters, so compare against **both** and record which matches. Mark it `inferred`, not `verified`, until another discriminating quarter appears |
| 5. Stale RBA notes & absent CVM `BASE_PERIOD` | `h01hist.xlsx` Notes still claim `2022/2023` when `5206.0`/`ANA_AGG`/`GOPITT` are on `2023-24`; `ANA_AGG` lacks `BASE_PERIOD` | Infer CVM reference year empirically via `Original` `sum(M1.GPM) - sum(M3.GPM)` rather than parsing static notes |
| 6. Structural API coverage gaps | No ABS Series IDs in API; `18` Non-market series absent from `ANA_AGG`; Farm/Non-farm GDP absent from `ANA_*` flows | Maintain explicit semantic key mapping; route missing series to Excel fallback in Phase 2. **[2nd pass]** Validate every configured key with `availableconstraint` or a bounded data probe; codelist membership is not evidence of data (`ANA_IND_GVA` industries `97`/`98`). Validate key arity against the DSD (the 6-position WPI key) |

---

## 7. Reproducible Verification Example (R)

The R script below runs the headline reconciliation checks end to end. It resolves the current `.xlsx` links from the ABS `/latest-release` pages, because ABS Excel download URLs embed the release month (e.g. `/sep-2026/`, `/jun-2026/`). That keeps it runnable across future release cycles.

**Requirements**

- R 4.0 or later, which `httr2` requires and which gives the `round()` behaviour described below. Only R 4.4.2 on macOS was tested.
- Two CRAN packages:
  - [`httr2`](https://httr2.r-lib.org/) (tested with 1.0.7) handles HTTP with an explicit user agent and timeout, and reads status codes and headers.
  - [`readxl`](https://readxl.tidyverse.org/) (tested with 1.4.3) reads `.xlsx` workbooks without Java or Excel. It returns each numeric cell as the double stored in the file.
- Everything else uses base R (`utils`, `stats`).

**Install and run**

```sh
Rscript -e 'install.packages(c("httr2", "readxl"), repos = "https://cloud.r-project.org")'
Rscript --vanilla reconcile.R      # save the script below as reconcile.R
```

The script makes 19 HTTP requests to `abs.gov.au`, `data.api.abs.gov.au` and `rba.gov.au`: 4 release pages, 4 workbooks, 4 RBA CSVs and 7 API queries. It writes the four ABS workbooks to R's session `tempdir()` and finishes in a few seconds.

**How the port handles precision, parsing and joins**

- **Workbooks.** `read_excel(..., col_names = FALSE, col_types = "list")` keeps each cell's own type:
  - The `Series ID` row is found by its label. The first-draft parser instead took the tenth non-empty row; both give row 10 in all four workbooks.
  - Dates arrive as UTC `POSIXct` and are converted with `as.Date(..., tz = "UTC")`.
  - Numbers arrive as the stored IEEE-754 double. For example, the `5206001` cell serialised as `1.1000000000000001` is read as exactly that value.
- **API and RBA CSVs.**
  - Both are read with `colClasses = "character"` and `na.strings = character(0)`. Blank `OBS_VALUE` cells (`OBS_STATUS=m`) therefore stay `""` and are dropped explicitly, and the raw 8-d.p. strings survive for decimal rounding.
  - RBA files have ragged rows, so the column count comes from `count.fields()`. Otherwise `read.csv()` would wrap long rows.
- **Rounding.**
  - `round_half_up()` does exact decimal `ROUND_HALF_UP` on the API *string*, not on the double.
  - Derived YoY values use base `round(x, 1)`. R 4.0.0 changed `round()` to pick the closest representable result, so R below 4.0 can differ at exact representational ties.
- **Joins.** A series is a named numeric vector keyed by period; a duplicated key keeps its last value. A comparison uses `intersect()` on the keys, sorted in byte order (`sort(method = "radix")`) so the result doesn't depend on locale, with an absolute tolerance of `1e-9`.

```r
# Reconcile ABS Data API output with ABS Excel time series and RBA CSV tables.
# Requires R >= 4.0 (tested on 4.4.2) and the httr2 and readxl packages.
# Run: Rscript --vanilla reconcile.R
suppressPackageStartupMessages({
  library(httr2)
  library(readxl)
})

t0 <- proc.time()[["elapsed"]]
UA <- "ABSdataAPI-reconcile/0.1"
API <- "https://data.api.abs.gov.au/rest/data/"

fetch <- function(url, path = NULL) {
  req <- req_timeout(req_user_agent(request(url), UA), 45)
  req_perform(req, path = path)
}

fetch_text <- function(url) {
  sub("^\\ufeff", "", resp_body_string(fetch(url), encoding = "UTF-8"))  # drop any BOM
}

read_csv_chr <- function(text) {
  # Read every column as character and keep blank cells as "" (not NA).
  utils::read.csv(text = text, colClasses = "character", na.strings = character(0),
                  check.names = FALSE, encoding = "UTF-8")
}

fetch_api <- function(path) read_csv_chr(fetch_text(paste0(API, path)))

resolve_abs_xlsx <- function(release_url, pattern) {
  html <- fetch_text(release_url)
  m <- regmatches(html, regexpr(paste0('href="[^"]*', pattern, '"'), html))
  if (length(m) == 0) stop("No match for ", pattern, " on ", release_url)
  href <- sub('^href="(.*)"$', "\\1", m)
  if (startsWith(href, "http")) href else paste0("https://www.abs.gov.au", href)
}

fetch_abs_data1 <- function(release_url, pattern) {
  # Returns a data.frame(sid, date, value) for every populated cell on the Data1 sheet.
  path <- tempfile(fileext = ".xlsx")
  fetch(resolve_abs_xlsx(release_url, pattern), path = path)
  x <- read_excel(path, sheet = "Data1", col_names = FALSE, col_types = "list",
                  .name_repair = "minimal")
  first <- vapply(x[[1]], function(v) if (is.character(v)) v else NA_character_, "")
  sid_row <- match("Series ID", first)  # row 10 in ABS time-series workbooks
  sids <- vapply(x[sid_row, -1], function(v) as.character(v[[1]]), "")
  data <- x[(sid_row + 1):nrow(x), ]
  dates <- as.Date(do.call(c, lapply(data[[1]], function(v) {
    if (inherits(v, "POSIXct")) v else as.POSIXct(NA, tz = "UTC")
  })), tz = "UTC")
  out <- lapply(seq_along(sids), function(j) {
    vals <- vapply(data[[j + 1]], function(v) if (is.numeric(v)) v else NA_real_, 0)
    keep <- !is.na(dates) & !is.na(vals)
    data.frame(sid = sids[j], date = dates[keep], value = vals[keep])
  })
  do.call(rbind, out)
}

parse_rba_csv <- function(url, mode) {
  text <- fetch_text(url)
  ncol <- max(utils::count.fields(textConnection(text), sep = ",", quote = "\"",
                                  blank.lines.skip = FALSE), na.rm = TRUE)
  rows <- utils::read.csv(text = text, header = FALSE, colClasses = "character",
                          na.strings = character(0), col.names = paste0("V", seq_len(ncol)),
                          fill = TRUE, blank.lines.skip = TRUE, encoding = "UTF-8")
  rows <- rows[apply(rows, 1, function(r) any(nzchar(trimws(r)))), ]
  first <- trimws(rows[[1]])
  sids <- unlist(rows[match("Series ID", first), ])
  pub_date <- rows[match("Publication date", first), 2]
  body <- rows[(match("Series ID", first) + 1):nrow(rows), ]
  body <- body[grepl("/", body[[1]], fixed = TRUE), ]
  dmy <- do.call(rbind, strsplit(body[[1]], "/", fixed = TRUE))
  mm <- as.integer(dmy[, 2])
  period <- if (mode == "M") sprintf("%s-%02d", dmy[, 3], mm) else
    sprintf("%s-Q%d", dmy[, 3], (mm - 1) %/% 3 + 1)
  out <- lapply(2:ncol, function(j) {
    v <- trimws(body[[j]])
    keep <- nzchar(v)
    data.frame(sid = rep(sids[j], sum(keep)), period = period[keep], value = as.numeric(v[keep]))
  })
  list(pub_date = pub_date, data = do.call(rbind, out))
}

to_q <- function(d) sprintf("%s-Q%d", format(d, "%Y"), (as.integer(format(d, "%m")) - 1) %/% 3 + 1)
to_m <- function(d) format(d, "%Y-%m")

# A "series" is a named numeric vector keyed by period; if a period repeats, the last value wins.
as_series <- function(keys, values) {
  keep <- !duplicated(keys, fromLast = TRUE)
  stats::setNames(values[keep], keys[keep])
}
api_series <- function(df) {
  df <- df[nzchar(df$OBS_VALUE), ]
  as_series(df$TIME_PERIOD, as.numeric(df$OBS_VALUE))
}
xl_series <- function(xl, sid, to) with(xl[xl$sid == sid, ], as_series(to(date), value))
rba_series <- function(rba, sid) with(rba$data[rba$data$sid == sid, ], as_series(period, value))

compare <- function(a, b, tol = 1e-9) {
  keys <- sort(intersect(names(a), names(b)), method = "radix")  # byte order, independent of locale
  bad <- keys[abs(a[keys] - b[keys]) > tol]
  list(n = length(keys), diffs = data.frame(period = bad, a = unname(a[bad]), b = unname(b[bad])))
}
fmt_diffs <- function(d) {
  if (nrow(d) == 0) return("[]")
  paste0("[", paste(sprintf("('%s', %s, %s)", d$period, format(d$a, nsmall = 1), format(d$b, nsmall = 1)), collapse = ", "), "]")
}

round_half_up <- function(x, digits) {
  # Exact decimal ROUND_HALF_UP of decimal strings: ties round away from zero, no binary error.
  neg <- startsWith(x, "-")
  a <- sub("^[-+]", "", x)
  int <- sub("\\..*$", "", a)
  frac <- ifelse(grepl(".", a, fixed = TRUE), sub("^[^.]*\\.", "", a), "")
  frac <- paste0(frac, strrep("0", pmax(0, digits + 1 - nchar(frac))))
  scaled <- as.numeric(paste0(int, substr(frac, 1, digits))) +
    (substr(frac, digits + 1, digits + 1) >= "5")
  ifelse(neg, -1, 1) * scaled / 10^digits
}

# 1. GDP / National Accounts (ANA_AGG vs 5206001.xlsx vs RBA H1 + BOP Terms of Trade)
gdp_rel <- paste0("https://www.abs.gov.au/statistics/economy/national-accounts/",
                  "australian-national-accounts-national-income-expenditure-and-product/latest-release")
x5206 <- fetch_abs_data1(gdp_rel, "5206001_Key_Aggregates\\.xlsx")
h1 <- parse_rba_csv("https://www.rba.gov.au/statistics/tables/csv/h1-data.csv", "Q")
ana <- fetch_api("ABS,ANA_AGG,1.0.0/M1+M3+M5.GPM+TTR.10+20.AUS.Q?format=csv")
ana_sel <- function(m, d, t) api_series(ana[ana$MEASURE == m & ana$DATA_ITEM == d & ana$TSEST == t, ])

api_gdp <- ana_sel("M1", "GPM", "20")
c_xl <- compare(api_gdp, xl_series(x5206, "A2304402X", to_q))
c_rba <- compare(api_gdp, rba_series(h1, "GGDPCVGDP"))

prev_year_q <- function(p) sprintf("%d-Q%s", as.integer(substr(p, 1, 4)) - 1, substr(p, 7, 7))
yoy_q <- function(s) {
  prev <- prev_year_q(names(s))
  ok <- prev %in% names(s)
  stats::setNames(round((s[ok] / s[prev[ok]] - 1) * 100, 1), names(s)[ok])
}
c_yoy <- compare(yoy_q(api_gdp), rba_series(h1, "GGDPCVGDPY"))

m1o <- ana_sel("M1", "GPM", "10"); m3o <- ana_sel("M3", "GPM", "10")
fy_diffs <- do.call(rbind, lapply(2020:2025, function(fy) {
  qs <- c(paste0(fy, c("-Q3", "-Q4")), paste0(fy + 1, c("-Q1", "-Q2")))
  if (!all(qs %in% names(m1o) & qs %in% names(m3o))) return(NULL)
  data.frame(fy = sprintf("%d-%s", fy, substr(fy + 1, 3, 4)),
             diff = round(sum(m1o[qs]) - sum(m3o[qs]), 1))
}))
ref_fy <- fy_diffs[which.min(abs(fy_diffs$diff)), ]

api_ttr <- ana_sel("M5", "TTR", "20")
rba_ttr <- rba_series(h1, "GOPITT")
c_ttr_ana <- compare(api_ttr, rba_ttr)
bop_ttr <- api_series(fetch_api("ABS,BOP,1.0.0/4.130.20.Q?format=csv"))
c_ttr_bop <- compare(bop_ttr, rba_ttr)

cat(sprintf("[GDP] ANA_AGG M1.GPM vs 5206001 A2304402X: %d quarters, diffs=%d\n", c_xl$n, nrow(c_xl$diffs)))
cat(sprintf("[GDP] ANA_AGG M1.GPM vs RBA H1 (%s) GGDPCVGDP: %d quarters, diffs=%d\n",
            h1$pub_date, c_rba$n, nrow(c_rba$diffs)))
cat(sprintf("[GDP] Recomputed YoY%% vs RBA H1 GGDPCVGDPY: %d quarters, diffs=%d\n", c_yoy$n, nrow(c_yoy$diffs)))
cat(sprintf("[GDP] Inferred CVM reference year (Original sum(M1)-sum(M3)): %s (diff=%s)\n",
            ref_fy$fy, format(ref_fy$diff, nsmall = 1)))
cat(sprintf("[ToT] ANA_AGG M5.TTR vs RBA H1 GOPITT: %d qtrs, diffs=%s\n", c_ttr_ana$n, fmt_diffs(c_ttr_ana$diffs)))
cat(sprintf("[ToT] BOP 4.130.20.Q vs RBA H1 GOPITT: %d qtrs, diffs=%s\n", c_ttr_bop$n, fmt_diffs(c_ttr_bop$diffs)))

# 2. Monthly CPI (CPI 2.0.0 vs 640101.xlsx vs RBA G4)
cpi_rel <- paste0("https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/",
                  "consumer-price-index-australia/latest-release")
x6401_1 <- fetch_abs_data1(cpi_rel, "640101\\.xlsx")
g4 <- parse_rba_csv("https://www.rba.gov.au/statistics/tables/csv/g4-data.csv", "M")
cm <- fetch_api("ABS,CPI,2.0.0/1+2+3.10001+999901.10+20.50.M?format=csv")
cm_sel <- function(m, i, t) api_series(cm[cm$MEASURE == m & cm$INDEX == i & cm$TSEST == t, ])

c_cm1 <- compare(cm_sel("1", "10001", "10"), xl_series(x6401_1, "A130393720C", to_m))
c_cm2 <- compare(cm_sel("1", "10001", "10"), rba_series(g4, "GCPIAGM"))
c_cm3 <- compare(cm_sel("3", "10001", "10"), rba_series(g4, "GCPIAGYPM"))
c_cm4 <- compare(cm_sel("2", "999901", "20"), rba_series(g4, "GCPIAGSAMP"))
cat(sprintf("[CPI-M] CPI(2.0.0) 1.10001.M vs 640101 A130393720C: %d months, diffs=%d\n", c_cm1$n, nrow(c_cm1$diffs)))
cat(sprintf("[CPI-M] CPI(2.0.0) vs RBA G4 (%s): Index=%d (diffs=%d), YoY=%d (diffs=%d), SA_MoM=%d (diffs=%d)\n",
            g4$pub_date, c_cm2$n, nrow(c_cm2$diffs), c_cm3$n, nrow(c_cm3$diffs), c_cm4$n, nrow(c_cm4$diffs)))

# 3. Quarterly CPI split (CPI 2.0.0 Q vs CPI_Q 1.0.0 vs Excel vs RBA G1) + BASE_PERIOD bug detector
x6401_17 <- fetch_abs_data1(cpi_rel, "6401017\\.xlsx")
x6401_ap <- fetch_abs_data1(cpi_rel, "64010Appendix1a\\.xlsx")
g1 <- parse_rba_csv("https://www.rba.gov.au/statistics/tables/csv/g1-data.csv", "Q")
cpi_q2 <- api_series(fetch_api("ABS,CPI,2.0.0/1.10001.10.50.Q?format=csv"))
cpi_q1_tm <- api_series(fetch_api("ABS,CPI_Q,1.0.0/3.999902.20.50.Q?format=csv"))

c_cq1 <- compare(cpi_q2, xl_series(x6401_17, "A2325846C", to_q))
c_cq2 <- compare(cpi_q2, rba_series(g1, "GCPIAG"))
c_tm_xl <- compare(cpi_q1_tm, xl_series(x6401_ap, "A3604511X", to_q))
c_tm_g1 <- compare(cpi_q1_tm, rba_series(g1, "GCPIOCPMTMYP"))
pre_diffs <- sum(c_tm_g1$diffs$period < "2001-Q3")
post_diffs <- sum(c_tm_g1$diffs$period >= "2001-Q3")

sa <- fetch_api("ABS,CPI,2.0.0/1..20.50.M?format=csv&startPeriod=2024-04&endPeriod=2024-04")
sa_apr24 <- sa[sa$BASE_PERIOD == "25" & nzchar(sa$OBS_VALUE) &
                 abs(as.numeric(sa$OBS_VALUE) - 100) < 0.01, ]

cat(sprintf("[CPI-Q] CPI(2.0.0) 1.10001.Q vs 6401017 A2325846C: %d qtrs, diffs=%d; vs RBA G1 (%s) GCPIAG: %d qtrs, diffs=%d\n",
            c_cq1$n, nrow(c_cq1$diffs), g1$pub_date, c_cq2$n, nrow(c_cq2$diffs)))
cat(sprintf("[CPI-Q] CPI_Q(1.0.0) 3.999902.Q vs Appendix1a A3604511X: %d qtrs, diffs=%d\n", c_tm_xl$n, nrow(c_tm_xl$diffs)))
cat(sprintf("[CPI-Q] CPI_Q(1.0.0) 3.999902.Q vs RBA G1 GCPIOCPMTMYP: %d qtrs, pre-2001Q3_diffs=%d, post-2001Q3_diffs=%d\n",
            c_tm_g1$n, pre_diffs, post_diffs))
cat(sprintf("[CPI-META] CPI(2.0.0) TSEST=20 M=1 series tagged BASE_PERIOD=25 yet 2024-04==100: %d series\n", nrow(sa_apr24)))

# 4. Labour Force 3-tier precision check (LF 1.0.0 vs RBA H5)
h5 <- parse_rba_csv("https://www.rba.gov.au/statistics/tables/csv/h5-data.csv", "M")
lf <- fetch_api("ABS,LF,1.0.0/M13.3.1599.20.AUS.M?format=csv")
lf <- lf[nzchar(lf$OBS_VALUE), ]
lf_float <- as_series(lf$TIME_PERIOD, as.numeric(lf$OBS_VALUE))
lf_r1 <- as_series(lf$TIME_PERIOD, round_half_up(lf$OBS_VALUE, 1))  # round the API string, not the double
rba_ur <- rba_series(h5, "GLFSURSA")
c_lf_raw <- compare(lf_float, rba_ur)
c_lf_r1 <- compare(lf_r1, rba_ur)
cat(sprintf("[LF] LF(1.0.0) M13 vs RBA H5 (%s) GLFSURSA (%d mos): raw_diffs=%d, DECIMALS=1_diffs=%d\n",
            h5$pub_date, c_lf_r1$n, nrow(c_lf_raw$diffs), nrow(c_lf_r1$diffs)))
cat(sprintf("Completed all checks in %.1fs\n", proc.time()[["elapsed"]] - t0))
```

### Observed Output (Executed Live 2026-11-23 UTC)

```text
[GDP] ANA_AGG M1.GPM vs 5206001 A2304402X: 268 quarters, diffs=0
[GDP] ANA_AGG M1.GPM vs RBA H1 (03-Sep-2026) GGDPCVGDP: 268 quarters, diffs=0
[GDP] Recomputed YoY% vs RBA H1 GGDPCVGDPY: 264 quarters, diffs=0
[GDP] Inferred CVM reference year (Original sum(M1)-sum(M3)): 2023-24 (diff=0.0)
[ToT] ANA_AGG M5.TTR vs RBA H1 GOPITT: 268 qtrs, diffs=[('2010-Q4', 96.0, 96.1)]
[ToT] BOP 4.130.20.Q vs RBA H1 GOPITT: 268 qtrs, diffs=[]
[CPI-M] CPI(2.0.0) 1.10001.M vs 640101 A130393720C: 30 months, diffs=0
[CPI-M] CPI(2.0.0) vs RBA G4 (29-Oct-2026): Index=30 (diffs=0), YoY=18 (diffs=0), SA_MoM=29 (diffs=0)
[CPI-Q] CPI(2.0.0) 1.10001.Q vs 6401017 A2325846C: 313 qtrs, diffs=0; vs RBA G1 (29-Oct-2026) GCPIAG: 313 qtrs, diffs=0
[CPI-Q] CPI_Q(1.0.0) 3.999902.Q vs Appendix1a A3604511X: 175 qtrs, diffs=0
[CPI-Q] CPI_Q(1.0.0) 3.999902.Q vs RBA G1 GCPIOCPMTMYP: 175 qtrs, pre-2001Q3_diffs=34, post-2001Q3_diffs=0
[CPI-META] CPI(2.0.0) TSEST=20 M=1 series tagged BASE_PERIOD=25 yet 2024-04==100: 18 series
[LF] LF(1.0.0) M13 vs RBA H5 (20-Nov-2026) GLFSURSA (585 mos): raw_diffs=585, DECIMALS=1_diffs=0
Completed all checks in 2.8s
```

**Run history on 2026-11-23 UTC.** All three runs printed the same 13 result lines; only the timing line differs.

| Run | Implementation | Timing |
|---|---|---|
| First pass | Original implementation, before the R port | `10.9s` |
| **[2nd pass]** Independent re-run, about 11:20 UTC | Same script, unchanged | `5.9s` |
| R port, about 12:50 UTC | This script | `2.8s` |

The headline parity results therefore reproduce for a second operator and a second implementation on the same day. Only same-day vintages are covered; see taxonomy A9.

### 7.1 Second-pass supplementary checks

**Install and run:** this script needs `httr2` only. Save it as `supp.R` and run `Rscript --vanilla supp.R`; it makes 22 requests.

S1–S6 reproduce the second-pass checks. S7 (WPI vs H4) and S8 (YoY knife-edges) were previously described as "run separately" and are now scripted as well. `frac_after()` reads the digits beyond the rounding position from the API string, so the knife-edge test isn't affected by binary floating-point error.

```r
# Second-pass supplementary checks. Requires R >= 4.0 (tested on 4.4.2) and the httr2 package.
# Run: Rscript --vanilla supp.R
suppressPackageStartupMessages(library(httr2))

API <- "https://data.api.abs.gov.au/rest/"
UA <- "ABSdataAPI-reconcile/0.1"

get <- function(url) {
  # Returns the response without raising on HTTP errors, so 403/404 can be reported.
  req <- req_error(req_timeout(req_user_agent(request(url), UA), 90), is_error = function(r) FALSE)
  req_perform(req)
}

rows <- function(path) {
  resp <- get(paste0(API, path))
  if (resp_status(resp) >= 400) return(sprintf("HTTP %d", resp_status(resp)))
  text <- sub("^\\ufeff", "", resp_body_string(resp, encoding = "UTF-8"))
  utils::read.csv(text = text, colClasses = "character", na.strings = character(0), check.names = FALSE)
}

series <- function(path) {
  df <- rows(path)
  df <- df[nzchar(df$OBS_VALUE), ]
  stats::setNames(as.numeric(df$OBS_VALUE), df$TIME_PERIOD)
}

frac_after <- function(x, digits) {
  # Fractional part left after shifting x by `digits` decimal places, read exactly from the string.
  frac <- ifelse(grepl(".", x, fixed = TRUE), sub("^[^.]*\\.", "", x), "")
  rest <- substring(frac, digits + 1)
  ifelse(nzchar(rest), as.numeric(paste0("0.", rest)), 0)
}

# S1. How many quarters discriminate between the two ABS Terms of Trade series?
ana <- series("data/ABS,ANA_AGG,1.0.0/M5.TTR.20.AUS.Q?format=csv")
bop <- series("data/ABS,BOP,1.0.0/4.130.20.Q?format=csv")
k <- sort(intersect(names(ana), names(bop)), method = "radix")
d <- k[ana[k] != bop[k]]
cat("S1 ANA_AGG vs BOP ToT diffs: ", paste(sprintf("(%s, %s, %s)", d, format(ana[d], nsmall = 1), format(bop[d], nsmall = 1)), collapse = " "), "\n", sep = "")

# S2. Is the DECIMALS attribute usable as a rounding rule outside LF?
for (path in c("ABS,CPI,2.0.0/1.10001.10.50.M", "ABS,CPI_Q,1.0.0/1.999902.20.50.Q",
               "ABS,ANA_AGG,1.0.0/M1.GPM.20.AUS.Q", "ABS,BOP,1.0.0/4.130.20.Q",
               "ABS,WPI,1.2.0/2.THRPEB.7.TOT.20.AUS.Q", "ABS,LF,1.0.0/M13.3.1599.20.AUS.M")) {
  rs <- rows(sprintf("data/%s?format=csv&lastNObservations=1", path))
  dec <- if ("DECIMALS" %in% names(rs)) unique(rs$DECIMALS) else "<no column>"
  cat("S2 ", sub("/.*", "", path), " DECIMALS = ", paste(sprintf("'%s'", dec), collapse = " "), "\n", sep = "")
}

# S3. Classify every CPI 2.0.0 monthly index series by the month where it equals 100
cpi <- rows("data/ABS,CPI,2.0.0/1...50.M?format=csv&startPeriod=2024-04&endPeriod=2025-09")
cpi$v <- as.numeric(cpi$OBS_VALUE)
at100 <- function(p) {
  hit <- cpi[cpi$TIME_PERIOD == p & cpi$v == 100, c("INDEX", "TSEST", "BASE_PERIOD")]
  paste(hit$INDEX, hit$TSEST, hit$BASE_PERIOD)
}
keys <- unique(cpi[c("INDEX", "TSEST", "BASE_PERIOD")])
id <- paste(keys$INDEX, keys$TSEST, keys$BASE_PERIOD)
a24 <- id %in% at100("2024-04"); s25 <- id %in% at100("2025-09")
keys$class <- ifelse(a24 & s25, "2024-04,2025-09", ifelse(a24, "2024-04", ifelse(s25, "2025-09", "neither")))
tab <- stats::aggregate(list(n = keys$INDEX), keys[c("TSEST", "BASE_PERIOD", "class")], length)
tab <- tab[order(tab$TSEST, tab$class, method = "radix"), ]
cat(sprintf("S3 TSEST=%s BASE_PERIOD=%s =100 at %s: %d\n", tab$TSEST, tab$BASE_PERIOD, tab$class, tab$n), sep = "")

# S4. Codelist codes are not proof of data: Farm/Non-farm in ANA_IND_GVA
txt <- resp_body_string(get(paste0(API, "availableconstraint/ABS,ANA_IND_GVA,1.0.0/all/all/INDUSTRY")))
vals <- regmatches(txt, gregexpr("<common:Value>[^<]*</common:Value>", txt))[[1]]
cat("S4 INDUSTRY codes with data: ", paste(gsub("</?common:Value>", "", vals), collapse = " "), "\n", sep = "")
farm <- rows("data/ABS,ANA_IND_GVA,1.0.0/....97+98..Q?format=csv")
cat("S4 Farm/Non-farm (97+98) data: ", if (is.character(farm)) farm else sprintf("%d rows", nrow(farm)), "\n", sep = "")

# S5. Knife-edge rounding cases (unrounded value within 1e-3 of a 1-dp boundary)
lf <- rows("data/ABS,LF,1.0.0/M3+M12+M13.3.1599.20.AUS.M?format=csv")
lf <- lf[nzchar(lf$OBS_VALUE), ]
edge <- lf[abs(frac_after(lf$OBS_VALUE, 1) - 0.5) < 0.001, ]
edge <- edge[order(edge$MEASURE, edge$TIME_PERIOD, method = "radix"), ]
cat("S5 LF knife-edge obs: ", paste(sprintf("(%s, %s, %s)", edge$MEASURE, edge$TIME_PERIOD, edge$OBS_VALUE), collapse = " "), "\n", sep = "")

# S6. When did RBA last rewrite each CSV? (server Last-Modified, UTC)
for (t in c("g1", "g4", "h1", "h4", "h5")) {
  resp <- get(sprintf("https://www.rba.gov.au/statistics/tables/csv/%s-data.csv", t))
  cat("S6 ", t, " ", resp_header(resp, "Last-Modified"), "\n", sep = "")
}

# S7. WPI vs RBA H4 with the corrected 7-position key (both use 1 d.p., so compare directly)
h4_text <- resp_body_string(get("https://www.rba.gov.au/statistics/tables/csv/h4-data.csv"))
h4_ncol <- max(utils::count.fields(textConnection(h4_text), sep = ",", quote = "\""), na.rm = TRUE)
h4 <- utils::read.csv(text = h4_text, header = FALSE, colClasses = "character", na.strings = character(0),
                      fill = TRUE, col.names = paste0("V", seq_len(h4_ncol)))
sid_row <- match("Series ID", trimws(h4$V1))
body <- h4[grepl("/", h4$V1, fixed = TRUE) & seq_len(nrow(h4)) > sid_row, ]
dmy <- do.call(rbind, strsplit(body$V1, "/", fixed = TRUE))
qtr <- sprintf("%s-Q%d", dmy[, 3], (as.integer(dmy[, 2]) - 1) %/% 3 + 1)
for (m in list(c("2", "GWPIQP"), c("3", "GWPIYP"))) {
  api <- series(sprintf("data/ABS,WPI,1.2.0/%s.THRPEB.7.TOT.20.AUS.Q?format=csv", m[1]))
  col <- which(unlist(h4[sid_row, ]) == m[2])
  v <- trimws(body[[col]]); rba <- stats::setNames(as.numeric(v[nzchar(v)]), qtr[nzchar(v)])
  k <- sort(intersect(names(api), names(rba)), method = "radix")
  cat(sprintf("S7 WPI MEASURE=%s vs H4 %s: %d qtrs (%s..%s), diffs=%d\n",
              m[1], m[2], length(k), k[1], k[length(k)], sum(abs(api[k] - rba[k]) > 1e-9)))
}

# S8. Derived YoY values within 0.0005 pp of a 1-dp rounding boundary
yoy_edges <- function(s, label) {
  s <- s[sort(names(s), method = "radix")]
  x <- (s[-(1:4)] / s[seq_len(length(s) - 4)] - 1) * 100
  e <- names(x)[abs((x * 10) %% 1 - 0.5) < 0.005]
  cat(sprintf("S8 %s: %d YoY values, edges: %s\n", label, length(x),
              paste(sprintf("%s=%.6f", e, x[e]), collapse = ", ")))
}
yoy_edges(series("data/ABS,ANA_AGG,1.0.0/M1.GPM.20.AUS.Q?format=csv"), "GDP CVM SA")
yoy_edges(series("data/ABS,CPI,2.0.0/1.10001.10.50.Q?format=csv"), "CPI Q original")
```

Observed output (2026-11-23 UTC):

```text
S1 ANA_AGG vs BOP ToT diffs: (2010-Q4, 96.0, 96.1)
S2 ABS,CPI,2.0.0 DECIMALS = ''
S2 ABS,CPI_Q,1.0.0 DECIMALS = ''
S2 ABS,ANA_AGG,1.0.0 DECIMALS = '<no column>'
S2 ABS,BOP,1.0.0 DECIMALS = '<no column>'
S2 ABS,WPI,1.2.0 DECIMALS = '<no column>'
S2 ABS,LF,1.0.0 DECIMALS = '1'
S3 TSEST=10 BASE_PERIOD=25 =100 at 2025-09: 158
S3 TSEST=20 BASE_PERIOD=25 =100 at 2024-04: 18
S3 TSEST=20 BASE_PERIOD=25 =100 at 2025-09: 34
S3 TSEST=20 BASE_PERIOD=25 =100 at neither: 53
S4 INDUSTRY codes with data: A B C D E F G H I J K L M N O P Q R S TOTAL
S4 Farm/Non-farm (97+98) data: HTTP 404
S5 LF knife-edge obs: (M12, 1982-04, 60.85001439) (M12, 2016-05, 64.84999226) (M12, 2024-10, 67.04993198) (M13, 1985-11, 7.95005315)
S6 g1 Thu, 29 Oct 2026 05:11:08 GMT
S6 g4 Thu, 29 Oct 2026 05:09:53 GMT
S6 h1 Thu, 03 Sep 2026 01:13:41 GMT
S6 h4 Thu, 19 Nov 2026 01:41:36 GMT
S6 h5 Fri, 20 Nov 2026 00:16:58 GMT
S7 WPI MEASURE=2 vs H4 GWPIQP: 116 qtrs (1997-Q4..2026-Q3), diffs=0
S7 WPI MEASURE=3 vs H4 GWPIYP: 113 qtrs (1998-Q3..2026-Q3), diffs=0
S8 GDP CVM SA: 264 YoY values, edges: 1970-Q1=8.250248, 2008-Q1=3.349850, 2016-Q1=2.650304, 2017-Q3=3.050229
S8 CPI Q original: 309 YoY values, edges: 1998-Q1=-0.150279, 2022-Q3=7.250210
```

S1–S6 match the second-pass results. S7 and S8 match the values cited in the text:
- **WPI vs H4 with the corrected key:** 116/116 and 113/113 quarters, 0 diffs ([§5](#5-cross-check-on-remaining-phase-1-dataflows-lf--wpi)).
- **YoY knife-edges:** GDP CVM SA values within 0.0005 pp of a rounding boundary at `1970-Q1`, `2008-Q1`, `2016-Q1` and `2017-Q3`, and quarterly CPI at `1998-Q1` and `2022-Q3`. All still match the RBA today.

### 7.2 Conversion parity checks for the R port

Before replacing the earlier scripts I checked that the R port reproduces them, so that a changed number could only come from live data:

1. **Same-time runs.** I ran the earlier main script and the R script back to back at about 12:50 UTC. All 13 result lines were identical, so the published numbers had not drifted since the first pass and the port introduced no change. I also ran the earlier supplementary script next to `supp.R`. S1–S6 agreed value for value; only the printing differs (R prints `(2010-Q4, 96.0, 96.1)` where the earlier script printed `[('2010-Q4', 96.0, 96.1)]`).
2. **Element-level comparison.** Both implementations dumped their parsed intermediate data, and I compared every value for exact double equality. That covered:
   - all `63,075` populated cells from the four ABS workbooks (`5206001`, `640101`, `6401017`, `64010Appendix1a`), keyed by series ID and date;
   - all `17,941` parsed cells and the `Publication date` of RBA `G1`, `G4`, `H1`, `H4` and `H5`;
   - `1,343` derived YoY values (`ANA_AGG` GDP CVM and nominal, `CPI` quarterly original, and the three `CPI_Q` series);
   - half-up rounding of all `434,959` distinct `LF(1.0.0)` observation strings from 1978 onward.

   There were **0 missing keys and 0 value mismatches**.
3. **Rounding edge cases.** `round_half_up()` matched decimal `ROUND_HALF_UP` on 12 synthetic inputs: negatives (`-0.25`, `-0.05`, `-1.5`), exact ties (`0.05`, `1.25`, `14563.25`), carries (`10.95` → `11.0`, `99.95` → `100.0`), near-ties (`0.149999999`, `-3.149999`), an integer (`2`) and the 8-d.p. `4.86483265`.

**Not checked:**
- Windows or Linux.
- R versions other than 4.4.2, or `httr2`/`readxl` versions other than those listed above.
- A non-UTF-8 session locale.
- A real release-day window (see Remaining Unverified Gaps 4 and 7).

The one-off harness used for check 2 is not included here, because it exercised the earlier implementation the R port replaces.

---

## 8. Recommendations: Which Reconciliation Checks Make Sense for Phase 1

Mapping directly onto the bounded-slice architecture in [abs-api-research.md §9](abs-api-research.md#9-suggested-poc-architecture), Phase 1 should run three lightweight, zero-key check tiers and defer full-cube Excel parsing and Indicator API parity to Phase 2:

### Tier 1 — Every Poll, Zero External Downloads (Fail-Fast Invariants)

1. **Strip `OBS_STATUS=m` blanks before completeness checks:** Do not treat expected missing observations (e.g., `1959-Q3` `M2.GPM` in `ANA_AGG`) as truncated transfers; count populated vs `OBS_STATUS=m` rows separately.
2. **Cache and verify SDMX codelists at startup:** Fetch `/rest/dataflow/ABS/{ID}/{ver}?references=all&detail=referencepartial&format=json-structure-2.0.0` (verified live) to resolve dimension codes and assert configured versions (`CPI 2.0.0`, `CPI_Q 1.0.0`, `LF 1.0.0`, `ANA_AGG 1.0.0`, `WPI 1.2.0`).
3. **Enforce reference-base invariants:**
   - Assert `CPI(2.0.0)` `1.10001.10.50.M` equals `100.0` at `2025-09` (catches unannounced CPI re-referencing immediately).
   - Quarantine or override `BASE_PERIOD` on the `18` `CPI(2.0.0)` `TSEST=20` monthly series anchored at `2024-04 = 100` so downstream SQL/Parquet consumers cannot mistake them for `Sep 2025 = 100`.
   - Compute `Original` (`TSEST=10`) `sum(M1.GPM) - sum(M3.GPM)` by Australian financial year on `ANA_AGG` to infer and stamp the active CVM reference year (`2023-24`) in provenance metadata.
   - **[2nd pass]** Scope the "equals 100 at base" assertion to `TSEST=10`. For SA series, flag only an exact `100` at a period other than the declared base (see [Discrepancy 5](#followed-discrepancy-5-base_period25-metadata-bug-in-cpi200)).
4. **[2nd pass] Validate configured keys at startup, not just versions:**
   - Check each key's arity against the DSD dimension count (the first draft's 6-position WPI key returned HTTP 403).
   - Check that each key has data via `GET /rest/availableconstraint/ABS,{ID},{ver}/{key}/all/{DIM}` (verified live) or a `lastNObservations=1` probe.
   - Codelist membership alone is not sufficient (`ANA_IND_GVA` `97`/`98`).

### Tier 2 — Release-Day (`T`) Canary Check Against 4 Headline ABS `.xlsx` Tables

Because ABS updates `abs.gov.au` spreadsheets at the 11:30 am embargo lift while the Beta Data API's arrival time is undocumented ([README evidence rules](../../README.md#evidence-rules)), running a lightweight canary check on release day (`CPI`, `ANA`, `LF`, `WPI`) against **only 4 dynamically resolved ABS `.xlsx` tables** (`640101.xlsx`, `6401017.xlsx`, `64010Appendix1a.xlsx`, `5206001_Key_Aggregates.xlsx`—optionally `62020001.xlsx` for `LF`) achieves two goals at once:
1. Confirms Data API parity before RBA updates its tables on `T+1` using `abs(a - b) <= 1e-6`.
2. Detects and logs when `abs.gov.au` Excel files reflect a newer period than the Data API, measuring real release-day API lag empirically rather than relying on the unverified `"~12:30pm"` community rule of thumb.

**[2nd pass] Order of checks on release day.** First compare the latest period in Excel with the latest period in the API.

- **The Excel file is ahead:** the API still holds the previous vintage. Every historical diff is then expected, because seasonal revisions arrive with the new release. Record `api_not_yet_updated` and retry; do not fail.
- **The latest periods are equal:** run the `<= 1e-6` comparison.

The 2026-11-23 audit ran days after each release, so it never saw this state. Without the ordering, the first live release-day run would probably produce a flood of false failures. This is an inference from how revisions propagate, not an observed failure.

### Tier 3 — `T+1` Canary Check Against RBA CSV Tables

Maintain a small, explicit `config/reconciliation_canaries.yaml` (~15–20 series across `CPI`, `CPI_Q`, `ANA_AGG`, `BOP` for `GOPITT`, `LF`, and `WPI`) specifying `sdmx_key`, `rba_table`, `rba_series_id`, `transform` (`direct`, `yoy_pct`, or `round_decimals`), `valid_from`, and `tolerance`.
- Trigger on `T+1` (or whenever a table's `Publication date` row advances).
- Restrict equality assertions to `Source: ABS` and post-`2003-Q1` `Source: ABS / RBA` series; **never** assert equality on `Source: RBA` analytical decompositions (`GCPIXVIYP`, `GCPITIYP`, etc.).
- Require `0` unexpected differences across the aligned overlap window.
- **[2nd pass]** Add `published_decimals` to each canary: `DECIMALS` exists only for `LF`, and the `transform: round_decimals` option needs an explicit target elsewhere. Classify one-unit, last-digit diffs whose unrounded value lies near a rounding boundary as `rounding_edge`, not as failures.
- **[2nd pass]** For `GOPITT`, configure **two** candidate sources (`BOP 4.130.20.Q` primary, `ANA_AGG M5.TTR.20.AUS.Q` secondary). Pass if either matches, and log which one did. The source attribution is inferred from a single quarter (see [Discrepancy 1](#followed-discrepancy-1-2010-q4-terms-of-trade-960-vs-961-and-imprecise-rba-attribution)).
- **[2nd pass]** Trigger on a change in the `Publication date` row, not on a fixed `T+1`. The observed lag is next day, between 11:13 and 16:11 Canberra time, but that is n = 1 per table.

### Defer to Phase 2 / Phase 3

- **Full-workbook Excel reconciliation & fallback ingestion:** Auditing all 15+ CPI workbooks (`640102`–`6401018`) and 45 National Accounts tables, or adding Excel fallback ingestion for the `18` missing Non-market series (`5206001`) and Farm/Non-farm GDP (`5206024`).
- **Indicator API parity:** Comparing `CPI(2.0.0)` / `CPI_Q(1.0.0)` / `ANA_AGG(1.0.0)` / `LF(1.0.0)` / `WPI(1.2.0)` against `CPI_H`, `CPI_Q_H`, `GDPE_H`, `LF_H`, and `WPI_H` once an Indicator API key is provisioned.
- **Automated revision classification:** Separating annual CVM reference-year roll-forwards or CPI re-referencing from routine seasonal-factor revisions across stored `(key, OBS_VALUE)` snapshots.
- **Re-implementing `Source: RBA` CPI decompositions:** Economists requiring RBA's GST-adjusted tradables/non-tradables or ex-volatile series should read them directly from RBA Table G1 rather than rebuilding custom weights/tax adjustments from ABS expenditure classes.

### Follow-Up Updates Flagged for Existing Project Notes (Not Applied Here)

For consideration during cross-review, four findings from this draft supersede details currently in [abs-api-research.md](abs-api-research.md):
1. **`CPI_Q(1.0.0)` vs `CPI(2.0.0)` `FREQ=Q` ([§4](abs-api-research.md#4-cpi-dataflows--corrected-status-verified-live-2026-11-23)):** They are disjoint (`TSEST=20` `2011-12=100` vs `TSEST=10` `Sep 2025=100`), so a quarterly pipeline needing both headline CPI and Trimmed Mean must ingest both rather than preferring `CPI_Q` alone.
2. **Codelist endpoints ([§8 item 2](abs-api-research.md#8-known-limitations--pitfalls)):** Both `/rest/dataflow/...?references=all` and `/rest/codelist/...` plus `format=csvfilewithlabels` are now live-verified.
3. **`BOP(1.0.0)` currentness ([§5](abs-api-research.md#5-rba-core-data-coverage--tested-vs-candidate-dataflows)):** Live data extends cleanly through `2026-Q2` despite the `NonProductionDataflow` flag.
4. **Phase 1 diagram typo ([§9](abs-api-research.md#phase-1--core-ingestion-pipeline-free-keyless)):** Lists `WPI(1.0.0)` whereas `§5` and live endpoints use `WPI(1.2.0)`. **[2nd pass]** Confirmed: `ABS,WPI,1.0.0` returns HTTP 404, and `GET /dataflow/ABS/WPI/all` lists only `1.2.0`.
5. **[2nd pass] `DECIMALS` and codelist-vs-data ([§8](abs-api-research.md#8-known-limitations--pitfalls)):**
   - `DECIMALS` is populated only in `LF`.
   - Codelist codes can exist with no data (`ANA_IND_GVA` industries `97`/`98`).
   - `availableconstraint` is a live, verified way to check which codes have data.
   - Both points are worth adding as pitfalls.
6. **[2nd pass] `NonProductionDataflow` (§5, §8.9):** the flag is also set on the live `CPI(2.0.0)` and `CPI_Q(1.0.0)`, as recorded in the [series continuity draft](abs-series-continuity.md), so it does not indicate currentness. This is consistent with item 3 above.

---

## 9. Primary Sources & Evidence Gaps

### Primary Sources Inspected Live (2026-11-23 UTC)

- **ABS Data API (SDMX 2.1 REST):** `https://data.api.abs.gov.au/rest/` — `dataflow?detail=allstubs`, `dataflow/ABS/CPI_Q?format=json-structure-2.0.0`, `dataflow/ABS/ANA_AGG/1.0.0?references=all&detail=referencepartial&format=json-structure-2.0.0`, `codelist/ABS/CL_CPI_INDEX/1.0.0?format=json-structure-2.0.0`, and data queries on `CPI(2.0.0)`, `CPI_Q(1.0.0)`, `ANA_AGG(1.0.0)`, `ANA_EXP(1.0.0)`, `ANA_INC(1.0.0)`, `ANA_IND_GVA(1.0.0)`, `BOP(1.0.0)`, `LF(1.0.0)`, and `WPI(1.2.0)`.
- **ABS API Updates & Release Pages:**
  - [ABS Data API User Guide — Updates](https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/updates)
  - [ABS Consumer Price Index, Australia (Sep 2026 / latest-release)](https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/consumer-price-index-australia/latest-release) — including `640101.xlsx`, `640106.xlsx`, `640107.xlsx`, `6401016.xlsx`, `6401017.xlsx`, `6401018.xlsx`, `64010Appendix1a.xlsx`, `64010Appendix2.xlsx`, `CPI-Quarterly.zip`, and `Quarterly CPI - September 2025 Conversion Factors .xlsx`.
  - [ABS Australian National Accounts: National Income, Expenditure and Product (Jun 2026 / latest-release)](https://www.abs.gov.au/statistics/economy/national-accounts/australian-national-accounts-national-income-expenditure-and-product/latest-release) — including `5206001_Key_Aggregates.xlsx`, `5206002_Expenditure_Volume_Measures.xlsx`, `5206003_Expenditure_Current_Price.xlsx`, `5206005_Expenditure_Implicit_Price_Deflators.xlsx`, and `5206024_Selected_Analytical_Series.xlsx`.
  - [ABS Wage Price Index, Australia (Sep 2026 / latest-release)](https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/wage-price-index-australia/latest-release)
  - [ABS Labour Force, Australia (Oct 2026 / latest-release)](https://www.abs.gov.au/statistics/labour/employment-and-unemployment/labour-force-australia/latest-release) — including `62020001.xlsx`.
- **RBA Statistical Tables Portal & Files:**
  - [RBA Statistical Tables](https://www.rba.gov.au/statistics/tables/) & [Changes to Statistical Tables](https://www.rba.gov.au/statistics/tables/changes-to-tables.html)
  - CSVs: `g1-data.csv`, `g4-data.csv`, `h1-data.csv`, `h4-data.csv`, `h5-data.csv`
  - Excel workbooks (`Notes` tabs): `g01hist.xlsx`, `g04hist.xlsx`, `h01hist.xlsx`

### Remaining Unverified Gaps

1. **ABS Indicator API (`indicator.api.abs.gov.au`) parity:** Not tested (no API key provisioned yet); whether `CPI_H` / `CPI_Q_H` / `GDPE_H` exhibit the same `BASE_PERIOD` mislabelling or `18`-series Non-market omission remains unverified.
2. **Pre-2002 RBA underlying inflation YoY% calculation:** The exact unpublished unrounded series or historical formula generating the `34` Trimmed Mean (`1983-Q1..2001-Q2`) and `33` Weighted Median (`1983-Q1..2002-Q4`) YoY% differences in RBA Table G1 was not found in public RBA documentation.
3. **Why `5302.0` (`BOP`) and `5206.0` (`ANA_AGG`) differ by `0.1` index points in `2010-Q4` Terms of Trade (`96.1` vs `96.0`):** Likely reflects an earlier rounding/revision vintage or separate unrounded IPD derivation between the two releases; the internal ABS compilation difference is not confirmed from public release notes.
4. **Release-day Data API arrival latency:** Not directly observed during a live 11:30 am release window on 2026-11-23 UTC; empirical lag measurement requires running the Tier 2 check on actual release days (e.g., `25/11/2026` October CPI or `2/12/2026` September National Accounts).
5. **[2nd pass] `GOPITT` source attribution:** only one quarter (2010-Q4) discriminates between `BOP` and `ANA_AGG`, and it is a rounding knife-edge. An older `5206.0` vintage at the RBA can't be ruled out. The attribution should be confirmed with the RBA or by a future discriminating quarter.
6. **[2nd pass] RBA update lag:** each table has one observed release (n = 1). `Last-Modified` shows only the last write, so the first publication time is unknown.
7. **[2nd pass] Release-day vintage skew:** the `api_not_yet_updated` classification in Tier 2 is a design inference. It needs to be validated on a real release day.
8. **[2nd pass] Analytical-series base:** the ABS methodology pages don't state a reference period for the 18 series anchored at April 2024 (see the [series continuity draft](abs-series-continuity.md), §4). Whether the ABS will correct the attribute or the data is unknown.
9. **[2nd pass] Not independently re-checked in this pass:**
   - the pre-2002 RBA G1 underlying YoY divergences (Discrepancy 6);
   - the `h01hist.xlsx` note text (Discrepancy 2);
   - the 18 missing Non-market series;
   - the `Source: RBA` divergence counts (Discrepancy 7).

   The §7 script reproduces only some of the counts behind them. Treat them as first-pass evidence.
