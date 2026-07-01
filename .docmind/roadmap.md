---
unique-name: roadmap
display-name: ROADMAP
category: GENERAL
description: This roadmap delivers a comprehensive analysis and remediation of the SI-to-NFS data migration for the collections (incassi) perimeter. The work starts with end-to-end reconciliation queries -- priori
---

# Roadmap: NFS Migration Analysis & Fix

## Overview

This roadmap delivers a comprehensive analysis and remediation of the SI-to-NFS data migration for the collections (incassi) perimeter. The work starts with end-to-end reconciliation queries -- prioritized by table volume -- that validate every migrated record between SI source and NFS final, giving immediate visibility into data integrity. It then progresses through infrastructure performance fixes (indexes, statistics, sequences) and systematic package-by-package data error analysis (registry, financial, supporting domains). All 32 requirements are mapped across 10 phases, producing executable SQL/PL-SQL scripts as deliverables.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Core Reconciliation Queries** - SI source vs NFS final row count, aggregate, and record-level reconciliation queries prioritized by table volume (completed 2026-02-12)
- [x] **Phase 2: Integrity Validation & Execution Guide** - Cross-schema referential integrity queries and reconciliation execution documentation (completed 2026-02-12)
- [ ] **Phase 3: Index & Statistics Analysis** - Identify missing indexes and generate statistics gathering scripts for all migration schemas
- [ ] **Phase 4: Sequence & Parallel Optimization** - Replace all MAX-based ID patterns with Oracle sequences and configure parallel execution
- [ ] **Phase 5: Registry Data Error Analysis** - Audit PKG_MIGRATION_REGISTRY and PKG_MIGRATION_REGISTRY_POL for data mapping errors
- [ ] **Phase 6: Taxonomy & Error Handling Analysis** - Audit PKG_MIGRATION_TAXONOMY plus classify exception handlers and idempotency across registry/taxonomy
- [ ] **Phase 7: Clearing & Invoice Data Error Analysis** - Audit highest-volume clearing and invoice packages for data mapping errors
- [ ] **Phase 8: Accounting & Mandates Data Error Analysis** - Audit contabilita, mandati, movimenti, and bonifici packages
- [ ] **Phase 9: Financial Cross-Cutting Analysis** - Hardcoded constant audit, exception handler classification, and cross-schema dependency map for all financial packages
- [ ] **Phase 10: Supporting Domains & Country Variants** - Audit supporting packages and produce country variant diffs with CZ&SK impact notes

## Phase Details

### Phase 1: Core Reconciliation Queries
**Goal**: Three tiers of SI-source-vs-NFS-final reconciliation queries are ready to run for every migration table, prioritized by table volume so the highest-impact tables are validated first
**Depends on**: Nothing (first phase)
**Requirements**: RECON-01, RECON-02, RECON-03
**Success Criteria** (what must be TRUE):
  1. Tier 1 row count reconciliation queries exist comparing SI source vs NFS final row counts for every migration table, ordered by volume priority: RATA_CED_DEB (41.2M) first, then PAREGGIO_FAT_ATT (23.8M), PAREGGIO_CED_DEB (722K), CONTABILE_FF (363K), MANDATI (326K), FATTURA_ATTIVA (321K), PAYMENT_NOTICE_DETAIL, PAREGGIO_NON_FAT (121K), then all remaining tables
  2. Tier 2 aggregate reconciliation queries exist checking SUM of monetary columns, MIN/MAX dates, and COUNT DISTINCT keys for all critical tables -- following the same volume-priority ordering
  3. Tier 3 record-level MINUS queries exist for all high-priority tables (RATA_CED_DEB through PAREGGIO_NON_FAT at minimum) enabling exact record comparison for discrepancy detection
  4. Each query is self-contained (includes necessary schema prefixes, DB link references) and executable by the team
**Plans:** 6 plans

Plans:
- [x] 01-01-PLAN.md -- Build SI-to-NFS table mapping reference with volume-priority ranking
- [x] 01-02-PLAN.md -- Generate Tier 1 row count queries for 8 high-priority tables
- [x] 01-03-PLAN.md -- Generate Tier 1 row count queries for all remaining migration tables
- [x] 01-04-PLAN.md -- Generate Tier 2 aggregate queries for critical financial tables
- [x] 01-05-PLAN.md -- Generate Tier 3 record-level MINUS queries for high-priority tables
- [x] 01-06-PLAN.md -- Validate query completeness against full table inventory

### Phase 2: Integrity Validation & Execution Guide
**Goal**: Cross-schema referential integrity is validated and the team has a complete guide for running all reconciliation queries
**Depends on**: Phase 1 (core reconciliation queries complete)
**Requirements**: RECON-04, RECON-05
**Success Criteria** (what must be TRUE):
  1. Cross-schema referential integrity queries exist detecting orphan records across NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION, NFS_LEGACY
  2. An execution guide documents how to run all reconciliation queries (Tier 1/2/3 plus integrity checks)
  3. The execution guide includes result interpretation instructions and discrepancy triage guidance
  4. The guide specifies recommended execution order (start with GR/PT small volumes, then scale to ITA)
**Plans:** 5 plans

Plans:
- [x] 02-01-PLAN.md -- Generate NFS_REGISTRY internal orphan detection queries (12 relationships)
- [x] 02-02-PLAN.md -- Generate NFS_PRODUCTS, NFS_COLLECTION, NFS_LEGACY orphan detection queries (24 relationships)
- [x] 02-03-PLAN.md -- Write comprehensive reconciliation execution guide (replaces README.md)
- [x] 02-04-PLAN.md -- Write result interpretation and discrepancy triage guide
- [x] 02-05-PLAN.md -- Update COVERAGE_CHECK.md with integrity coverage and Phase 2 verification

### Phase 3: Index & Statistics Analysis
**Goal**: The team has a complete picture of missing database infrastructure and ready-to-run DDL scripts for indexes and statistics
**Depends on**: Nothing (independent of reconciliation phases)
**Requirements**: PERF-01, PERF-02, PERF-05
**Success Criteria** (what must be TRUE):
  1. A complete index coverage report exists identifying every missing index on JOIN columns, FK columns, and WHERE predicates across NFS_MIGRATION staging, intermediate, and NFS final tables
  2. A DBMS_STATS gathering script exists covering all staging, intermediate, and final tables across NFS_MIGRATION, NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION schemas -- including the 4 commented-out calls at lines 70984, 71017, 72511, 72544
  3. CREATE INDEX DDL scripts are ready for execution covering staging table load columns, validation rule columns, and final table FK/lookup columns
  4. Each script includes a header documenting which tables/columns are targeted and why
**Plans**: TBD

Plans:
- [ ] 03-01: Catalog existing indexes and identify gaps on staging tables
- [ ] 03-02: Catalog existing indexes and identify gaps on intermediate and final tables
- [ ] 03-03: Generate DBMS_STATS gathering script for all schemas
- [ ] 03-04: Generate CREATE INDEX DDL scripts
- [ ] 03-05: Produce index coverage report with gap analysis

### Phase 4: Sequence & Parallel Optimization
**Goal**: All MAX-based ID generation anti-patterns are replaced with Oracle sequences and parallel execution is configured for high-volume tables
**Depends on**: Phase 3 (index/statistics context informs parallel configuration)
**Requirements**: PERF-03, PERF-04
**Success Criteria** (what must be TRUE):
  1. CREATE SEQUENCE DDL exists for all 42 SELECT MAX(id) patterns, with CACHE 10000 for high-volume tables (RATA_CED_DEB 41M, PAREGGIO_FAT_ATT 23M) and CACHE 1000 for lower-volume tables
  2. Each sequence DDL includes the replacement SQL showing old MAX pattern and new NEXTVAL usage
  3. DBMS_PARALLEL_EXECUTE configuration recommendations exist with chunk sizing and parallel degree guidance for all high-volume domains
  4. The parallel configuration extends the existing clearing implementation pattern to all applicable domains
**Plans**: TBD

Plans:
- [ ] 04-01: Catalog all 42 SELECT MAX patterns with table volumes and context
- [ ] 04-02: Generate CREATE SEQUENCE DDL with appropriate CACHE sizes
- [ ] 04-03: Generate replacement SQL for each MAX-to-sequence conversion
- [ ] 04-04: Produce DBMS_PARALLEL_EXECUTE configuration recommendations
- [ ] 04-05: Document sequence initialization strategy (setting START WITH based on current MAX)

### Phase 5: Registry Data Error Analysis
**Goal**: PKG_MIGRATION_REGISTRY and PKG_MIGRATION_REGISTRY_POL are fully audited with all data mapping errors documented and corrected INSERT logic provided
**Depends on**: Phase 3 (statistics context), Phase 4 (sequence replacements referenced in audit)
**Requirements**: DREG-01, DREG-02, DREG-03
**Success Criteria** (what must be TRUE):
  1. An entity-type mapping matrix exists cross-referencing all hardcoded constants (ADT_V2_ID, ADS_V2_ID, DOT_V2_ID) in PKG_MIGRATION_REGISTRY against NFS reference data
  2. A line-by-line audit report exists for PKG_MIGRATION_REGISTRY (~14K lines) documenting every incorrect mapping, wrong sequence, and missing join in INSERT logic
  3. A line-by-line audit report exists for PKG_MIGRATION_REGISTRY_POL documenting the same categories of errors
  4. For each identified error, the corrected PL/SQL logic is provided alongside the original
**Plans**: TBD

Plans:
- [ ] 05-01: Build entity-type mapping matrix for registry packages
- [ ] 05-02: Audit PKG_MIGRATION_REGISTRY staging and validation logic
- [ ] 05-03: Audit PKG_MIGRATION_REGISTRY loading_final and transfer logic
- [ ] 05-04: Audit PKG_MIGRATION_REGISTRY_POL (full package)
- [ ] 05-05: Compile registry data error report with corrected logic

### Phase 6: Taxonomy & Error Handling Analysis
**Goal**: PKG_MIGRATION_TAXONOMY is audited, and exception handling behavior and idempotency characteristics are fully documented for all registry/taxonomy packages
**Depends on**: Phase 5 (registry audit establishes patterns used here)
**Requirements**: DREG-04, DREG-05, DREG-06
**Success Criteria** (what must be TRUE):
  1. PKG_MIGRATION_TAXONOMY (~2.5K lines) is fully audited for entity-type mapping validation and INSERT logic errors
  2. Every WHEN OTHERS handler in registry/taxonomy packages is classified as RAISE, LOG-and-continue, or silent-swallow -- including the 2 explicit "then null" handlers at lines 19385 and 19915
  3. Idempotency analysis documents what happens on re-run for each registry/taxonomy package (duplicates created, records updated, errors thrown, or correctly skipped)
  4. Recommendations exist for each silent-swallow handler and each non-idempotent path
**Plans**: TBD

Plans:
- [ ] 06-01: Audit PKG_MIGRATION_TAXONOMY entity-type mappings and INSERT logic
- [ ] 06-02: Classify all exception handlers in registry/taxonomy packages
- [ ] 06-03: Analyze idempotency behavior for PKG_MIGRATION_REGISTRY
- [ ] 06-04: Analyze idempotency behavior for PKG_MIGRATION_REGISTRY_POL and PKG_MIGRATION_TAXONOMY
- [ ] 06-05: Compile error handling and idempotency report with recommendations

### Phase 7: Clearing & Invoice Data Error Analysis
**Goal**: The highest-volume migration packages (clearing/pareggio and invoice/fatture) are fully audited with all data mapping errors documented
**Depends on**: Phase 5 (registry audit reveals entity-type patterns reused here), Phase 6 (error handling classification pattern established)
**Requirements**: DFIN-01, DFIN-02, DFIN-03
**Success Criteria** (what must be TRUE):
  1. An entity-type mapping matrix exists for PKG_MIGRATION_CLEARING validating all hardcoded constants for PAREGGIO_FAT_ATT (23M rows), PAREGGIO_CED_DEB (722K), and PAREGGIO_NON_FAT (121K)
  2. A data population audit exists for PKG_MIGRATION_CLEARING covering all INSERT logic for clearing/pareggio tables
  3. A data population audit exists for PKG_MIGRATION_FATTURE and PKG_MIGRATION_FATT_ATTIVE covering invoice migration logic (FATTURA_ATTIVA 321K rows)
  4. The known 605/901 Payment Notice bug is fully documented with root cause and fix
**Plans**: TBD

Plans:
- [ ] 07-01: Build entity-type mapping matrix for clearing packages
- [ ] 07-02: Audit PKG_MIGRATION_CLEARING data population logic
- [ ] 07-03: Audit PKG_MIGRATION_FATTURE data population logic
- [ ] 07-04: Audit PKG_MIGRATION_FATT_ATTIVE data population logic (including 605/901 fix)
- [ ] 07-05: Compile clearing and invoice data error report

### Phase 8: Accounting & Mandates Data Error Analysis
**Goal**: Contabilita, mandati, movimenti, and bonifici migration packages are fully audited with all data mapping errors documented
**Depends on**: Phase 7 (clearing audit provides cross-reference for shared financial entity mappings)
**Requirements**: DFIN-04, DFIN-05, DFIN-06, DFIN-07
**Success Criteria** (what must be TRUE):
  1. PKG_MIGRATION_CONTABILITA is fully audited for data population errors (CONTABILE_FF 363K rows)
  2. PKG_MIGRATION_MANDATI is fully audited for data population errors (MANDATI 326K rows)
  3. PKG_MIGRATION_MOVIMENTI is fully audited for data population errors
  4. PKG_MIGRATION_BONIFICI is fully audited for data population errors
  5. For each package, corrected PL/SQL logic is provided alongside each identified error
**Plans**: TBD

Plans:
- [ ] 08-01: Audit PKG_MIGRATION_CONTABILITA data population logic
- [ ] 08-02: Audit PKG_MIGRATION_MANDATI data population logic
- [ ] 08-03: Audit PKG_MIGRATION_MOVIMENTI data population logic
- [ ] 08-04: Audit PKG_MIGRATION_BONIFICI data population logic
- [ ] 08-05: Compile accounting and mandates data error report

### Phase 9: Financial Cross-Cutting Analysis
**Goal**: All cross-cutting concerns across financial packages are catalogued -- hardcoded constants validated, exception handlers classified, and cross-schema dependencies mapped
**Depends on**: Phase 7, Phase 8 (individual package audits complete; this phase synthesizes cross-cutting patterns)
**Requirements**: DFIN-08, DFIN-09, DFIN-10
**Success Criteria** (what must be TRUE):
  1. A complete catalog exists of all C_LEGAL_ENTITY, C_ABI_CSE_*, C_YEAR_* constants across all financial packages with validation of each value
  2. Every WHEN OTHERS handler across all financial packages is classified as RAISE, LOG-and-continue, or silent-swallow
  3. A cross-schema dependency map documents all NFS_LEGACY lookups and OLDROWID2NEW mapping mechanisms used in financial packages
  4. Recommendations exist for each invalid constant value, each silent-swallow handler, and each unsafe cross-schema dependency
**Plans**: TBD

Plans:
- [ ] 09-01: Catalog and validate all hardcoded constants across financial packages
- [ ] 09-02: Classify all exception handlers in financial packages
- [ ] 09-03: Map cross-schema dependencies and NFS_LEGACY lookups
- [ ] 09-04: Map OLDROWID2NEW bridging mechanisms
- [ ] 09-05: Compile financial cross-cutting analysis report with recommendations

### Phase 10: Supporting Domains & Country Variants
**Goal**: Supporting packages are audited, country variant divergences are documented, and CZ&SK cross-cutting impact notes are complete
**Depends on**: Phase 9 (financial cross-cutting patterns inform country variant analysis)
**Requirements**: DSUP-01, DSUP-02, DSUP-03, DSUP-04, DSUP-05, DSUP-06
**Success Criteria** (what must be TRUE):
  1. PKG_DEAL_STL_LAL and PKG_DEAL_STL_LAL_OLD are fully audited for deal/settlement/legal action logic errors
  2. PKG_MIGRATION_LOAN, PKG_MIGRATION_CIGCUP, and PKG_COEXISTENCE_OLD2NEW are fully audited for data population errors
  3. A systematic diff report exists for every IT-vs-SK package variant (PKG_MIGRATION_BONIFICI vs PKG_MIGRATION_BONIFICI_SK, PKG_MIGRATION_CLEARING vs PKG_MIGRATION_CLEARING_SK, etc.)
  4. CZ&SK cross-cutting impact notes flag every ITA issue that will also affect the CZ&SK migration
  5. Each country-variant divergence is classified as intentional (country-specific logic) or accidental (missed bug fix)
**Plans**: TBD

Plans:
- [ ] 10-01: Audit PKG_DEAL_STL_LAL and PKG_DEAL_STL_LAL_OLD
- [ ] 10-02: Audit PKG_MIGRATION_LOAN data population logic
- [ ] 10-03: Audit PKG_MIGRATION_CIGCUP and PKG_COEXISTENCE_OLD2NEW
- [ ] 10-04: Produce country variant diff report (IT vs SK for all package pairs)
- [ ] 10-05: Compile CZ&SK cross-cutting impact notes
- [ ] 10-06: Classify variant divergences as intentional vs accidental

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Core Reconciliation Queries | 6/6 | Complete | 2026-02-12 |
| 2. Integrity Validation & Execution Guide | 5/5 | Complete | 2026-02-12 |
| 3. Index & Statistics Analysis | 0/5 | Not started | - |
| 4. Sequence & Parallel Optimization | 0/5 | Not started | - |
| 5. Registry Data Error Analysis | 0/5 | Not started | - |
| 6. Taxonomy & Error Handling Analysis | 0/5 | Not started | - |
| 7. Clearing & Invoice Data Error Analysis | 0/5 | Not started | - |
| 8. Accounting & Mandates Data Error Analysis | 0/5 | Not started | - |
| 9. Financial Cross-Cutting Analysis | 0/5 | Not started | - |
| 10. Supporting Domains & Country Variants | 0/6 | Not started | - |