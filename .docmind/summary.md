---
unique-name: summary
display-name: SUMMARY
category: GENERAL
description: **Project:** NFS Migration Analysis & Fix (SI-to-NFS, BFF Factoring System)
---

# Project Research Summary

**Project:** NFS Migration Analysis & Fix (SI-to-NFS, BFF Factoring System)
**Domain:** Oracle PL/SQL Data Migration Optimization & Analysis
**Researched:** 2026-02-09
**Confidence:** HIGH

## Executive Summary

The NFS migration project is a large-scale Oracle PL/SQL data migration system (~100K lines) that transfers financial factoring data from legacy SI country schemas to a modern multi-schema NFS architecture across four target schemas (NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION, NFS_LEGACY). The research reveals this is fundamentally a **data engineering correctness and performance optimization problem**, not a greenfield development project. The existing migration runs in production for ITA, GR, and PT, but contains critical performance anti-patterns (MAX-based ID generation, row-by-row cursor processing, missing statistics) that make it unsuitable for high-volume deployments and contains known data mapping bugs (the 605/901 sequence error is a symptom of systemic entity-type mapping problems).

The recommended approach is **surgical optimization through static code analysis**: catalog infrastructure gaps (indexes, statistics, sequences), audit package logic for data mapping errors, and design end-to-end reconciliation queries that validate SI source vs NFS final correctness. All work is code-analysis-only with no live database access. The toolset is entirely Oracle-native (DBMS_PARALLEL_EXECUTE, DBMS_STATS, Oracle sequences, DBMS_COMPARISON) — no external ETL tools or third-party frameworks are needed.

The primary risks are: (1) SELECT MAX anti-pattern creating O(n^2) performance degradation on 41M-row tables plus concurrency bugs in parallel execution, (2) entity-type mapping errors propagating silently across all 25+ packages due to hardcoded constants rather than centralized lookup tables, (3) cursor-based error handlers that swallow failures causing phantom success, and (4) cross-schema commits without transaction boundaries creating non-idempotent re-runs. Mitigation follows a three-phase approach: infrastructure fixes first (indexes, sequences, statistics), then package-by-package data error analysis with particular focus on entity-type mappings, then comprehensive SI-to-NFS reconciliation queries to validate completeness and accuracy.

## Key Findings

### Recommended Stack

The migration is Oracle 19c PL/SQL native and should remain so — no language or platform change is needed. The optimization stack is entirely Oracle built-in features: DBMS_PARALLEL_EXECUTE for chunk-based parallel processing (already partially implemented), Oracle sequences to replace MAX-based ID generation (42 occurrences must be converted), DBMS_STATS for optimizer statistics (currently missing), direct-path INSERT with APPEND hint for high-volume loads, and DBMS_COMPARISON for formal reconciliation. The anti-patterns found in the current code are well-documented Oracle mistakes with well-documented Oracle solutions.

**Core technologies:**
- **Oracle Database 19c** — long-term support release, already in production. All optimization techniques work on 19c. No database version upgrade needed.
- **DBMS_PARALLEL_EXECUTE** — built-in chunking framework already used for clearing tables. Must be extended to all high-volume domains (41M-row RATA_CED_DEB, 23M-row PAREGGIO_FAT_ATT).
- **Oracle Sequences with large CACHE** — replace all `SELECT MAX(id)+1` patterns. CACHE 10000 for 40M+ row tables eliminates the O(n^2) full-table-scan problem and provides concurrency safety.
- **DBMS_STATS** — gather statistics on staging tables after bulk load and on final tables after migration. Currently missing entirely (four DBMS_STATS calls are commented out). Without current statistics, the CBO generates catastrophic plans.
- **Direct-path INSERT (APPEND hint)** — for initial staging loads. Bypasses buffer cache, 3-10x faster. Combine with NOLOGGING on staging tables.
- **DBMS_COMPARISON** — Oracle's formal table comparison framework for SI-vs-NFS reconciliation. Complements custom MINUS queries.

### Expected Features

Analysis is code-only with no live database access. Deliverables are SQL/DDL scripts for the team to execute.

**Must have (table stakes):**
- **TS-1: Migration Table Inventory** — complete SI-to-staging-to-final table mapping across all 25 packages, covering all 5 schemas.
- **TS-2: Row Count Reconciliation (SI vs NFS final)** — end-to-end row count validation. Currently only staging-vs-final exists.
- **TS-3: Performance Bottleneck Identification** — systematic catalog of SELECT MAX (42), missing indexes, absent statistics, cursor-loop anti-patterns.
- **TS-4: Performance Fix Scripts** — ready-to-execute DDL for indexes, sequences, DBMS_STATS calls.
- **TS-5: Data Population Logic Audit** — line-by-line review of INSERT logic to find the next 605/901-type bugs before they hit production.
- **TS-6: Error Handling Assessment** — classify all WHEN OTHERS handlers (30 occurrences) as RAISE vs LOG-and-continue vs silent-swallow.
- **TS-7: Key Field Comparison Queries** — beyond row counts, validate amounts, dates, FK relationships at field level.
- **TS-8: Aggregate Checksum Queries** — SUM of monetary columns for financial integrity validation.

**Should have (differentiators):**
- **D-1: MAX-to-Sequence Replacement Scripts** — highest-impact fix. Converts all 42 MAX patterns to sequences with transaction-safety guarantees.
- **D-4: Hardcoded Constant Audit** — catalog all hardcoded entity types, legal entities, year values, country codes. Critical for CZ&SK migration prep.
- **D-6: Orphan Record Detection** — queries to find broken FK relationships across NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION, NFS_LEGACY.

**Defer (v2+):**
- **D-2: Cursor-to-Bulk Conversion** — high effort, better addressed in CZ&SK new implementation rather than retrofitting ITA.
- **D-5: Parallel Execution Safety Analysis** — important but secondary to getting core reconciliation working.
- **D-8: Data Type Mismatch Detection** — valuable but lower priority than entity-type mapping bugs.

### Architecture Approach

The migration follows a five-hop architecture: (1) SI source to NFS_MIGRATION staging via DB link, (2) staging cleanup (mgr_data_adjustment), (3) validation (validate_rule), (4) staging to NFS_MIGRATION intermediate domain tables (loading_final), (5) intermediate to NFS final schemas (Transfer via row-by-row cursor loops). Data flows through 25+ packages into four target schemas. Each package follows a standardized step-driven pattern managed through MGR_REQUEST_STEP configuration tables.

Analysis decomposes into three orthogonal components: (1) Infrastructure Analysis produces DDL scripts (indexes, sequences, statistics) applied before/after migration runs, (2) Package-by-Package Data Error Analysis modifies package body logic to fix mapping bugs, and (3) End-to-End Reconciliation produces read-only SQL queries run on-demand for verification. The three deliverable areas are largely independent and can be developed in parallel, though reconciliation benefits from knowing correct mappings from data error analysis.

**Major components:**
1. **Infrastructure (Performance) Analysis** — indexes, statistics, sequences, parallel configuration. Cross-cutting, applies to all packages. Produces DDL/config scripts run BEFORE/AFTER migration. 6 sub-components.
2. **Package-by-Package Data Error Analysis** — logic review per package. 25 packages x 5 hops each. Produces corrected PL/SQL package bodies. Focuses on entity-type mappings (Pitfall 2), MAX patterns (Pitfall 1), exception handlers (Pitfall 3).
3. **End-to-End Reconciliation** — SI source vs NFS final validation. Produces SQL query scripts run ON DEMAND. Three-tier model: row counts (fast, broad), aggregates (medium, targeted), record-level MINUS (slow, precise).

### Critical Pitfalls

1. **SELECT MAX(id) for ID generation is not concurrency-safe** — 42 occurrences create O(n^2) performance degradation plus race conditions in DBMS_PARALLEL_EXECUTE parallel chunks. Prevention: replace with Oracle sequences using CACHE 10000 for high-volume tables. Detection: grep for `SELECT\s+MAX\s*\(` followed by `INTO`. Addresses in: Performance Analysis (Phase 1) and Data Error Analysis (Phase 2).

2. **Entity-type mapping errors propagate silently** — the known 605/901 bug is a symptom. Hardcoded constants (ADT_V2_ID, ADS_V2_ID, DOT_V2_ID) scattered across 25 packages. No centralized mapping table. Prevention: build complete entity-type mapping matrix, cross-reference every hardcoded constant against NFS reference data, create NFS_MIGRATION.ENTITY_TYPE_MAP table. Addresses in: Data Error Analysis (Phase 2).

3. **Cursor-based row-by-row processing masks data errors through partial commits** — 181 cursor loops, 30 WHEN OTHERS handlers, two explicit `exception when others then null;` patterns. Allows "phantom success" where migration reports DONE but has silently skipped rows. Prevention: classify all exception handlers as RAISE vs LOG-and-continue vs silent-swallow. Add row-level error tracking table. Addresses in: Data Error Analysis (Phase 2) and Reconciliation (Phase 3).

4. **Cross-schema dependencies without transactional boundaries** — migration writes to four schemas (NFS_MIGRATION, NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION) with scattered COMMIT statements. Failure after committing to one schema but before another creates permanent inconsistency. Prevention: map every COMMIT location to its procedure and schemas modified. Add savepoints before cross-schema write groups. Addresses in: Data Error Analysis (Phase 2) and Reconciliation (Phase 3) for cross-schema integrity checks.

5. **Country-specific code duplication means fixing one country does not fix others** — PKG_COEXISTENCE_OLD2NEW vs PKG_COEXISTENCE_OLD2NEW_SK are copy-pasted. C_LEGAL_ENTITY used 2,458 times. Bug fixes in ITA do not automatically apply to GR/PT/SK. Prevention: diff all country variants, catalog divergences, document fixes as applying to ALL variants. Critical for CZ&SK prep. Addresses in: Data Error Analysis (Phase 2) across all package variants.

## Implications for Roadmap

Based on the five-hop architecture and the dependency between infrastructure fixes, logic correctness, and reconciliation validation, the natural phase structure follows the analysis components. Infrastructure analysis can proceed first because it requires understanding table structures but not package logic, and its results (indexes, statistics) benefit all subsequent testing. Data error analysis follows, with registry/taxonomy first due to high dependency (all other packages reference COUNTERPART and taxonomy data). Reconciliation comes last because it needs to know correct mappings from data error analysis and benefits from performance optimizations for query execution speed.

### Phase 1: Infrastructure & Performance Analysis
**Rationale:** Cross-cutting analysis that produces DDL scripts usable immediately. Does not require deep package logic understanding. Index and statistics improvements benefit all subsequent analysis work (faster test query execution).
**Delivers:**
- Complete index coverage report (520 existing indexes + missing indexes on join columns)
- DBMS_STATS gathering script for all staging/intermediate/final tables
- Sequence replacement strategy for all 42 MAX patterns
- DBMS_PARALLEL_EXECUTE configuration recommendations
**Addresses:** TS-3 (Performance Bottleneck Identification), TS-4 (Performance Fix Scripts), D-1 (MAX-to-Sequence Replacement)
**Avoids:** Pitfall 1 (MAX anti-pattern), Pitfall 7 (missing statistics)

### Phase 2: Registry & Taxonomy Data Error Analysis
**Rationale:** Highest dependency packages (PKG_MIGRATION_REGISTRY ~14K lines, PKG_MIGRATION_TAXONOMY ~2.5K lines). All other packages reference NFS_REGISTRY.COUNTERPART for ID resolution. Errors here cascade to all downstream packages.
**Delivers:**
- Entity-type mapping matrix for registry/taxonomy
- Corrected package bodies for PKG_MIGRATION_REGISTRY, PKG_MIGRATION_REGISTRY_POL, PKG_MIGRATION_TAXONOMY
- Exception handler classification (RAISE vs LOG-and-continue vs silent-swallow)
- Idempotency analysis (does it check "already migrated?")
**Uses:** Oracle sequences (from Phase 1), DBMS_STATS (from Phase 1)
**Implements:** Package-by-Package Data Error Analysis component (first iteration)
**Addresses:** TS-5 (Data Population Logic Audit), TS-6 (Error Handling Assessment), TS-1 (Table Inventory for registry domain)
**Avoids:** Pitfall 2 (entity-type mapping), Pitfall 3 (cursor error handlers), Pitfall 4 (cross-schema commits)

### Phase 3: Core Financial Domains Data Error Analysis
**Rationale:** High-volume tables (RATA_CED_DEB: 41M rows, PAREGGIO_FAT_ATT: 23M rows). Largest business impact. Six packages: CLEARING, FATTURE, FATT_ATTIVE, CONTABILITA, MANDATI, MOVIMENTI, BONIFICI.
**Delivers:**
- Entity-type mapping matrix for financial domains
- Corrected package bodies for 6 core financial packages
- Hardcoded constant audit (C_LEGAL_ENTITY, C_ABI_CSE_*, C_YEAR_*)
- Cross-schema dependency map (NFS_LEGACY lookups)
**Addresses:** TS-5 (Data Population Logic Audit), D-4 (Hardcoded Constant Audit), TS-9 (Package Execution Order)
**Avoids:** Pitfall 2 (entity-type mapping in high-volume domains), Pitfall 5 (country-specific duplication)

### Phase 4: Supporting Domains & Country Variants
**Rationale:** Lower volume, fewer dependencies. DEAL_STL_LAL, LOAN, CIGCUP, COEXISTENCE packages plus all _SK suffix variants.
**Delivers:**
- Data error analysis for supporting packages
- Country variant diff report (IT vs SK vs OLD variants)
- CZ&SK cross-cutting impact notes
**Addresses:** TS-10 (CZ&SK Cross-Cutting Impact), remaining TS-5 (Data Population Logic Audit for all packages)
**Avoids:** Pitfall 5 (country-specific duplication) by explicitly diffing variants

### Phase 5: End-to-End Reconciliation
**Rationale:** Requires knowledge of correct table mappings (from Phases 2-4) and benefits from performance optimizations (from Phase 1) for query speed. Currently only staging-to-final reconciliation exists; this phase adds SI-to-NFS end-to-end validation.
**Delivers:**
- Tier 1 (row count) reconciliation queries: SI source vs NFS_MIGRATION staging vs NFS final
- Tier 2 (aggregate) reconciliation queries: SUM of amounts, MIN/MAX dates, COUNT DISTINCT keys
- Tier 3 (record-level) MINUS queries for critical tables
- Cross-schema referential integrity queries (orphan detection)
**Addresses:** TS-2 (Row Count Reconciliation), TS-7 (Key Field Comparison), TS-8 (Aggregate Checksums), D-6 (Orphan Record Detection)
**Avoids:** Pitfall 10 (reconciliation only covers staging-to-final, not end-to-end)

### Phase Ordering Rationale

- **Infrastructure first** because index and statistics improvements benefit all subsequent query execution (both analysis work and final reconciliation queries). The 42 MAX patterns are easy to identify statically without understanding package logic.
- **Registry/taxonomy second** because every other package depends on COUNTERPART and taxonomy IDs. Fixing data errors here prevents cascading errors downstream.
- **Core financial third** because these are the highest-volume, highest-business-impact domains. By this phase, the analysis pattern (entity-type mapping matrix, exception handler classification, hardcoded constant audit) is established and can be applied systematically.
- **Supporting domains fourth** because they are lower volume and have fewer cross-package dependencies. Country variants belong here because they are compared against already-analyzed IT packages.
- **Reconciliation last** because it requires correct table mappings (output of Phases 2-4) and benefits from performance context (output of Phase 1). End-to-end reconciliation validates the correctness of all prior fixes.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (Core Financial):** High complexity due to cross-schema NFS_LEGACY lookups. May need additional research into NFS_LEGACY bridge table structure and the OLDROWID2NEW mapping mechanism. Standard pattern analysis is sufficient initially, but if the bridge logic proves more complex than cursory review suggests, `/gsd:research-phase` for "Oracle cross-schema bridging patterns" may be warranted.
- **Phase 5 (Reconciliation):** DBMS_COMPARISON framework usage for formal reconciliation. The research confirms DBMS_COMPARISON exists in Oracle 19c and works via DB links, but column name mismatches between SI and NFS schemas may require view-based abstraction. If reconciliation queries prove unmanageably complex, `/gsd:research-phase` for "Oracle DBMS_COMPARISON with schema mapping" may be helpful.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Infrastructure):** Index creation, DBMS_STATS usage, sequence replacement are well-documented Oracle standard practices. Official Oracle 19c documentation (already fetched) provides complete guidance.
- **Phase 2 (Registry/Taxonomy):** Standard PL/SQL code analysis. No specialized research needed beyond what is already in PITFALLS.md and ARCHITECTURE.md.
- **Phase 4 (Supporting Domains):** Follows the same pattern as Phase 3 but with smaller scope. No additional research needed.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All recommendations verified against Oracle 19c official documentation via WebFetch. DBMS_PARALLEL_EXECUTE, DBMS_STATS, sequences, direct-path INSERT, DBMS_COMPARISON all confirmed in Oracle 19c PL/SQL Packages Reference and SQL Tuning Guide. |
| Features | HIGH | All findings derived from direct analysis of nfs_migration.sql (99,465 lines). MAX patterns (42), cursor loops (181), WHEN OTHERS (30), EXECUTE IMMEDIATE (345) all confirmed via grep. Table stakes and differentiators prioritized based on PROJECT.md known issues (605/901 bug, performance on high-volume tables). |
| Architecture | HIGH | Five-hop flow confirmed via direct code reading. 25 packages catalogued. MGR_REQUEST_STEP orchestration pattern analyzed. Cross-schema dependency map derived from package body SQL. Component boundaries (Infrastructure vs Data Error vs Reconciliation) align with natural work partitioning. |
| Pitfalls | HIGH | All 13 pitfalls derived from actual code analysis, not generic guidance. SELECT MAX at 42 locations confirmed. Entity-type mapping errors confirmed via known 605/901 bug. Exception handlers classified. Cross-schema commits confirmed. Country duplication (PKG_*_SK variants) confirmed. |

**Overall confidence:** HIGH

### Gaps to Address

The research is code-analysis-only with no live database access. Several assumptions need validation when implementation begins:

- **Actual table row counts** — PROJECT.md provides estimates (RATA_CED_DEB: 41M, PAREGGIO_FAT_ATT: 23M) but these are projections. Actual volumes in production SI schemas may differ. Reconciliation queries should be tested with real volumes to confirm performance.
- **DB link latency** — the migration accesses SI schemas via database links (dblink parameter in loading_staging). Network latency between SI and NFS databases affects staging load performance. DBMS_STATS recommendations assume local tables; statistics on remote tables accessed via DB link may need different approach.
- **NFS reference data completeness** — entity-type mapping analysis assumes NFS lookup tables (ADT_V2_ID, ADS_V2_ID, DOT_V2_ID reference data) are complete. If NFS reference data is missing types that SI uses, the mapping matrix will reveal gaps but cannot prescribe values. Requires business domain expertise to resolve.
- **DBMS_PARALLEL_EXECUTE chunk sizing** — recommendations suggest ROWID-based chunking and parallel degree 8-16, but optimal settings depend on database server CPU/memory resources and I/O subsystem characteristics. Settings should be tested incrementally (start with degree 4, measure, increase).
- **Sequence CACHE sizing** — CACHE 10000 recommended for 40M+ row tables, but the actual optimal cache size depends on concurrent session count and migration execution pattern. If migration uses DBMS_PARALLEL_EXECUTE with degree 16, the cache must accommodate 16 parallel sessions plus some headroom. Test and adjust.
- **Column name mapping SI-to-NFS** — reconciliation queries assume column names are documented or can be inferred from INSERT statement column lists. Some SI-to-NFS mappings may involve transformations (e.g., SI column split into multiple NFS columns, or vice versa). Requires domain knowledge to map correctly.

How to handle during planning/execution:
- Request actual row counts from DBA for all high-volume tables before finalizing index and parallel execution recommendations.
- Test DB link performance with a sample staging load query (SELECT COUNT(*) vs SELECT * with ROWNUM <= 100000) to characterize latency and throughput.
- During Phase 2-4 data error analysis, query NFS reference data tables directly to validate entity-type constants exist.
- During Phase 5 reconciliation, start with Tier 1 (row count) queries on small datasets (GR/PT volumes) to validate query logic before scaling to ITA volumes.

## Sources

### Primary (HIGH confidence)
- **Direct codebase analysis** — `C:\Desktop\BFF\Code Review\NFS\Migrazione\nfs_migration.sql` (99,465 lines, exported 2025-12-16). All package structures, cursor patterns, MAX occurrences, exception handlers, EXECUTE IMMEDIATE usage, schema references confirmed via direct code reading and grep pattern matching.
- **Oracle 19c PL/SQL Packages Reference** — verified via WebFetch. DBMS_PARALLEL_EXECUTE task/chunk/run_task API, FINISHED/FINISHED_WITH_ERROR status codes, RESUME_TASK for retry confirmed.
- **Oracle 19c SQL Tuning Guide** — verified via WebFetch. DBMS_STATS.GATHER_TABLE_STATS, AUTO_SAMPLE_SIZE, online statistics gathering, real-time statistics confirmed.
- **Oracle 19c Performance Tuning Guide** — verified via WebFetch. SESSION_TRACE_ENABLE, TKPROF, AWR snapshots, ADDM analysis modes, Real-Time ADDM triggers confirmed.
- **Oracle 19c SQL Language Reference** — verified via WebFetch. INSERT...SELECT, APPEND hint, NOLOGGING mode, parallel DML, LOG ERRORS INTO, MERGE statement syntax confirmed.
- **Project context** — `.planning/PROJECT.md` (requirements, known 605/901 bug, volume estimates, CZ&SK migration scope, perimeter constraints).

### Secondary (MEDIUM confidence)
- **Codebase architecture documents** — `.planning/codebase/ARCHITECTURE.md`, `.planning/codebase/STRUCTURE.md`, `.planning/codebase/CONCERNS.md`. These are Claude-generated analysis documents, not primary source code, but cross-reference well with direct code analysis.
- **Existing CZ&SK analysis** — `ANALISI_MIGRAZIONE_BONIFICI_CZSK.md` (mentioned in FEATURES.md). Confirmed C_LEGAL_ENTITY=41 hardcoded in _SK package, validation phases SKIP'd. This is a prior analysis artifact, not original source code.

### Tertiary (LOW confidence)
- None. All findings are traced to either direct source code analysis or official Oracle 19c documentation. No third-party blogs, forum posts, or unverified community sources were used.

---
*Research completed: 2026-02-09*
*Ready for roadmap: yes*