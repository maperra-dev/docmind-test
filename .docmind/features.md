---
unique-name: features
display-name: FEATURES
category: GENERAL
description: **Domain:** Oracle PL/SQL Data Migration Analysis (SI-to-NFS Financial Factoring System)
---

# Feature Landscape

**Domain:** Oracle PL/SQL Data Migration Analysis (SI-to-NFS Financial Factoring System)
**Researched:** 2026-02-09
**Confidence:** HIGH (based on direct codebase analysis of ~100K lines of migration PL/SQL)

---

## Table Stakes

Features that any competent migration analysis must include. Missing any of these means the analysis is incomplete and untrustworthy.

### TS-1: Migration Table Inventory & Mapping

| Aspect | Detail |
|--------|--------|
| **What** | Complete catalog of every source (SI) table, staging (NFS_MIGRATION) table, and target (NFS) table involved in the migration, with column-level mapping |
| **Why Expected** | Without this, you cannot verify that data flows correctly from source to destination. The known 605/901 sequence error proves that mapping errors exist. |
| **Complexity** | Medium |
| **Notes** | Must cover all 5+ target schemas: NFS_REGISTRY, NFS_COLLECTION, NFS_LEGACY, NFS_PRODUCTS, NFS_MIGRATION. Requires cross-referencing package INSERT statements against DDL. The migration uses DB links to SI, so source table access is indirect. |

### TS-2: Row Count Reconciliation (SI Source vs NFS Final)

| Aspect | Detail |
|--------|--------|
| **What** | SQL queries that compare record counts at SI source (via DB link) against NFS final destination tables, per migration scope (BONIFICI, MANDATI, FATTURE, etc.) |
| **Why Expected** | The most basic data integrity check. Currently only staging-vs-final reconciliation exists. The gap between SI source and NFS final is the highest risk area. |
| **Complexity** | Medium |
| **Notes** | Must account for filtering logic (legal entity, year ranges, ABI codes). Row counts must be segmented by meaningful business dimensions (year, legal entity, scope) to catch partial failures. Requires understanding each package's WHERE clause filters. |

### TS-3: Performance Bottleneck Identification

| Aspect | Detail |
|--------|--------|
| **What** | Systematic identification of all performance anti-patterns in the migration code: missing indexes, absent table statistics, MAX()-based ID generation, row-by-row cursor processing, unnecessary commits |
| **Why Expected** | Migration is known to be slow on high-volume tables (RATA_CED_DEB: 41M rows, PAREGGIO_FAT_ATT: 24M rows). Performance issues are a primary motivator for this project. |
| **Complexity** | Medium |
| **Notes** | Every package body contains `SELECT max(xxx_id) INTO l_xxx_id FROM schema.TABLE` after INSERT to get the generated ID. This is O(n) per row on unindexed identity columns. Combined with cursor-based row-by-row processing, this creates O(n^2) behavior. |

### TS-4: Performance Fix Scripts (Indexes & Statistics)

| Aspect | Detail |
|--------|--------|
| **What** | Ready-to-execute DDL scripts: CREATE INDEX for missing indexes on join/filter columns, DBMS_STATS.GATHER_TABLE_STATS calls, sequence replacement recommendations |
| **Why Expected** | Analysis without actionable fix scripts is just a report. The team needs scripts they can run directly. |
| **Complexity** | Medium-High |
| **Notes** | Must cover both NFS_MIGRATION staging tables and NFS final tables. Index recommendations must be based on actual WHERE clauses and JOIN conditions found in the package bodies. Statistics gathering should target tables accessed via DB links as well as local tables. |

### TS-5: Data Population Logic Audit

| Aspect | Detail |
|--------|--------|
| **What** | Line-by-line review of every INSERT...SELECT and cursor-based INSERT in all migration packages to verify: correct source columns, correct join conditions, correct literal/constant mappings, correct FK references |
| **Why Expected** | The known 605/901 bug (wrong sequence ID in Payment Notice join) proves that population errors exist. Where there is one bug, there are more. |
| **Complexity** | High |
| **Notes** | 25+ packages, each with multiple INSERT procedures. Focus areas: (a) sequence/ID references in WHERE clauses, (b) hardcoded constants (C_LEGAL_ENTITY=41, C_ABI_CSE_*='05000'), (c) cross-schema FK references (e.g., NFS_REGISTRY.WORKFLOW, NFS_PRODUCTS.products_manager), (d) NVL/COALESCE default values. |

### TS-6: Error Handling Assessment

| Aspect | Detail |
|--------|--------|
| **What** | Review of exception handling patterns across all packages: WHEN OTHERS clauses, PRAGMA AUTONOMOUS_TRANSACTION logging, rollback behavior, error propagation |
| **Why Expected** | Silent data loss is the worst outcome in a financial migration. Swallowed exceptions mean records can be silently skipped. |
| **Complexity** | Medium |
| **Notes** | Multiple patterns observed: (a) check_start() returns t_start_step(0, null) on WHEN OTHERS -- silently fails, (b) parallel_exec leaves failed tasks for "post-mortem" but doesn't raise, (c) some exception handlers only DBMS_OUTPUT.PUT_LINE (lost in batch jobs). Need to catalog every exception path and classify as: propagates, logs-and-continues, or silently swallows. |

### TS-7: Key Field Comparison Queries

| Aspect | Detail |
|--------|--------|
| **What** | SQL queries that compare key business fields (not just row counts) between SI source and NFS final: amounts, dates, status codes, foreign key relationships |
| **Why Expected** | Row counts can match while data is wrong. A migration that moves the right number of rows with wrong amounts is worse than one that clearly fails. Financial data integrity requires field-level verification. |
| **Complexity** | High |
| **Notes** | Priority fields: monetary amounts (SUM checks), dates (MIN/MAX range checks), status/type codes (DISTINCT value distribution), FK integrity (orphan check). Must be per-table and per-scope. |

### TS-8: Aggregate Checksum Queries

| Aspect | Detail |
|--------|--------|
| **What** | Aggregate comparison queries: SUM of monetary columns, COUNT DISTINCT of key dimensions, hash-based row matching for sampled verification |
| **Why Expected** | Standard practice for financial data migration. Auditors expect proof that total amounts are preserved end-to-end. |
| **Complexity** | Medium |
| **Notes** | For each table with monetary columns, produce: SUM(amount) grouped by year/legal_entity at source vs destination. Flag any delta > 0.01 (cent-level precision for EUR). |

### TS-9: Package Execution Order & Dependency Analysis

| Aspect | Detail |
|--------|--------|
| **What** | Document the correct execution order of migration packages and identify inter-package dependencies (e.g., REGISTRY must run before CLEARING because CLEARING references counterpart IDs created by REGISTRY) |
| **Why Expected** | Running packages in wrong order causes FK violations or orphaned records. The orchestration logic is embedded in MGR_REQUEST_STEP configuration, not in code. |
| **Complexity** | Medium |
| **Notes** | The step-based workflow (check_start / set_status_req / MGR_REQUEST_STEP) governs execution flow. Must map: which scopes/subscopes exist, what order they run, which share staging tables. REGISTRY and TAXONOMY are prerequisites for everything else. |

### TS-10: CZ&SK Cross-Cutting Impact Notes

| Aspect | Detail |
|--------|--------|
| **What** | Flag every finding that also applies to the upcoming CZ&SK migration, particularly: hardcoded ITA constants, missing parameterization, _SK package variant gaps |
| **Why Expected** | CZ&SK migration is the next phase. Every issue found in ITA code that has a hardcoded assumption will need to be addressed in _SK variants too. |
| **Complexity** | Low (additive to other analysis) |
| **Notes** | The BONIFICI_SK analysis already found C_LEGAL_ENTITY=41 hardcoded in the _SK package. All other _SK variants likely have the same issue. Only BONIFICI_SK has been analyzed so far. |

---

## Differentiators

Features that elevate the analysis from "adequate" to "catches issues before they become production incidents." Not strictly expected, but high value.

### D-1: MAX()-to-Sequence Replacement Scripts

| Aspect | Detail |
|--------|--------|
| **What** | Ready-to-deploy replacement code that converts all `SELECT MAX(id) INTO l_id FROM table` patterns to use RETURNING clauses or Oracle sequences, with transaction-safety guarantees |
| **Value Proposition** | MAX() on identity columns is the single largest performance bottleneck. In PKG_MIGRATION_REGISTRY alone, MAX() is called per-row in a cursor loop processing hundreds of thousands of counterparts. Replacing with RETURNING INTO eliminates full table scans and provides concurrency safety. |
| **Complexity** | High |
| **Notes** | The INSERT...RETURNING INTO pattern is straightforward, but must be applied consistently across all packages. Some packages use DBMS_PARALLEL_EXECUTE which complicates the replacement because concurrent MAX() calls can return the same value (race condition causing duplicate key errors or silent overwrites). |

### D-2: Cursor-to-Bulk Processing Conversion Analysis

| Aspect | Detail |
|--------|--------|
| **What** | Identify all cursor-based row-by-row INSERT loops that could be converted to FORALL/BULK COLLECT or set-based INSERT...SELECT, with estimated performance improvement |
| **Value Proposition** | Row-by-row processing with PL/SQL-to-SQL context switching is 10-100x slower than bulk operations. For 41M-row tables, this is the difference between hours and minutes. |
| **Complexity** | High |
| **Notes** | Not all cursors can be trivially converted -- some have conditional logic (IF l_is_inserted = 1 THEN INSERT... ELSE l_ctp_id := r_tables.ctp_id). These need case-by-case analysis. The parallel_exec pattern already handles some bulk work via DBMS_PARALLEL_EXECUTE, but the "finals" procedures (mgr_finals_with_curs, mgr_finals_inc_fat_with_curs) are all cursor-based. |

### D-3: Validation Rules Coverage Gap Analysis

| Aspect | Detail |
|--------|--------|
| **What** | Audit the existing validation rules engine (V1-V8 in PKG_UTILITY.verify_rule) against actual data quality requirements: what rules exist, what rules are missing, what scopes have SKIP'd validation phases |
| **Value Proposition** | The CZ&SK BONIFICI analysis showed that "Validate Rules" and "Report Validate" phases are SKIP'd. If validation is skipped, bad data flows through unchecked. Identifying the gap between what V1-V8 covers and what the data actually needs catches silent corruption. |
| **Complexity** | Medium |
| **Notes** | Current rules: V1=NOT NULL, V2=numeric, V3=valid date, V4=alphanumeric, V5=field length, V6=ITA tax ID format, V7=ITA fiscal code format, V8=REA/CCIAA format. Missing: FK referential integrity validation, cross-field consistency (e.g., currency vs country), business rule validation (e.g., amounts should be positive), duplicate detection. |

### D-4: Hardcoded Constant Parameterization Audit

| Aspect | Detail |
|--------|--------|
| **What** | Comprehensive inventory of every hardcoded constant across all packages (C_LEGAL_ENTITY=41, C_ABI_CSE_*='05000', C_PAESE='ITALIA', C_COD_PAESE='086', C_MARKET=1, C_V2_LE_CODE='BFFIT', C_YEAR_FATT_ATT_ELAB=2023) and whether each is properly overridden at runtime from MGR_REQUEST/MGR_CONFIGURATION |
| **Value Proposition** | The #1 cause of "works in ITA, fails in CZ/SK" bugs. Even for ITA re-runs, hardcoded year=2023 will cause problems in future migrations. |
| **Complexity** | Medium |
| **Notes** | Some constants ARE properly overridden (C_LEGAL_ENTITY is reassigned from MGR_REQUEST in some procedures but not all). The danger is inconsistency: a procedure might read from config at the top but a sub-procedure lower in the call stack still uses the package-level hardcoded value. |

### D-5: Parallel Execution Safety Analysis

| Aspect | Detail |
|--------|--------|
| **What** | Verify that DBMS_PARALLEL_EXECUTE chunk definitions produce non-overlapping data partitions, that parallel tasks cannot interfere with each other, and that the MAX()-based ID generation is safe under concurrency |
| **Value Proposition** | DBMS_PARALLEL_EXECUTE with MAX()-based ID generation is a concurrency bug waiting to happen. Two parallel chunks could INSERT simultaneously, both call MAX(), get the same value, and one overwrites the other. |
| **Complexity** | High |
| **Notes** | The parallel_exec procedure uses LEAST(in_deg, 20) for parallelism. Chunking is by SQL query (create_chunks_by_sql). Need to verify: (a) chunks are mutually exclusive, (b) no shared mutable state (MAX() is shared mutable state), (c) error handling propagates correctly across parallel jobs. |

### D-6: Orphan Record Detection Queries

| Aspect | Detail |
|--------|--------|
| **What** | SQL queries that detect records in NFS target tables that reference non-existent parent records (broken FK relationships), indicating migration ordering errors or partial failures |
| **Value Proposition** | Migration packages create parent-child relationships across schemas (e.g., COUNTERPART -> LINK_CTP_RTP_RST -> ADDRESS). If parent creation fails silently, children become orphans. Standard FK constraints may be disabled during migration for performance. |
| **Complexity** | Medium |
| **Notes** | Priority relationships: WORKFLOW references, COUNTERPART->COUNTERPART_INFO, DEAL->STOCK_LOAD, BANK_TRANSFER_HEADER->BANK_TRANSFER_DETAIL. These span NFS_REGISTRY, NFS_COLLECTION, NFS_LEGACY, and NFS_PRODUCTS schemas. |

### D-7: Idempotency & Re-runnability Analysis

| Aspect | Detail |
|--------|--------|
| **What** | Determine whether each migration package can be safely re-run (idempotent) or whether re-running causes duplicates, and provide fix recommendations |
| **Value Proposition** | Migration failures require re-runs. If re-running a package doubles the data instead of resuming from the failure point, the team has a bigger problem than they started with. |
| **Complexity** | Medium-High |
| **Notes** | Some packages have NOT EXISTS checks (e.g., mgr_finals_inc_fat_with_curs checks nfs_legacy.doi_rata_ced_deb). Others do not. The "Clear Table Elab" step suggests tables are truncated before re-run, but this must be verified for each package. The IS_TRANSF flag in MGR_COUNTERPART suggests a transfer-tracking mechanism. |

### D-8: Data Type & Precision Mismatch Detection

| Aspect | Detail |
|--------|--------|
| **What** | Compare column data types and precision between SI source, NFS_MIGRATION staging, and NFS target tables to detect silent truncation or conversion issues |
| **Value Proposition** | VARCHAR2(50) in SI mapped to VARCHAR2(30) in NFS silently truncates. NUMBER(10,2) to NUMBER(8,2) silently loses large amounts. These are data corruption bugs that pass row count reconciliation. |
| **Complexity** | Medium |
| **Notes** | V5 validation rule already checks field length, but only for configured columns. This analysis covers ALL columns systematically, including numeric precision and date format differences. |

---

## Anti-Features

Features to explicitly NOT build. These either waste effort, add risk, or are out of scope.

### AF-1: Automated Migration Re-write

| Anti-Feature | Automated rewrite of migration packages (e.g., converting all cursor logic to bulk processing, replacing all MAX() calls) |
|--------------|-----------|
| **Why Avoid** | The migration code is in production for ITA, GR, and PT. A wholesale rewrite introduces regression risk. The team needs surgical fixes, not a new codebase to test. Additionally, this is code-analysis-only -- no live DB access for testing rewrites. |
| **What to Do Instead** | Produce targeted fix scripts for specific performance bottlenecks and identified bugs. Recommend bulk processing as a future phase for CZ&SK new implementation. |

### AF-2: NFS Staging-to-NFS Final Reconciliation

| Anti-Feature | Queries comparing NFS_MIGRATION staging tables to NFS final tables |
|--------------|-----------|
| **Why Avoid** | These already exist (per PROJECT.md: "existing queries already cover this"). Duplicating this work adds no value. |
| **What to Do Instead** | Focus exclusively on SI source vs NFS final (the end-to-end gap that is NOT currently covered). Reference existing staging reconciliation where useful but do not recreate it. |

### AF-3: CZ&SK Migration Implementation

| Anti-Feature | Actually implementing the _SK package variants for CZ&SK countries |
|--------------|-----------|
| **Why Avoid** | Explicitly out of scope per project constraints. CZ&SK is a separate future phase. Only the BONIFICI_SK package has been analyzed (and partially corrected). |
| **What to Do Instead** | Document cross-cutting findings that affect CZ&SK. Flag hardcoded ITA constants. Note which _SK variants exist vs which are missing. This informs the future phase without doing its work. |

### AF-4: Live Database Testing or Execution

| Anti-Feature | Running any analysis queries or fix scripts against live/test databases |
|--------------|-----------|
| **Why Avoid** | No database access. This is a code-analysis-only project. Scripts are delivered for the team to execute. |
| **What to Do Instead** | All deliverables are SQL/PL-SQL scripts with clear execution instructions, prerequisites, and expected outcomes documented in comments. |

### AF-5: Full Application-Level Testing Framework

| Anti-Feature | Building an automated test harness, CI/CD pipeline, or regression test suite for the migration |
|--------------|-----------|
| **Why Avoid** | Over-engineering for the project scope. The migration is a one-time (per country) batch operation, not a continuously-running application. |
| **What to Do Instead** | Provide reconciliation queries that serve as manual verification scripts. These are effectively one-shot tests that the team runs after each migration execution. |

### AF-6: Non-Incassi Perimeter Analysis

| Anti-Feature | Analyzing migration packages for perimeters other than incassi (collections) |
|--------------|-----------|
| **Why Avoid** | Explicitly out of scope. The project is scoped to the collections perimeter only. |
| **What to Do Instead** | If analysis incidentally discovers cross-perimeter issues (e.g., shared PKG_UTILITY bugs), note them but do not investigate further. |

---

## Feature Dependencies

```
TS-1 (Table Inventory)
  |
  +---> TS-2 (Row Count Reconciliation) -- needs table mapping to write queries
  |       |
  |       +---> TS-7 (Key Field Comparison) -- extends row counts to field level
  |       |
  |       +---> TS-8 (Aggregate Checksums) -- extends row counts to amount verification
  |
  +---> TS-5 (Population Logic Audit) -- needs mapping to verify against
  |       |
  |       +---> D-1 (MAX-to-Sequence Fix) -- fixes found during audit
  |       |
  |       +---> D-4 (Hardcoded Constant Audit) -- subset of population audit
  |       |
  |       +---> D-7 (Idempotency Analysis) -- requires understanding population logic
  |
  +---> D-8 (Data Type Mismatch) -- needs column mapping to compare types
  |
  +---> D-6 (Orphan Detection) -- needs FK relationships from mapping

TS-3 (Performance Identification)
  |
  +---> TS-4 (Performance Fix Scripts) -- fixes for identified issues
  |       |
  |       +---> D-1 (MAX-to-Sequence Fix) -- most critical performance fix
  |
  +---> D-2 (Cursor-to-Bulk Analysis) -- performance improvement area
  |
  +---> D-5 (Parallel Execution Safety) -- related to performance + correctness

TS-6 (Error Handling Assessment) -- independent, can run in parallel

TS-9 (Execution Order) -- independent, informs all other analysis

TS-10 (CZ&SK Notes) -- additive, produced alongside all other features

D-3 (Validation Rules Gap) -- requires TS-5 (population audit) context
```

---

## MVP Recommendation

### Phase 1 -- Foundation (must complete first)

1. **TS-1: Table Inventory & Mapping** -- everything else depends on this
2. **TS-9: Package Execution Order** -- informs analysis ordering

### Phase 2 -- Core Analysis (parallel workstreams)

Workstream A (Performance):
3. **TS-3: Performance Bottleneck Identification**
4. **TS-4: Performance Fix Scripts**
5. **D-1: MAX()-to-Sequence Replacement Scripts** (highest-impact fix)

Workstream B (Correctness):
6. **TS-5: Data Population Logic Audit** (the largest single effort)
7. **TS-6: Error Handling Assessment**
8. **D-4: Hardcoded Constant Audit** (overlaps with TS-5)

### Phase 3 -- Verification

9. **TS-2: Row Count Reconciliation Queries** (SI vs NFS end-to-end)
10. **TS-7: Key Field Comparison Queries**
11. **TS-8: Aggregate Checksum Queries**
12. **D-6: Orphan Record Detection**

### Phase 4 -- Advanced (if time permits)

13. **D-2: Cursor-to-Bulk Conversion Analysis**
14. **D-3: Validation Rules Coverage Gap**
15. **D-5: Parallel Execution Safety**
16. **D-7: Idempotency Analysis**
17. **D-8: Data Type Mismatch Detection**

### Defer

- **D-2 (Cursor-to-Bulk)**: High effort, best addressed in CZ&SK new implementation rather than retrofitting ITA packages
- **D-5 (Parallel Safety)**: Important but secondary to getting core reconciliation working
- **D-8 (Data Type Mismatch)**: Valuable but lower priority than population logic bugs

---

## Complexity Budget

| Feature | Estimated Effort | Packages to Review | Output Volume |
|---------|------------------|--------------------|---------------|
| TS-1 | 1-2 days | All 25+ | 1 master mapping document |
| TS-2 | 2-3 days | All 25+ | ~30 reconciliation SQL scripts |
| TS-3 | 1-2 days | All 25+ | 1 analysis report |
| TS-4 | 1-2 days | N/A (DDL) | ~20 index DDLs + stats scripts |
| TS-5 | 3-5 days | All 25+ | Per-package audit reports |
| TS-6 | 1 day | All 25+ | 1 error handling report |
| TS-7 | 2-3 days | All 25+ | ~30 field comparison SQL scripts |
| TS-8 | 1-2 days | High-volume tables | ~10 aggregate check SQL scripts |
| TS-9 | 0.5 days | MGR_REQUEST_STEP config | 1 dependency diagram |
| TS-10 | Additive | All _SK variants | Notes within each report |
| D-1 | 2-3 days | All packages with MAX() | Per-package fix scripts |
| D-2 | 2-3 days | All cursor-based packages | Conversion feasibility report |
| D-3 | 1 day | PKG_UTILITY + MGR_CONFIGURATION | Gap analysis report |
| D-4 | 1 day | All packages | Constants inventory |
| D-5 | 1-2 days | Packages using parallel_exec | Safety analysis report |
| D-6 | 1 day | All cross-schema relationships | ~15 orphan detection SQL scripts |
| D-7 | 1-2 days | All packages | Idempotency matrix |
| D-8 | 1-2 days | All mapped tables | Type comparison matrix |

**Total estimated effort: 20-35 days** for comprehensive coverage, or **10-15 days** for MVP (Phases 1-3).

---

## Sources

- **Direct codebase analysis**: `C:\Desktop\BFF\Code Review\NFS\Migrazione\nfs_migration.sql` (~99,465 lines)
  - Package bodies analyzed: PKG_MIGRATION_BONIFICI (line 39941), PKG_MIGRATION_CLEARING (line 47832), PKG_MIGRATION_FATTURE (line 61076), PKG_MIGRATION_MANDATI (line 68229), PKG_MIGRATION_REGISTRY (line 73714), PKG_MIGRATION_TAXONOMY (line 95258), PKG_UTILITY (line 97796)
  - MAX() pattern confirmed at lines: 73860, 73908, 95318, 95341, 95366, 95393
  - Validation rules engine (V1-V8) confirmed at lines: 97905-97995
  - Parallel execution pattern confirmed in all major packages
- **Existing analysis**: `ANALISI_MIGRAZIONE_BONIFICI_CZSK.md` (CZ&SK BONIFICI migration analysis)
- **Project definition**: `.planning/PROJECT.md` (requirements, known issues, constraints, volume data)
- **Confidence**: HIGH -- all findings based on direct source code inspection, not documentation or hearsay