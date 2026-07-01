---
unique-name: readme
display-name: README
category: GENERAL
description: SI source vs NFS final reconciliation queries for the collections (incassi) migration.
---

# Reconciliation Query Suite

SI source vs NFS final reconciliation queries for the collections (incassi) migration.

Covers **19 migration tables** across 4 NFS target schemas with 28 SQL files and documentation.

## Getting Started

| Guide | Purpose |
|-------|---------|
| **[EXECUTION_GUIDE.md](EXECUTION_GUIDE.md)** | **Start here.** Full step-by-step instructions: parameter resolution, execution order, country escalation, performance tuning. |
| [TRIAGE_GUIDE.md](TRIAGE_GUIDE.md) | Interpreting results, investigating discrepancies, handling orphan records. |
| [COVERAGE_CHECK.md](COVERAGE_CHECK.md) | Coverage validation matrix confirming all 19 tables are covered at appropriate tiers. |

## Directory Structure

```
reconciliation/
  00_table_mapping.sql       -- Master SI-to-NFS mapping reference (19 tables)
  tier1-row-counts/          -- Row count comparisons: SI vs staging vs NFS final (9 files)
  tier2-aggregates/          -- Financial aggregate comparisons: SUM, MIN/MAX, COUNT DISTINCT (6 files)
  tier3-minus/               -- Record-level MINUS for discrepancy detection (5 files)
  integrity/                 -- Cross-schema orphan detection: FK integrity validation (4 files)
```

## Quick Parameter Reference

All queries use three parameters. See [EXECUTION_GUIDE.md](EXECUTION_GUIDE.md) Section 3 for full resolution instructions.

| Parameter | What | Example |
|-----------|------|---------|
| `:req_id` | Migration request ID from `MGR_REQUEST.REQ_NM_ID` | `123` |
| `:len_id` | Legal entity ID: IT=41, ES=44, PT=45, GR=47, SK=52/53 | `41` |
| `{dblink}` | DB link: IT=FACT_BFF, GR=FACT_BGR, PT=FACT_BPT, SK=FACT_BSK | `FACT_BFF` |

## Project Context

This suite was generated as part of the **NFS Migration Analysis** project (Phases 1 and 2):
- **Phase 1:** Core reconciliation queries (Tier 1/2/3) -- row counts, aggregates, record-level MINUS
- **Phase 2:** Integrity validation (cross-schema orphan detection) and execution documentation