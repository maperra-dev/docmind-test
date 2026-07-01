---
unique-name: pitfalls
display-name: PITFALLS
category: GENERAL
description: **Domain:** Oracle PL/SQL data migration analysis for financial factoring system (SI to NFS)
---

# Domain Pitfalls

**Domain:** Oracle PL/SQL data migration analysis for financial factoring system (SI to NFS)
**Researched:** 2026-02-09
**Confidence:** HIGH (based on direct codebase analysis of 99,465-line nfs_migration.sql)

---

## Critical Pitfalls

Mistakes that cause data corruption, silent failures, or require full re-migration.

### Pitfall 1: SELECT MAX(id) for Identity Generation Is Not Concurrency-Safe

**What goes wrong:** The migration uses `SELECT MAX(column) INTO variable` at least 42 times across the codebase to generate new IDs (e.g., `SELECT max(ctp_id) INTO l_ctp_id FROM NFS_REGISTRY.COUNTERPART`). When parallel sessions or DBMS_PARALLEL_EXECUTE chunks run concurrently, two sessions can read the same MAX value before either inserts, producing duplicate IDs. Even in single-session mode, the full-table scan for MAX degrades to O(n) per row inserted, making it O(n^2) for the entire migration.

**Why it happens:** The original GUPTA/SI system was single-user or low-concurrency. Developers replicated the pattern without considering that NFS target tables grow during migration and that DBMS_PARALLEL_EXECUTE (confirmed at line ~40005) runs multiple chunks simultaneously.

**Consequences:**
- **Duplicate primary keys** causing ORA-00001 violations that silently break chunks in DBMS_PARALLEL_EXECUTE (chunk status = PROCESSED_WITH_ERROR, but migration continues).
- **Wrong foreign key references** when a child record references an ID that was also assigned to a different parent.
- **Performance cliff** on tables like RATA_CED_DEB (41M rows) and PAREGGIO_FAT_ATT (23M rows) -- each INSERT triggers a full scan of the growing target table.

**Prevention:**
- **In analysis phase:** Catalog every `SELECT MAX(...)` occurrence with the table it scans and the context (parallel-safe or not). Cross-reference with DBMS_PARALLEL_EXECUTE usage to identify which MAX calls are concurrency-exposed.
- **In fix phase:** Replace each MAX with a proper Oracle SEQUENCE (the schema already defines sequences like SQ_ACD, SQ_ACL, etc. -- verify they cover all MAX usage points). For the `NFS_REGISTRY.COUNTERPART.ctp_id` case (line 73860), the identity column's auto-increment sequence should be used directly.

**Detection (warning signs):**
- Grep for `SELECT\s+MAX\s*\(` followed by `INTO` in PL/SQL blocks.
- Check if any MAX target tables are also targets of DBMS_PARALLEL_EXECUTE chunks.
- Look for `ORA-00001` in MGR_LOG_ELAB error records.

**Which analysis phase should address it:** Performance Analysis (Phase 1) for the O(n^2) scan problem; Data Error Analysis (Phase 2) for the concurrency/duplicate-ID risk.

---

### Pitfall 2: Entity Type Mapping Errors Propagate Silently Across All Downstream Tables

**What goes wrong:** The known 605/901 sequence error (Payment Notice detail uses entity-type sequence 605 instead of 901 for invoice lookup) is a symptom of a systemic problem: the migration maps SI entity types (codice sequences, type codes, status codes) to NFS entity types through hardcoded constants and cursor-based lookups. A single wrong mapping constant in a package header propagates to every row processed by that package's procedures. Because the migration has 25+ packages and the entity-type mappings are scattered (not centralized), the same type of error can exist in multiple packages without detection.

**Why it happens:**
- SI and NFS use different numbering schemes for the same business concepts (e.g., SI "tipo_documento" codes vs. NFS "ADT_V2_ID" codes).
- Mappings are embedded inline in SELECT statements and CASE expressions rather than in a single mapping/lookup table.
- No automated cross-validation exists to confirm that the mapped NFS entity type actually exists in the NFS reference data.

**Consequences:**
- **Orphaned foreign keys** -- migrated records reference NFS type IDs that do not exist in NFS lookup tables, causing application-level errors when NFS/Quarkus microservices try to load the data.
- **Wrong business behavior** -- records assigned to the wrong entity type will be processed incorrectly by NFS business logic (e.g., an invoice treated as a mandate).
- **Multiplied scope** -- if one mapping is wrong, statistically similar mappings in other packages are also suspect. With 25+ packages, this can affect thousands of records silently.

**Prevention:**
- **In analysis phase:** Build a complete entity-type mapping matrix: for each package, list every hardcoded type constant (ADT_V2_ID, ADS_V2_ID, DOT_V2_ID, etc.) and cross-reference it against NFS reference data. Flag any constant that does not exist in the target NFS lookup table.
- **In fix phase:** Create a centralized NFS_MIGRATION.ENTITY_TYPE_MAP table and rewrite inline mappings to use it, enabling a single point of validation.
- **In reconciliation phase:** Add a "referential integrity check" query for every migrated table: `SELECT COUNT(*) FROM migrated_table t WHERE NOT EXISTS (SELECT 1 FROM nfs_lookup l WHERE l.type_id = t.type_id)`.

**Detection (warning signs):**
- Any hardcoded numeric literal in a migration SELECT that represents an entity type (look for patterns like `= 605`, `= 901`, `= 'DEF'`).
- Cursor queries that join to NFS_LEGACY lookup tables with no `WHERE EXISTS` validation.
- The `COE_V2_ID` and `DOT_V2_ID` columns being set to string literals like `'DEF'` without validation.

**Which analysis phase should address it:** Data Error Analysis (Phase 2) as the primary phase. Reconciliation (Phase 3) as the validation phase.

---

### Pitfall 3: Cursor-Based Row-by-Row Processing Masks Data Errors Through Partial Commits

**What goes wrong:** The migration processes data in cursor FOR loops (181 cursor loop occurrences found) with COMMIT statements inside or immediately after loop completion. When an exception occurs mid-loop, the WHEN OTHERS handler (30 occurrences) typically logs the error and either continues to the next row or rolls back only the current batch. This means the migration can "succeed" (status = DONE) while having skipped/corrupted an unknown number of rows.

**Why it happens:**
- Row-by-row processing was necessary in the SI/GUPTA era for memory constraints.
- WHEN OTHERS handlers are used as a safety net, but many simply log and continue: `exception when others then log_elab(...)`.
- Two handlers explicitly swallow errors: `exception when others then null;` (lines 19385, 19915).
- The MGR_REQUEST status tracking only records DONE/ERROR at the request level, not at the individual row level.

**Consequences:**
- **Phantom success** -- migration reports completion but has silently skipped rows. This is the most dangerous failure mode because it is invisible until downstream systems process the data.
- **Inconsistent parent-child relationships** -- if a parent row fails but child processing continues (or vice versa), the NFS database has orphaned records.
- **Impossible reconciliation** -- without knowing which rows were skipped, row-count reconciliation between SI and NFS will show unexplained deltas.

**Prevention:**
- **In analysis phase:** For each cursor loop, trace the exception handling path. Classify handlers as: (a) RAISE (re-throws, safe), (b) LOG+CONTINUE (dangerous -- rows silently skipped), (c) NULL (most dangerous -- errors invisible). Document the count of each type per package.
- **In reconciliation phase:** Design row-level reconciliation that compares SI source counts against NFS final counts per entity, per legal entity. Any delta must be explainable.
- **In fix phase:** For LOG+CONTINUE handlers, add row-level error tracking (insert failed row keys into a MGR_MIGRATION_ERRORS table).

**Detection (warning signs):**
- `WHEN OTHERS THEN` not followed by `RAISE` in cursor processing blocks.
- Row count deltas between SI source and NFS final that do not correspond to known validation rejections.
- The `exception when others then null;` pattern (confirmed at lines 19385, 19915).

**Which analysis phase should address it:** Data Error Analysis (Phase 2) for handler classification; Reconciliation (Phase 3) for detecting the damage.

---

### Pitfall 4: Cross-Schema Dependencies Without Transactional Boundaries

**What goes wrong:** The migration writes to four different Oracle schemas in a single logical transaction: NFS_MIGRATION (staging), NFS_REGISTRY (counterparts, taxonomy -- 806 references), NFS_PRODUCTS (documents, instalments -- 330 references), NFS_COLLECTION (bank transfers -- 95 references), and reads from NFS_LEGACY (268 references). COMMIT statements are scattered throughout procedures (not at consistent transaction boundaries), and ROLLBACK only appears in exception handlers. If a procedure fails after committing to NFS_REGISTRY but before committing the corresponding NFS_PRODUCTS records, the cross-schema data is permanently inconsistent.

**Why it happens:**
- Oracle does not support distributed transactions across schemas on the same instance without explicit savepoint management.
- The migration was built incrementally, with each package handling its own commits.
- The RESET procedures (RESET_NFS_MIGRATION, RESET_NFS_REGISTRY_ANAGRAFICA, etc.) use DELETE-based cleanup, not TRUNCATE, making re-runs slow on large tables.

**Consequences:**
- **Cross-schema inconsistency** -- NFS_REGISTRY has the counterpart but NFS_PRODUCTS is missing the linked documents.
- **Non-idempotent re-runs** -- running the migration again after a partial failure can produce duplicate records in schemas that already committed.
- **Orphaned cleanup** -- the RESET procedures delete from NFS_REGISTRY tables in a specific order (line 19252-19273), but if the deletion order does not match foreign key dependencies, ORA-02292 (child record found) blocks cleanup.

**Prevention:**
- **In analysis phase:** Map every COMMIT location to its procedure and identify which schemas have been modified before that COMMIT. Document "commit scope" for each procedure.
- **In fix phase:** Introduce explicit savepoints before each cross-schema write group, and ensure ROLLBACK TO SAVEPOINT on failure.
- **In reconciliation phase:** Cross-schema referential integrity checks (e.g., every NFS_PRODUCTS.accounting_document must have a corresponding NFS_REGISTRY.COUNTERPART).

**Detection (warning signs):**
- COMMIT statements inside cursor loops (rather than after complete entity processing).
- Multiple RESET procedures that must be run in a specific order.
- EXECUTE IMMEDIATE 'TRUNCATE TABLE...' for some tables (line 19020-19058) but DELETE for others -- inconsistent cleanup strategy.

**Which analysis phase should address it:** Architecture/dependency mapping (pre-Phase 1); Reconciliation (Phase 3) for cross-schema integrity checks.

---

### Pitfall 5: Country-Specific Code Duplication Means Fixing One Country Does Not Fix Others

**What goes wrong:** The migration has separate package variants per country: PKG_COEXISTENCE_OLD2NEW and PKG_COEXISTENCE_OLD2NEW_SK are nearly identical (lines 19924-19978 vs 19982-20036) with the same structure and constants but potentially different logic paths. PKG_DEAL_STL_LAL and PKG_DEAL_STL_LAL_OLD are another copy pair. The C_LEGAL_ENTITY constant is used 2,458 times across the file. When a bug is fixed in the ITA version, the identical fix must be manually applied to GR, PT, and SK versions. History shows this does not happen reliably.

**Why it happens:**
- Each country was onboarded at different times (ITA first, then GR/PT, SK planned).
- Rather than parameterizing country-specific logic, developers copy-pasted entire packages and changed the legal entity constant.
- No automated test suite verifies behavioral equivalence across country variants.

**Consequences:**
- **Fix regression** -- a bug fixed in PKG_DEAL_STL_LAL for ITA still exists in the GR/PT variant.
- **Analysis scope explosion** -- every pitfall found must be checked across all country variants (multiplied by the number of packages).
- **CZ&SK migration risk** -- the upcoming CZ&SK migration will likely be created by copying from the ITA/SK variants, inheriting all unfixed bugs.

**Prevention:**
- **In analysis phase:** Identify all package pairs/groups that are country variants. Diff them to catalog divergences. Any divergence is either (a) a legitimate country-specific rule, or (b) a bug that was fixed in one copy but not another.
- **In fix phase:** Document fixes as applying to ALL country variants, not just the one where the bug was discovered.
- **In CZ&SK prep:** Flag which fixes from ITA/GR/PT must be pre-applied to the CZ&SK template before it is created.

**Detection (warning signs):**
- Package names ending in `_SK`, `_OLD`, or with country suffixes.
- Identical procedure signatures in different packages.
- The same bug pattern (e.g., SELECT MAX) appearing in one variant but already fixed in another.

**Which analysis phase should address it:** Data Error Analysis (Phase 2) -- must audit all variants, not just ITA.

---

## Moderate Pitfalls

### Pitfall 6: Dynamic SQL (EXECUTE IMMEDIATE) Hides Compile-Time Errors

**What goes wrong:** The migration uses EXECUTE IMMEDIATE 345 times. Dynamic SQL bypasses Oracle's compile-time validation -- column name typos, wrong table names, and type mismatches only surface at runtime. The validation rules procedure (lines 37280-37338) builds SQL strings dynamically from MGR_CONFIGURATION table data, meaning a configuration error produces a syntactically valid but semantically wrong query.

**Prevention:**
- **In analysis phase:** Extract all EXECUTE IMMEDIATE strings and attempt to parse them statically. Flag any that reference columns or tables not in the DDL.
- **In fix phase:** Where possible, replace EXECUTE IMMEDIATE with static SQL. Where dynamic SQL is necessary (the validation engine), add a "dry run" mode that explains the generated SQL without executing it.

**Detection (warning signs):**
- EXECUTE IMMEDIATE with string concatenation (SQL injection risk and validation-bypass risk).
- The `CASE r_fields.CFG_V2_RULE WHEN 'V1' THEN...` block (line 37297) -- any new rule added to MGR_CONFIGURATION that does not match a CASE branch falls through to `ELSE` which generates a no-op `SELECT 1 FROM DUAL` -- silently skipping validation.

**Which analysis phase should address it:** Data Error Analysis (Phase 2).

---

### Pitfall 7: Missing Table Statistics Cause Optimizer Plan Flips

**What goes wrong:** The migration gathers statistics on some staging tables (lines 61823, 61829) but has commented-out statistics gathering for critical tables: `-- DBMS_STATS.GATHER_TABLE_STATS('NFS_MIGRATION', 'MOVIMENTO_VALORIZZ')` (line 70984) and `--DBMS_STATS.GATHER_TABLE_STATS('NFS_MIGRATION', 'ACCOUNTING_LOG')` (line 71017). These commented-out calls appear in duplicate at lines 72511 and 72544, confirming they were intentionally disabled rather than forgotten. Without current statistics on tables that grow from 0 to millions of rows during migration, the Oracle optimizer uses default assumptions (small table) and chooses nested-loop joins instead of hash joins, causing orders-of-magnitude performance degradation.

**Prevention:**
- **In analysis phase:** Catalog every DBMS_STATS call (active and commented-out) and the table it targets. Cross-reference with the high-volume tables list.
- **In fix phase:** Add DBMS_STATS.GATHER_TABLE_STATS calls at strategic points: (a) after initial bulk load into staging, (b) after staging-to-final transfer, (c) before any large join operation.

**Detection (warning signs):**
- Commented-out `DBMS_STATS` calls (confirmed at 4 locations).
- Procedures that bulk-insert millions of rows but do not gather stats before the next query that reads those rows.
- Performance that is acceptable for GR/PT (small volumes) but catastrophic for ITA (large volumes).

**Which analysis phase should address it:** Performance Analysis (Phase 1).

---

### Pitfall 8: RESET Procedures Use DELETE Instead of TRUNCATE for Large Tables

**What goes wrong:** The RESET_NFS_REGISTRY_ANAGRAFICA procedure (line 19249) uses `DELETE FROM NFS_REGISTRY.COUNTERPART` and similar DELETE statements for 20+ tables. DELETE generates undo/redo for every row, is logged, and respects foreign key constraints. On tables with millions of rows, this takes hours and fills the undo tablespace. By contrast, the RESET_NFS_MIGRATION procedure (line 19096) uses `EXECUTE IMMEDIATE 'TRUNCATE TABLE...'` for its tables. The inconsistency means re-running the migration requires hours of cleanup for the registry/products schemas.

**Prevention:**
- **In analysis phase:** Document which RESET procedures use DELETE vs. TRUNCATE. Flag any DELETE on a table with >100K expected rows.
- **In fix phase:** Convert DELETE to TRUNCATE where FK constraints allow (disable FKs, truncate, re-enable). For tables with cross-schema FKs that cannot be disabled, use `DELETE /*+ PARALLEL(4) */` with periodic commits.

**Detection (warning signs):**
- RESET procedures that take >10 minutes to complete.
- ORA-30036 (unable to extend undo segment) during RESET operations.

**Which analysis phase should address it:** Performance Analysis (Phase 1).

---

### Pitfall 9: No Idempotency Guard on Migration Procedures

**What goes wrong:** If a migration run fails mid-way and is restarted, most procedures will re-process records that were already successfully migrated. The `check_start` function (line 20085) checks request status (DONE/IN_PROGRESS/ERROR) but this is at the request level, not the entity level. The OLDROWID2NEW mapping table (NFS_LEGACY.OLDROWID2NEW) tracks some old-to-new ID relationships, but it is only used in 3 joins (lines 13839, 13940, 15331) -- the vast majority of procedures do not check "has this record already been migrated?"

**Prevention:**
- **In analysis phase:** For each procedure, determine: (a) Does it check whether the target record already exists before inserting? (b) Does it use MERGE/UPSERT or INSERT with ON DUPLICATE KEY logic? (c) What happens if run twice for the same data?
- **In fix phase:** Add idempotency checks using MERGE statements or pre-check queries against the OLDROWID2NEW mapping.

**Detection (warning signs):**
- ORA-00001 (unique constraint violated) when restarting a failed migration.
- Duplicate records in NFS target tables after a restart.
- The `IS_TRANSF` flag in MGR_COUNTERPART (line 73810) -- this is a transfer flag, but check if it is reliably set on success and reset on failure.

**Which analysis phase should address it:** Data Error Analysis (Phase 2).

---

### Pitfall 10: Reconciliation Only Covers Staging-to-Final, Not End-to-End

**What goes wrong:** The existing reconciliation compares NFS_MIGRATION staging tables against NFS final tables (NFS_PRODUCTS, NFS_REGISTRY, NFS_COLLECTION). This misses two critical failure points: (a) SI source to NFS_MIGRATION staging (data lost or transformed incorrectly during DB link extraction), and (b) data type/precision loss during transformation (e.g., SI NUMBER(18,4) truncated to NFS NUMBER(15,2) for financial amounts).

**Prevention:**
- **In reconciliation phase:** Design three-tier reconciliation:
  1. **Tier 1 (Completeness):** SI source row count vs. NFS final row count per entity, per legal entity.
  2. **Tier 2 (Accuracy):** SI source aggregate amounts (SUM of financial columns) vs. NFS final aggregate amounts. Any delta indicates data transformation errors.
  3. **Tier 3 (Referential):** For every FK relationship in NFS, verify that the referenced parent record exists.
- **SI access consideration:** Since this is code-analysis-only (no live DB), the reconciliation queries must be designed to be run by the team. Include both the SI-side query and the NFS-side query, clearly documented.

**Detection (warning signs):**
- Row count deltas between tiers that exceed known validation-rejection counts.
- Financial amount sums that differ by >0.01 between SI and NFS.
- The DB link parameter in `loading_staging` (line 20066: `in_dblink`) -- verify the DB link is used consistently and does not silently fail on connectivity issues.

**Which analysis phase should address it:** Reconciliation (Phase 3) -- this is the primary purpose of that phase.

---

## Minor Pitfalls

### Pitfall 11: Mixed Case Table Names Cause Portability Issues

**What goes wrong:** Some tables use quoted mixed-case names: `"TBL_rec_doi_values"` (line 19051), `"TBL_rec_acd_values"` (line 19052). Oracle stores unquoted identifiers in uppercase and quoted identifiers in the exact case specified. Any SQL that references these tables without the exact quoted form will fail with ORA-00942 (table or view does not exist). This creates fragile code where a missing double-quote breaks a query.

**Prevention:** In analysis phase, catalog all mixed-case table names and verify that every reference uses consistent quoting.

**Which analysis phase should address it:** Data Error Analysis (Phase 2).

---

### Pitfall 12: Hardcoded Schema Qualifiers Prevent Environment Portability

**What goes wrong:** The migration hardcodes schema names: `NFS_MIGRATION`, `NFS_REGISTRY`, `NFS_PRODUCTS`, `NFS_COLLECTION`, `NFS_LEGACY`. If the target environment uses different schema names (e.g., for CZ&SK or a test environment), every reference must be manually updated.

**Prevention:** This is a lower-priority concern for the current analysis (ITA/GR/PT schemas are stable), but flag it as a risk for CZ&SK preparation.

**Which analysis phase should address it:** CZ&SK cross-cutting notes.

---

### Pitfall 13: SEQUENCE Start Values May Collide with Existing Data

**What goes wrong:** The MIG_COUNTERPART_SEQ sequence starts at 9,000,000 (line 19226: `create sequence NFS_MIGRATION.MIG_COUNTERPART_SEQ start with 9000000`). If the NFS_REGISTRY.COUNTERPART table already has records with ctp_id > 9,000,000 from previous migration runs or application inserts, the sequence will generate colliding IDs. The RESET procedure drops and recreates the sequence (lines 19222-19226), but does not check the current MAX(ctp_id) before setting the start value.

**Prevention:** In fix phase, modify sequence creation to use `START WITH (SELECT MAX(ctp_id)+1 FROM NFS_REGISTRY.COUNTERPART)` or equivalent dynamic logic.

**Which analysis phase should address it:** Performance Analysis (Phase 1, sequence management) and Data Error Analysis (Phase 2, collision risk).

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Performance Analysis | Focusing only on indexes while ignoring the 42 SELECT MAX calls and commented-out DBMS_STATS | Treat MAX-to-SEQUENCE replacement and statistics gathering as equal priority to index creation |
| Data Error Analysis (ITA) | Fixing the 605/901 error in isolation without checking for similar mapping errors in other packages | Build a complete entity-type mapping matrix across ALL packages before fixing any single error |
| Data Error Analysis (variants) | Auditing only the ITA package variant and assuming GR/PT/SK are identical | Diff all country variants; any divergence is a suspect bug |
| Reconciliation Design | Writing reconciliation queries that compare row counts only | Must include financial amount aggregates (SUM), referential integrity checks, and per-legal-entity breakdowns |
| Reconciliation Execution | Designing SI-side queries that assume DB link access from NFS | Since this is code-analysis-only, provide standalone SQL that the team can run separately on SI and NFS, then compare results externally |
| Fix Script Delivery | Delivering fix scripts without rollback procedures | Every fix script must have a corresponding UNDO script or at minimum a pre-execution backup query |
| CZ&SK Preparation | Assuming CZ&SK can copy from "fixed" ITA packages | CZ&SK template must be created AFTER all ITA fixes are verified, not branched from the current broken code |
| Parallel Execution | Testing fixes with small datasets (GR/PT volumes) and assuming they work at ITA scale | Performance fixes must be validated with ITA volume estimates (RATA_CED_DEB: 41M rows, PAREGGIO_FAT_ATT: 23M rows) |

---

## Cascading Risk Assessment

The pitfalls above are not independent. They cascade:

```
SELECT MAX anti-pattern (Pitfall 1)
  --> wrong IDs generated (Pitfall 2 amplified)
    --> child records reference wrong parents
      --> reconciliation shows unexplained deltas (Pitfall 10)
        --> but cursor error handlers swallow the root cause (Pitfall 3)
          --> fix applied to ITA but not GR/PT (Pitfall 5)
```

**The correct analysis order is therefore:**
1. Map the full entity-type landscape and identify all mapping constants (Pitfall 2 -- foundational)
2. Catalog all SELECT MAX and assess concurrency risk (Pitfall 1 -- performance + correctness)
3. Classify all exception handlers to understand error visibility (Pitfall 3 -- determines what we can trust)
4. Design reconciliation that accounts for known error patterns (Pitfall 10 -- validation)
5. Apply analysis to all country variants (Pitfall 5 -- scope completeness)

---

## Sources

- **Primary:** Direct analysis of `C:/Desktop/BFF/Code Review/NFS/Migrazione/nfs_migration.sql` (99,465 lines, exported 2025-12-16)
- **Project context:** `C:/Desktop/BFF/Code Review/Migrazione/.planning/PROJECT.md`
- **Pattern counts verified by grep:**
  - SELECT MAX: 42 occurrences
  - CURSOR declarations: 51
  - Cursor FOR loops: 181
  - EXECUTE IMMEDIATE: 345
  - WHEN OTHERS: 30
  - BULK COLLECT/FORALL: 9 (extremely low for a 99K-line migration)
  - NFS_LEGACY references: 268
  - NFS_PRODUCTS references: 330
  - NFS_REGISTRY references: 806
  - NFS_COLLECTION references: 95
  - LEGAL_ENTITY/LEN_ID references: 2,458
  - INDEX creations: 534
- **Confidence:** HIGH -- all findings are from the actual migration source code, not generic guidance