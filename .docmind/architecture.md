---
unique-name: architecture
display-name: ARCHITECTURE
category: GENERAL
description: **Domain:** SI-to-NFS Oracle Data Migration Analysis
---

# Architecture Patterns

**Domain:** SI-to-NFS Oracle Data Migration Analysis
**Researched:** 2026-02-09
**Confidence:** HIGH (based on direct analysis of 99,465-line `nfs_migration.sql`)

---

## System Overview

The migration system involves **four Oracle schemas** and a fifth external schema accessed via database link:

```
SI Database (source)          NFS Database (target)
+--------------------+        +---------------------------------------------+
|                    |        |                                             |
| SI Country Schema  | dblink | NFS_MIGRATION Schema (~99K lines)          |
| (IT/ES/PT/GR/SK)  |------->| - Staging tables (per domain)              |
|                    |        | - Migration packages (25+)                 |
|                    |        | - MGR_* management tables                  |
|                    |        | - V_QUAD_* reconciliation views            |
|                    |        | - QUAD_* analysis tables                   |
+--------------------+        |                                             |
                              +-----|-------|-------|------------------------+
                                    |       |       |
                              +-----v-+ +---v---+ +-v---------+  +----------+
                              | NFS_  | | NFS_  | | NFS_      |  | NFS_     |
                              |REGIST-| |PRODUC-| | COLLEC-   |  | LEGACY   |
                              |RY     | |TS     | | TION      |  |          |
                              +-------+ +-------+ +-----------+  +----------+
                              Counter-   Documents  Clearings     Mapping/
                              parts,     Invoices   Bank xfers    bridge
                              Taxonomy   Loans      Mandates      tables
```

## Migration Package Architecture

### Package Inventory (25 packages, 14 domains)

| Package | Lines (approx.) | Target Schema(s) | Domain |
|---------|-----------------|-------------------|--------|
| PKG_MIGRATION_REGISTRY | ~14,000 | NFS_REGISTRY | Counterparts, contacts, links |
| PKG_MIGRATION_REGISTRY_POL | ~7,500 | NFS_REGISTRY | Poland-specific registry |
| PKG_MIGRATION_CLEARING | ~1,800 | NFS_COLLECTION, NFS_LEGACY | Clearing/pareggio entries |
| PKG_MIGRATION_CLEARING_SK | ~1,800 | NFS_COLLECTION, NFS_LEGACY | SK clearing |
| PKG_MIGRATION_BONIFICI | ~3,300 | NFS_COLLECTION | Bank transfers |
| PKG_MIGRATION_BONIFICI_SK | ~3,300 | NFS_COLLECTION | SK bank transfers |
| PKG_MIGRATION_MANDATI | ~2,400 | NFS_COLLECTION | Mandates |
| PKG_MIGRATION_MOVIMENTI | ~1,500 | NFS_COLLECTION | Movements |
| PKG_MIGRATION_MOVIMENTI_SK | ~1,500 | NFS_COLLECTION | SK movements |
| PKG_MIGRATION_CONTABILITA | ~1,500 | NFS_PRODUCTS | Accounting documents |
| PKG_MIGRATION_CONTABILITA_SK | ~1,400 | NFS_PRODUCTS | SK accounting |
| PKG_MIGRATION_FATTURE | ~2,800 | NFS_PRODUCTS, NFS_LEGACY | Invoices (fatture) |
| PKG_MIGRATION_FATTURE_SK | ~2,800 | NFS_PRODUCTS, NFS_LEGACY | SK invoices |
| PKG_MIGRATION_FATT_ATTIVE | ~1,500 | NFS_PRODUCTS | Active invoices |
| PKG_MIGRATION_FATT_ATTIVE_SK | ~1,500 | NFS_PRODUCTS | SK active invoices |
| PKG_MIGRATION_DEAL_STL_LAL | ~1,800 | NFS_PRODUCTS, NFS_LEGACY | Deals, stock loads, legal actions |
| PKG_MIGRATION_DEAL_STL_LAL_SK | ~1,800 | NFS_PRODUCTS, NFS_LEGACY | SK deals |
| PKG_MIGRATION_TAXONOMY | ~2,500 | NFS_REGISTRY | Category/subcategory hierarchy |
| PKG_MIGRATION_LOAN | ~1,500 | NFS_PRODUCTS | Loan instalments/tranches |
| PKG_MIGRATION_CIGCUP | ~1,200 | NFS_PRODUCTS | CIG/CUP codes |
| PKG_COEXISTENCE_OLD2NEW | ~6,400 | NFS_COLLECTION, NFS_PRODUCTS, NFS_LEGACY | Coexistence bridge |
| PKG_COEXISTENCE_OLD2NEW_SK | ~6,400 | NFS_COLLECTION, NFS_PRODUCTS, NFS_LEGACY | SK coexistence |
| PKG_DEAL_STL_LAL | ~1,800 | NFS_PRODUCTS | Deal helper (non-migration) |
| PKG_DEAL_STL_LAL_OLD | ~1,800 | NFS_PRODUCTS | Deal helper (old version) |
| PKG_UTILITY | ~1,700 | NFS_MIGRATION | Utility/helper functions |

### Standard Package Internal Structure

Every migration package follows the same step-driven pattern managed through `MGR_REQUEST_STEP`:

```
main_migration
    |
    |--- FOR EACH pending MGR_REQUEST (status = INS or DONE, scope = '<DOMAIN>')
    |       |
    |       |--- WHILE check_start returns next step:
    |       |       |
    |       |       |--- CASE next_step
    |       |       |       |
    |       |       |       |--- 'Clear Table Elab'  --> clear_tabelab()
    |       |       |       |       DELETE from staging tables WHERE req_nm_id = N
    |       |       |       |
    |       |       |       |--- 'Loading Staging'   --> loading_staging()
    |       |       |       |       For each MGR_CONFIGURATION entity:
    |       |       |       |         - Build dynamic SQL: INSERT INTO staging_table
    |       |       |       |           SELECT columns FROM source_table@dblink
    |       |       |       |         - Optionally filter by year range
    |       |       |       |         - Chunk by 1M rows via SEQ_NO ranges
    |       |       |       |       Then: mgr_data_adjustment()
    |       |       |       |         - Run configured adjustments
    |       |       |       |
    |       |       |       |--- 'Validate Rules'    --> validate_rule()
    |       |       |       |       Apply MGR_VALIDATE_RULE checks
    |       |       |       |
    |       |       |       |--- 'Report Validate'   --> report_validate()
    |       |       |       |       Write validation results to VLD_REPORT_T
    |       |       |       |
    |       |       |       |--- 'Loading Final'     --> loading_final()
    |       |       |       |       Domain-specific INSERT INTO NFS_MIGRATION.*
    |       |       |       |       tables from staging with transformations
    |       |       |       |       Uses DBMS_PARALLEL_EXECUTE (4-10 threads)
    |       |       |       |       Chunks by year or key ranges
    |       |       |       |
    |       |       |       |--- 'Transfer'          --> mgr_finals_*_with_curs()
    |       |       |       |       Row-by-row cursor loop:
    |       |       |       |         - Read from NFS_MIGRATION intermediate tables
    |       |       |       |         - Generate new IDs (sequence or SELECT MAX)
    |       |       |       |         - INSERT INTO NFS_COLLECTION/PRODUCTS/REGISTRY
    |       |       |       |         - INSERT INTO NFS_LEGACY bridge tables
    |       |       |       |       Also uses DBMS_PARALLEL_EXECUTE for clearing
    |       |       |       |
    |       |       |       |--- 'End'               --> set_status_req(END)
```

### Data Flow: Five-Hop Architecture

```
HOP 1: SI --> NFS_MIGRATION staging    (loading_staging via @dblink)
HOP 2: NFS_MIGRATION staging cleanup   (mgr_data_adjustment)
HOP 3: Validation pass                 (validate_rule + report_validate)
HOP 4: NFS_MIGRATION intermediate      (loading_final: staging --> domain tables)
HOP 5: NFS_MIGRATION --> NFS_* final   (Transfer: row-by-row cursor to target schemas)
```

The five-hop design means each domain's data passes through five distinct stages, each of which can introduce or mask errors. Analysis must trace data through all five hops.

---

## Component Boundaries for Analysis

### Component 1: Infrastructure Analysis (Cross-Cutting)

**Scope:** The structural foundation shared by all 25 packages.

| Sub-Component | What It Analyzes | Key Artifacts |
|---------------|------------------|---------------|
| **Index coverage** | Missing indexes on staging tables (NFS_MIGRATION.*), intermediate tables, and join columns | CREATE INDEX DDL in migration SQL (520 indexes found, but many staging tables lack them) |
| **Statistics management** | Absence of DBMS_STATS calls (zero found) | All 25 package bodies |
| **Sequence vs MAX** | 42 `SELECT MAX` patterns replacing proper sequence usage | PKG_MIGRATION_REGISTRY (most instances), PKG_MIGRATION_TAXONOMY, PKG_COEXISTENCE_OLD2NEW |
| **Parallel execution** | DBMS_PARALLEL_EXECUTE configuration (degree, chunking) | PKG_MIGRATION_CLEARING, PKG_MIGRATION_BONIFICI, PKG_MIGRATION_FATTURE |
| **Index manipulation** | SET_INDEX_UNUSABLE_TABLE / SET_INDEX_REBUILD_TABLE procedures | Standalone procedures in NFS_MIGRATION |
| **Bulk processing** | Only 9 FORALL/BULK COLLECT instances across 99K lines | All package bodies |

**Boundary:** This component produces DDL scripts (indexes, sequences, statistics) that are applied BEFORE or AFTER migration runs. It does not modify package logic.

### Component 2: Package-by-Package Data Error Analysis

**Scope:** Each of the 25 migration packages analyzed individually for logic errors.

| Sub-Component | What It Analyzes | Key Concerns |
|---------------|------------------|--------------|
| **loading_staging correctness** | Dynamic SQL construction, column mapping, dblink queries | Column list built from MGR_CONFIGURATION + exclusions; wrong columns = silent data corruption |
| **mgr_data_adjustment correctness** | Data transformations applied post-staging | Hard-coded adjustments (e.g., Spain req_id=904 special case in clearing) |
| **validate_rule effectiveness** | Whether MGR_VALIDATE_RULE rules catch actual errors | Rules stored in MGR_VALIDATE_RULE table; need to check completeness |
| **loading_final correctness** | INSERT-SELECT logic from staging to intermediate tables | Complex JOINs against NFS_LEGACY tables; wrong joins = orphaned/duplicated records |
| **Transfer correctness** | Row-by-row cursor logic writing to final NFS schemas | ID generation (MAX vs sequence), field mapping, NFS_LEGACY bridge records |

**Analysis pattern per package:**

```
For each PKG_MIGRATION_*:
  1. Identify source tables (SI) and target tables (NFS_*)
  2. Trace column mappings through all 5 hops
  3. Flag transformation logic errors (wrong sequences, missing joins, data type mismatches)
  4. Flag missing validation rules
  5. Identify country-specific customizations (if/else on req_id)
  6. Produce corrected PL/SQL where errors found
```

**Boundary:** This component modifies package body logic. It does NOT add infrastructure (indexes, statistics).

### Component 3: End-to-End Reconciliation

**Scope:** Verifying row counts and key field values from SI source all the way to NFS final tables.

| Sub-Component | What It Analyzes | Key Artifacts |
|---------------|------------------|--------------|
| **Existing V_QUAD_* views** | NFS_MIGRATION staging vs NFS final reconciliation (already exists for ~12 domains) | V_QUAD_ALL_PERC and domain-specific views |
| **SI source vs NFS_MIGRATION staging** | Row count comparison between SI origin and staging tables | Requires dblink queries; currently missing entirely |
| **SI source vs NFS final** | End-to-end integrity: does every SI record arrive correctly in NFS? | Currently missing entirely; this is the highest-value analysis |
| **Key field reconciliation** | Beyond row counts: do amounts, dates, foreign keys match? | Aggregate checks (SUM of amounts), MINUS queries for mismatches |
| **Domain-level reconciliation** | Per-domain queries organized by migration package | One reconciliation set per package |

**Three-tier reconciliation model:**

```
Tier 1: Row Counts (fast, broad)
  SI source COUNT(*) vs NFS_MIGRATION staging COUNT(*) vs NFS final COUNT(*)
  Per table, per request_id, per country

Tier 2: Aggregate Checks (medium, targeted)
  SUM(amount), MIN/MAX(date), COUNT(DISTINCT key) comparisons
  Identifies systematic over/under-counting

Tier 3: Record-Level MINUS (slow, precise)
  SI_table MINUS NFS_final_table on key columns
  Identifies exactly which records are missing/extra/wrong
```

**Boundary:** This component produces read-only SQL queries. It does NOT modify data or package logic.

---

## How the Three Deliverable Areas Relate

```
                    +----------------------------------+
                    |     Migration Package Codebase    |
                    |     (25 packages, ~99K lines)     |
                    +--------+--------+--------+-------+
                             |        |        |
              +--------------+   +----+----+   +--------------+
              |                  |         |                   |
     +--------v--------+  +-----v-----+  +--------v--------+
     |  PERFORMANCE     |  |  DATA      |  |  RECONCILIATION |
     |  ANALYSIS        |  |  ERROR     |  |  QUERIES        |
     |                  |  |  ANALYSIS  |  |                  |
     | Indexes          |  | Per-pkg    |  | SI vs staging   |
     | Statistics       |  | logic      |  | staging vs final|
     | Sequences        |  | review     |  | SI vs NFS final |
     | Parallel config  |  | Fix scripts|  | Row + aggregate |
     | Bulk processing  |  |            |  | + record-level  |
     +--------+---------+  +-----+------+  +--------+--------+
              |                  |                   |
              v                  v                   v
     DDL/config scripts   Modified PL/SQL     SQL query scripts
     (run BEFORE/AFTER    (replace existing    (run ON DEMAND
      migration)           packages)            for verification)
```

### Dependency Graph Between Deliverable Areas

```
Performance Analysis <------- Data Error Analysis
       |                             |
       |  (Performance fixes don't   |  (Logic fixes may change
       |   depend on logic fixes,    |   which tables/columns
       |   but logic fixes benefit   |   need indexes)
       |   from perf fixes being     |
       |   applied first for         |
       |   testing speed)            |
       |                             |
       +----------+  +---------------+
                  |  |
                  v  v
          Reconciliation Queries
          (Must know BOTH the correct
           target structure AND have
           perf context for query design)
```

**Key insight:** Performance and Data Error analyses are largely independent and can be done in parallel, but Reconciliation benefits from both being complete first (to know which tables exist, what the correct mappings are, and to design queries that run efficiently).

---

## Management/Orchestration Layer

The migration is orchestrated through management tables in NFS_MIGRATION:

| Table | Purpose | Role in Analysis |
|-------|---------|------------------|
| `MGR_REQUEST` | One row per migration run (request_id, scope, country, status, dblink) | Central audit trail; all queries filter by REQ_NM_ID |
| `MGR_REQUEST_STEP` | Step definitions per scope (defines the Clear/Load/Validate/Final/Transfer/End sequence) | Defines migration flow; analysis must understand step ordering |
| `MGR_REQUEST_STATUS` | Status codes (INS, PROG, DONE, END, ERROR) | Status tracking; useful for identifying failed runs |
| `MGR_CONFIGURATION` | Maps source entities to staging tables per scope/subscope/dblink | Core metadata; loading_staging is data-driven from this |
| `MGR_CFG_DBLINK` | Country-specific dblink configuration (paese, sigla, codice) | Country resolution; critical for multi-country analysis |
| `MGR_COLS_EXCLUDED` | Columns to skip during staging load | Column filtering; may hide important data |
| `MGR_COLS_WHERE` | Filter columns for year-range partitioning during staging | Date-based filtering; critical for incremental loads |
| `MGR_VALIDATE_RULE` | Validation rules applied after staging | Rule completeness is part of data error analysis |
| `MGR_LOG_ELAB` | Execution log (autonomous transactions, survives rollbacks) | Audit trail for debugging failed migrations |
| `MGR_LOG_QUADRATURE` | Reconciliation results | Existing reconciliation infrastructure |
| `MGR_SEQ_VAL` | Sequence value tracking | Related to MAX-based ID generation problem |

---

## Staging Table Architecture

Staging tables in NFS_MIGRATION mirror SI source structure with additional columns:

| Added Column | Purpose |
|--------------|---------|
| `REQ_NM_ID` | Links to MGR_REQUEST; partitions data by migration run |
| `SEQ_NO` | Auto-generated sequence number for chunking (identity column via ISEQ$$_*) |
| `ROW_ID` | Tracks original SI rowid for reconciliation |

Intermediate domain tables (CLEARING, BANK_TRANSFER, CONTABILE_FF, etc.) exist in NFS_MIGRATION with the same added columns, serving as a transformation stage before final insert into target schemas.

**COE_* tables** (COE_CLEARING, COE_CONTABILE_FF, etc.) are coexistence copies that mirror the intermediate tables for the old-to-new coexistence bridge.

---

## Cross-Schema Dependency Map

```
PKG_MIGRATION_REGISTRY --> NFS_REGISTRY.COUNTERPART
                       --> NFS_REGISTRY.LINK_CTP_RTP_RST
                       --> NFS_REGISTRY.CONTACT
                       --> NFS_REGISTRY.LINK_CTP_CTP

PKG_MIGRATION_CLEARING --> NFS_COLLECTION.CLEARING (via NFS_COLLECTION.SQ_CLE sequence)
                       --> NFS_LEGACY.OLDROWID2NEW (bridge)
                       --> NFS_LEGACY.DOI_RATA_CED_DEB (lookup)
                       --> NFS_LEGACY.DOI_CONTABILE_FF (lookup)
                       --> NFS_LEGACY.BTD_PAGAMENTO_FCD_FAA (lookup)
                       --> NFS_LEGACY.BTD_RISTR_PAGAMENTO (lookup)

PKG_MIGRATION_BONIFICI --> NFS_COLLECTION.BANK_TRANSFER*
                       --> NFS_COLLECTION.LINK_BTR_BTD
                       --> NFS_COLLECTION.LINK_BTD_BDS
                       --> NFS_COLLECTION.MATCHING_PROPOSAL
                       --> NFS_LEGACY.BAT_PAGAMENTO_FCD_FAA
                       --> NFS_LEGACY.BTD_*

PKG_MIGRATION_FATTURE  --> NFS_PRODUCTS.DOCUMENT_INSTALMENT (lookup)
                       --> NFS_LEGACY.DOI_RATA_CED_DEB
                       --> NFS_LEGACY.DOI_FATTURA_ATTIVA

PKG_MIGRATION_CONTAB.  --> NFS_PRODUCTS.ACCOUNTING_DOCUMENT (lookup)
                       --> NFS_LEGACY.DOI_CONTABILE_FF

PKG_MIGRATION_TAXONOMY --> NFS_REGISTRY.SUPERCATEGORY
                       --> NFS_REGISTRY.CATEGORY
                       --> NFS_REGISTRY.SUBCATEGORY
                       --> NFS_REGISTRY.SUBCATEGORYDATA
                       --> NFS_REGISTRY.COUNTERPART_TAXONOMY_DATA
```

**Critical dependency:** Registry must be migrated BEFORE clearing, bank transfers, invoices, and all other domains because they reference NFS_REGISTRY.COUNTERPART and NFS_REGISTRY.LINK_CTP_RTP_RST for ID resolution. Taxonomy should run before or alongside registry since counterpart taxonomy data references the category hierarchy.

---

## Suggested Analysis Order

Based on the architecture, analysis should follow this order:

### Phase 1: Infrastructure (Performance Analysis)
**Why first:** Index and statistics analysis requires understanding table structures but not package logic. Results benefit all subsequent testing.

1. Catalog all NFS_MIGRATION staging tables and their indexes (520 existing)
2. Identify missing indexes on join columns used in loading_final and Transfer steps
3. Catalog all 42 SELECT MAX patterns and map to proper sequences
4. Create DBMS_STATS gathering script for all staging and intermediate tables
5. Assess DBMS_PARALLEL_EXECUTE configuration (degree, chunk sizes)
6. Identify FORALL/BULK COLLECT optimization opportunities (only 9 instances in 99K lines)

### Phase 2: Registry + Taxonomy (Data Error Analysis, highest dependency)
**Why second:** All other packages depend on registry data. Errors here cascade everywhere.

1. Analyze PKG_MIGRATION_REGISTRY (largest package, ~14K lines)
2. Analyze PKG_MIGRATION_REGISTRY_POL
3. Analyze PKG_MIGRATION_TAXONOMY
4. Focus on SELECT MAX patterns in counterpart/contact ID generation
5. Check CSE_ANAGRAFICA_DETTAGLIO SELECT MAX(rowid) patterns (7 instances)

### Phase 3: Core Financial Domains (Data Error Analysis)
**Why third:** These are the high-volume tables where errors have the most impact.

1. PKG_MIGRATION_CLEARING (RATA_CED_DEB: 41.2M rows, PAREGGIO_FAT_ATT: 23.8M rows)
2. PKG_MIGRATION_FATTURE + PKG_MIGRATION_FATT_ATTIVE (FATTURA_ATTIVA: 321K rows)
3. PKG_MIGRATION_CONTABILITA (CONTABILE_FF: 363K rows)
4. PKG_MIGRATION_MANDATI (MANDATI: 326K rows)
5. PKG_MIGRATION_MOVIMENTI
6. PKG_MIGRATION_BONIFICI

### Phase 4: Supporting Domains (Data Error Analysis)
**Why fourth:** Lower volume, fewer dependencies.

1. PKG_MIGRATION_DEAL_STL_LAL (deals, stock loads, legal actions)
2. PKG_MIGRATION_LOAN
3. PKG_MIGRATION_CIGCUP
4. PKG_COEXISTENCE_OLD2NEW (coexistence bridge)

### Phase 5: SK Variants (Data Error Analysis)
**Why fifth:** SK packages mirror IT packages with country-specific adjustments. Analyze differences only.

1. All _SK suffix packages (compare to IT counterparts)

### Phase 6: Reconciliation Queries
**Why last:** Requires knowledge of all table structures, correct mappings, and transformation logic from prior phases.

1. SI source vs NFS_MIGRATION staging (Tier 1: row counts)
2. NFS_MIGRATION staging vs NFS final (Tier 1: row counts) -- extends existing V_QUAD_* views
3. SI source vs NFS final (Tier 1: row counts, Tier 2: aggregates)
4. Record-level MINUS queries (Tier 3) for critical tables

---

## Anti-Patterns to Avoid in Analysis Scripts

### Anti-Pattern 1: Analyzing Package Logic Without Understanding MGR_CONFIGURATION
**What:** Jumping straight into package body SQL without understanding the metadata tables.
**Why bad:** loading_staging is entirely data-driven from MGR_CONFIGURATION. Column mappings, exclusions, and filter conditions are not in the package code -- they are in configuration tables.
**Instead:** Start every package analysis by querying MGR_CONFIGURATION for that scope/subscope.

### Anti-Pattern 2: Treating IT and SK as Identical
**What:** Assuming _SK packages are just copies with different country constants.
**Why bad:** Some packages have significant logic differences (e.g., Spain special case on req_id=904 in clearing). SK packages may have unique adjustments.
**Instead:** Diff IT and SK package bodies to identify actual differences.

### Anti-Pattern 3: Ignoring the COE_* Tables
**What:** Analyzing only the main migration flow and ignoring coexistence tables.
**Why bad:** PKG_COEXISTENCE_OLD2NEW is ~6,400 lines and writes to COE_* mirror tables. These are used for parallel running of old and new systems. Errors here break coexistence.
**Instead:** Include COE_* tables in reconciliation and data error analysis.

---

## Patterns to Follow in Analysis

### Pattern 1: Trace-Through Analysis
For each package, trace a single record from SI source through all 5 hops to NFS final, documenting every transformation.

### Pattern 2: Join Verification
For every LEFT JOIN in loading_final and Transfer steps, verify that the join columns have matching data types and that NULLs from missed joins are handled correctly (many NVL patterns exist but may be incomplete).

### Pattern 3: Volume-Weighted Priority
Analyze high-volume table paths first. RATA_CED_DEB at 41.2M rows means even a 0.01% error rate produces 4,100+ incorrect records.

---

## Sources

- Direct analysis of `C:\Desktop\BFF\Code Review\NFS\Migrazione\nfs_migration.sql` (99,465 lines)
- Codebase architecture document at `C:\Desktop\BFF\Code Review\.planning\codebase\ARCHITECTURE.md`
- Codebase structure document at `C:\Desktop\BFF\Code Review\.planning\codebase\STRUCTURE.md`
- Codebase concerns document at `C:\Desktop\BFF\Code Review\.planning\codebase\CONCERNS.md`
- Project definition at `C:\Desktop\BFF\Code Review\Migrazione\.planning\PROJECT.md`

*Architecture analysis: 2026-02-09*