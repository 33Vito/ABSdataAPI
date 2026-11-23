# ABSdataAPI

Proof of concept for automated ingestion of Australian Bureau of Statistics (ABS) data for Reserve Bank of Australia (RBA) economists.

The pipeline will poll the ABS Data API, a free and keyless SDMX 2.1 REST service at `https://data.api.abs.gov.au/rest/`, for RBA-core indicators such as CPI, Labour Force, National Accounts, WPI and International Trade. It will resolve codelist codes to readable labels and write CSV/Parquet output with provenance metadata.

**Status:** research phase. There is no pipeline code yet.

## Start here

| Document | Role |
| :--- | :--- |
| [ABS API research notes](docs/research/abs-api-research.md) | **Implementation evidence.** Live-tested endpoints, verified dataflows (§5), release timing (§6), pitfalls (§8) and the Phase 1 architecture (§9). |
| [Central bank ingestion report](docs/research/central-bank-ingestion/report.md) | **Design rationale.** How the RBA, Fed, Bundesbank and BoK (with the BoE as a benchmark) ingest national statistics, and why that supports a bounded, auditable pull. |
| [Research notes index](docs/research/central-bank-ingestion/report.md#research-notes) | The five cited source notes behind the report. |

## Evidence rules

Only claims that the research notes verified count as facts. Everything else stays labelled as unverified.

- **Verified:** the base URL is `https://data.api.abs.gov.au/rest/` and there is no auth key. The API uses SDMX 2.1. The ABS gateway documents a 30-second timeout and a 10 MB response limit. The verified dataflow IDs and versions are listed in §5 of the ABS notes.
- **Release time:** under the ABS [Policy on Pre-Release Access](https://www.abs.gov.au/about/legislation-and-policy/policy-pre-release-access-abs-statistics), the website generally releases at 11:30am Canberra time, and the Indicator API publishes when the embargo lifts.
- **Not guaranteed:** the ABS doesn't document when data reaches the Beta Data API after the embargo lifts, so that time varies. "~12:30pm" is a community assumption. Poll from 12:00pm Canberra time on release days and accept variable latency.
- **Unverified:** the 1M-observation cap per request and the "CSV required above 100K observations" rule come from community repositories. Don't treat either as a hard limit. Size requests to the documented gateway limits and test them empirically.
- **Unreliable:** don't use `updatedAfter` in production. Use bounded `startPeriod`/`endPeriod` slices and value comparison instead.
- Candidate dataflow IDs must be checked against the metadata endpoint before use.

When a new probe changes one of these facts, update the ABS research notes first. Then update this section, `CLAUDE.md`, and any config that copies the fact.

## Repository layout

```
README.md                  Human entry point; canonical layout, naming and evidence rules
AGENTS.md                  Short instructions for coding agents
CLAUDE.md                  Claude Code–specific instructions
docs/
  research/
    abs-api-research.md    ABS API implementation evidence
    central-bank-ingestion/
      report.md            Cross-jurisdiction design rationale
      notes/               Source notes for the report, one per jurisdiction or theme
```

### Planned Python layout (not created yet)

Add these when code is first written. Don't create empty placeholders before then.

```
pyproject.toml             Package metadata and dependencies
src/absdataapi/            Pipeline package (fetch, codelist resolution, storage, provenance)
tests/                     Test suite
config/                    Dataflow list (verified IDs and versions from ABS notes §5), not hardcoded
outputs/                   Generated CSV/Parquet output (gitignored)
```

## Naming and placement

- File and directory names use lowercase kebab-case (`south-korea-bok.md`). They contain no spaces, capitals or underscores. Python modules are the exception and follow PEP 8 (`snake_case.py`).
- Research and reference material goes under `docs/research/`. A multi-file study gets its own directory with `report.md` and a `notes/` folder.
- Generated data goes in `outputs/` and is never committed.
- Use relative links between documents. After you move or rename a file, update every link that points to it and check that each one still resolves, including `#section` anchors.
