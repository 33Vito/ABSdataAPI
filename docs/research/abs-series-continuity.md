# ABS Series Continuity Across Metadata Changes — Research Draft

> Status: **second-pass draft (cross-review strengthened).** Not yet reconciled with [abs-api-research.md](abs-api-research.md).
> Probes: run live on 2026-11-23 UTC against `https://data.api.abs.gov.au/rest/` (first pass from 11:06 UTC; second-pass cross-checks from 11:25 UTC).
> Primary case: the November 2025 CPI transition (`CPI` 1.1.0 → 2.0.0, `CPI_M` ceased, `CPI_Q` reinstated), plus non-CPI counterexamples (`ANA_AGG`, `WPI`, `LF`/`LF_EDU`, `HSI_M`/`RT`).

## Summary

The question is how ABSdataAPI can tell that an observation fetched today belongs to the same economic series as one fetched last year, after the ABS has changed the metadata under it.

The CPI transition—and cross-checks across 15 ABS dataflows—show that **no single SDMX identifier carries series identity on its own**, and **version pinning alone does not prevent silent in-place rebases or stops**:

| Identifier | Did it stay stable? | Evidence (§3) |
| :--- | :--- | :--- |
| Dataflow ID (`CPI`) | Yes, but the version under it changed from 1.1.0 to 2.0.0, and an unversioned request silently switched to 2.0.0 | F1 |
| Series key (`1.10001.10.50.Q`) | Same key, same concept, new level (rebased by ABS factor `0.6945` for Australia All groups) | F2 |
| Series key across dataflows (`10001` in `CPI_M` vs `CPI`) | Same code, **different statistical series** (not a rebase) | F3 |
| Codelist ID | Renamed: `CL_CPI_INDEX_17` → `CL_CPI_INDEX` | F4 |
| Code meaning | `CL_BASE_PERIOD` code `25` means "Sep 2025 = 100.0" in v1.1.0 (`CPI 2.0.0`), "2025 = 100.0" in v2.0.0 (`CPI_M`), and "2024-25 = 100.0" in v1.0.0 (live `WPI 1.2.0`) | F5 |
| `BASE_PERIOD` attribute | Says Sep 2025 = 100 across all 18 monthly SA aggregates, groups, and analytical series (`999901`–`999903`), which actually equal 100 at April 2024 | F6 |
| Codelist labels | Wording changed ("Weighted average of eight capital cities" → "Australia"), and 8 hierarchy parents moved | F4 |
| Pinned version (`1.0.0` in `ANA_AGG` / `LF_EDU`) | `ANA_AGG 1.0.0` rolls its CVM reference year in place (`2023-24`); monthly `LF_EDU 1.0.0` stopped after `2026-03` with no `CEASED` notice or version change | F8 |

**Recommendation in one line:** Phase 1 should identify a series by *(agency, dataflow ID, exact pinned version string, full series key)*, resolve labels strictly against the exact codelist version referenced by that DSD, keep a small hand-maintained **lineage file** for cross-version/cross-flow links, and pair version pinning with staleness alerts and base-type-aware invariants to catch silent in-place rebases and stops. See [§5](#5-what-phase-1-should-do).

## 1. Why CPI

The research notes ([§4](abs-api-research.md#4-cpi-dataflows--corrected-status-verified-live-2026-11-23)) already recorded three CPI dataflows with different statuses and a claimed "September 2025 = 100" base. CPI is also the RBA's most-watched release, and its trimmed mean is a core policy input. Any continuity failure here would be expensive.

## 2. Method

1. Fetched each dataflow with `references=all&detail=full`, then compared the DSDs, the codelists each DSD references, and every version of those codelists (`/codelist/ABS/{id}/all`).
2. Fetched overlapping observations of the same concept from different dataflows and versions, and tested whether one is a pure rescaling of the other.
3. Compared the API evidence with the ABS's own change log and methodology papers ([Sources](#8-sources)).

All checks run in R using base R plus `httr2` and `jsonlite` (see [§6](#6-reproduction) for install notes and the runnable script).

## 3. Findings: what changed and how we know

### F1. A new major version replaced the old one under the same dataflow ID, and both remain live

- `GET /dataflow/ABS/CPI/all` lists **two versions, 1.1.0 and 2.0.0**. Both still serve data.
- `ABS,CPI,1.1.0` is the old quarterly-only flow. Its history runs from 1948-Q3 to 2025-Q3 on the 2011-12 = 100 reference, with 1 decimal place and no `BASE_PERIOD` attribute. Asking for 2025-Q4 onward returns `404 NoRecordsFound`.
- `ABS,CPI,2.0.0` holds monthly data from 2024-04 and quarterly data from 1948-Q3, with a `BASE_PERIOD` attribute and 2 decimal places.
- **An unversioned request (`/data/ABS,CPI/...`) resolves to 2.0.0.** A client that did not pin the version would have seen the All-groups index drop from 143.6 to 99.73 between two polls, with no error.
- The ABS [API updates page](https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/updates) (November 2025 entry) confirms "CPI version 1.1.0 was updated to 2.0.0" and "CPI_M 1.2.0 and CPI_M_H discontinued".
- Keeping the old version is **not consistent across dataflows**. Auditing 15 economic dataflows (`CPI`, `CPI_Q`, `CPI_M`, `LF`, `ANA_AGG`, `WPI`, `ITGS`, `BOP`, `QBIS`, `CAPEX`, `LABOUR_ACCT_Q`, `RT`, `HSI_M`, `BUSINESS_TURNOVER`, `PPI_FD`) shows:
  - Only **5 of 15** retain multiple live versions: `CPI` (`1.1.0`, `2.0.0`), `HSI_M` (`1.0.0`, `1.4.0`, `1.6.0`), `PPI_FD` (`1.0.0`, `1.1.0`), `LABOUR_ACCT_Q` (`1.0.0`, `1.1` — note 2-part version strings also occur), and `BUSINESS_TURNOVER` (`1.0.0`, `1.2.0`, `1.3.0`).
  - Older versions of `WPI` (`1.0.0`, `1.1.0`) return 404, and `CPI_M` lists only `1.2.0` even though DSDs `1.0.0` and `1.1.0` exist. A pipeline therefore cannot rely on the old version staying available after a change.
  - Worse, **superseded minor and major versions of live flows freeze silently with no `CEASED` prefix** in their description (`CPI 1.1.0` frozen at `2025-Q3`; `HSI_M 1.0.0` at `2024-05` and `1.4.0` at `2025-06`; `PPI_FD 1.0.0` at `2023-Q4`; `LABOUR_ACCT_Q 1.0.0` at `2024-Q4`). Pinning a minor version isolates a pipeline from breaking schema changes, but it also cuts off new releases unless minor bumps are alerted.

### F2. The quarterly index was re-referenced as a pure rebase, but the published percentage changes were revised

**Level.** Comparing `CPI 1.1.0` and `CPI 2.0.0` for `1.10001.10.50.Q` (All groups, original, Australia) over all 309 overlapping quarters (1948-Q3 to 2025-Q3) shows a single factor *k* ∈ [0.694489, 0.694509] fits every quarter once rounding of both series is allowed for (1 dp in the old series, 2 dp in the new; 100/*k* ≈ 143.99, the old-basis level of September 2025). ABS's published [conversion factors workbook](https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/consumer-price-index-australia/sep-2026/Quarterly%20CPI%20-%20September%202025%20Conversion%20Factors%20.xlsx) (released 28/01/2026) directly confirms **`0.6945`** for All groups Australia (`A2325846C`). Crucially, the workbook publishes **distinct conversion factors per capital city and expenditure class** (for example, `0.6975` Melbourne, `0.6814` Brisbane, `0.7263` Darwin, and `0.5570` for Beef and veal, Australia `A2327241X`), so a rebase factor is a per-series property, not a dataflow constant.

**Movements.** The published quarterly % change (`MEASURE=2`) **was revised** in the early history:

| Quarters compared | Differ by ≥ 0.15 pp |
| :--- | :--- |
| From 1948 | 96 of 308 |
| From 1980 | 11 of 183 |
| From 2000 | 0 of 103 |

The largest gaps are in the late 1940s and 1950s (for example, 1949-Q4: 0.0 % before, 1.4 % after). The old index was published to 1 dp at levels near 4, so its published movements were rounding artefacts. The new 2 dp index on a ~2.6 level is more precise, which shows it was rebuilt from unrounded data, not rescaled from the rounded published series.

The ABS states that the movements "have not been revised, aside from minor impacts of rounding" ([re-referencing paper](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/re-referencing-quarterly-consumer-price-index)). That is true for recent decades, but for the 1950s "minor" amounts to more than 1 pp.

**Splice.** From 2025-Q4 the quarterly figure equals the mean of the three monthly figures (2026-Q1: 101.70 vs mean 101.693). Before that it does not (2024-Q2: 96.41 vs mean 96.363). The ABS confirms: "From the December quarter 2025, the quarterly index numbers are calculated as the average of the three monthly index series." One series key in one dataflow therefore holds **two compilation methods**, joined at 2025-Q3/Q4, and nothing in the observation-level metadata marks the join.

### F3. The same series key in a successor dataflow is a different statistic

`CPI_M 1.2.0` (Monthly CPI *Indicator*) and `CPI 2.0.0` monthly overlap from 2024-04 to 2025-09 for key `1.10001.10.50.M`. The ratio between them **is not constant** (0.7771 to 0.7820), so this is not a rebase. Month-on-month changes differ by up to 0.52 pp; January 2025 is −0.24 % in the indicator and +0.28 % in the complete CPI.

The ABS explains why. In the Indicator, prices collected quarterly were "rolled forward" between collection months. About 50 % of the basket weight was priced monthly, against 87 % in the complete Monthly CPI ([introducing paper](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/introducing-consumer-price-indexes-new-monthly-time-series)). The pages read do not say that the two monthly histories are linked or comparable.

The same applies to the analytical series. `999902` (Trimmed mean) means three different things depending on where it appears:

| Where | What it is |
| :--- | :--- |
| `CPI 2.0.0`, FREQ=M | Monthly trimmed mean from 2024-04, new compilation |
| `CPI_Q 1.0.0` / `CPI 1.1.0`, FREQ=Q | Quarterly trimmed mean on the pre-October 2025 compilation basis |
| `CPI_M` | No `999902` at all; it carried `999905` "Annual trimmed mean" instead |

The quarterly trimmed mean continues in `CPI_Q`. The newly read [quarterly trimmed mean explanatory note](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/quarterly-trimmed-mean-pre-october-2025-compilation-basis-explanatory-note) specifies continuation **"through to at least the June quarter 2027"** (rather than an open-ended "around mid-2027"), and documents an internal source break inside `CPI_Q` itself: from `2025-Q4` onward, **Child care**, **Long day care fees** (part of Preschool and primary education), and **Insurance** use new monthly data sources because the pre-transition quarterly sources ceased.

**Comparing `CPI 1.1.0` vs `CPI_Q 1.0.0` across the full overlap (`1982-Q1..2025-Q3`) challenges the assumption that differences are only routine seasonal re-estimation:**

| Series (`TSEST=20`, `REGION=50`, `FREQ=Q`) | Index diffs (`1982-Q1..2025-Q3`) | Diff span & max index gap | Published QoQ% / YoY% diffs |
| :--- | :--- | :--- | :--- |
| `999901` All groups SA | `9 / 156` (`1986-Q4..2025-Q3`) | `2020-Q4..2025-Q2`, max `0.1000` pts | — |
| Sampled SA ECs (`114122`, `40008`, `30014`) | `13/145`, `9/181`, `15/213` | `2020-Q4+`, `2021-Q2+`, `2022-Q1+` | — |
| `999902` Trimmed mean | **`162 / 175`** | **`1982-Q1..2025-Q3`**, max `0.2978` pts | QoQ `29/174` (`24` pre-2003, max `0.1` pp); YoY `49/171` (`41` pre-2003, max `0.2` pp) |
| `999903` Weighted median | **`133 / 175`** | **`1982-Q1..2025-Q3`**, max **`0.5012`** pts | QoQ `44/174` (`37` pre-2003, max `0.2` pp); YoY `79/171` (`62` pre-2003, max **`0.3` pp**) |

`999901` All groups SA and the sampled SA expenditure classes (`114122` Vegetables, `40008` Other cereal products, `30014` Rents) differ only from `2020-Q4` onward, matching the concurrent seasonal adjustment described in [CPI Methodology, Sep 2026](https://www.abs.gov.au/methodologies/consumer-price-index-australia-methodology/sep-2026) and the [seasonal adjustment paper](https://www.abs.gov.au/about/key-priorities/big-data-timely-insights-phase-2/complete-monthly-measure-cpi/seasonal-adjustment-new-complete-monthly-consumer-price-index-cpi). By contrast, `999902` and `999903` diverge all the way back to **`1982-Q1`** (with `31` and `46` index-ratio step-changes `> 0.0002`, while both versions keep the exact `2011-12` mean `= 100.000000`). Routine recent-window seasonal re-estimation alone does not explain those pre-2003 shifts; the underlying cause in ABS's `CPI_Q` rebuild remains unconfirmed.

### F4. Codelists were renamed, relabelled and restructured

The DSDs reference different codelists:

| Dimension | `CPI 2.0.0` | `CPI_M 1.2.0` | `CPI_Q 1.0.0` |
| :--- | :--- | :--- | :--- |
| `MEASURE` | `CL_CPI_MEASURES` 1.1.0 (7 codes) | `CL_CPI_M_MEASURES` 1.0.0 | `CL_CPI_MEASURES` 1.0.0 (5 codes) |
| `INDEX` | `CL_CPI_INDEX` 1.0.0 (166) | `CL_CPI_INDEX_17` 1.3.0 (170) | `CL_CPI_INDEX_17` 1.1.0 (168) |
| `REGION` | `CL_CPI_REGION` 1.1.0 | `CL_CPI_REGION` 1.0.0 | `CL_CPI_REGION` 1.0.0 |
| `BASE_PERIOD` | `CL_BASE_PERIOD` 1.1.0 | `CL_BASE_PERIOD` 2.0.0 | *(attribute absent)* |

The ABS update log confirms "CL_CPI_INDEX_17 1.1.0 was updated to CL_CPI_INDEX 1.0.0". The codelist **ID** changed, not just the version.

Comparing `CL_CPI_INDEX` 1.0.0 with `CL_CPI_INDEX_17` 1.3.0:

- **Codes kept their identity well.** All 166 codes in the new list exist in the old list, and 164 of them have identical labels. The 2 changes are wording only (`Non-discretionary` → `Non-Discretionary`, and the `999904` label).
- **The hierarchy moved.** 8 codes have new parents. For example, `40055` moved from `20003` to `30016`, and `132300` gained parent `104100`. Any aggregate built by walking the tree would change.
- **Some codes were dropped:** `104099`, `131198`, `131199`, `999905`.

Labels also changed wording between `CL_CPI_MEASURES` versions. Code `3` reads "Percentage Change from Corresponding Quarter of the Previous Year" in 1.0.0 and "Percentage change from previous year" in 1.1.0. Region `50` changed from "Weighted average of eight capital cities" to "Australia". **Labels are display text, not identity.**

### F5. The same code means different things in different versions of one codelist

This is the most dangerous finding for a label resolver.

| `CL_BASE_PERIOD` code | v1.0.0 (9 DSDs incl. live `WPI 1.2.0`) | v1.1.0 (used by live `CPI 2.0.0`) | v2.0.0 (used by ceased `CPI_M` 1.0.0–1.2.0) |
| :--- | :--- | :--- | :--- |
| `12` | 2011-12 = 100.0 | 2012 = 100.0 | 2012 = 100.0 |
| `17` | 2016-17 = 100.0 | Sep 2017 = 100.0 | September 2017 = 100.0 |
| `25` | 2024-25 = 100.0 | **Sep 2025 = 100.0** | **2025 = 100.0** |

Version 2.0.0 is the *higher* version but belongs to the *ceased* flow. A resolver that fetches "latest `CL_BASE_PERIOD`" would label every current CPI observation "2025 = 100.0". That is wrong: the data equal 100 at 2025-09, not on a 2025 annual average. One that used v1.0.0 would say "2024-25 = 100.0".

Crucially, checking `/codelist/ABS/CL_BASE_PERIOD/{ver}?references=parents` shows **`CL_BASE_PERIOD 1.0.0` is also active in production across 9 DSDs—including live Phase 1 `WPI(1.2.0)`**, `PPI(1.0.0/1.1.0/1.1.3)`, `LCI(1.0.0)`, `ITPI_EXP(1.0.0)`, and `ITPI_IMP(1.0.0)` (even though `WPI 1.2.0` currently emits an empty `BASE_PERIOD` column in data rows). A single Phase 1 pipeline therefore encounters **both `CL_BASE_PERIOD 1.0.0` and `1.1.0` concurrently** across core indicators; caching a single version per codelist ID would mislabel one of them.

All three versions are marked `isFinal=true`. The ABS update log entry reads "CL_BASE_PERIOD 1.1.0 updated to CL_BASE_PERIOD 1.1.0", which looks like a typo, so the log alone cannot be used to reconstruct which version is meant.

### F6. The `BASE_PERIOD` attribute does not always match the data—and naive `v == 100` detection fails

In `CPI 2.0.0`, every `MEASURE=1` monthly observation probed carries `BASE_PERIOD=25` (Sep 2025 = 100). Exhaustive auditing across all `TSEST=10` (158 series) and `TSEST=20` (105 series) Australia (`REGION=50`, `FREQ=M`) index series proves the mismatch is systematic, not limited to three analytical rows:

- **`TSEST=10` (Original, 158 series):** All `158` equal `100.00` at `2025-09`.
- **`TSEST=20` (Seasonally Adjusted, 105 series):** **`18` aggregate, group, and analytical series equal `100.0` at `2024-04`** (their start month) while carrying `BASE_PERIOD=25`: all 11 CPI Groups (`20001`–`20006`, `115486`, `115488`, `115489`, `115493`, `126670`), `102675` Tradables, `102676` Non-tradables, `104101` Goods, `104104` Services, and `999901`–`999903` (where `2025-09` sits at `104.00`, `104.6361`, and `105.0466`). The remaining 87 expenditure-class SA series (which have history back to `2017-09`) sit on the `Sep 2025 = 100` basis.
- Even after reading [CPI Methodology, Sep 2026](https://www.abs.gov.au/methodologies/consumer-price-index-australia-methodology/sep-2026) (*"Monthly and quarterly CPI data have an index reference period of September 2025 = 100.00"*) and the [seasonal adjustment paper](https://www.abs.gov.au/about/key-priorities/big-data-timely-insights-phase-2/complete-monthly-measure-cpi/seasonal-adjustment-new-complete-monthly-consumer-price-index-cpi) (which mentions splicing pre-April 2024 backcasts during seasonal adjustment without stating an April 2024 = 100 base), the contradiction between `BASE_PERIOD=25` and the `2024-04 = 100` data values remains unresolved.

Meanwhile `CPI_Q` and `CPI 1.1.0` carry **no** `BASE_PERIOD` attribute. The ABS states the quarterly SA series "remain on the 2011-12 index reference period", verified by their 4-quarter `2011-Q3..2012-Q2` mean equalling `100.0000`.

**Counterexample test on Recommendation #8 (`v == 100` detection):** Scanning each series for `OBS_VALUE == 100` is **not** a reliable standalone base-period detector across ABS datasets:

| Slice tested | Total series | Outcome under `v == 100` scan | Why naive `v == 100` fails |
| :--- | :--- | :--- | :--- |
| `CPI 2.0.0` `M=1`, `TSEST=10`, `M` | `158` | `146` hit `100` only at `2025-09`; **`12` hit `100` in multiple months** | Flat/stepped series (e.g., `115496` Secondary education is `100` across all 12 months `2025-02..2026-01`; `20005` Transport hits `100` at both `2024-07` and `2025-09`) |
| `CPI 2.0.0` `M=1`, `TSEST=20`, `M` | `105` | **`50` never hit `100`**; **`3` hit `100` in a single wrong month**; `1` (`102675`) hits `100` twice (`2024-04`, `2025-06`); `17` only `2024-04`; `34` only `2025-09` | Seasonal adjustment shifts `2025-09` off `100.0`; random crossings happen to land on `100.0` (`114122` Vegetables at `2024-05`, `40055` at `2024-03`, `40083` at `2025-11`) |
| Annual FY-mean flows (`CPI_Q`, `WPI 1.2.0`, `ANA_AGG` `M5.TTR`) | All tested | **Zero individual quarters equal `100.0`** | FY-average bases set the 4-quarter mean to `100.0` (e.g., `WPI 1.2.0` `FY 2008-09 = [98.6, 99.8, 100.5, 101.1]`, mean `100.0000`) |

### F7. Status signals: "CEASED", `NonProductionDataflow`, and `dataConstraints`

- **Ceased status** appears only as free text at the start of the dataflow description (`"CEASED - Dataset: CPI_M. …"`, `"CEASED - Dataset: RT. …"`, `"CEASED - Dataset: BUSINESS_TURNOVER. …"`), whereas superseded versions of active flows (`CPI 1.1.0`, `HSI_M 1.0.0/1.4.0`, `PPI_FD 1.0.0`, `LABOUR_ACCT_Q 1.0.0`) and paused flows (`LF_EDU 1.0.0`) carry **no `CEASED` prefix**.
- **`NonProductionDataflow=true` is set on all 15 of the 15 audited flows**—both live primary flows (`CPI 2.0.0`, `CPI_Q 1.0.0`, `LF 1.0.0`, `ANA_AGG 1.0.0`, `WPI 1.2.0`, `BOP 1.0.0`, `ITGS 1.2.0`) and ceased flows. It is a blanket Beta-API annotation and carries zero status signal.
- **Dual `Actual` `dataConstraints` (`CR_A_{id}` / `CR_B_{id}`):** Fetching `?references=all` returns two `dataConstraints` entries carrying `TIME_PERIOD` `timeRange` bounds, **but they alternate between two release slots and can disagree with served data** (e.g., `CPI 2.0.0` lists `CR_A = 2026-09-30` and `CR_B = 2026-08-31`; `LF_EDU 1.0.0` lists `CR_A = 2026-05-31` and `CR_B = 2026-03-31` even though live monthly observations end at `2026-03`). Treat `dataConstraints` as a diagnostic hint, not proof of data availability.
- After a flow or period stops, requesting new periods returns `404 NoRecordsFound`—identical to a release that has not reached the API yet.

### F8. Counterexamples beyond CPI: in-place rebases and silent stops without version bumps

Testing non-CPI dataflows refutes any assumption that continuity breaks always appear as new dataflow versions or `CEASED` descriptions:

1. **Annual in-place re-referencing in National Accounts (`ANA_AGG 1.0.0`):** Chain-volume GDP (`M1.GPM`) in `ANA_AGG 1.0.0` has no `BASE_PERIOD` attribute and stays pinned on version `1.0.0`, yet the ABS rolls the chain-volume reference year forward annually in place (rescaling every historical quarter back to `1959-Q3`). For `Original` (`TSEST=10`), the identity `sum(M1.GPM) == sum(M3.GPM)` proves the active reference year is **`2023-24`** (`2,680,025` vs `2,680,025`, `diff = 0.0`, compared to `+66,702.0` in `2022-23` and `-61,970.0` in `2024-25`). Version pinning does not isolate a pipeline from this rebase.
2. **Silent transition stop under Labour Force modernisation (`LF_EDU 1.0.0` vs `LF 1.0.0`):** In [Upcoming changes to Labour Force Survey outputs](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/upcoming-changes-labour-force-survey-outputs) (`19/03/2026`), the ABS announced a progressive survey transition across April–August 2026 where headline estimates continue while selected topics pause or move to the Labour Supplementary Survey (explicitly mentioning educational items). Live probes confirm headline `LF(1.0.0)` kept all `744 / 744` populated series keys continuous from `2026-03` to `2026-10` (and `LF_AGES`, `LF_HOURS`, `LF_STAT_EMP`, `LF_UNDER 1.0.1`, `LF_WEEKLY_HOURS` all extend to `2026-10`), whereas monthly **`LF_EDU(1.0.0)` stopped after `2026-03` with no `CEASED` prefix and no version change**.
3. **Experimental-to-headline evolution without `CEASED` on old minor versions (`RT` → `HSI_M`):** Retail Trade `RT(1.0.0)` carries `"CEASED - "` (ended `2025-06`), while its successor `HSI_M` evolved across `1.0.0` → `1.4.0` → `1.6.0` where each older minor version froze silently without `CEASED` text.

## 4. What remains ambiguous

1. **The reference period of the 18 `CPI 2.0.0` monthly SA series.** The data equal `100` at `2024-04`, the attribute says `Sep 2025 = 100`, and neither [CPI Methodology, Sep 2026](https://www.abs.gov.au/methodologies/consumer-price-index-australia-methodology/sep-2026) (which states `September 2025 = 100.00` for monthly and quarterly CPI) nor the [seasonal adjustment paper](https://www.abs.gov.au/about/key-priorities/big-data-timely-insights-phase-2/complete-monthly-measure-cpi/seasonal-adjustment-new-complete-monthly-consumer-price-index-cpi) explicitly documents an April 2024 base for those 18 series. It remains unclear whether the attribute or the data will be corrected.
2. **Which `CL_BASE_PERIOD` labels are intended.** v1.1.0 and v2.0.0 disagree on code `25`, and the ABS change log is self-contradictory. I inferred from the data that v1.1.0's "Sep 2025" is correct for `CPI 2.0.0`.
3. **The quarterly trimmed mean from 2025-Q4 to 2026-Q1.** The re-referencing happened from the December 2025 release (January 2026), and `CPI_Q` was reinstated in April 2026. `CPI_Q` now holds 2025-Q4 and 2026-Q1, but I cannot tell from today's API whether those quarters could be reached through the API before April 2026.
4. **Why `CPI_Q` and `CPI 1.1.0` trimmed mean / weighted median diverge back to `1982-Q1`.** While `999901` All groups SA and expenditure classes show only post-`2020-Q4` seasonal-factor revisions, `999902` and `999903` diverge throughout `1982–2002` (up to `0.50` index points / `0.3 pp` YoY). The ABS papers read explain the `2025-Q4` Child care/Insurance source change, not the `1982–2002` shift.
5. **How long old versions stay available.** `CPI 1.1.0`, `HSI_M 1.0.0/1.4.0`, `PPI_FD 1.0.0`, and `LABOUR_ACCT_Q 1.0.0` are still served, while `WPI 1.0.0/1.1.0` are gone. No ABS retention policy was found; the updates page describes none.
6. **When `CPI_Q` will end.** The [quarterly trimmed mean explanatory note](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/quarterly-trimmed-mean-pre-october-2025-compilation-basis-explanatory-note) commits to publishing it "through to at least the June quarter 2027", with no final end date announced.
7. **Post-transition Labour Force changes & Indicator API continuity.** `LF(1.0.0)` remained intact through `2026-10`, whereas `LF_EDU(1.0.0)` silently stopped after `2026-03` and the mid-2026 post-transition article cited in the March 2026 paper has not been checked. Separately, the production Indicator API (`indicator.api.abs.gov.au`) remains untested without an API key (so whether `CPI_H`, `CPI_Q_H`, or `GDPE_H` share `CPI 2.0.0`'s `BASE_PERIOD` bug or freeze behaviour is unknown).

## 5. What Phase 1 should do

These recommendations add to the Phase 1 architecture in [abs-api-research.md §9](abs-api-research.md#9-suggested-poc-architecture); they don't replace it.

1. **Series identity = (agency, dataflow ID, exact version string, full series key).** Store all four on every row and treat version strings as opaque (not parsed semver, since `LABOUR_ACCT_Q` uses `1.1`). Never send an unversioned request (F1). Never treat the same key in a different dataflow, or a different version, as the same series by default (F2, F3).
2. **Resolve labels against the exact codelist the pinned DSD references.** Fetch the structure with `references=all` for the pinned version and resolve against that response only. Key cached codelists strictly by `(codelist_id, codelist_version)`—never store a single "latest" `CL_BASE_PERIOD` globally, because live `WPI 1.2.0` (`1.0.0`) and `CPI 2.0.0` (`1.1.0`) resolve code `25` to different text in the same pipeline (F4, F5). Record the codelist URN (ID and version) with each label in provenance metadata.
3. **Keep codes as the stored value and labels as derived display text.** Diffs and joins use codes. A label change is logged, not treated as a new series (F4).
4. **Run a structural check before each fetch:**
   - Compare `GET /dataflow/ABS/{id}/all?detail=allstubs` with the pinned version (alert on **any** new major or minor version, because superseded minor versions like `HSI_M 1.0.0/1.4.0` freeze silently with no `CEASED` prefix), and compare the DSD's referenced codelist IDs and versions with the stored snapshot.
   - Check the description for a `CEASED` prefix.
   - On any difference, **alert and keep fetching the pinned version**. Never switch automatically.
   - Don't use `NonProductionDataflow` as a status signal, and don't treat `dataConstraints` (`CR_A`/`CR_B`) as sole freshness proof (F7).
5. **Treat `404 NoRecordsFound` on a release day as unresolved, and alert on multi-cycle staleness** (F7, F8). If the structure check shows `CEASED`, record the series as ended; otherwise retry within the release-day polling window, and raise a **staleness alert** if `max(TIME_PERIOD)` falls behind `> 1` scheduled release cycle so silent stops like `LF_EDU(1.0.0)` are surfaced promptly.
6. **Keep revisions within one version—and add an in-place rebase detector.** Key the §9 value comparison on the full identity from item 1 so cross-version rebases (`CPI 1.1.0 → 2.0.0`) don't flood revision logs (F2). Within a single pinned version, flag a bulk historical shift (`> 50%` of periods changing at near-constant ratio, or `ANA_AGG` `Original` `sum(M1.GPM) == sum(M3.GPM)` jumping to a new FY) as an **in-place rebase** rather than thousands of per-observation revisions (F8).
7. **Add a small hand-maintained lineage file** in `config/`, next to the dataflow list, for each concept the RBA tracks. Each link records the predecessor, the successor, the relation type (`rebase`, `break`, `replacement`, `split`), per-series conversion factor where applicable (e.g., `0.6945` applies to Australia All groups, not cities or components), the evidence, and the date. Seed it with the CPI case, for example:

   | Concept | From | To | Relation | Note |
   | :--- | :--- | :--- | :--- | :--- |
   | CPI All groups, Q, original | `CPI 1.1.0` `1.10001.10.50.Q` | `CPI 2.0.0` `1.10001.10.50.Q` | rebase (factor `0.6945`) | From 2025-Q4, quarterly is the mean of monthly |
   | CPI All groups, M | `CPI_M 1.2.0` `1.10001.10.50.M` | `CPI 2.0.0` `1.10001.10.50.M` | break | Indicator vs complete CPI; overlap 2024-04 to 2025-09 |
   | Trimmed mean, Q | `CPI 1.1.0` `1.999902.20.50.Q` | `CPI_Q 1.0.0` `1.999902.20.50.Q` | replacement | 2011-12 basis; through at least `2027-Q2`; `2025-Q4+` updates Child care/Insurance sources |
   | Trimmed mean, M | (none) | `CPI 2.0.0` `1.999902.20.50.M` | new series | Starts 2024-04; data equal 100 at 2024-04 |

   The pipeline only *reads* this file, to label outputs and to warn. Splicing or chain-linking is left to economists and deferred past Phase 1.
8. **Record the reference period using base-type-aware invariants instead of a naive `v == 100` scan.** Store `BASE_PERIOD` as delivered (or `"not stated in API"` where absent), plus an explicit invariant check matched to the dataset's base convention (F6, F8):
   - *Point-in-time month bases (`CPI 2.0.0`):* assert `v[2025-09] == 100.0` on `TSEST=10` Original series, and flag `TSEST=20` series where `v[first_period] == 100.0` (`2024-04`) while `BASE_PERIOD=25` (quarantine `BASE_PERIOD` on those 18 series).
   - *Financial-year average bases (`CPI_Q 1.0.0`, `WPI 1.2.0`, `ANA_AGG` Terms of Trade):* check that the 4-quarter FY mean equals `100.0` (e.g., `2011-12` for `CPI_Q`, `2008-09` for `WPI`).
   - *Chain-volume dollar levels (`ANA_AGG 1.0.0`):* infer the active CVM reference year each release from the `Original` (`TSEST=10`) minimum of `abs(sum(M1.GPM) - sum(M3.GPM))` across Australian financial years.

## 6. Reproduction

Requires **R >= 4.1** (for the base pipe `|>` and base `utils::read.csv(..., colClasses = "character", na.strings = character(0))` so coded dimensions and blank `BASE_PERIOD` attributes stay uncoerced) plus two lightweight CRAN dependencies:

- **`httr2`**: handles HTTP timeouts and captures `404 NoRecordsFound` responses cleanly without aborting (`req_error(is_error = \(resp) FALSE)`).
- **`jsonlite`**: parses SDMX-JSON 2.0.0 dataflow metadata (`fromJSON(..., simplifyVector = FALSE)`) for `F8`/`9.` ceased-prefix checks.

Install dependencies once and run the script from a shell:

```r
install.packages(c("httr2", "jsonlite"))
```

```sh
Rscript --vanilla repro.R
```

Output on 2026-11-23 UTC (verified live on `R 4.4.2`, `httr2 1.0.7`, `jsonlite 1.8.9`, matching all 9 checks and 19 output lines of the original live run):

```text
1. 309 quarters 1948-Q3..2025-Q3; k in [0.694489, 0.694509]; pure rebase: TRUE
2. from 1948: 96/308 quarterly % changes differ by >= 0.15 pp
2. from 1980: 11/183 quarterly % changes differ by >= 0.15 pp
2. from 2000: 0/103 quarterly % changes differ by >= 0.15 pp
3. CPI_M vs CPI monthly 10001: ratio 0.7771..0.7820 (constant => rebase)
4. All groups: BASE_PERIOD=25, equals 100 at [2025-09]
4. Trimmed mean: BASE_PERIOD=25, equals 100 at [2024-04]
5. CPI_M after Sep 2025: HTTP 404
5. WPI 1.0.0: HTTP 404
6. CPI 2.0.0 M=1 TSEST=10 multi-100=12/158; TSEST=20 never-100=50/105, apr24==100=18, wrong-single-100=[114122, 40055, 40083]
7. 999901 All groups SA vs CPI_Q: index_diffs=9/156 (2020-Q4..2025-Q2, max=0.1000)
7. 999902 Trimmed mean vs CPI_Q: index_diffs=162/175 (1982-Q1..2025-Q3, max=0.2978)
7. 999903 Weighted median vs CPI_Q: index_diffs=133/175 (1982-Q1..2025-Q3, max=0.5012)
8. ANA_AGG 1.0.0 Original sum(M1-M3) by FY: {2022-23: 66702.0, 2023-24: 0.0, 2024-25: -61970.0}
8. WPI 1.2.0 BASE_PERIOD="", exact_100=[], FY08-09 mean=100.0000
9. HSI_M 1.0.0: ceased_prefix=FALSE, max_period=2024-05
9. HSI_M 1.6.0: ceased_prefix=FALSE, max_period=2026-09
9. LF_EDU 1.0.0: ceased_prefix=FALSE, max_period=2026-03
9. LF 1.0.0: ceased_prefix=FALSE, max_period=2026-10
```

```r
# repro.R — Reproduce the CPI and cross-flow continuity checks (R >= 4.1).
library(httr2)
library(jsonlite)

BASE <- "https://data.api.abs.gov.au/rest/"

get_csv <- function(path) {
  resp <- request(paste0(BASE, "data/", path)) |>
    req_timeout(60) |>
    req_error(is_error = function(resp) FALSE) |>
    req_perform()
  status <- resp_status(resp)
  if (status != 200L) {
    return(sprintf("HTTP %d", status))
  }
  read.csv(
    text = resp_body_string(resp),
    colClasses = "character",
    na.strings = character(0),
    stringsAsFactors = FALSE,
    check.names = FALSE
  )
}

get_json <- function(path) {
  resp <- request(paste0(BASE, path)) |>
    req_timeout(60) |>
    req_perform()
  fromJSON(resp_body_string(resp), simplifyVector = FALSE)
}

as_series <- function(df) {
  keep <- nzchar(df$OBS_VALUE)
  vals <- as.numeric(df$OBS_VALUE[keep])
  names(vals) <- df$TIME_PERIOD[keep]
  vals
}

old <- as_series(get_csv("ABS,CPI,1.1.0/1.10001.10.50.Q?format=csv"))   # 2011-12 = 100, 1 dp
new <- as_series(get_csv("ABS,CPI,2.0.0/1.10001.10.50.Q?format=csv"))   # Sep 2025 = 100, 2 dp
common <- sort(intersect(names(old), names(new)), method = "radix")

# 1. Pure rebase test: is there one factor k that fits every quarter within rounding?
lo <- max((new[common] - 0.005) / (old[common] + 0.05))
hi <- min((new[common] + 0.005) / (old[common] - 0.05))
cat(sprintf(
  "1. %d quarters %s..%s; k in [%.6f, %.6f]; pure rebase: %s\n",
  length(common), common[1], common[length(common)], lo, hi, lo <= hi
))

# 2. Published quarterly % changes (MEASURE=2) before and after re-referencing
p_old <- as_series(get_csv("ABS,CPI,1.1.0/2.10001.10.50.Q?format=csv"))
p_new <- as_series(get_csv("ABS,CPI,2.0.0/2.10001.10.50.Q?format=csv"))
p_common <- intersect(names(p_old), names(p_new))
for (start in c("1948", "1980", "2000")) {
  cc <- p_common[p_common >= start]
  n <- sum(abs(p_old[cc] - p_new[cc]) >= 0.15)
  cat(sprintf("2. from %s: %d/%d quarterly %% changes differ by >= 0.15 pp\n", start, n, length(cc)))
}

# 3. Same INDEX code, different series: Monthly CPI Indicator vs complete Monthly CPI
ind <- as_series(get_csv("ABS,CPI_M,1.2.0/1.10001.10.50.M?format=csv&startPeriod=2024-04"))
cmp_ <- as_series(get_csv("ABS,CPI,2.0.0/1.10001.10.50.M?format=csv&endPeriod=2025-09"))
m_common <- sort(intersect(names(ind), names(cmp_)), method = "radix")
ratios <- cmp_[m_common] / ind[m_common]
cat(sprintf(
  "3. CPI_M vs CPI monthly 10001: ratio %.4f..%.4f (constant => rebase)\n",
  min(ratios), max(ratios)
))

# 4. BASE_PERIOD attribute vs where the series actually equals 100
meta_checks <- list(
  list(code = "10001", tsest = "10", name = "All groups"),
  list(code = "999902", tsest = "20", name = "Trimmed mean")
)
for (mc in meta_checks) {
  rows <- get_csv(sprintf("ABS,CPI,2.0.0/1.%s.%s.50.M?format=csv", mc$code, mc$tsest))
  s <- as_series(rows)
  at100 <- sort(names(s)[s == 100], method = "radix")
  cat(sprintf(
    "4. %s: BASE_PERIOD=%s, equals 100 at [%s]\n",
    mc$name, rows$BASE_PERIOD[1], paste(at100, collapse = ", ")
  ))
}

# 5. Ceased vs not-yet-published both look like 404
cat(sprintf("5. CPI_M after Sep 2025: %s\n", get_csv("ABS,CPI_M,1.2.0/1.10001.10.50.M?format=csv&startPeriod=2025-10")))
cat(sprintf("5. WPI 1.0.0: %s\n", get_csv("ABS,WPI,1.0.0/all?format=csv&startPeriod=2026-Q2&endPeriod=2026-Q2")))

# 6. Stress-test v == 100 across all CPI 2.0.0 M=1, REGION=50, FREQ=M series
cpi_m1 <- get_csv("ABS,CPI,2.0.0/1...50.M?format=csv")
cpi_m1 <- cpi_m1[nzchar(cpi_m1$OBS_VALUE), , drop = FALSE]
cpi_m1$VAL <- as.numeric(cpi_m1$OBS_VALUE)

t10_list <- split(cpi_m1[cpi_m1$TSEST == "10", , drop = FALSE], cpi_m1$INDEX[cpi_m1$TSEST == "10"])
t10_multi <- sum(vapply(t10_list, function(d) sum(d$VAL == 100) > 1L, logical(1)))

t20_list <- split(cpi_m1[cpi_m1$TSEST == "20", , drop = FALSE], cpi_m1$INDEX[cpi_m1$TSEST == "20"])
t20_at100 <- lapply(t20_list, function(d) sort(d$TIME_PERIOD[d$VAL == 100], method = "radix"))
t20_never <- sum(lengths(t20_at100) == 0L)
t20_apr24 <- sum(vapply(t20_list, function(d) any(d$TIME_PERIOD == "2024-04" & d$VAL == 100), logical(1)))
t20_wrong1 <- sort(
  names(t20_at100)[vapply(t20_at100, function(ps) length(ps) == 1L && !(ps %in% c("2024-04", "2025-09")), logical(1))],
  method = "radix"
)
cat(sprintf(
  "6. CPI 2.0.0 M=1 TSEST=10 multi-100=%d/%d; TSEST=20 never-100=%d/%d, apr24==100=%d, wrong-single-100=[%s]\n",
  t10_multi, length(t10_list), t20_never, length(t20_list), t20_apr24, paste(t20_wrong1, collapse = ", ")
))

# 7. CPI 1.1.0 vs CPI_Q 1.0.0 across 999901, 999902, 999903
q_checks <- list(
  list(code = "999901", name = "All groups SA"),
  list(code = "999902", name = "Trimmed mean"),
  list(code = "999903", name = "Weighted median")
)
for (qc in q_checks) {
  o_i <- as_series(get_csv(sprintf("ABS,CPI,1.1.0/1.%s.20.50.Q?format=csv", qc$code)))
  n_i <- as_series(get_csv(sprintf("ABS,CPI_Q,1.0.0/1.%s.20.50.Q?format=csv", qc$code)))
  cc_i <- sort(intersect(names(o_i), names(n_i)), method = "radix")
  diffs <- abs(n_i[cc_i] - o_i[cc_i])
  hit <- cc_i[diffs > 1e-9]
  cat(sprintf(
    "7. %s %s vs CPI_Q: index_diffs=%d/%d (%s..%s, max=%.4f)\n",
    qc$code, qc$name, length(hit), length(cc_i), hit[1], hit[length(hit)], max(diffs[diffs > 1e-9])
  ))
}

# 8. In-place FY reference years: ANA_AGG 1.0.0 CVM identity & WPI 1.2.0 FY-mean base
ana <- get_csv("ABS,ANA_AGG,1.0.0/M1+M3.GPM.10.AUS.Q?format=csv&startPeriod=2022-Q3&endPeriod=2026-Q2")
ana <- ana[nzchar(ana$OBS_VALUE), , drop = FALSE]
ana_m1 <- as_series(ana[ana$MEASURE == "M1", , drop = FALSE])
ana_m3 <- as_series(ana[ana$MEASURE == "M3", , drop = FALSE])
fy_labels <- character()
for (fy in 2022:2024) {
  qs <- c(sprintf("%d-Q3", fy), sprintf("%d-Q4", fy), sprintf("%d-Q1", fy + 1L), sprintf("%d-Q2", fy + 1L))
  fy_diff <- round(sum(ana_m1[qs] - ana_m3[qs]), 1)
  fy_labels <- c(fy_labels, sprintf("%d-%02d: %.1f", fy, (fy + 1L) %% 100L, fy_diff))
}
cat(sprintf("8. ANA_AGG 1.0.0 Original sum(M1-M3) by FY: {%s}\n", paste(fy_labels, collapse = ", ")))

wpi <- get_csv("ABS,WPI,1.2.0/1.THRPEB.7.TOT.10.AUS.Q?format=csv")
wpi_s <- as_series(wpi)
wpi_100 <- sort(names(wpi_s)[wpi_s == 100], method = "radix")
wpi_0809 <- mean(wpi_s[c("2008-Q3", "2008-Q4", "2009-Q1", "2009-Q2")])
cat(sprintf(
  "8. WPI 1.2.0 BASE_PERIOD=\"%s\", exact_100=[%s], FY08-09 mean=%.4f\n",
  wpi$BASE_PERIOD[1], paste(wpi_100, collapse = ", "), wpi_0809
))

# 9. Silent freezes & stops without CEASED: HSI_M 1.0.0 vs 1.6.0, LF_EDU 1.0.0 vs LF 1.0.0
stop_checks <- list(
  list(fid = "HSI_M", ver = "1.0.0", qs = "all?format=csv&lastNObservations=1"),
  list(fid = "HSI_M", ver = "1.6.0", qs = "all?format=csv&lastNObservations=1"),
  list(fid = "LF_EDU", ver = "1.0.0", qs = "all?format=csv&startPeriod=2025-01"),
  list(fid = "LF", ver = "1.0.0", qs = "M13.3.1599.20.AUS.M?format=csv&startPeriod=2025-01")
)
for (sc in stop_checks) {
  df_meta <- get_json(sprintf("dataflow/ABS/%s/%s?format=json-structure-2.0.0", sc$fid, sc$ver))$data$dataflows[[1]]
  desc <- if (is.null(df_meta$description)) "" else df_meta$description
  ceased <- startsWith(desc, "CEASED")
  s <- as_series(get_csv(sprintf("ABS,%s,%s/%s", sc$fid, sc$ver, sc$qs)))
  max_p <- tail(sort(names(s), method = "radix"), 1L)
  cat(sprintf("9. %s %s: ceased_prefix=%s, max_period=%s\n", sc$fid, sc$ver, ceased, max_p))
}
```

The structure comparisons in F4, F5, F7, and F8 were run against these endpoints:

```
GET /dataflow/ABS/{ID}/all?format=json-structure-2.0.0
GET /dataflow/ABS/{CPI|CPI_M|CPI_Q|...}/{ver}?references=all&detail=full   (or detail=referencepartial)
GET /codelist/ABS/{CL_BASE_PERIOD|CL_CPI_INDEX|CL_CPI_INDEX_17|CL_CPI_MEASURES|CL_CPI_REGION}/all
GET /codelist/ABS/CL_BASE_PERIOD/1.0.0?references=parents&detail=allstubs   → used by 9 DSDs incl. WPI 1.0.0/1.2.0
GET /codelist/ABS/CL_BASE_PERIOD/1.1.0?references=parentsandsiblings&detail=allstubs   → used by DSD CPI 2.0.0
GET /codelist/ABS/CL_BASE_PERIOD/2.0.0?references=parents&detail=allstubs             → used by DSD CPI_M 1.0.0, 1.1.0, 1.2.0
```

## 7. Notes for reconciling with `abs-api-research.md`

The README says a new probe that changes a fact should be applied to the research notes first. This draft was scoped to one file, so these are listed here for the cross-review and **have not been applied**:

| Section | Current text | Evidence from this draft |
| :--- | :--- | :--- |
| [§9](abs-api-research.md#9-suggested-poc-architecture) | `WPI(1.0.0)` | Only WPI 1.2.0 exists, and 1.0.0 returns 404 (F1). §5 already says 1.2.0. |
| [§4](abs-api-research.md#4-cpi-dataflows--corrected-status-verified-live-2026-11-23) | CPI_M "superseded by CPI(2.0.0) which covers the same monthly data" | Not the same data. It is a different statistic, with month-on-month gaps of up to 0.52 pp (F3). |
| §4 | "Use `CPI_Q(1.0.0)` for dedicated quarterly tables" | `CPI_Q` holds only seasonally adjusted Australia series (`TSEST=20`) on the 2011-12 basis, with no original All-groups series. Headline quarterly CPI is `CPI 2.0.0` FREQ=Q (F3, F6). |
| §4 | CPI 2.0.0 historical depth "1972+" | Quarterly All groups starts 1948-Q3. Monthly starts 2024-04. |
| §4, §5 | "Base: September 2025 = 100" | True for the headline series. Not true for 18 monthly SA series (groups, goods/services, tradables/non-tradables, All groups SA, trimmed mean, weighted median — all April 2024 = 100 in the data) (F6). |
| [§8.9](abs-api-research.md#8-known-limitations--pitfalls), §5 | `NonProductionDataflow` "present on CPI_M, BUSINESS_TURNOVER and RT"; "flag present — verify currentness" | Present on **15 / 15** audited dataflows including live `CPI 2.0.0`, `CPI_Q 1.0.0`, `LF 1.0.0`, `ANA_AGG 1.0.0`, `WPI 1.2.0`, and `BOP 1.0.0` (which is current through `2026-Q2`). It carries no currentness signal (F7). |
| §5, §11 log | Flows listed with a single version (`CPI`, `PPI_FD`, `LABOUR_ACCT_Q`, `HSI_M`) | Five audited flows retain multiple live versions (`CPI 1.1.0/2.0.0`, `HSI_M 1.0.0/1.4.0/1.6.0`, `PPI_FD 1.0.0/1.1.0`, `LABOUR_ACCT_Q 1.0.0/1.1`, `BUSINESS_TURNOVER 1.0.0/1.2.0/1.3.0`); superseded versions freeze silently without `CEASED` (F1). |
| §5 (`LF`) | Labour Force listed without transition caveats | Headline `LF 1.0.0` stayed continuous (`744/744` keys) through `2026-10`, whereas `LF_EDU 1.0.0` silently stopped after `2026-03` during the April–August 2026 survey transition (F8). |

## 8. Sources

**Primary: ABS Data API** (live, 2026-11-23 UTC). These are the endpoints in [§6](#6-reproduction).

**Primary: ABS web pages and workbooks**

- [Data API updates](https://www.abs.gov.au/statistics/application-programming-interfaces-apis/data-api-user-guide/updates): November 2025 entry (CPI 1.1.0 → 2.0.0, CPI_M discontinued, base Sep 2025 = 100, codelist changes), April 2026 entry (CPI_Q reinstated), and May 2026 entry (Labour Force modernisation).
- [Re-referencing the quarterly CPI](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/re-referencing-quarterly-consumer-price-index) (28/01/2026): earlier 2011-12 = 100 reference at 1 dp, movements unrevised "aside from minor impacts of rounding", quarterly as the average of monthly from the December quarter 2025, and SA series kept on 2011-12.
- [Quarterly CPI - September 2025 Conversion Factors (.xlsx)](https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/consumer-price-index-australia/sep-2026/Quarterly%20CPI%20-%20September%202025%20Conversion%20Factors%20.xlsx) (28/01/2026): official series-level conversion factors from `2011-12` to `September 2025` (`0.6945` for `A2325846C`, varying by capital city and expenditure class).
- [Introducing the CPI's new monthly time series](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/introducing-consumer-price-indexes-new-monthly-time-series) (18/11/2025): re-referenced to September 2025 = 100.00, series back to April 2024, Indicator "rolled forward" prices, 50 % vs 87 % monthly-priced weight. *Read in part.*
- [Extended analysis and trimmed mean](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/introducing-consumer-price-indexs-new-monthly-time-series-extended-analysis-and-trimmed-mean) (26/11/2025): monthly trimmed mean back to April 2024, quarterly trimmed mean on the pre-October 2025 basis "for at least 18 months".
- [Quarterly Trimmed mean (pre-October 2025 compilation basis) explanatory note](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/quarterly-trimmed-mean-pre-october-2025-compilation-basis-explanatory-note): confirms continuation through at least June quarter 2027 and documents `2025-Q4+` new monthly data sources for Child care, Long day care fees, and Insurance.
- [Seasonal adjustment in the new complete Monthly CPI](https://www.abs.gov.au/about/key-priorities/big-data-timely-insights-phase-2/complete-monthly-measure-cpi/seasonal-adjustment-new-complete-monthly-consumer-price-index-cpi): monthly price collection from April 2024 with spliced pre-April 2024 backcasts for seasonal adjustment; does not specify an April 2024 = 100 base for the 18 monthly SA series.
- [CPI methodology, Sep 2026](https://www.abs.gov.au/methodologies/consumer-price-index-australia-methodology/sep-2026): concurrent seasonal adjustment revisions and statement that monthly and quarterly CPI data use `September 2025 = 100.00`.
- [CPI latest release](https://www.abs.gov.au/statistics/economy/price-indexes-and-inflation/consumer-price-index-australia/latest-release) (September 2026 edition): "Index reference period: September 2025 = 100.0."
- [Upcoming changes to Labour Force Survey outputs](https://www.abs.gov.au/statistics/detailed-methodology-information/information-papers/upcoming-changes-labour-force-survey-outputs) (19/03/2026): April–August 2026 survey transition, continuity of headline series, and paused/retired outputs (including educational items).

**Remaining unread / unverified follow-up:** the mid-2026 post-transition Labour Force article referenced in the March 2026 paper, the remainder of the November 2025 introducing paper, and the key-gated Indicator API (`indicator.api.abs.gov.au`).
