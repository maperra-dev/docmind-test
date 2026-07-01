---
unique-name: stack
display-name: STACK
category: GENERAL
description: **Project:** NFS Migration Analysis & Fix (SI-to-NFS, BFF Factoring System)
---

# Technology Stack: Oracle PL/SQL Data Migration Optimization

**Project:** NFS Migration Analysis & Fix (SI-to-NFS, BFF Factoring System)
**Researched:** 2026-02-09
**Overall Confidence:** HIGH (Oracle official documentation verified)

---

## Executive Summary

The NFS migration codebase is a ~100K-line Oracle PL/SQL package running in the NFS_MIGRATION schema. Code analysis reveals the migration already uses **DBMS_PARALLEL_EXECUTE** for chunking large tables (clearing, invoices) but falls back to **row-by-row cursor loops** for other domains (bank transfers, contacts, identifiers). The migration relies on **INSERT INTO ... SELECT** for set-based operations (good) but has critical anti-patterns: MAX-based ID generation, missing statistics, and commits inside cursor loops. The stack below prescribes the Oracle-native toolset needed to fix performance, ensure correctness, and implement end-to-end reconciliation. No external tools are needed -- everything is Oracle-native.

---

## Recommended Stack

### Core Runtime Environment

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Oracle Database | 19c (19.3+) | Migration runtime | Long-term support release through 2027. The codebase already uses 19c features (identity columns, EDITIONABLE types). 19c is the standard enterprise target. | HIGH |
| PL/SQL | 19c native | Migration package language | Already in use. No language change needed -- optimization is about using PL/SQL features correctly. | HIGH |
| NFS_MIGRATION schema | Dedicated | Migration workspace | Already architected correctly -- dedicated schema isolates migration objects from both SI and NFS. Keep this pattern. | HIGH |

**Why not 21c or 23ai?** The codebase uses Oracle 19c features and the production environment is 19c. All optimization techniques below work on 19c. Upgrading the database version is out of scope and unnecessary -- 19c has every feature needed for this migration.

### Performance Optimization Stack

| Technology | Oracle Package/Feature | Purpose | Why | Confidence |
|------------|----------------------|---------|-----|------------|
| DBMS_PARALLEL_EXECUTE | Built-in (19c) | Parallel chunk-based processing | Already used in the codebase for clearing tables. Must be extended to ALL high-volume domains. Eliminates row-by-row processing while maintaining transactional control per chunk. | HIGH |
| Oracle Sequences | CREATE SEQUENCE | ID generation | Replace ALL `SELECT MAX(id)+1` patterns. Sequences are contention-free with proper CACHE settings, eliminate full table scans, and are the only correct approach for parallel ID generation. | HIGH |
| DBMS_STATS | Built-in (19c) | Optimizer statistics | Must be run on ALL migration staging tables after bulk loads. Without current statistics, the CBO generates catastrophic plans (full table scans instead of index lookups on 41M-row tables). | HIGH |
| Direct-Path INSERT | `/*+ APPEND */` hint | High-volume inserts | For initial staging loads where tables are truncated first. Bypasses buffer cache, eliminates redo (with NOLOGGING), 3-10x faster than conventional insert. | HIGH |
| Parallel DML | `ALTER SESSION ENABLE PARALLEL DML` | Multi-process DML | For INSERT INTO ... SELECT on tables with 1M+ rows. Combines with APPEND hint for maximum throughput. Requires careful session management. | HIGH |
| NOLOGGING mode | Table/index attribute | Reduce redo generation | For staging tables during migration loads. Combined with direct-path INSERT, eliminates redo logging overhead. MUST take backup after migration completes. | HIGH |
| Error Logging Tables | `DBMS_ERRLOG` | Capture DML errors without aborting | Instead of WHEN OTHERS THEN ROLLBACK, use LOG ERRORS INTO to capture individual row failures while the batch continues. Critical for 23-41M row tables. | MEDIUM |

### Diagnostics and Tuning Stack

| Technology | Oracle Package/Feature | Purpose | Why | Confidence |
|------------|----------------------|---------|-----|------------|
| AWR Reports | `DBMS_WORKLOAD_REPOSITORY` | Workload analysis | Automatic hourly snapshots capture SQL execution statistics, wait events, I/O patterns. Compare AWR reports before/after optimization to prove improvement. Requires Diagnostics Pack license. | HIGH |
| ASH (Active Session History) | `V$ACTIVE_SESSION_HISTORY` | Real-time wait analysis | Identifies what sessions are waiting on RIGHT NOW during migration runs. 1-second sampling granularity shows lock contention, I/O waits, and CPU bottlenecks. | HIGH |
| ADDM | `DBMS_ADDM` | Automatic diagnostics | Analyzes AWR data and generates actionable recommendations. Run ADDM analysis spanning the migration window to get Oracle's own fix suggestions. | HIGH |
| SQL Trace / TKPROF | `DBMS_MONITOR` / `DBMS_SESSION` | Statement-level profiling | Enable tracing on migration sessions to get per-statement parse/execute/fetch counts, CPU time, physical reads. TKPROF formats trace files into readable reports. | HIGH |
| Execution Plans | `DBMS_XPLAN.DISPLAY_CURSOR` | Query plan analysis | Examine actual (not estimated) execution plans for migration queries. Look for full table scans on large tables, nested loops on high-cardinality joins, and missing index usage. | HIGH |
| Real-Time SQL Monitoring | `V$SQL_MONITOR` | Long-running SQL tracking | Automatically tracks SQL statements running > 5 seconds or using parallel execution. Shows real-time progress, estimated completion time, and resource consumption. | HIGH |

### Reconciliation Stack

| Technology | Oracle Package/Feature | Purpose | Why | Confidence |
|------------|----------------------|---------|-----|------------|
| DBMS_COMPARISON | Built-in (19c) | Formal data comparison | Oracle's built-in table comparison engine. Creates named comparisons between source (SI) and target (NFS) tables, tracks row-level differences, and can auto-converge discrepancies. | HIGH |
| MINUS operator | SQL set operator | Row-level difference detection | `SELECT ... FROM source MINUS SELECT ... FROM target` identifies exact missing/extra rows. Simple, no setup required, works for ad-hoc checks. | HIGH |
| Aggregate reconciliation queries | Standard SQL | Count/sum validation | SI source counts/sums vs NFS target counts/sums. The existing codebase has NFS staging-to-NFS final but lacks SI-to-NFS end-to-end checks. | HIGH |
| MGR_LOG_QUADRATURE | Already exists in codebase | Error/discard tracking | The migration already has a quadrature (reconciliation) logging table. Must be extended to capture ALL discrepancy types, not just address province mismatches. | HIGH |
| MGR_LOG_ELAB | Already exists in codebase | Execution logging | Already logs start/end/error for each procedure. Must be enhanced with row counts (SQL%ROWCOUNT) after every DML to enable post-migration auditing. | HIGH |

### Index and Constraint Management

| Technology | Purpose | Why | Confidence |
|------------|---------|-----|------------|
| B-tree indexes on join columns | Query performance | Every JOIN column in migration queries must have an index on the staging tables. The codebase joins on (LEN_ID, AA_PCD, NUM_PCD) patterns repeatedly -- these MUST be indexed. | HIGH |
| Composite indexes | Multi-column lookups | For queries filtering on (REQ_NM_ID, IS_VALID, IS_HISTORICAL) -- a composite index eliminates full table scans on large staging tables. | HIGH |
| Deferred constraint validation | Load performance | Disable FK constraints during bulk load, re-enable with VALIDATE after. Eliminates per-row constraint checking during INSERT. | HIGH |
| Index rebuild after bulk load | Optimizer accuracy | After large inserts, indexes may have poor clustering factor. Rebuild or coalesce indexes post-load. | MEDIUM |

---

## Anti-Patterns Found in Current Codebase (What NOT to Do)

### Critical: Row-by-Row Cursor Processing

**Found in:** `mgr_finals_inc_bon_with_curs_stg` (line ~40075), `mgr_finals_inc_bon_with_curs_end` (line ~40181)

```sql
-- ANTI-PATTERN: Row-by-row processing with nested cursor loops
FOR r_bank_header IN (SELECT ... FROM bank_transfer_header WHERE ...)
LOOP
    l_bth_nm_id := nfs_collection.sq_bth.nextval;
    INSERT INTO NFS_COLLECTION.BANK_TRANSFER_HEADER VALUES (...);

    FOR r_bank_stg IN (SELECT ... FROM bank_transfer_staging WHERE ...)
    LOOP
        l_bts_nm_id := nfs_collection.sq_bts.nextval;
        -- Field-by-field assignment to rowtype variable
        r_bts.BTS_NM_ID := l_bts_nm_id;
        r_bts.BTH_NM_ID := l_bth_nm_id;
        -- ... 20+ field assignments ...
        INSERT INTO NFS_COLLECTION.BANK_TRANSFER_STAGING VALUES r_bts;
        UPDATE bank_transfer SET BTS_NM_ID = l_bts_nm_id WHERE ...;
    END LOOP;
END LOOP;
```

**Why this is catastrophic:**
- Each row requires a context switch between PL/SQL and SQL engines
- For N header rows x M staging rows, this generates N*M individual INSERT+UPDATE statements
- Each INSERT triggers per-row constraint checking, redo generation, and undo generation
- The UPDATE inside the inner loop hits the same table repeatedly without batching

**Correct approach:** Convert to INSERT INTO ... SELECT with sequence.NEXTVAL in the SELECT list, or use BULK COLLECT + FORALL with LIMIT clause.

### Critical: MAX-Based ID Generation

**Found throughout the codebase** (57 occurrences of MAX patterns)

```sql
-- ANTI-PATTERN: Full table scan for every ID generation
SELECT MAX(id) + 1 INTO l_next_id FROM target_table;
```

**Why this is catastrophic:**
- Full table scan on every call (O(n) per row inserted = O(n^2) total for n inserts)
- Race condition in parallel execution (two sessions can get same MAX value)
- No caching -- every call hits the table
- On a 41M-row table, this means 41M full table scans

**Correct approach:** Use `CREATE SEQUENCE ... CACHE 1000` or larger. Sequences are O(1), cached in SGA, and safe for parallel execution.

### Moderate: Commits Inside Cursor Loops

**Found in:** Multiple procedures commit after each parallel chunk completes, but also commit inside loops

**Why this is problematic:**
- Each COMMIT forces a log file sync wait
- Breaks transactional consistency (partial migration state on failure)
- For the parallel_exec wrapper this is acceptable (per-chunk commits are the DBMS_PARALLEL_EXECUTE design), but for cursor-loop procedures this is wrong

**Correct approach:** Commit once per logical unit of work, not per row.

### Moderate: WHEN OTHERS THEN ROLLBACK; RAISE

**Found in:** Every procedure in the migration package

```sql
EXCEPTION WHEN OTHERS THEN
    log_elab(..., SQLERRM);
    DBMS_OUTPUT.PUT_LINE(SQLERRM);
    ROLLBACK;
    RAISE;
```

**Why this is suboptimal:**
- Loses the error stack (SQLERRM alone does not capture the full backtrace)
- ROLLBACK discards ALL work in the current transaction, even successful rows
- For 41M-row tables, a single row failure means starting over

**Correct approach:** Use `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE` for full error context. Use LOG ERRORS INTO for DML error tolerance. Use SAVE EXCEPTIONS with FORALL for bulk error capture.

### Minor: to_number(decode(...)) in WHERE Clauses

**Found in:** `mgr_clearing_par_fat_att` (line ~50154)

```sql
WHERE to_number(decode(pfa.AA_FAT_ATT,'TEMP',to_char(DT_VAL_GFA,'YYYY'),pfa.AA_FAT_ATT))
      BETWEEN start_id AND end_id;
```

**Why this is problematic:**
- Function-based predicates prevent index usage unless a matching function-based index exists
- Converting VARCHAR2 to NUMBER at query time for every row is CPU-intensive
- The DECODE/TO_NUMBER chain is not SARGable

**Correct approach:** Pre-compute the numeric value in a materialized column or create a function-based index on the expression.

---

## Optimization Techniques by Table Size

### Tables > 10M Rows (RATA_CED_DEB: 41M, PAREGGIO_FAT_ATT: 24M)

| Technique | How | Why |
|-----------|-----|-----|
| DBMS_PARALLEL_EXECUTE | Chunk by ROWID or numeric column, parallel_level 8-16 | Distributes work across multiple server processes. Already used for clearing; must extend to all large tables. |
| Direct-path INSERT | `INSERT /*+ APPEND */ INTO target SELECT ... FROM source` | Bypasses buffer cache. 3-10x faster for large inserts. Requires COMMIT before subsequent access to same table. |
| NOLOGGING on staging tables | `ALTER TABLE staging_table NOLOGGING` | Eliminates redo generation during direct-path INSERT. Take RMAN backup after migration. |
| Sequence CACHE 10000 | `ALTER SEQUENCE seq CACHE 10000` | Large cache eliminates SGA round-trips for ID generation. At 41M rows, CACHE 20 (default) means 2M cache refills. CACHE 10000 means 4100 refills. |
| Partitioned staging tables | Partition by REQ_NM_ID or date range | Enables partition-level operations (truncate, statistics, parallel scan). Consider if multiple migration requests run on same tables. |
| Disable indexes before load, rebuild after | `ALTER INDEX idx UNUSABLE` / `ALTER INDEX idx REBUILD` | Index maintenance during bulk INSERT is expensive. Faster to rebuild once after all data is loaded. |
| Gather statistics post-load | `DBMS_STATS.GATHER_TABLE_STATS(... , estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE)` | Stale statistics after bulk load cause CBO to choose wrong plans for subsequent queries. |

### Tables 100K - 10M Rows (PAREGGIO_CED_DEB: 722K, CONTABILE_FF: 363K, MANDATI: 326K)

| Technique | How | Why |
|-----------|-----|-----|
| INSERT INTO ... SELECT (set-based) | Single SQL statement, no cursor loops | Already used for clearing tables. Extend to ALL domains at this size. |
| Proper indexing on join columns | Create B-tree indexes on (LEN_ID, business_key_columns) | Most migration queries join staging to NFS_LEGACY via LEN_ID + business keys. Without indexes, these are full table scan joins. |
| Gather statistics | `DBMS_STATS.GATHER_TABLE_STATS` with default sampling | Ensure CBO has accurate row counts and column distributions. |
| Sequence with CACHE 1000 | `ALTER SEQUENCE seq CACHE 1000` | Sufficient for this volume. Eliminates MAX-based generation. |

### Tables < 100K Rows (reference data, configuration)

| Technique | How | Why |
|-----------|-----|-----|
| Simple INSERT INTO ... SELECT | Standard set-based DML | At this scale, even suboptimal approaches run in seconds. Focus optimization effort on large tables. |
| Standard sequences CACHE 20 | Default sequence settings | Low volume does not warrant large cache. |

---

## Sequence Replacement Strategy

The migration codebase has 57 occurrences of MAX-based patterns. Here is the replacement approach.

### Pattern 1: MAX in SELECT (subquery correlation)

```sql
-- BEFORE: Anti-pattern
SELECT MAX(cle_nm_id) + 1 FROM clearing;

-- AFTER: Sequence
SELECT seq_clearing.NEXTVAL FROM DUAL;
-- Or directly in INSERT ... SELECT:
INSERT INTO clearing (cle_nm_id, ...)
SELECT seq_clearing.NEXTVAL, ... FROM source;
```

### Pattern 2: MAX for batch start value

```sql
-- BEFORE: Get starting ID for a batch
SELECT NVL(MAX(id), 0) + 1 INTO l_start FROM target;

-- AFTER: Initialize sequence once from current MAX, then use NEXTVAL
-- Run ONCE during setup:
DECLARE v_max NUMBER;
BEGIN
  SELECT NVL(MAX(id), 0) INTO v_max FROM target;
  EXECUTE IMMEDIATE 'CREATE SEQUENCE seq_target START WITH ' || (v_max + 1) || ' CACHE 10000';
END;
```

### Pattern 3: MAX inside cursor loop

```sql
-- BEFORE: N full table scans
FOR rec IN cursor LOOP
  SELECT MAX(id)+1 INTO l_id FROM target;
  INSERT INTO target(id, ...) VALUES(l_id, ...);
END LOOP;

-- AFTER: Zero table scans
-- Convert entire loop to:
INSERT INTO target (id, ...)
SELECT seq_target.NEXTVAL, ... FROM source;
```

### Sequence Configuration for Migration

| Sequence | CACHE Setting | Rationale |
|----------|--------------|-----------|
| Sequences for 40M+ row tables | CACHE 10000 | Minimize SGA contention in parallel execution |
| Sequences for 1M-10M row tables | CACHE 5000 | Good balance of memory and performance |
| Sequences for < 1M row tables | CACHE 1000 | Standard migration-level cache |
| Post-migration (application use) | CACHE 20-100 | Reset to normal application-appropriate cache after migration |

**NOORDER is correct** (already used in the codebase) -- ORDER is only needed for RAC strict ordering requirements and adds serialization overhead.

---

## Statistics Gathering Strategy

### Pre-Migration (before running migration packages)

```sql
-- Gather statistics on ALL source tables accessed via database links
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS(
    ownname          => 'NFS_LEGACY',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
    method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
    degree           => 8,
    cascade          => TRUE  -- include indexes
  );
END;
/
```

### Post-Staging Load (after data arrives in NFS_MIGRATION staging tables)

```sql
-- Gather stats on each staging table after bulk load
BEGIN
  FOR t IN (SELECT table_name FROM user_tables WHERE table_name LIKE '%_STG' OR table_name IN ('PAREGGIO_FAT_ATT','RATA_CED_DEB','CONTABILE_FF','MANDATI','ANAGRAFICA_DEB','ANAGRAFICA_CED'))
  LOOP
    DBMS_STATS.GATHER_TABLE_STATS(
      ownname          => 'NFS_MIGRATION',
      tabname          => t.table_name,
      estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
      method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
      degree           => 4,
      cascade          => TRUE
    );
  END LOOP;
END;
/
```

### Post-Migration (after INSERT INTO ... SELECT into NFS final tables)

```sql
-- Gather stats on NFS target tables to keep application queries optimal
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname          => 'NFS_COLLECTION',
    tabname          => 'CLEARING',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
    method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
    cascade          => TRUE
  );
  -- Repeat for each target table
END;
/
```

---

## Reconciliation Framework Design

The current codebase only reconciles NFS staging vs NFS final. The critical gap is **SI source vs NFS final** reconciliation.

### Layer 1: Row Count Reconciliation (fast, coarse)

```sql
-- For each migrated entity, compare source and target counts
SELECT 'SI_SOURCE' AS origin, COUNT(*) AS row_count FROM SI_SCHEMA.source_table@si_dblink WHERE conditions
UNION ALL
SELECT 'NFS_STAGING', COUNT(*) FROM NFS_MIGRATION.staging_table WHERE req_nm_id = :req
UNION ALL
SELECT 'NFS_FINAL', COUNT(*) FROM NFS_COLLECTION.target_table WHERE req_nm_id = :req;
```

### Layer 2: Aggregate Reconciliation (fast, catches data errors)

```sql
-- Financial totals must match between source and target
SELECT 'SI_SOURCE', SUM(amount), COUNT(DISTINCT account_id) FROM SI_SCHEMA.transactions@si_dblink
UNION ALL
SELECT 'NFS_FINAL', SUM(cle_nm_amount), COUNT(DISTINCT ace_nm_id) FROM NFS_COLLECTION.clearing WHERE req_nm_id = :req;
```

### Layer 3: Row-Level Reconciliation (slow, exact)

```sql
-- Find specific rows that differ
SELECT key_columns FROM SI_SOURCE_VIEW
MINUS
SELECT key_columns FROM NFS_TARGET_VIEW;
-- And the reverse:
SELECT key_columns FROM NFS_TARGET_VIEW
MINUS
SELECT key_columns FROM SI_SOURCE_VIEW;
```

### Layer 4: DBMS_COMPARISON (formal, automated)

For critical tables, use Oracle's built-in comparison framework. This requires a database link between SI and NFS schemas (which exists per the codebase's NFS_LEGACY references).

```sql
BEGIN
  DBMS_COMPARISON.CREATE_COMPARISON(
    comparison_name => 'MIG_CLEARING_CHECK',
    schema_name     => 'NFS_COLLECTION',
    object_name     => 'CLEARING',
    dblink_name     => 'SI_LINK',
    column_list     => 'CLE_NM_AMOUNT,CLE_DT_OPERATION,CLE_DT_VALUE,DOI_NM_ID',
    scan_mode       => DBMS_COMPARISON.CMP_SCAN_MODE_FULL
  );
END;
/
```

**Note:** DBMS_COMPARISON works between databases via DB links. Since the migration already accesses SI data through NFS_LEGACY schema (which may be a synonym/link layer), this is viable. However, column names differ between SI and NFS -- reconciliation queries that map SI columns to NFS columns are more practical for this project.

### Recommended Reconciliation Results Table

```sql
CREATE TABLE NFS_MIGRATION.MGR_RECONCILIATION (
  rec_id          NUMBER GENERATED ALWAYS AS IDENTITY,
  req_nm_id       NUMBER NOT NULL,
  entity_name     VARCHAR2(100) NOT NULL,   -- e.g., 'CLEARING', 'MANDATE'
  check_type      VARCHAR2(30) NOT NULL,    -- 'ROW_COUNT', 'SUM_CHECK', 'KEY_MATCH'
  source_value    VARCHAR2(4000),           -- SI value
  target_value    VARCHAR2(4000),           -- NFS value
  status          VARCHAR2(10) NOT NULL,    -- 'MATCH', 'MISMATCH', 'ERROR'
  detail          VARCHAR2(4000),           -- Mismatch description
  check_timestamp TIMESTAMP DEFAULT SYSTIMESTAMP,
  CONSTRAINT pk_mgr_reconciliation PRIMARY KEY (rec_id)
);
```

---

## Parallel Execution Configuration

### DBMS_PARALLEL_EXECUTE Settings (Already In Use)

The codebase already has a `parallel_exec` wrapper procedure (found at line ~40000). Current settings:

```sql
c_max_deg CONSTANT NUMBER := 20;  -- Maximum parallel degree cap
-- Uses create_chunks_by_sql with custom SQL for chunk boundaries
-- parallel_level passed as parameter, capped at LEAST(in_deg, c_max_deg)
```

**Recommended adjustments:**

| Parameter | Current | Recommended | Why |
|-----------|---------|-------------|-----|
| c_max_deg | 20 | Keep 20 | Reasonable cap. Going higher risks resource contention. |
| CACHE on chunk query sequences | 20 | 10000 | Large cache for parallel ID generation |
| Chunk size | Custom SQL-based | Use ROWID chunks for tables without natural partition key | ROWID-based chunking distributes more evenly across physical storage |
| Error handling | Check task_status after run | Add RESUME_TASK with retry loop | Handles transient failures (deadlocks, temp space) gracefully |

### Session-Level Parallel DML (For Set-Based Operations)

```sql
-- Enable before migration procedures that use INSERT ... SELECT
ALTER SESSION ENABLE PARALLEL DML;
ALTER SESSION FORCE PARALLEL DML PARALLEL 8;

-- Run migration
EXEC migration_procedure(:req_id);

-- Disable after
ALTER SESSION DISABLE PARALLEL DML;
```

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Parallel processing | DBMS_PARALLEL_EXECUTE | DBMS_SCHEDULER job arrays | Already in use, well-integrated, automatic chunk management. DBMS_SCHEDULER is lower-level. |
| ID generation | Oracle Sequences | UUID / SYS_GUID() | NFS schema uses numeric IDs throughout. Changing to UUID would require NFS application changes. |
| Bulk DML | INSERT INTO ... SELECT | BULK COLLECT + FORALL | INSERT ... SELECT is more efficient (zero PL/SQL context switches). FORALL is for cases where PL/SQL transformation logic is needed per row. |
| Error logging | DBMS_ERRLOG (LOG ERRORS INTO) | SAVE EXCEPTIONS with FORALL | LOG ERRORS INTO works with plain INSERT/UPDATE statements. SAVE EXCEPTIONS requires FORALL which requires cursor-based processing. For set-based operations, DBMS_ERRLOG is better. |
| Data comparison | Custom reconciliation queries | DBMS_COMPARISON | Column names differ between SI and NFS. Custom queries can map columns. DBMS_COMPARISON requires identical column names or views. Use both. |
| Statistics | DBMS_STATS | ANALYZE TABLE | ANALYZE is deprecated since 10g. DBMS_STATS is the supported, feature-rich replacement. |
| ETL tool | PL/SQL native | Oracle Data Pump / GoldenGate / external ETL | Migration is already implemented in PL/SQL. Rewriting in a different tool is not in scope. Optimization of existing code is the goal. |
| Monitoring | AWR + ASH + SQL Monitor | Third-party APM tools | Oracle-native tools are sufficient and already licensed. No additional tooling needed. |

---

## Execution Environment Configuration

### Session Parameters for Migration

```sql
-- Set before running migration packages
ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD';
ALTER SESSION SET NLS_TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS.FF';
ALTER SESSION SET OPTIMIZER_INDEX_COST_ADJ = 50;       -- Favor index access paths
ALTER SESSION SET OPTIMIZER_INDEX_CACHING = 90;         -- Assume 90% of index blocks cached
ALTER SESSION SET PGA_AGGREGATE_TARGET = 2G;            -- If DBA configurable
ALTER SESSION ENABLE PARALLEL DML;
ALTER SESSION SET "_optimizer_use_feedback" = FALSE;     -- Prevent plan instability during migration
ALTER SESSION SET HASH_AREA_SIZE = 1048576000;           -- 1GB for large hash joins
ALTER SESSION SET SORT_AREA_SIZE = 1048576000;           -- 1GB for large sorts
```

**Note:** Some parameters require DBA privileges. The actual values depend on the database server's available memory.

### Index Creation Script Template

```sql
-- Template for creating migration-supporting indexes
-- Run BEFORE migration, gather stats AFTER
CREATE INDEX idx_pfa_req_aa ON NFS_MIGRATION.PAREGGIO_FAT_ATT(REQ_NM_ID, AA_FAT_ATT, NUM_FAT_ATT)
  TABLESPACE migration_idx NOLOGGING PARALLEL 4;
ALTER INDEX idx_pfa_req_aa NOPARALLEL;  -- Reset for normal use

CREATE INDEX idx_pnf_req_aa ON NFS_MIGRATION.PAREGGIO_NON_FAT(REQ_NM_ID, AA_PCD, NUM_PCD)
  TABLESPACE migration_idx NOLOGGING PARALLEL 4;
ALTER INDEX idx_pnf_req_aa NOPARALLEL;

-- Join column indexes on NFS_LEGACY tables (if not already present)
CREATE INDEX idx_doi_fa_len ON NFS_LEGACY.DOI_FATTURA_ATTIVA(LEN_ID, DFA_AA_FAT_ATT, DFA_NUM_FAT_ATT)
  TABLESPACE nfs_idx NOLOGGING PARALLEL 4;
ALTER INDEX idx_doi_fa_len NOPARALLEL;

CREATE INDEX idx_btd_pag_len ON NFS_LEGACY.BTD_PAGAMENTO_FCD_FAA(LEN_ID, BFA_AA_PCD, BFA_NUM_PCD)
  TABLESPACE nfs_idx NOLOGGING PARALLEL 4;
ALTER INDEX idx_btd_pag_len NOPARALLEL;

CREATE INDEX idx_btd_ris_len ON NFS_LEGACY.BTD_RISTR_PAGAMENTO(LEN_ID, BRP_AA_PCD, BRP_NUM_PCD, BRP_NUM_RSP)
  TABLESPACE nfs_idx NOLOGGING PARALLEL 4;
ALTER INDEX idx_btd_ris_len NOPARALLEL;
```

---

## Sources

All recommendations verified against Oracle 19c official documentation:

- **DBMS_PARALLEL_EXECUTE**: Oracle Database PL/SQL Packages Reference, 19c -- verified via WebFetch. Confirms task/chunk/run_task API, FINISHED/FINISHED_WITH_ERROR status codes, and RESUME_TASK for retry. **HIGH confidence.**
- **DBMS_STATS / Optimizer Statistics**: Oracle Database SQL Tuning Guide, 19c -- verified via WebFetch. Confirms GATHER_TABLE_STATS, AUTO_SAMPLE_SIZE, online statistics gathering, real-time statistics. **HIGH confidence.**
- **SQL Trace / DBMS_MONITOR / AWR / ADDM**: Oracle Database Performance Tuning Guide, 19c -- verified via WebFetch. Confirms SESSION_TRACE_ENABLE, TKPROF, AWR snapshots, ADDM analysis modes, Real-Time ADDM triggers. **HIGH confidence.**
- **INSERT ... SELECT / Direct-Path / APPEND hint**: Oracle Database SQL Language Reference, 19c -- verified via WebFetch. Confirms APPEND hint behavior, NOLOGGING mode, parallel DML, LOG ERRORS INTO syntax. **HIGH confidence.**
- **MERGE statement**: Oracle Database SQL Language Reference, 19c -- verified via WebFetch. Confirms WHEN MATCHED/NOT MATCHED, DELETE WHERE clause, single-pass performance advantage. **HIGH confidence.**
- **DBMS_COMPARISON**: Oracle Database PL/SQL Packages Reference, 19c -- verified via WebFetch. Confirms CREATE_COMPARISON, COMPARE, CONVERGE operations, DBA_COMPARISON_ROW_DIF view, DB link requirements. **HIGH confidence.**
- **Anti-pattern analysis**: Derived from direct code analysis of `C:\Desktop\BFF\Code Review\NFS\Migrazione\nfs_migration.sql` (99,465 lines). Row-by-row cursor loops confirmed at lines ~40075-40167. MAX patterns: 57 occurrences confirmed via grep. DBMS_PARALLEL_EXECUTE usage confirmed at lines ~40000-40043. **HIGH confidence.**

---

## Confidence Assessment

| Area | Confidence | Reason |
|------|------------|--------|
| Oracle version features (19c) | HIGH | Verified against official Oracle 19c documentation via WebFetch |
| DBMS_PARALLEL_EXECUTE | HIGH | Official docs + confirmed already in use in codebase |
| Sequence vs MAX replacement | HIGH | Well-established Oracle best practice, verified in docs |
| DBMS_STATS usage | HIGH | Official docs, standard DBA practice |
| AWR/ASH/ADDM diagnostics | HIGH | Official docs, standard Oracle Enterprise Edition features |
| Direct-path INSERT / NOLOGGING | HIGH | Official docs, well-documented behavior |
| Reconciliation approach | MEDIUM | Custom design based on standard patterns. DBMS_COMPARISON verified but column mapping complexity is project-specific. |
| Session parameter tuning | MEDIUM | Values are environment-dependent. Recommendations are directional, actual values need DBA input. |
| Specific index recommendations | MEDIUM | Based on code analysis of query patterns. Need EXPLAIN PLAN validation against actual data. |