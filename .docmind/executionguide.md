---
unique-name: executionguide
display-name: EXECUTION GUIDE
category: GENERAL
description: Complete reference for running the SI-to-NFS reconciliation query suite -- from parameter resolution through cross-reference analysis.
---

# Reconciliation Execution Guide

Complete reference for running the SI-to-NFS reconciliation query suite -- from parameter resolution through cross-reference analysis.

**Audience:** Anyone who needs to validate data integrity for the collections (incassi) migration.
**Last updated:** 2026-02-12

---

## 1. Overview and Purpose

### What This Suite Validates

The reconciliation query suite compares data at every stage of the SI-to-NFS collections migration pipeline:

```
SI Source (FACT_BFF/BGR/BPT/BSK)
  --> Staging (NFS_MIGRATION)
    --> Intermediate (NFS_MIGRATION.DOI_*)
      --> Bridge (NFS_LEGACY.DOI_*)
        --> Final (NFS_PRODUCTS / NFS_COLLECTION / NFS_REGISTRY / NFS_LEGACY)
```

It answers one question: **does the NFS final data match the SI source data?**

### Scope

- **19 migration tables** across 4 NFS target schemas (NFS_REGISTRY, NFS_PRODUCTS, NFS_COLLECTION, NFS_LEGACY)
- **5 countries:** Italy (IT), Greece (GR), Portugal (PT), Slovakia (SK), Spain (ES)
- **4 validation tiers:**
  - **Tier 1** -- Row counts: do the source and target have the same number of records?
  - **Tier 2** -- Financial aggregates: do SUM of monetary columns, MIN/MAX dates, COUNT DISTINCT keys match?
  - **Tier 3** -- Record-level MINUS: which exact records exist in one side but not the other?
  - **Integrity** -- Cross-schema orphan detection: are all foreign key relationships intact in NFS final?

### Relationship to V_QUAD Views

V_QUAD views (`V_QUAD_*_PERC`) cover **staging-to-final** validation -- they compare NFS_MIGRATION staging against NFS final tables using column-mapping logic. This suite extends that coverage to include **SI-to-final** comparison AND **cross-schema referential integrity** validation that V_QUAD does not address.

### What Is NOT Covered

- **Staging-to-final only:** V_QUAD views handle this natively in the database
- **Non-incassi perimeter:** Tables outside the collections migration scope
- **CZ and SK variants:** Tier 3 MINUS queries are based on V_QUAD_*_PERC views (IT/GR/PT). Slovakia uses V_QUAD_*_PERC_SK variants with different join logic; SK-specific Tier 3 queries would need adaptation

---

## 2. Prerequisites

### Schema Access

You need SELECT grants on the following schemas:

| Schema | Purpose |
|--------|---------|
| NFS_MIGRATION | Staging tables, configuration (MGR_CFG_DBLINK, MGR_REQUEST), V_QUAD views |
| NFS_REGISTRY | Final registry tables (COUNTERPART, LINK_CTP_RTP_RST, etc.) |
| NFS_PRODUCTS | Final product tables (DOCUMENT_INSTALMENT, CLEARING) |
| NFS_COLLECTION | Final collection tables (CLEARING, ACCOUNTING_LOG) |
| NFS_LEGACY | Bridge tables (DOI_*, BAT_*, BTD_*, LEA_*, STL_*, DEA_*) |

### DB Link Availability

The following DB links must be accessible from your execution schema:

| DB Link | Country | Legal Entity ID (LEN_ID) | Status |
|---------|---------|--------------------------|--------|
| FACT_BFF | Italy (IT) | 41 | Confirmed |
| FACT_BGR | Greece (GR) | 47 | Confirmed |
| FACT_BPT | Portugal (PT) | 45 | Confirmed |
| FACT_BSK | Slovakia (SK) | 52, 53 | Confirmed |
| *(resolve below)* | Spain (ES) | 44 | **Unconfirmed** -- resolve with query below |

To resolve Spain's DB link name:
```sql
SELECT MCD_V2_DBLINK, MCD_V2_PAESE, MCD_V2_LEGAL_ENTITY
  FROM NFS_MIGRATION.MGR_CFG_DBLINK
 WHERE MCD_V2_LEGAL_ENTITY = '44';
```

### Oracle Client

All queries are Oracle 19c compatible SQL. No PL/SQL blocks, no DBMS packages.

### Bind Variable Support

Queries use `:req_id` and `:len_id` bind variables plus `{dblink}` text substitution. Your SQL tool must support Oracle bind variables:
- **SQL*Plus:** Use `DEFINE` or `ACCEPT` for text substitution, `:variable` for bind
- **SQL Developer:** Prompted automatically on execution
- **DBeaver:** Supports bind variable prompts
- **DataGrip:** Supports bind variable prompts

---

## 3. Parameter Resolution

Before running any query, you must resolve three parameters: the DB link name, the legal entity ID, and the request ID. Follow these steps in order.

### Step 1: Resolve DB Link and Legal Entity Mapping

```sql
SELECT MCD_V2_DBLINK    AS dblink_name,
       MCD_V2_PAESE     AS country_name,
       MCD_V2_SIGLA_PAESE AS country_code,
       MCD_V2_LEGAL_ENTITY AS len_id
  FROM NFS_MIGRATION.MGR_CFG_DBLINK
 ORDER BY MCD_V2_LEGAL_ENTITY;
```

**Expected output:**

| dblink_name | country_name | country_code | len_id |
|-------------|--------------|--------------|--------|
| FACT_BFF | ITALIA | IT | 41 |
| *(resolve)* | SPAGNA | ES | 44 |
| FACT_BPT | PORTOGALLO | PT | 45 |
| FACT_BGR | GRECIA | GR | 47 |
| FACT_BSK | SLOVACCHIA | SK | 52 |
| FACT_BSK | SLOVACCHIA | SK | 53 |

Note: Slovakia has **two legal entities** (52 and 53) sharing one DB link. Run queries twice for SK -- once with `:len_id = 52` and once with `:len_id = 53`.

### Step 2: Resolve Request IDs Per Country and Scope

```sql
SELECT REQ_NM_ID        AS req_id,
       REQ_V2_SCOPE     AS scope,
       REQ_V2_DBLINK    AS dblink,
       REQ_NM_LEN_ID    AS len_id,
       REQ_NM_YEAR_FROM,
       REQ_NM_YEAR_TO
  FROM NFS_MIGRATION.MGR_REQUEST
 ORDER BY REQ_V2_SCOPE, REQ_V2_DBLINK;
```

**Important:** Multiple `req_id` values may exist per country/scope pair (from historical migration runs). Use the most recent request ID, or ask the migration team which specific request to reconcile.

### Step 3: Substitute Parameters in Queries

For each query file, substitute:

| Parameter | Replace With | Example |
|-----------|-------------|---------|
| `:req_id` | The actual `REQ_NM_ID` value from Step 2 | `123` |
| `:len_id` | The legal entity ID from Step 1 | `41` (for Italy) |
| `{dblink}` | The actual DB link name from Step 1 | `FACT_BFF` (for Italy) |

**For Slovakia:** Run queries twice -- once with `len_id = 52` and once with `len_id = 53`.

**For Spain:** First resolve the DB link name from Step 1, then substitute `{dblink}` with that value.

**Exception:** CLEARING queries in the integrity suite do **not** use `:len_id` because the CLEARING table lacks a LEN_ID column. Only `:req_id` and `{dblink}` are needed.

---

## 4. Complete File Inventory

### Root Files

| File | Description |
|------|-------------|
| `00_table_mapping.sql` | Master SI-to-NFS mapping reference for all 19 migration tables. Authoritative source for schema names, column names, filters, and volume-priority ordering. |
| `README.md` | Quick-start guide and directory overview. Points to this file for full execution instructions. |
| `EXECUTION_GUIDE.md` | This file. Comprehensive step-by-step execution reference. |
| `COVERAGE_CHECK.md` | Coverage validation matrix confirming all 19 tables are covered at the appropriate tiers. |

### tier1-row-counts/ -- Row Count Comparisons (9 files)

SI source vs staging vs NFS final record counts. Start here for every reconciliation run.

| File | Table(s) | Volume | Description |
|------|----------|--------|-------------|
| `01_rata_ced_deb.sql` | RATA_CED_DEB | 41.2M | Highest-volume table. Three-point count: SI source, staging, NFS final (via DOI_RATA_CED_DEB bridge). COD_CFC != '017' filter. |
| `02_pareggio_fat_att.sql` | PAREGGIO_FAT_ATT | 23.8M | No IS_VALID column. ROW_ID-based join. NFS final via V_QUAD approach (shared CLEARING target). |
| `03_pareggio_ced_deb.sql` | PAREGGIO_CED_DEB | 722K | NFS final via V_QUAD approach (shared CLEARING target). |
| `04_contabile_ff.sql` | CONTABILE_FF | 363K | AA_CBL NOT IN ('TEMP','AZL') filter. Bridge: DOI_CONTABILE_FF. |
| `05_mandato.sql` | MANDATO | 326K | Table name singular (MANDATO), scope plural (MANDATI). Complex join includes PAREGGIO_CED_DEB + DOI_RATA_CED_DEB subquery. |
| `06_fattura_attiva.sql` | FATTURA_ATTIVA | 321K | Bridge: DOI_FATTURA_ATTIVA. |
| `07_payment_notice.sql` | PAYMENT_NOTICE_DETAIL | derived | No SI source. Derived from PAREGGIO_CED_DEB + RATA_CED_DEB during mandato migration. Staging-vs-final + MANDATO cross-reference. |
| `08_pareggio_non_fat.sql` | PAREGGIO_NON_FAT | 121K | No IS_VALID column. ROW_ID-based join. NFS final via V_QUAD approach (shared CLEARING target). |
| `09_remaining_tables.sql` | 11 tables | lower | Consolidated file covering: PAGAMENTO_FCD_FAA, RISTR_PAGAMENTO, BONIFICI_ENTRATA, MOVIMENTO_VALORIZZ, ACCOUNTING_LOG, AZIONE_LEGALE, CARICO, OPERAZIONE_MAPROS, ANAGRAFICA_CED, ANAGRAFICA_DEB, PAREGGIO_CBL. |

### tier2-aggregates/ -- Financial Aggregate Comparisons (6 files)

SUM of monetary columns, MIN/MAX dates, COUNT DISTINCT keys. Run for tables showing Tier 1 discrepancies, or for full validation.

| File | Table(s) | Description |
|------|----------|-------------|
| `01_rata_ced_deb_agg.sql` | RATA_CED_DEB | 8 monetary SUM expressions (IMP_RCD, COMMISSIONE, INTERESSI, etc.) |
| `02_pareggio_fat_att_agg.sql` | PAREGGIO_FAT_ATT | 2 monetary SUM expressions |
| `03_contabile_ff_agg.sql` | CONTABILE_FF | 10 monetary SUM expressions (COMMISSIONE, INTERESSI, SPESE, IMP_NETTO, IMP_COMP_INC) -- direct-sum columns, no DECODE needed |
| `04_mandato_agg.sql` | MANDATO | 6 monetary SUM expressions |
| `05_fattura_attiva_agg.sql` | FATTURA_ATTIVA | 8 monetary SUM expressions. Nested DECODE for Italy-specific COD_CED=96 recipient resolution. |
| `06_pagamento_fcd_faa_agg.sql` | PAGAMENTO_FCD_FAA, PAREGGIO_CED_DEB, PAREGGIO_NON_FAT | 3 sections (A: PAG, B: PCD, C: PNF). 213 lines total. |

### tier3-minus/ -- Record-Level MINUS Queries (5 files)

Bidirectional MINUS: staging-not-in-final + final-not-in-staging. Run for tables showing Tier 1 or Tier 2 discrepancies.

| File | Table(s) | Description |
|------|----------|-------------|
| `01_rata_ced_deb_minus.sql` | RATA_CED_DEB | Bidirectional MINUS. V_QUAD column expressions copied verbatim. 448 lines. |
| `02_pareggio_fat_att_minus.sql` | PAREGGIO_FAT_ATT | Uses aa_pcd >= 2010 filter from V_QUAD (not IS_VALID). 290 lines. |
| `03_contabile_ff_minus.sql` | CONTABILE_FF | Bidirectional MINUS. 310 lines. |
| `04_mandato_minus.sql` | MANDATO | Complex join: PAREGGIO_CED_DEB + DOI_RATA_CED_DEB subquery for PND_NM_AMOUNT. 489 lines. |
| `05_fattura_attiva_minus.sql` | FATTURA_ATTIVA | Nested DECODE for Italy COD_CED=96. 407 lines. |

### integrity/ -- Cross-Schema Orphan Detection (4 files)

LEFT JOIN + IS NULL orphan detection for FK relationships across NFS target schemas. Run independently of Tier 1-3 results.

| File | Schema | Relationships | Description |
|------|--------|---------------|-------------|
| `01_registry_orphans.sql` | NFS_REGISTRY | 12 | COUNTERPART hub + 8 satellite tables + COUNTERPART_TAXONOMY_DATA + reverse orphan check. All logical FKs (no database constraints). 433 lines. |
| `02_products_orphans.sql` | NFS_PRODUCTS | 9 | **Pre-flight FK constraint status check (Section 0)** + CTP_NM_ID_ISSUER/RECIPIENT (NO FK constraint, HIGH RISK) + 3 cross-schema + 4 internal. 394 lines. |
| `03_collection_orphans.sql` | NFS_COLLECTION | 9 | CLEARING.DOI_NM_ID (HIGH RISK -- FK disabled during migration Transfer) + 5 LCR_ID downstream from LINK_CTP_RTP_RST hub. 349 lines. |
| `04_legacy_orphans.sql` | NFS_LEGACY | 6 | 3 critical DOI_* bridge tables (41.2M + 363K + 321K chains) + 3 medium-priority bridge tables + consolidated 8-table LEN_ID check. 328 lines. |

---

## 5. Recommended Execution Order

Structure your reconciliation run as six phases. Complete each phase before moving to the next.

### Phase A: Pre-Flight Checks (5 minutes)

**Purpose:** Confirm that all prerequisites are in place before running any reconciliation queries.

1. **Resolve parameters** -- Run the three queries from Section 3 above. Confirm:
   - All DB links return data (no ORA-02019 errors)
   - You have valid `req_id` values for the target country
   - `len_id` values match expected legal entities

2. **Check V_QUAD status** -- Quick overview of current reconciliation state:
   ```sql
   SELECT *
     FROM NFS_MIGRATION.V_QUAD_ALL_PERC
    WHERE COUNTRY = '{country}'
    ORDER BY TABELLA;
   ```

3. **Run FK constraint status check** -- From `integrity/02_products_orphans.sql`, Section 0:
   ```sql
   SELECT owner, constraint_name, table_name, status, validated
     FROM all_constraints
    WHERE constraint_type = 'R'
      AND owner IN ('NFS_PRODUCTS', 'NFS_COLLECTION', 'NFS_LEGACY', 'NFS_REGISTRY')
      AND status = 'DISABLED'
    ORDER BY owner, table_name;
   ```

4. **If any FKs are disabled:** STOP and report to the migration team before proceeding. Disabled FKs during reconciliation mean orphan detection results will be unreliable -- new orphans could be created while you are checking.

### Phase B: Tier 1 Row Counts (~10 minutes per country)

**Purpose:** Quick pass/fail for every migration table.

**Files:** `tier1-row-counts/01_rata_ced_deb.sql` through `tier1-row-counts/09_remaining_tables.sql`

**Procedure:**
1. Run files in numerical order: 01, 02, 03, ..., 09
2. Start with GR or PT (smallest volumes, fastest execution)
3. For each table, record:
   - SI_SOURCE_COUNT
   - STAGING_COUNT
   - NFS_FINAL_COUNT
4. **Expected:** All three counts match (within year-range filtering tolerance)
5. **If any count mismatch:** Note the table and continue -- investigate in Phase C

**Known behaviors that are NOT discrepancies:**
- ANAGRAFICA_CED/DEB: NFS_FINAL > SI_SOURCE is expected (1:N COUNTERPART expansion)
- PAYMENT_NOTICE_DETAIL: No SI source count available (derived table)
- Year-range filtering: Small deltas may be explained by `REQ_NM_YEAR_FROM`/`REQ_NM_YEAR_TO` in MGR_REQUEST

### Phase C: Tier 2 Aggregates (~10 minutes per country)

**Purpose:** Quantify the financial magnitude of any row count discrepancies.

**Files:** `tier2-aggregates/01_rata_ced_deb_agg.sql` through `tier2-aggregates/06_pagamento_fcd_faa_agg.sql`

**When to run:**
- Run ONLY for tables that showed Tier 1 discrepancies, OR
- Run for all tables if doing full validation

**Procedure:**
1. Run files in numerical order
2. Compare SUM of monetary columns between SI source, staging, and NFS final
3. **Expected:** All sums match exactly -- accounting data must be precise
4. **If any sum mismatch:** This is a **financial data integrity issue** -- flag as **HIGH severity**

### Phase D: Tier 3 Record-Level MINUS (~15-30 minutes per country for IT, <5 minutes for GR/PT)

**Purpose:** Identify exact records that differ between staging and NFS final.

**Files:** `tier3-minus/01_rata_ced_deb_minus.sql` through `tier3-minus/05_fattura_attiva_minus.sql`

**When to run:**
- Run ONLY for tables that showed Tier 1 or Tier 2 discrepancies
- Or when full record-level validation is required

**Procedure:**
1. Run files in numerical order
2. Each file produces two result sets:
   - **Staging-not-in-final:** Records in NFS_MIGRATION staging but missing from NFS final
   - **Final-not-in-staging:** Records in NFS final but not in NFS_MIGRATION staging
3. **Expected:** 0 rows in both directions
4. **Any rows returned:** These are the exact records that differ -- use for root cause analysis

**Note:** PAREGGIO_CED_DEB and PAREGGIO_NON_FAT do not have individual Tier 3 files. Their V_QUAD views (`V_QUAD_PAR_CED_DEB_PERC`, `V_QUAD_PAR_NON_FAT_PERC`) exist in the database and can be queried directly if needed.

### Phase E: Cross-Schema Integrity (~10-20 minutes per country)

**Purpose:** Detect orphan records -- child rows referencing non-existent parent rows across NFS schemas.

**Files:** `integrity/01_registry_orphans.sql` through `integrity/04_legacy_orphans.sql`

**When to run:**
- Run independently of Tier 1-3 results (these check FK integrity, not row counts)
- Always run as part of a complete reconciliation

**Procedure:**
1. Run in order: `01_registry_orphans.sql` first (foundation schema), then `02_products`, `03_collection`, `04_legacy`
2. Each file contains summary-count queries (quick) and detail queries (run only if summary shows orphans)
3. **Expected:** 0 orphans for every relationship
4. **If orphans found:** See `TRIAGE_GUIDE.md` for investigation workflow

**Key risks to watch:**
- `02_products_orphans.sql` relationships 1-2: CTP_NM_ID_ISSUER and CTP_NM_ID_RECIPIENT have **NO FK constraints** -- orphans here are invisible to the database
- `03_collection_orphans.sql` relationship 1: CLEARING.DOI_NM_ID FK is **disabled during migration Transfer** -- orphans can accumulate during the migration window

### Phase F: Cross-Reference (manual analysis)

**Purpose:** Correlate findings across all validation tiers.

**Procedure:**
1. Compare Tier 1-3 discrepancies with Phase E orphan results
2. Orphan counts may explain row count mismatches (e.g., orphan CLEARING rows could inflate NFS final counts)
3. Financial aggregate mismatches should correlate with specific orphan records found in Phase E
4. Document findings in a country-specific results file

---

## 6. Country Escalation Strategy

Run countries in this order, from smallest to largest, to validate query correctness before tackling high-volume data.

### Round 1: Greece (GR) or Portugal (PT)

**Purpose:** Validate that all queries execute correctly, parameters are right, results are interpretable.

| Attribute | Value |
|-----------|-------|
| Volume | 1K-50K rows per table |
| LEN_ID | GR: 47, PT: 45 |
| DB Link | GR: FACT_BGR, PT: FACT_BPT |
| Expected time per tier | <5 minutes |
| Total expected time | ~30 minutes |

Start here. If queries fail or produce unexpected results, debug against GR/PT data before scaling up.

### Round 2: Slovakia (SK)

**Purpose:** Test multi-legal-entity handling.

| Attribute | Value |
|-----------|-------|
| Volume | Medium |
| LEN_ID | 52 AND 53 (run each separately) |
| DB Link | FACT_BSK |
| Expected time per tier | ~10 minutes |
| Total expected time | ~60 minutes (two passes) |

Run each legal entity separately. Compare results between LEN_ID 52 and 53 for consistency.

### Round 3: Italy (IT)

**Purpose:** Full-scale production reconciliation.

| Attribute | Value |
|-----------|-------|
| Volume | RATA_CED_DEB: 41.2M, PAREGGIO_FAT_ATT: 23.8M |
| LEN_ID | 41 |
| DB Link | FACT_BFF |
| Expected time per tier | Tier 1: 10 min, Tier 2: 10 min, Tier 3: 15-30 min |
| Total expected time | ~90 minutes |

Consider running during off-peak hours. See Section 7 for performance tuning.

### Round 4: Spain (ES) -- Last

**Purpose:** Validate after DB link is resolved.

| Attribute | Value |
|-----------|-------|
| Volume | Unknown |
| LEN_ID | 44 |
| DB Link | **Resolve first** (see Prerequisites, Section 2) |
| Expected time per tier | Unknown -- estimate from volume after first Tier 1 run |
| Total expected time | Unknown |

**Before running:** Resolve the Spain DB link name by querying `MGR_CFG_DBLINK`. Then substitute `{dblink}` with the resolved name in all query files.

---

## 7. Performance Notes for Large Tables

### IT-Scale Considerations

| Table | Volume | Concern | Mitigation |
|-------|--------|---------|------------|
| RATA_CED_DEB | 41.2M | Tier 3 MINUS may take 15-30 minutes | Ensure sufficient TEMP tablespace |
| PAREGGIO_FAT_ATT | 23.8M | Same as above | Same mitigation |
| CLEARING | 23M+ | Integrity LEFT JOIN can be expensive | Consider PARALLEL hint (see below) |
| DOCUMENT_INSTALMENT | 40M+ | Integrity LEFT JOIN can be expensive | Consider PARALLEL hint (see below) |

### PARALLEL Hint

If any query runs longer than 30 minutes:

1. **First check indexes:** Verify that join columns have indexes:
   ```sql
   SELECT index_name, column_name
     FROM all_ind_columns
    WHERE table_owner = '{schema}'
      AND table_name = '{table}'
    ORDER BY index_name, column_position;
   ```

2. **Then add PARALLEL hint** if indexes exist but query is still slow:
   ```sql
   SELECT /*+ PARALLEL(4) */ ...
   ```
   Place the hint immediately after `SELECT` in the query.

3. **TEMP tablespace:** Large MINUS operations require sorting. If you get ORA-01652 (unable to extend temp segment), request TEMP tablespace increase from the DBA team.

### General Tips

- Run Tier 1 first -- it completes in seconds even for IT-scale tables and tells you which tables need deeper investigation
- Avoid running Tier 3 MINUS for all tables on IT unless specifically needed -- focus on tables that showed Tier 1/2 discrepancies
- Integrity queries can run in parallel with Tier 1-3 (they check different things)

---

## 8. Quick Reference Card

One-page summary for experienced users.

### Parameter Quick-Ref

```
:req_id  = MGR_REQUEST.REQ_NM_ID (from Step 2)
:len_id  = Legal entity ID: IT=41, ES=44, PT=45, GR=47, SK=52/53
{dblink} = DB link name: IT=FACT_BFF, GR=FACT_BGR, PT=FACT_BPT, SK=FACT_BSK
```

### Execution Summary

| Step | What | Files | Time (IT) | Time (GR/PT) |
|------|------|-------|-----------|--------------|
| A | Pre-flight checks | (inline SQL) | 5 min | 5 min |
| B | Tier 1 row counts | tier1-row-counts/01-09 | 10 min | 5 min |
| C | Tier 2 aggregates | tier2-aggregates/01-06 | 10 min | 5 min |
| D | Tier 3 MINUS | tier3-minus/01-05 | 30 min | 5 min |
| E | Integrity checks | integrity/01-04 | 20 min | 5 min |
| F | Cross-reference | manual analysis | 15 min | 5 min |
| **Total** | | | **~90 min** | **~30 min** |

### Country Order

```
GR/PT (smallest) --> SK (medium, 2 LEN_IDs) --> IT (largest) --> ES (resolve dblink first)
```

### Decision Tree

```
Tier 1 row counts
  |
  +--> Counts match? --> PASS (move to next table)
  |
  +--> Counts differ?
        |
        +--> Run Tier 2 aggregates
              |
              +--> Sums match? --> Row count delta explained by year-range filtering or 1:N expansion
              |
              +--> Sums differ? --> HIGH SEVERITY
                    |
                    +--> Run Tier 3 MINUS to identify exact records
                          |
                          +--> Cross-reference with integrity orphan results (Phase E)

Integrity checks (run independently)
  |
  +--> 0 orphans? --> PASS
  |
  +--> Orphans found? --> See TRIAGE_GUIDE.md
```

### File Count Summary

| Directory | Files | Lines |
|-----------|-------|-------|
| reconciliation/ (root) | 4 | ~1,125 |
| tier1-row-counts/ | 9 | ~1,332 |
| tier2-aggregates/ | 6 | ~646 |
| tier3-minus/ | 5 | ~1,944 |
| integrity/ | 4 | ~1,504 |
| **Total** | **28** | **~6,551** |