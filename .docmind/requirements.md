---
unique-name: requirements
display-name: REQUIREMENTS
category: GENERAL
description: **Defined:** 2026-02-09
---

# Requirements: NFS Migration Analysis & Fix

**Defined:** 2026-02-09
**Core Value:** Every record migrated from SI to NFS must be correct and verifiable -- data integrity is non-negotiable.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Performance Infrastructure

- [ ] **PERF-01**: Complete index coverage report for all NFS_MIGRATION staging, intermediate, and NFS final tables -- identify missing indexes on JOIN columns, FK columns, and WHERE clause predicates
- [ ] **PERF-02**: DBMS_STATS gathering script for all staging, intermediate, and final tables across NFS_MIGRATION, NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION schemas -- including the 4 commented-out calls found at lines 70984, 71017, 72511, 72544
- [ ] **PERF-03**: Replace all 42 SELECT MAX(id) patterns with Oracle sequences -- generate CREATE SEQUENCE DDL with CACHE 10000 for high-volume tables (RATA_CED_DEB 41M, PAREGGIO_FAT_ATT 23M) and CACHE 1000 for lower-volume tables
- [ ] **PERF-04**: DBMS_PARALLEL_EXECUTE configuration recommendations -- extend existing clearing implementation to all high-volume domains with chunk sizing and parallel degree guidance
- [ ] **PERF-05**: CREATE INDEX DDL scripts ready for execution -- covering staging table load columns, validation rule columns, and final table FK/lookup columns

### Data Error Analysis -- Registry & Taxonomy

- [ ] **DREG-01**: Entity-type mapping matrix for PKG_MIGRATION_REGISTRY -- cross-reference all hardcoded entity-type constants (ADT_V2_ID, ADS_V2_ID, DOT_V2_ID) against NFS reference data
- [ ] **DREG-02**: Data population logic audit for PKG_MIGRATION_REGISTRY (~14K lines) -- line-by-line review of INSERT logic for incorrect mappings, wrong sequences, missing joins
- [ ] **DREG-03**: Data population logic audit for PKG_MIGRATION_REGISTRY_POL -- same audit as DREG-02 for the POL variant
- [ ] **DREG-04**: Data population logic audit for PKG_MIGRATION_TAXONOMY (~2.5K lines) -- entity-type mapping validation and INSERT logic review
- [ ] **DREG-05**: Exception handler classification for registry/taxonomy packages -- classify all WHEN OTHERS handlers as RAISE vs LOG-and-continue vs silent-swallow (including the 2 explicit `then null` at lines 19385, 19915)
- [ ] **DREG-06**: Idempotency analysis for registry/taxonomy packages -- document what happens on re-run (duplicates? updates? errors?)

### Data Error Analysis -- Core Financial Domains

- [ ] **DFIN-01**: Entity-type mapping matrix for PKG_MIGRATION_CLEARING -- validate all hardcoded constants for pareggio tables (PAREGGIO_FAT_ATT 23M rows, PAREGGIO_CED_DEB 722K, PAREGGIO_NON_FAT 121K)
- [ ] **DFIN-02**: Data population logic audit for PKG_MIGRATION_CLEARING -- INSERT logic review for all clearing/pareggio tables
- [ ] **DFIN-03**: Data population logic audit for PKG_MIGRATION_FATTURE and PKG_MIGRATION_FATT_ATTIVE -- invoice migration logic review (FATTURA_ATTIVA 321K rows)
- [ ] **DFIN-04**: Data population logic audit for PKG_MIGRATION_CONTABILITA -- accounting migration logic review (CONTABILE_FF 363K rows)
- [ ] **DFIN-05**: Data population logic audit for PKG_MIGRATION_MANDATI -- mandate migration logic review (MANDATI 326K rows)
- [ ] **DFIN-06**: Data population logic audit for PKG_MIGRATION_MOVIMENTI -- movement migration logic review
- [ ] **DFIN-07**: Data population logic audit for PKG_MIGRATION_BONIFICI -- bank transfer migration logic review
- [ ] **DFIN-08**: Hardcoded constant audit across all financial packages -- catalog all C_LEGAL_ENTITY, C_ABI_CSE_*, C_YEAR_* constants and validate values
- [ ] **DFIN-09**: Exception handler classification for all financial packages -- same classification as DREG-05
- [ ] **DFIN-10**: Cross-schema dependency map -- document all NFS_LEGACY lookups and OLDROWID2NEW mapping mechanisms

### Data Error Analysis -- Supporting Domains & Country Variants

- [ ] **DSUP-01**: Data population logic audit for PKG_DEAL_STL_LAL and PKG_DEAL_STL_LAL_OLD -- deal/settlement/legal action logic review
- [ ] **DSUP-02**: Data population logic audit for PKG_MIGRATION_LOAN -- loan migration logic review
- [ ] **DSUP-03**: Data population logic audit for PKG_MIGRATION_CIGCUP -- CIG/CUP migration logic review
- [ ] **DSUP-04**: Data population logic audit for PKG_COEXISTENCE_OLD2NEW -- coexistence data path review
- [ ] **DSUP-05**: Country variant diff report -- systematic diff between IT packages and _SK variants (PKG_MIGRATION_BONIFICI vs PKG_MIGRATION_BONIFICI_SK, PKG_MIGRATION_CLEARING vs PKG_MIGRATION_CLEARING_SK, etc.)
- [ ] **DSUP-06**: CZ&SK cross-cutting impact notes -- flag all issues found in ITA analysis that will also affect the CZ&SK migration

### Reconciliation

- [ ] **RECON-01**: Tier 1 row count reconciliation queries -- SI source vs NFS final row counts for every migration table
- [ ] **RECON-02**: Tier 2 aggregate reconciliation queries -- SUM of monetary columns, MIN/MAX dates, COUNT DISTINCT keys for critical tables
- [ ] **RECON-03**: Tier 3 record-level MINUS queries -- exact record comparison for discrepancy detection on high-priority tables
- [ ] **RECON-04**: Cross-schema referential integrity queries -- orphan record detection across NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION, NFS_LEGACY
- [ ] **RECON-05**: Reconciliation execution guide -- documentation for the team on how to run queries, interpret results, and triage discrepancies

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Performance Deep Optimization

- **PERF-V2-01**: Cursor-to-bulk conversion -- replace row-by-row cursor loops with FORALL/BULK COLLECT for all 181 cursor instances
- **PERF-V2-02**: Direct-path INSERT/APPEND hint implementation for all staging loads
- **PERF-V2-03**: Session-level parallel DML configuration scripts with NOLOGGING mode

### Advanced Analysis

- **ADV-V2-01**: Parallel execution safety analysis -- verify DBMS_PARALLEL_EXECUTE chunk safety with MAX patterns
- **ADV-V2-02**: Data type mismatch detection -- SI vs NFS column type compatibility audit
- **ADV-V2-03**: DBMS_COMPARISON-based automated reconciliation framework

## Out of Scope

| Feature | Reason |
|---------|--------|
| CZ&SK migration implementation | Separate future project -- only cross-cutting notes captured here |
| NFS staging-to-NFS final reconciliation | Already exists and working |
| Live database execution | Code analysis only -- scripts delivered for team to run |
| Non-incassi perimeter migration | Outside current scope -- collections only |
| Automated migration rewrite | Too high-risk -- surgical fixes to existing code preferred |
| Test framework creation | Deliverables are analysis + scripts, not test suites |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| RECON-01 | Phase 1: Core Reconciliation Queries | Pending |
| RECON-02 | Phase 1: Core Reconciliation Queries | Pending |
| RECON-03 | Phase 1: Core Reconciliation Queries | Pending |
| RECON-04 | Phase 2: Integrity Validation & Execution Guide | Pending |
| RECON-05 | Phase 2: Integrity Validation & Execution Guide | Pending |
| PERF-01 | Phase 3: Index & Statistics Analysis | Pending |
| PERF-02 | Phase 3: Index & Statistics Analysis | Pending |
| PERF-05 | Phase 3: Index & Statistics Analysis | Pending |
| PERF-03 | Phase 4: Sequence & Parallel Optimization | Pending |
| PERF-04 | Phase 4: Sequence & Parallel Optimization | Pending |
| DREG-01 | Phase 5: Registry Data Error Analysis | Pending |
| DREG-02 | Phase 5: Registry Data Error Analysis | Pending |
| DREG-03 | Phase 5: Registry Data Error Analysis | Pending |
| DREG-04 | Phase 6: Taxonomy & Error Handling Analysis | Pending |
| DREG-05 | Phase 6: Taxonomy & Error Handling Analysis | Pending |
| DREG-06 | Phase 6: Taxonomy & Error Handling Analysis | Pending |
| DFIN-01 | Phase 7: Clearing & Invoice Data Error Analysis | Pending |
| DFIN-02 | Phase 7: Clearing & Invoice Data Error Analysis | Pending |
| DFIN-03 | Phase 7: Clearing & Invoice Data Error Analysis | Pending |
| DFIN-04 | Phase 8: Accounting & Mandates Data Error Analysis | Pending |
| DFIN-05 | Phase 8: Accounting & Mandates Data Error Analysis | Pending |
| DFIN-06 | Phase 8: Accounting & Mandates Data Error Analysis | Pending |
| DFIN-07 | Phase 8: Accounting & Mandates Data Error Analysis | Pending |
| DFIN-08 | Phase 9: Financial Cross-Cutting Analysis | Pending |
| DFIN-09 | Phase 9: Financial Cross-Cutting Analysis | Pending |
| DFIN-10 | Phase 9: Financial Cross-Cutting Analysis | Pending |
| DSUP-01 | Phase 10: Supporting Domains & Country Variants | Pending |
| DSUP-02 | Phase 10: Supporting Domains & Country Variants | Pending |
| DSUP-03 | Phase 10: Supporting Domains & Country Variants | Pending |
| DSUP-04 | Phase 10: Supporting Domains & Country Variants | Pending |
| DSUP-05 | Phase 10: Supporting Domains & Country Variants | Pending |
| DSUP-06 | Phase 10: Supporting Domains & Country Variants | Pending |

**Coverage:**
- v1 requirements: 32 total
- Mapped to phases: 32
- Unmapped: 0

---
*Requirements defined: 2026-02-09*
*Last updated: 2026-02-10 after roadmap revision (reconciliation prioritized to Phases 1-2)*