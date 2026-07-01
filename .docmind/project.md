---
unique-name: project
display-name: PROJECT
category: GENERAL
description: Comprehensive analysis and remediation of the SI-to-NFS data migration package for the collections (incassi) perimeter. The migration is already implemented for ITA, GR, and PT but suffers from perfor
---

# NFS Migration Analysis & Fix

## What This Is

Comprehensive analysis and remediation of the SI-to-NFS data migration package for the collections (incassi) perimeter. The migration is already implemented for ITA, GR, and PT but suffers from performance issues, data population errors, and insufficient reconciliation. This project produces analysis reports with working fix scripts (SQL/PL-SQL) to resolve all three problem areas.

## Core Value

Every record migrated from SI to NFS must be correct and verifiable — data integrity is non-negotiable.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Performance analysis: identify all causes of slow migration (indexes, statistics, MAX-based sequences, query plans)
- [ ] Performance fix scripts: create index DDLs, sequence replacements, and statistics gathering procedures
- [ ] Data error analysis: audit all migration table population logic for incorrect mappings, wrong sequences, missing joins
- [ ] Data error fix scripts: corrected PL/SQL procedures for each identified error (including the 605/901 Payment Notice fix)
- [ ] Reconciliation queries: SI source vs NFS final for all migration tables (end-to-end row counts, key field comparisons, aggregate checks)
- [ ] CZ&SK cross-cutting notes: flag any issues that will also affect the upcoming CZ&SK migration

### Out of Scope

- CZ&SK migration implementation — separate future phase, only cross-cutting notes here
- NFS staging-to-NFS final reconciliation — existing queries already cover this
- Live database execution — code analysis only, scripts delivered for team to run
- Migration for non-incassi perimeters

## Context

**Existing codebase analysis**: `.planning/codebase/` contains the overall NFS and SI codebase mapping (at `C:\Desktop\BFF\Code Review\.planning\codebase`).

**Migration package source**: `C:\Desktop\BFF\Code Review\NFS\Migrazione` contains the actual PL/SQL migration code.

**CZ&SK analysis**: `C:\Desktop\BFF\Code Review\Migrazione` contains the CZ&SK migration analysis (not yet implemented).

**Key volume tables (ITA)**:
| Table | Row Count |
|-------|-----------|
| CONTABILE_FF | 362,738 |
| MANDATI | 325,583 |
| PAREGGIO_CED_DEB | 722,309 |
| PAREGGIO_FAT_ATT | 23,830,849 |
| PAREGGIO_NON_FAT | 121,072 |
| FATTURA_ATTIVA | 321,305 |
| RATA_CED_DEB | 41,227,963 |

**Known issues**:
- Missing indexes and table statistics
- MAX-based ID generation instead of Oracle sequences (causes contention and full table scans)
- Payment Notice detail: query uses sequence 605 instead of 901 for invoice lookup
- Reconciliation queries only compare NFS staging vs NFS final, not SI source vs NFS final

**Data models**: SI and NFS data models available in the codebase analysis.

## Constraints

- **Access**: Code analysis only — no live/test database access. Scripts must be delivered for team execution.
- **Scope**: Collections (incassi) perimeter only — ITA/GR/PT migration code.
- **Deliverables**: All outputs go to `C:\Desktop\BFF\Code Review\Migrazione`.
- **Database**: Oracle PL/SQL environment.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| All three problem areas equally prioritized | User confirmed no single area takes precedence | — Pending |
| SI source vs NFS final reconciliation | Most valuable comparison — end-to-end data integrity verification | — Pending |
| All migration tables analyzed (not just high-volume) | Comprehensive coverage preferred to catch all population errors | — Pending |
| CZ&SK as separate phase | Keep focused on existing implementation, note cross-cutting issues | — Pending |

---
*Last updated: 2026-02-09 after initialization*