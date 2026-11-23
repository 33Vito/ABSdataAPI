# ABSdataAPI

> ABS automated data ingestion PoC for economists at the Reserve Bank of Australia.

Read [README.md](README.md) for the project overview, repository layout, naming rules and evidence rules. Read [AGENTS.md](AGENTS.md) for the rules that apply to all agents. This file only adds instructions specific to Claude.

## Model

Use `claude-sonnet-5-5` for all coding in this project (pinned, not alias).

## Project context

Goal: build a Python pipeline that polls the ABS Data API (free, keyless SDMX 2.1 REST) for RBA-core economic indicators (CPI, Labour Force, National Accounts, WPI, International Trade, etc.), resolves codelist codes to readable labels, and writes structured CSV/Parquet with provenance metadata.

Operational constraints (API base URL, release timing, gateway limits, which claims are verified) are kept in one place, the [README "Evidence rules"](README.md#evidence-rules) section. Follow it, and don't copy those facts into this file.

## Files

- `docs/research/abs-api-research.md`: ABS API implementation evidence (verified dataflows §5, Phase 1 architecture §9)
- `docs/research/central-bank-ingestion/report.md`: design rationale from five central banks
- `docs/research/central-bank-ingestion/notes/`: source notes for the report
