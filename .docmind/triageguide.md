---
unique-name: triageguide
display-name: TRIAGE GUIDE
category: GENERAL
description: > **Companion to:** `EXECUTION_GUIDE.md` -- Run queries first (following EXECUTION_GUIDE.md), then use this guide to interpret results.
---

# Reconciliation Result Interpretation & Triage Guide

> **Companion to:** `EXECUTION_GUIDE.md` -- Run queries first (following EXECUTION_GUIDE.md), then use this guide to interpret results.
>
> **Scope:** All reconciliation queries in `reconciliation/tier1-row-counts/`, `tier2-aggregates/`, `tier3-minus/`, and `integrity/`.
>
> **Audience:** Migration team members interpreting query output during SI-to-NFS reconciliation validation.

---

## Table of Contents

1. [How to Read This Guide](#1-how-to-read-this-guide)
2. [Expected Results -- What "Clean" Looks Like](#2-expected-results----what-clean-looks-like)
3. [Common Discrepancy Patterns](#3-common-discrepancy-patterns)
4. [Orphan Triage Workflows](#4-orphan-triage-workflows)
5. [Severity Classification Matrix](#5-severity-classification-matrix)
6. [Escalation Decision Tree](#6-escalation-decision-tree)

---

## 1. How to Read This Guide

This guide complements `EXECUTION_GUIDE.md`. The workflow is:

1. **Run queries** following `EXECUTION_GUIDE.md` (parameter resolution, execution order, output capture).
2. **Compare results** against the "Expected Results" table below (Section 2).
3. **If discrepancies exist**, look up the pattern in "Common Discrepancy Patterns" (Section 3).
4. **If orphans are found**, follow the step-by-step workflow in "Orphan Triage Workflows" (Section 4).
5. **Classify severity** using the matrix (Section 5).
6. **Decide next steps** using the escalation decision tree (Section 6).

### Severity Levels

| Level | Meaning | Response Time |
|-------|---------|---------------|
| **CRITICAL** | Blocks go-live. Data loss or financial integrity compromised. | Immediate -- stop validation and investigate. |
| **HIGH** | Must fix before production. Orphan references or broken reconciliation chains. | Investigate within 24 hours. |
| **MEDIUM** | Investigate, may be acceptable. Row count deltas on medium-volume tables. | Investigate within 1 week. Document findings. |
| **LOW** | Informational, likely acceptable. Expected behaviors (year filters, 1:N expansion). | Document and accept. |

### Notation Conventions

- `SI_SOURCE` = original SI schema table (accessed via DB link)
- `STAGING` = NFS_MIGRATION staging table
- `NFS_FINAL` = target NFS schema table (NFS_PRODUCTS, NFS_COLLECTION, NFS_REGISTRY, NFS_LEGACY)
- `:req_id` = MGR_REQUEST.REQ_NM_ID bind variable
- `:len_id` = Legal entity ID bind variable
- `{dblink}` = DB link placeholder (country-specific)

---

## 2. Expected Results -- What "Clean" Looks Like

A **clean** reconciliation means every query tier returns results within expected tolerances. Use this table as a quick reference after running each query tier.

| Query Tier | Clean Result | What It Means |
|------------|--------------|---------------|
| **Tier 1 Row Counts** | SI_SOURCE = STAGING = NFS_FINAL (or NFS_FINAL >= SI_SOURCE for 1:N registry tables) | Every record was loaded and transferred |
| **Tier 2 Aggregates** | SUM/MIN/MAX/COUNT DISTINCT match exactly across all three layers | Financial amounts are preserved precisely |
| **Tier 3 MINUS** | 0 rows in both directions (staging-not-in-final AND final-not-in-staging) | No records lost or duplicated |
| **Integrity Orphans** | 0 orphan count for every relationship | All cross-schema references are valid |
| **Pre-Flight FK Check** | 0 rows in disabled constraint query | All FK constraints are active |

### Tier-Specific Notes

**Tier 1 -- Row Counts:**
- Registry tables (ANAGRAFICA_CED, ANAGRAFICA_DEB) are expected to show NFS_FINAL > SI_SOURCE because the registry migration produces 1:N COUNTERPART + LINK_CTP_RTP_RST records per SI source row.
- Staging counts may be less than SI_SOURCE if year-range filtering is intentional (see Pattern A below).
- CLEARING-target tables (PAREGGIO_FAT_ATT, PAREGGIO_CED_DEB, PAREGGIO_NON_FAT, MOVIMENTO_VALORIZZ) use V_QUAD-derived NFS_FINAL counts.

**Tier 2 -- Aggregates:**
- Monetary columns using sign decoding (DECODE) must match exactly. Even a 0.01 difference on high-volume tables is significant.
- Tables without DECODE (CONTABILE_FF direct-sum columns) should match with zero tolerance.

**Tier 3 -- MINUS:**
- Both directions matter: staging-not-in-final detects data loss; final-not-in-staging detects phantom records.
- V_QUAD tables use the view's column expressions for comparison, not raw column values.

**Integrity Orphans:**
- Any non-zero orphan count requires investigation.
- Relationships without FK constraints (CTP_NM_ID_ISSUER, CTP_NM_ID_RECIPIENT) are highest risk because the database does NOT prevent orphan creation.

**Pre-Flight FK Check:**
- Must return 0 rows. Any disabled FK constraint means the migration Transfer step did not complete properly.
- Run this BEFORE orphan detection queries -- disabled FKs explain many orphan findings.

---

## 3. Common Discrepancy Patterns

For each pattern: **Symptom** (what you see), **Root Cause** (why it happens), **Severity**, and **Action** (what to do).

### Pattern A: Tier 1 -- Staging count < SI source count

- **Symptom:** STAGING_COUNT < SI_SOURCE_COUNT
- **Root cause:** Year-range filtering during `loading_staging`. `MGR_REQUEST` has `REQ_NM_YEAR_FROM` / `REQ_NM_YEAR_TO` that excludes older records.
- **Severity:** LOW -- expected behavior if year-range filtering is intentional
- **Action:**
  1. Check `REQ_NM_YEAR_FROM` / `REQ_NM_YEAR_TO` values for the `:req_id`.
  2. If filter is intentional, document as accepted delta.
  3. If unintentional, investigate `loading_staging` logic.
  4. Calculate the exclusion percentage: `(SI_SOURCE - STAGING) / SI_SOURCE * 100`. If >10%, request team confirmation (see Section 6).

### Pattern B: Tier 1 -- NFS final count > SI source count (Registry tables)

- **Symptom:** NFS_FINAL_COUNT > SI_SOURCE_COUNT for ANAGRAFICA_CED or ANAGRAFICA_DEB
- **Root cause:** Registry migration produces 1:N COUNTERPART + LINK_CTP_RTP_RST records per SI source row. This is **normal**.
- **Severity:** LOW -- expected 1:N expansion
- **Action:**
  1. No action needed.
  2. Document the expansion ratio (e.g., 1:2.3 for typical registry).
  3. If ratio seems abnormally high (>1:10), request team confirmation (see Section 6).

### Pattern C: Tier 2 -- Monetary SUM mismatch with doubled values

- **Symptom:** NFS_FINAL SUM is approximately 2x the SI_SOURCE SUM
- **Root cause:** Missing `DECODE(sign_column, '+', amount, -amount)` in the aggregate. Negative amounts are being added instead of subtracted.
- **Severity:** HIGH -- if the query is correct, this indicates a data population error in the migration code
- **Action:**
  1. First verify the query uses correct sign handling (check for DECODE on sign columns).
  2. If the query is missing DECODE: fix the query, re-run, compare again.
  3. If the query is correct (DECODE present and correct): escalate as a data mapping bug -- the migration code is not applying sign correctly.
  4. Cross-reference with the V_QUAD view definition to verify expected sign handling.

### Pattern D: Tier 3 -- Rows appear in staging-not-in-final only

- **Symptom:** MINUS query returns rows that exist in staging but NOT in NFS final
- **Root cause:** Transfer step may have failed or been interrupted. Records loaded to staging but not transferred to final schema.
- **Severity:** CRITICAL -- data loss (records exist in staging but were not migrated to final)
- **Action:**
  1. Check `MGR_REQUEST` status for the affected scope (`:req_id`).
  2. Check if `MANAGE_INDEX.ENABLE_FK_DOCUMENT` completed successfully.
  3. Check if the Transfer step logged any errors.
  4. **Escalate immediately** -- do not continue validation until resolved.
  5. Document exact row count and sample IDs from the MINUS output.

### Pattern E: Tier 3 -- Rows appear in final-not-in-staging only

- **Symptom:** MINUS query returns rows that exist in NFS final but NOT in staging
- **Root cause:** Either (1) a previous migration run left records in final, or (2) another process inserted records directly into NFS final.
- **Severity:** MEDIUM -- unexpected but not necessarily data loss
- **Action:**
  1. Check if records belong to a different `REQ_NM_ID` (leftover from a previous migration run).
  2. Check `DELETE_DATE` on the NFS final side -- soft-deleted records may appear as phantoms.
  3. If records belong to the current `:req_id`, investigate the Transfer step for duplicates.
  4. Document findings and discuss with the team.

### Pattern F: Tier 1 -- NFS final count via V_QUAD does not match staging count (CLEARING-target tables)

- **Symptom:** For PAREGGIO_FAT_ATT, PAREGGIO_CED_DEB, PAREGGIO_NON_FAT, MOVIMENTO_VALORIZZ: V_QUAD-derived NFS final counts differ from staging counts
- **Root cause:** Multiple SI source tables share the NFS_COLLECTION.CLEARING target. V_QUAD isolates per-source counts using column mapping logic. If the column mapping is incorrect, counts will be wrong.
- **Severity:** MEDIUM to HIGH depending on magnitude
- **Action:**
  1. Cross-reference with the V_QUAD view definitions in the database.
  2. Verify `TABLE_ORIG` column (if available on staging) to trace source.
  3. Compare the delta magnitude: small deltas (<1%) may indicate V_QUAD filter differences; large deltas (>5%) indicate a mapping problem.
  4. If V_QUAD-derived counts show different deltas than direct staging queries, request team confirmation (see Section 6).

### Pattern G: Tier 2 -- COUNT DISTINCT mismatch on key columns

- **Symptom:** COUNT DISTINCT on a key column (e.g., DOI_NM_ID, CTP_NM_ID) differs across layers
- **Root cause:** Either (1) duplicates were introduced during migration, or (2) the key column mapping is incorrect.
- **Severity:** HIGH -- indicates potential data integrity issue
- **Action:**
  1. Run a `GROUP BY key_column HAVING COUNT(*) > 1` query on the layer with the higher distinct count to find duplicates.
  2. If duplicates exist in NFS_FINAL but not in STAGING: Transfer step created duplicates.
  3. If duplicates exist in STAGING but not in SI_SOURCE: `loading_staging` created duplicates.
  4. Escalate as a data integrity bug.

### Pattern H: Integrity -- Orphan count matches across multiple relationships for the same parent table

- **Symptom:** Multiple relationships referencing the same parent table (e.g., NFS_REGISTRY.COUNTERPART) all show orphans, often with similar counts
- **Root cause:** The parent table itself is incomplete -- a batch of records was never migrated.
- **Severity:** HIGH to CRITICAL -- systematic migration failure
- **Action:**
  1. Check the parent table's Tier 1 row count (SI_SOURCE vs NFS_FINAL).
  2. If NFS_FINAL < SI_SOURCE: parent table migration is incomplete. Resolve at the parent level first.
  3. Do NOT investigate individual orphan relationships until the parent table is reconciled.
  4. Escalate as a migration completeness issue.

---

## 4. Orphan Triage Workflows

These workflows cover the three highest-risk orphan relationships identified during integrity query development. Each workflow provides a step-by-step investigation path from summary count to root cause.

### Workflow 1: Orphan CTP_NM_ID_ISSUER / CTP_NM_ID_RECIPIENT (NO FK constraint)

**Source file:** `reconciliation/integrity/02_products_orphans.sql` (relationships #1 and #2)

**Risk level:** HIGH -- No database FK constraint exists. The database does NOT prevent orphan creation. These are the highest-risk columns in the entire migration.

**Context:** `NFS_PRODUCTS.ACCOUNTING_DOCUMENT_DATA` stores CTP_NM_ID_ISSUER and CTP_NM_ID_RECIPIENT columns that logically reference `NFS_REGISTRY.COUNTERPART.CTP_ID`, but no FK constraint enforces this relationship.

**Investigation steps:**

1. **Run the summary count** from `integrity/02_products_orphans.sql` (Section 1 and Section 2).
   - If `orphan_count = 0`: no action needed. Move to next relationship.
   - If `orphan_count > 0`: continue to step 2.

2. **Run the detail query** to get specific orphan CTP IDs.
   - Note the `orphan_ctp_id` values and `LEN_ID` for each orphan row.

3. **For each orphan CTP_NM_ID:**

   a. **Check if counterpart was ever migrated:**
   ```sql
   SELECT CTP_ID, CTP_NM_CODE, DELETE_DATE
     FROM NFS_REGISTRY.COUNTERPART
    WHERE CTP_ID = :orphan_ctp_id;
   ```
   - If **not found**: the counterpart migration failed for this record. Go to step 3b.
   - If **found with DELETE_DATE set**: the counterpart exists but is soft-deleted. The orphan detection query's `DELETE_DATE IS NULL` filter correctly excludes it. This is a valid orphan -- the parent was soft-deleted but the child still references it. Go to step 3d.
   - If **found with DELETE_DATE IS NULL**: the orphan detection query may have a bug. Re-check the JOIN condition.

   b. **Check NFS_MIGRATION staging for the original record:**
   ```sql
   SELECT *
     FROM NFS_MIGRATION.ANAGRAFICA_CED
    WHERE CTP_NM_ID = :orphan_ctp_id
      AND REQ_NM_ID = :req_id;
   -- Also check ANAGRAFICA_DEB if the CTP is a debtor
   ```
   - If found in staging but not in NFS_REGISTRY: the Transfer step failed for this counterpart.
   - If not found in staging: the `loading_staging` step failed for this counterpart.

   c. **Check if the ACCOUNTING_DOCUMENT_DATA record belongs to a different legal entity** than the expected COUNTERPART:
   ```sql
   SELECT ad.LEN_ID AS document_len_id, ctp.LEN_ID AS counterpart_len_id
     FROM NFS_PRODUCTS.ACCOUNTING_DOCUMENT ad
     JOIN NFS_PRODUCTS.ACCOUNTING_DOCUMENT_DATA add_ ON ad.ADD_NM_ID_CURRENT = add_.ADD_NM_ID
     LEFT JOIN NFS_REGISTRY.COUNTERPART ctp ON ctp.CTP_ID = add_.CTP_NM_ID_ISSUER
    WHERE add_.CTP_NM_ID_ISSUER = :orphan_ctp_id;
   ```
   - Cross-entity references may indicate a data mapping error.

   d. **Classify and act:**
   - If counterpart was never migrated: **Severity HIGH**. Re-migrate the missing counterpart or flag the accounting document for manual review.
   - If counterpart was soft-deleted: **Severity MEDIUM**. Document -- the child references a logically-deleted parent. May be acceptable depending on business rules.
   - If cross-entity mismatch: **Severity HIGH**. Escalate as a data mapping bug.

### Workflow 2: Orphan CLEARING.DOI_NM_ID (FK disabled during migration)

**Source file:** `reconciliation/integrity/03_collection_orphans.sql` (relationship #1)

**Risk level:** CRITICAL -- FK constraint `FK_DOI_CLE` exists but is disabled during the migration Transfer step. If not re-enabled, orphans can be created silently.

**Context:** `NFS_COLLECTION.CLEARING.DOI_NM_ID` references `NFS_PRODUCTS.DOCUMENT_INSTALMENT.DOI_NM_ID`. The FK (`FK_DOI_CLE`) is temporarily disabled by `MANAGE_INDEX` during Transfer.

**Investigation steps:**

1. **Run the summary count** from `integrity/03_collection_orphans.sql` (Section 1).
   - If `orphan_count = 0`: no action needed. Move to next relationship.
   - If `orphan_count > 0`: continue to step 2.

2. **Check FK constraint status first:**

   a. Run the pre-flight query from `integrity/02_products_orphans.sql` (Section 0).

   b. **If FK_DOI_CLE is DISABLED:**
   - The migration Transfer step did not complete properly.
   - To find all violating rows without blocking other operations:
   ```sql
   ALTER TABLE NFS_COLLECTION.CLEARING
     ENABLE NOVALIDATE CONSTRAINT FK_DOI_CLE;
   -- NOVALIDATE finds violations without blocking existing data
   ```
   - **Severity:** MEDIUM (known issue -- FK was left disabled)
   - Action: Document. Re-run `MANAGE_INDEX.ENABLE_FK_DOCUMENT` to properly re-enable.

   c. **If FK_DOI_CLE is ENABLED:**
   - Orphans should NOT exist (the constraint prevents them).
   - Verify the orphan detection query is correct (check JOIN condition, DELETE_DATE filter).
   - If query is correct and orphans exist with FK enabled: **Severity CRITICAL**. The constraint may have been enabled with NOVALIDATE, leaving pre-existing violations.

3. **For each orphan DOI_NM_ID:**

   a. **Check if DOCUMENT_INSTALMENT exists but was soft-deleted:**
   ```sql
   SELECT DOI_NM_ID, DELETE_DATE
     FROM NFS_PRODUCTS.DOCUMENT_INSTALMENT
    WHERE DOI_NM_ID = :orphan_doi;
   ```

   b. **Check if the DOI was part of a failed Transfer:**
   ```sql
   SELECT *
     FROM NFS_MIGRATION.CLEARING
    WHERE CLE_NM_ID = :orphan_cle_nm_id;
   ```

4. **Severity classification:**
   - Orphans with FK **enabled**: CRITICAL
   - Orphans with FK **disabled** (known issue): MEDIUM
   - Orphans on soft-deleted parent: MEDIUM (document and assess)

5. **Resolution:** Re-run Transfer step or manually reconcile CLEARING records against DOCUMENT_INSTALMENT.

### Workflow 3: Orphan DOI_NM_ID in NFS_LEGACY bridge tables

**Source file:** `reconciliation/integrity/04_legacy_orphans.sql` (relationships #1-3)

**Risk level:** HIGH -- Bridge tables are the critical link between SI legacy keys and NFS IDs. If the bridge is broken, the entire reconciliation chain for the affected table is invalid.

**Context:** Three bridge tables in NFS_LEGACY reference `NFS_PRODUCTS.DOCUMENT_INSTALMENT.DOI_NM_ID`:
- `DOI_RATA_CED_DEB` (highest volume -- supports 41.2M RATA_CED_DEB reconciliation)
- `DOI_CONTABILE_FF` (supports 363K CONTABILE_FF reconciliation)
- `DOI_FATTURA_ATTIVA` (supports 321K FATTURA_ATTIVA reconciliation)

**Investigation steps:**

1. **Run the summary counts** from `integrity/04_legacy_orphans.sql` (Sections 1, 2, 3).
   - If all `orphan_count = 0`: no action needed.
   - If any `orphan_count > 0`: continue to step 2 for the affected table(s).

2. **For each affected bridge table, examine orphan records:**

   a. **For DOI_RATA_CED_DEB:**
   ```sql
   SELECT drc.DOI_NM_ID, drc.LEN_ID, drc.RCD_NM_ID
     FROM NFS_LEGACY.DOI_RATA_CED_DEB drc
    WHERE drc.DOI_NM_ID = :orphan_doi
      AND drc.LEN_ID = :len_id;
   ```

   b. **For DOI_CONTABILE_FF:**
   ```sql
   SELECT dcf.DOI_NM_ID, dcf.LEN_ID, dcf.CFF_NM_ID
     FROM NFS_LEGACY.DOI_CONTABILE_FF dcf
    WHERE dcf.DOI_NM_ID = :orphan_doi
      AND dcf.LEN_ID = :len_id;
   ```

   c. **For DOI_FATTURA_ATTIVA:**
   ```sql
   SELECT dfa.DOI_NM_ID, dfa.LEN_ID, dfa.FAT_NM_ID
     FROM NFS_LEGACY.DOI_FATTURA_ATTIVA dfa
    WHERE dfa.DOI_NM_ID = :orphan_doi
      AND dfa.LEN_ID = :len_id;
   ```

3. **Cross-reference with Tier 1 row counts:**
   - If `DOI_RATA_CED_DEB` bridge count != `DOCUMENT_INSTALMENT` count, this confirms the orphan is due to a missing DOCUMENT_INSTALMENT record (not a bridge-side error).
   - Check if the DOI was created during `loading_final` but deleted before Transfer completed.

4. **Check the parent DOCUMENT_INSTALMENT:**
   ```sql
   SELECT DOI_NM_ID, DELETE_DATE
     FROM NFS_PRODUCTS.DOCUMENT_INSTALMENT
    WHERE DOI_NM_ID = :orphan_doi;
   ```
   - If not found: DOCUMENT_INSTALMENT was never created. Check `loading_final` logic.
   - If found with DELETE_DATE: soft-deleted parent. Document.

5. **Severity:** HIGH -- breaks the entire reconciliation chain for the affected table.

6. **Resolution:** Re-run Transfer step for the affected scope, or investigate `loading_final` logic for the missing DOCUMENT_INSTALMENT records.

---

## 5. Severity Classification Matrix

Use this matrix to classify any finding from the reconciliation suite. When multiple criteria apply, use the **highest** severity level.

| Severity | Criteria | Examples | Action |
|----------|----------|----------|--------|
| **CRITICAL** | Financial amount discrepancy >0.01% on high-volume tables; data loss (records in staging not in final); FK-enforced orphans found | Tier 2 SUM mismatch on RATA_CED_DEB; Tier 3 shows lost records; CLEARING orphans with FK_DOI_CLE enabled | Stop migration validation. File bug immediately. Investigate root cause before proceeding. |
| **HIGH** | Orphan counterpart references (no FK constraint); bridge table integrity broken; FK constraints left disabled post-migration | CTP_NM_ID orphans in ACCOUNTING_DOCUMENT_DATA; DOI_NM_ID orphans in NFS_LEGACY bridge tables; disabled FK_DOI_CLE | Investigate within 24 hours. May require re-migration of affected records. Do not proceed to production until resolved. |
| **MEDIUM** | Row count mismatches on medium-volume tables; orphans on enforced-FK relationships with FK currently enabled (indicates temporary disable issue); V_QUAD count discrepancies | Tier 1 delta on CONTABILE_FF; CLEARING orphans with FK currently enabled; V_QUAD vs direct staging count mismatch | Investigate within 1 week. Document findings. May be acceptable if explained by known patterns. |
| **LOW** | Year-range filtering deltas; 1:N registry expansion; static reference data mismatches (LEN_ID, CUR_V2_ID) | Staging < SI source due to year filter; ANAGRAFICA NFS > SI (1:N expansion); CURRENCY orphans in NFS_LEGACY | Document and accept. These are expected behaviors, not bugs. |

### Volume-Based Severity Adjustment

For tables with reconciliation volume priority (from `00_table_mapping.sql`):

| Volume Tier | Tables | Severity Multiplier |
|-------------|--------|---------------------|
| Tier A (>1M rows) | RATA_CED_DEB (41.2M), PAREGGIO_FAT_ATT (23.8M) | Any discrepancy is at least HIGH |
| Tier B (100K-1M) | PAREGGIO_CED_DEB (722K), CONTABILE_FF (363K), MANDATO (326K), FATTURA_ATTIVA (321K), PAREGGIO_NON_FAT (121K) | Apply standard matrix |
| Tier C (<100K) | MOVIMENTO_VALORIZZ, PAREGGIO_CBL, ACCOUNTING_LOG, PAGAMENTO_FCD_FAA, remaining tables | Downgrade by one level unless CRITICAL |

---

## 6. Escalation Decision Tree

### When to STOP investigating (accept the finding)

Stop investigating and document as "accepted" when:

- The delta is **fully explained** by a known pattern from Section 3 (Patterns A-H).
- The orphan count is **0** for the relationship.
- The mismatch is on a **LOW-severity reference data column** (LEN_ID, CUR_V2_ID in NFS_LEGACY).
- The discrepancy is within expected tolerances:
  - Registry 1:N expansion ratio between 1:1 and 1:10
  - Year-range filter excludes <10% of SI source records
  - V_QUAD-derived counts match within 0.1% of direct staging queries

### When to FILE A BUG

File a bug immediately when:

- **Tier 2 monetary SUM mismatch** that is NOT explained by sign handling (Pattern C).
- **Tier 3 shows records in staging-not-in-final** -- data loss (Pattern D). Severity: CRITICAL.
- **CTP_NM_ID orphans > 0** -- financial documents reference invalid counterparties (Workflow 1). Severity: HIGH.
- **DOI_NM_ID orphans > 0** on bridge tables -- reconciliation chain broken (Workflow 3). Severity: HIGH.
- **FK constraints found DISABLED** post-migration -- `MANAGE_INDEX` issue (Pre-Flight check). Severity: HIGH.
- **Any CRITICAL severity finding** from the matrix (Section 5).
- **Duplicates found** via COUNT DISTINCT mismatch (Pattern G). Severity: HIGH.

### When to REQUEST TEAM CONFIRMATION

Request team input (do not file a bug yet, do not accept) when:

- **Registry 1:N expansion ratio seems abnormally high** (>1:10) -- may indicate a data duplication issue or incorrect ratio expectation.
- **Year-range filter excludes more records than expected** (>10% of SI source) -- may indicate incorrect `REQ_NM_YEAR_FROM/TO` configuration.
- **V_QUAD-derived counts show different deltas** than direct staging queries -- may indicate V_QUAD view definition mismatch.
- **Spain DB link resolution** -- team must provide the dblink name (`{dblink_ES}` placeholder).
- **Multiple schemas show correlated orphans** for the same parent entity -- may indicate systematic migration failure requiring architectural review.
- **Pattern not covered** by Sections 3-4 -- unknown discrepancy type that doesn't match any documented pattern.

### Decision Flowchart

```
Finding from query output
        |
        v
Is orphan count = 0?
  YES --> Document as clean. STOP.
  NO  --> Continue.
        |
        v
Does it match a known pattern (Section 3, A-H)?
  YES --> Is severity LOW?
            YES --> Accept and document. STOP.
            NO  --> Apply severity action from matrix. Continue below.
  NO  --> Request team confirmation. STOP.
        |
        v
Is severity CRITICAL?
  YES --> File bug immediately. Stop validation.
  NO  --> Continue.
        |
        v
Is severity HIGH?
  YES --> File bug. Investigate within 24h. Continue validation on other tables.
  NO  --> Continue.
        |
        v
Is severity MEDIUM?
  YES --> Document. Investigate within 1 week. Continue validation.
  NO  --> Document and accept (LOW). Continue validation.
```

---

## Appendix: Quick Reference -- File to Section Mapping

| Integrity File | Relationships | Key Workflows |
|----------------|---------------|---------------|
| `integrity/01_registry_orphans.sql` | 12 NFS_REGISTRY relationships (COUNTERPART hub, 8 satellites, taxonomy, reverse check) | Pattern H (correlated orphans) |
| `integrity/02_products_orphans.sql` | Section 0: Pre-flight FK check; Sections 1-9: NFS_PRODUCTS relationships | Workflow 1 (CTP_NM_ID orphans), Pre-flight check |
| `integrity/03_collection_orphans.sql` | 9 NFS_COLLECTION relationships (CLEARING hub, LCR downstream) | Workflow 2 (CLEARING.DOI_NM_ID orphans) |
| `integrity/04_legacy_orphans.sql` | 6 NFS_LEGACY bridge relationships + consolidated LEN_ID check | Workflow 3 (DOI_NM_ID bridge orphans) |

| Reconciliation Tier | Files | Key Patterns |
|---------------------|-------|--------------|
| `tier1-row-counts/01-09` | 19 tables covered | Patterns A, B, F |
| `tier2-aggregates/01-06` | 8 monetary tables | Pattern C, G |
| `tier3-minus/01-05` | 5 V_QUAD tables | Patterns D, E |

---

*This guide is part of the SI-to-NFS migration reconciliation suite.*
*Companion document: `EXECUTION_GUIDE.md` (query execution instructions)*
*Reference: `00_table_mapping.sql` (master table mapping)*