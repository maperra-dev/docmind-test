---
unique-name: state
display-name: STATE
category: GENERAL
description: See: .planning/PROJECT.md (updated 2026-02-09)
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-09)

**Core value:** Every record migrated from SI to NFS must be correct and verifiable -- data integrity is non-negotiable.
**Current focus:** Phase 2 COMPLETE -- ready for Phase 3 (Index & Statistics Analysis)

## Current Position

Phase: 2 of 10 (Integrity Validation & Execution Guide) -- COMPLETE
Plan: 5 of 5 in current phase -- PHASE COMPLETE
Status: Phase complete
Last activity: 2026-02-12 -- Completed 02-05-PLAN.md (Coverage Validation & Phase 2 Completion)

Progress: [======================........................................] 22%
Phase 1:  [======] 6/6 COMPLETE
Phase 2:  [=====] 5/5 COMPLETE

## Performance Metrics

**Velocity:**
- Total plans completed: 11
- Average duration: 3.5min
- Total execution time: 0.68 hours (41min)

**By Phase:**

| Phase | Plans | Total | Avg/Plan | Status |
|-------|-------|-------|----------|--------|
| 01-core-reconciliation-queries | 6/6 | 23min | 3.8min | COMPLETE |
| 02-integrity-validation-execution-guide | 5/5 | 18min | 3.6min | COMPLETE |

**Recent Trend:**
- Last 10 plans: 01-02 (4min), 01-03 (4min), 01-04 (4min), 01-05 (7min), 01-06 (2min), 02-01 (2min), 02-02 (4min), 02-03 (3min), 02-04 (3min), 02-05 (6min)
- Trend: stable at ~3.5min/plan

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- [Roadmap]: 10-phase comprehensive structure derived from 5 requirement categories
- [Roadmap]: Research recommends registry/taxonomy before financial domains due to COUNTERPART dependency cascade
- [Roadmap Revision]: Reconciliation queries promoted to Phases 1-2 per user priority -- immediate data integrity visibility before infrastructure or code analysis work
- [Roadmap Revision]: Reconciliation queries prioritized by table volume: RATA_CED_DEB (41.2M) > PAREGGIO_FAT_ATT (23.8M) > PAREGGIO_CED_DEB (722K) > CONTABILE_FF (363K) > MANDATI (326K) > FATTURA_ATTIVA (321K) > PAREGGIO_NON_FAT (121K) > remaining tables
- [01-01]: Volume-priority ranking established as canonical reconciliation sequence
- [01-01]: Three-tier directory structure (row counts -> aggregates -> MINUS) adopted as escalation path
- [01-01]: All queries parameterized with :req_id, :len_id, {dblink} -- no hardcoded legal entity IDs
- [01-01]: Spain DB link left as placeholder {dblink_ES} pending team confirmation
- [01-02]: NFS_LEGACY.DOI_* bridge tables used as NFS_FINAL count proxy for tables with bridge (RATA_CED_DEB, CONTABILE_FF, FATTURA_ATTIVA)
- [01-02]: V_QUAD views used for NFS_FINAL derivation on CLEARING-target tables (PAREGGIO_FAT_ATT, PAREGGIO_CED_DEB, PAREGGIO_NON_FAT)
- [01-02]: PAREGGIO_FAT_ATT and PAREGGIO_NON_FAT confirmed as no-IS_VALID tables (ROW_ID-based)
- [01-02]: PAYMENT_NOTICE_DETAIL reconciliation is two-point only with MANDATO cross-reference
- [01-03]: NFS_LEGACY bridge tables used for NFS_FINAL counts (BAT_PAGAMENTO_FCD_FAA, BTD_RISTR_PAGAMENTO, LEA_AZIONE_LEGALE, STL_CARICO, DEA_OPERAZIONE_MAPROS)
- [01-03]: ACCOUNTING_LOG is NFS-domain intermediate (not SI source) -- DDL-verified, maps to NFS_COLLECTION.ACCOUNTING_LOG
- [01-03]: IS_VALID absent from MOVIMENTO_VALORIZZ, PAREGGIO_CBL, ACCOUNTING_LOG -- DDL-verified, REQ_NM_ID-only filtering
- [01-03]: Registry tables (ANAGRAFICA_CED/DEB) produce 1:N COUNTERPART records -- NFS_FINAL > SI_SOURCE is expected
- [01-03]: PAREGGIO_CBL NFS_FINAL count deferred to V_QUAD approach due to shared NFS_COLLECTION.CLEARING
- [01-04]: DDL-verified column names: DT_PFA_GFA (not DT_GFA) for PAREGGIO_FAT_ATT, DT_PCD_GCD for PAREGGIO_CED_DEB
- [01-04]: PAGAMENTO_FCD_FAA uses COD_CED/COD_DEB (not COD_AGT) per DDL
- [01-04]: CONTABILE_FF direct-sum columns (COMMISSIONE, INTERESSI, SPESE, IMP_NETTO, IMP_COMP_INC) confirmed as no-DECODE
- [01-05]: V_QUAD column expressions copied verbatim -- no simplification of production-tested transformation logic
- [01-05]: PAREGGIO_FAT_ATT uses aa_pcd >= 2010 filter from V_QUAD (not IS_VALID)
- [01-05]: FATTURA_ATTIVA nested DECODE for Italy-specific COD_CED=96 recipient resolution preserved exactly
- [01-05]: MANDATO complex join pattern includes PAREGGIO_CED_DEB + DOI_RATA_CED_DEB subquery for PND_NM_AMOUNT
- [01-06]: All 19 migration tables confirmed covered at Tier 1 (100%)
- [01-06]: 8 monetary tables confirmed covered at Tier 2 (100% of applicable)
- [01-06]: 5 V_QUAD tables confirmed covered at Tier 3 (100% of applicable)
- [01-06]: PAREGGIO_CED_DEB and PAREGGIO_NON_FAT Tier 3 classified as optional enhancement (V_QUAD views exist in DB)
- [01-06]: Phase 1 all 4 success criteria (SC1-SC4) confirmed FULLY MET
- [02-01]: All NFS_REGISTRY internal FK relationships treated as logical (no database constraints) -- queries are safety net
- [02-01]: Reverse check included for COUNTERPARTs with no LINK_CTP_RTP_RST rows (incomplete migration detection)
- [02-02]: Pre-flight FK constraint status check placed in 02_products_orphans.sql only (first file in execution order)
- [02-02]: TABLE_ORIG excluded from CLEARING detail queries -- exists only on NFS_MIGRATION staging, not NFS_COLLECTION final
- [02-02]: Consolidated UNION ALL LEN_ID check for all 8 NFS_LEGACY tables (single efficient query)
- [02-02]: No :len_id filter on CLEARING queries -- CLEARING lacks LEN_ID column
- [02-03]: EXECUTION_GUIDE.md replaces README.md as authoritative execution reference (510 lines vs 40-line signpost)
- [02-03]: 6-phase execution order: pre-flight (mandatory) -> tier 1 -> tier 2 (conditional) -> tier 3 (conditional) -> integrity (independent) -> cross-reference
- [02-03]: Country escalation: GR/PT first (validation), then SK (multi-LEN_ID), then IT (production scale), then ES (pending dblink)
- [02-03]: Performance threshold: PARALLEL(4) hint for queries exceeding 30 minutes; check indexes first
- [02-04]: 8 discrepancy patterns documented (A-H) -- added Pattern G (COUNT DISTINCT) and Pattern H (correlated orphans) beyond plan spec
- [02-04]: Volume-based severity adjustment: Tier A tables (>1M rows) automatically at least HIGH severity
- [02-04]: TRIAGE_GUIDE.md references EXECUTION_GUIDE.md as companion document (forward reference until 02-03 completes)
- [02-05]: COVERAGE_CHECK.md extended with sections 7-10 for Phase 2 integrity coverage (456 lines total)
- [02-05]: All 36 FK relationships mapped to specific integrity SQL files and section numbers
- [02-05]: Phase 2 success criteria SC1-SC4 all verified FULLY MET with file evidence
- [02-05]: Reconciliation suite totals: 29 files, ~6,954 lines across Phase 1 + Phase 2

### Pending Todos

None.

### Blockers/Concerns

- [Research]: Phase 7 (Clearing) may reveal NFS_LEGACY bridge logic more complex than expected -- may need research
- [Research]: Phase 1 (Reconciliation) DBMS_COMPARISON with schema name mismatches may require view-based abstraction

## Session Continuity

Last session: 2026-02-12
Stopped at: Phase 2 COMPLETE. Ready for Phase 3 (Index & Statistics Analysis)
Resume file: .planning/phases/02-integrity-validation-execution-guide/02-05-SUMMARY.md