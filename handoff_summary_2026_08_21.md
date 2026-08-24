# iDEE Pipeline Builder — Session Handoff (August 21, 2026)

## Overview

This session focused on two major changes to the iDEE framework:
1. **Renaming all "bitbucket" references to "external"** — cosmetic but pipeline-breaking
2. **Fixing a metadata resolver bug** introduced by the earlier `task_id` refactoring

Plus two small cleanup items at the end.

---

## Project Root

```
/Workspace/Users/iduran3@its.jnj.com/idee_for_databricks_DAB_testing/
├── idee_dbx_framework/   (framework source — installed via %pip)
│   └── src/idee_dbx/
│       ├── main.py
│       ├── config_parser.py
│       ├── config_validator.py
│       └── runtime/
│           ├── executor.py
│           ├── config_resolver.py   ← BUG FIX HERE
│           ├── contracts.py
│           ├── transformer.py
│           └── ...
├── idee_dbx_app/
│   ├── config/active/
│   │   ├── customer_pipeline_combined.yaml   ← UPDATED
│   │   └── contracts_pipeline.yaml
│   ├── ddl/
│   │   ├── 04_create_bronze_customer_external.sql
│   │   ├── 05_create_bronze_customer_landing.sql
│   │   └── ... (01_create_bronze_customer.sql DELETED)
│   └── notebooks/
│       ├── pipeline_builder.ipynb
│       ├── transform_customer_dedup.ipynb   ← UPDATED
│       └── transform_contract_enrichment.py
```

---

## Change 1: Rename "bitbucket" → "external"

### Why
The user wanted to generalize the data source naming — no longer tied to "Bitbucket" branding. The DDL file was already renamed to `04_create_bronze_customer_external.sql` (creating table `bronze_customer_external`), but downstream references still pointed to the old `bronze_customer_bitbucket` name.

### Files Changed

#### A. `customer_pipeline_combined.yaml` (ID: 475720801161281)
Path: `/idee_dbx_app/config/active/customer_pipeline_combined.yaml`

| Line(s) | Before | After |
|---------|--------|-------|
| ~36 | `# CONNECTOR — fetch source files from Bitbucket before running` | `# CONNECTOR — fetch source files from external data source before running` |
| ~54-58 | `# TASK 1: Bitbucket → Bronze (Bitbucket source)` / `task_name: "bitbucket_to_bronze"` | `# TASK 1: External Source → Bronze` / `task_name: "external_to_bronze"` |
| ~61 | `source_name: "Customer CSV from Bitbucket"` | `source_name: "Customer CSV from External Source"` |
| ~68 | `target_name: "Bronze Customer Bitbucket"` | `target_name: "Bronze Customer External"` |
| ~74 | `description: "Raw customer records from Bitbucket source"` | `description: "Raw customer records from external data source"` |
| ~99 | `# Raw CSV from the landing Volume. Same schema as the Bitbucket file.` | `# ...Same schema as the external source file.` |
| ~150 | `source_name: "Bronze Customer Bitbucket (primary ref)"` | `source_name: "Bronze Customer External (primary ref)"` |
| ~304 | `description: "Daily ETL: Fetch from Bitbucket + Landing → ..."` | `description: "Daily ETL: Fetch from External Source + Landing → ..."` |
| ~312 | `source: bitbucket_and_landing` | `source: external_and_landing` |

#### B. `transform_customer_dedup` notebook (ID: 475720801161290)
Path: `/idee_dbx_app/notebooks/transform_customer_dedup.ipynb`

| Cell | Before | After |
|------|--------|-------|
| Cell 1 (markdown) | `Reads BOTH bronze tables (bitbucket + landing)` | `Reads BOTH bronze tables (external + landing)` |
| Cell 2 (setup) | `BRONZE_BITBUCKET = "...bronze_customer_bitbucket"` | `BRONZE_EXTERNAL = "...bronze_customer_external"` |
| Cell 2 (setup) | `print(f"Source 1 (BB) : {BRONZE_BITBUCKET}")` | `print(f"Source 1 (Ext) : {BRONZE_EXTERNAL}")` |
| Cell 3 (read) | `df_bb = read_bronze(BRONZE_BITBUCKET, "bitbucket")` | `df_ext = read_bronze(BRONZE_EXTERNAL, "external")` |
| Cell 3 (read) | `df_bitbucket = df_bb.select(...)` | `df_external = df_ext.select(...)` |
| Cell 3 (read) | `print(f"Bitbucket rows : {df_bitbucket.count()}")` | `print(f"External rows  : {df_external.count()}")` |
| Cell 4 (union) | `combined_df = df_bitbucket.unionByName(df_landing)` | `combined_df = df_external.unionByName(df_landing)` |
| Cell 5 (write) | `Check that {BRONZE_BITBUCKET}` | `Check that {BRONZE_EXTERNAL}` |

### Impact
Without this change, the pipeline failed at Task 3 (bronze_combined_to_silver) because the transform notebook tried to `spark.table("...bronze_customer_bitbucket")` which no longer existed.

---

## Change 2: config_resolver.py Bug Fix (task_id lookup)

### Why
A prior session refactored the metadata model so that `pipeline_id` is now shared by ALL tasks in the same YAML (one pipeline_id per YAML file), while each task gets its own unique `task_id`. However, the runtime resolver was still querying child metadata tables by `pipeline_id` — meaning it got back ALL datasets for the entire YAML and grabbed `[0]`, which could be the wrong task.

This caused the contracts pipeline to fail: `contracts_etl_landing_to_bronze` (column_mode: star) was picking up the dataset row for `contracts_etl_silver_to_gold` (column_mode: notebook), so it tried to call a notebook transformer instead of pass-through.

### File Changed

#### `config_resolver.py` (ID: 475720801161352)
Path: `/idee_dbx_framework/src/idee_dbx/runtime/config_resolver.py`

| Line | Before | After |
|------|--------|-------|
| 140-143 | `pipeline = pipelines[0]` | `pipeline = pipelines[0]` |
| | `resolved_id = pipeline["pipeline_id"]` | `resolved_id = pipeline["pipeline_id"]` |
| | | `resolved_task_id = pipeline.get("task_id") or resolved_id` |
| | `datasets = _rows(spark, f"SELECT * FROM {prefix}.{TABLE_DATASET_CONFIG} WHERE pipeline_id = '{resolved_id}'")` | `datasets = _rows(spark, f"SELECT * FROM {prefix}.{TABLE_DATASET_CONFIG} WHERE task_id = '{resolved_task_id}'")` |

### Impact
Fixed the contracts pipeline — each task now resolves to its OWN dataset metadata row. All 7 tasks (4 customer + 3 contracts) execute correctly.

---

## Change 3: Delete legacy `bronze_customer` DDL

### Why
The old `01_create_bronze_customer.sql` DDL created a table that is no longer used by any pipeline. It was a holdover from the original single-source design.

### Action
- **Deleted file**: `/idee_dbx_app/ddl/01_create_bronze_customer.sql` (ID: 475720801161289)
- **Dropped table**: `medtech_md_scion_dev.idee_app.bronze_customer`

### Impact
The DDL step now creates 7 app tables instead of 8. No pipeline references this table.

---

## Change 4: Email DQ Rule — warn → quarantine

### Why
The value "not-an-email" was passing through to silver/gold because the email format rule only issued a warning. The user wants invalid emails quarantined.

### File Changed

#### `customer_pipeline_combined.yaml` (ID: 475720801161281)

| Line | Before | After |
|------|--------|-------|
| ~196-198 | `rule_expression: "email_address IS NULL OR email_address LIKE '%@%'"` | (unchanged) |
| | `action: warn` | `action: quarantine` |
| | `severity: warning` | `severity: error` |

### Impact
Any row where `email_address` is NOT NULL and doesn't contain `@` is now quarantined (moved to a quarantine table) instead of just logged as a warning. This catches values like "not-an-email".

---

## Final Pipeline Output (after all fixes)

Both YAMLs run successfully:

```
=== contracts_pipeline.yaml (3 tasks) ===
  ✓ Task 1: contracts_etl_landing_to_bronze    | star mode
  ✓ Task 2: contracts_etl_bronze_to_silver     | explicit mode
  ✓ Task 3: contracts_etl_silver_to_gold       | notebook mode

=== customer_pipeline_combined.yaml (4 tasks) ===
  ✓ Task 1: customer_etl_combined_external_to_bronze      | star mode (API source)
  ✓ Task 2: customer_etl_combined_landing_to_bronze       | star mode (cloud_files)
  ✓ Task 3: customer_etl_combined_bronze_combined_to_silver | notebook mode (dedup)
  ✓ Task 4: customer_etl_combined_silver_to_gold          | explicit mode
```

Customer pipeline DQ (Task 3): invalid emails now quarantined instead of warned.

---

## Key Architecture Context (for next AI)

- **pipeline_id**: One per YAML file (shared by all tasks in that file)
- **task_id**: One per task within a YAML (hash of `pipeline_name + task_name`)
- **pipeline_config table**: One row per TASK (has both `pipeline_id` and `task_id`)
- **Child tables** (source_config, target_config, dataset_config, etc.): Keyed by `task_id`
- **Resolver lookup**: Finds pipeline_config row by `pipeline_name`, then uses `task_id` to find children
- **Framework install**: `%pip install ../../idee_dbx_framework` (source install, no wheel needed during dev)
- **Catalog**: `medtech_md_scion_dev`, schemas: `idee_metaschema` (metadata), `idee_app` (business data)
- **Secret scope**: `idee_secret_keys` (owned by iduran3, key `01_token` for external API)

---

## Files Summary (quick reference)

| File | ID | Change |
|------|----|--------|
| `customer_pipeline_combined.yaml` | 475720801161281 | Renamed bitbucket→external, email rule warn→quarantine |
| `transform_customer_dedup.ipynb` | 475720801161290 | Renamed all bitbucket vars/refs to external |
| `config_resolver.py` | 475720801161352 | Fixed child table lookup: pipeline_id → task_id |
| `01_create_bronze_customer.sql` | 475720801161289 | DELETED (legacy unused table) |
