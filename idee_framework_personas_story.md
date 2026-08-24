# iDEE Framework — Three Personas Story

## Overview

This document tells the story of the iDEE Pipeline Builder framework from three perspectives. Use this as a demo narrative or onboarding guide.

---

## Persona 1: The Veteran (Experienced Employee, New to the Framework)

### Who they are
Sarah has been at the company for 4 years. She knows the data, knows the business rules, and has built plenty of Spark notebooks manually. Her team just told her: "We're standardizing on iDEE. Here's the repo."

### Her first reaction
"Great, another framework. What do I actually have to do differently?"

### What's hers to touch

| Asset | Location | Her responsibility |
| --- | --- | --- |
| YAML config file | config/active/*.yaml | Define her pipeline: source, target, column mappings, DQ rules |
| DDL files | ddl/*.sql | Define her target table schemas (CREATE TABLE IF NOT EXISTS) |
| Custom notebooks | notebooks/transform_*.py | Write complex transforms (JOINs, dedup, enrichment) |
| Landing volume data | /Volumes/.../landing/ | Upload or manage her source files |

### What she does NOT touch
- The framework source code (idee_dbx_framework/)
- Metadata tables (idee_metaschema.*)
- The pipeline_builder orchestrator notebook
- Schema validation files (schema/pipeline_config_schema_*.yaml)

### How she gets started (5 steps)

1. Copy an existing YAML - She grabs customer_pipeline_combined.yaml, duplicates it, renames it for her domain (e.g., inventory_pipeline.yaml).

2. Edit the pipeline section - Changes pipeline_name, description, owner_email, execution_order.

3. Define her tasks - For each hop (landing to bronze to silver to gold), she fills in:
   - source: where data comes from (cloud_files path, delta_table, or api)
   - target: where it lands (fully qualified table name)
   - dataset: what column_mode to use (star, explicit, or notebook)
   - quality: DQ rules (expressions + actions: warn/quarantine/drop/fail)

4. Write her DDLs - One CREATE TABLE IF NOT EXISTS per target table in the ddl/ folder. Numbers prefix for ordering (06_create_bronze_inventory.sql).

5. Run the pipeline - Opens pipeline_builder, hits Run All. Her config is auto-discovered, validated, parsed into metadata, and executed.

### What she says after her first successful run
"Oh - I didn't write a single line of Spark. I just described what I wanted and it ran. The DQ rules caught two bad rows and quarantined them. That's actually nice."

### When she needs a custom notebook
- Combining multiple source tables (like customer dedup across external + landing)
- Complex enrichment JOINs (like gold_contract joining gold_customer)
- Business logic that can't be expressed in a YAML column mapping

She sets column_mode: notebook, points notebook_path to her transform notebook, and the framework calls it via dbutils.notebook.run(), passing it a staging table to write results into.

---

## Persona 2: The New Hire (First Week at the Company, First Time on Databricks)

### Who they are
James just joined as a junior data engineer. He's used Python and SQL in school but has never seen Databricks, Unity Catalog, or a production ETL framework.

### His first reaction
"There are so many folders. What is a catalog? What's a schema? What's a Volume?"

### What he needs to understand first

The mental model (5 minutes):

    Source Data (CSV/API)
        -> Bronze (raw, as-is)      = SELECT * from source
        -> Silver (cleaned, typed)  = column renaming OR custom notebook
        -> Gold (business-ready)    = explicit columns, enrichment

Key Databricks concepts (10 minutes):
- Catalog = top-level container (like a database server). Ours: medtech_md_scion_dev
- Schema = a namespace within a catalog (like a database). We use two:
  - idee_metaschema = framework internals (he doesn't touch this)
  - idee_app = business data (bronze/silver/gold tables live here)
- Volume = a folder on cloud storage accessible via /Volumes/... (where CSVs land)
- Notebook = a runnable document (like Jupyter) - the pipeline_builder is one

### His learning path

Week 1 - Observer:
- Read an existing YAML (customer_pipeline_combined.yaml) top to bottom
- Look at the bronze/silver/gold tables in Catalog Explorer
- Run the pipeline_builder and watch the output - see which tasks succeed, how many rows flow

Week 2 - Small change:
- Add a new DQ rule to an existing task (e.g., "phone_number must be 10 digits")
- Change a severity from warn to quarantine and see the row disappear from gold
- Look at pipeline_audit table to see the run history

Week 3 - First pipeline:
- Copy a YAML, define a simple 2-task pipeline (landing to bronze to silver)
- Write the DDLs for his new tables
- Run it, debug any validation errors (the framework tells him exactly what's wrong)

Week 4 - Advanced:
- Write a custom notebook transform
- Add column_mode: explicit with type casts
- Understand execution_order and cross-pipeline dependencies

### What he says after Week 1
"I don't fully understand how the framework works internally, but I can read the YAML and see exactly what the pipeline does. That's way better than reading 500 lines of PySpark and guessing which part does what."

---

## Persona 3: The Framework Engineer (Admin / Platform Team)

### Who they are
Maria is on the platform team. She built the iDEE framework, maintains the metadata tables, and helps other teams onboard. When something goes wrong at the infrastructure level, she's the one who gets paged.

### Her responsibilities

| Layer | What she owns | What she does |
| --- | --- | --- |
| Framework source | idee_dbx_framework/src/idee_dbx/ | Writes and tests the core engine: parsers, validators, executors, readers, writers |
| Metadata schema | idee_metaschema.* (9 tables) | Designs table schemas, manages migrations (adding task_id, etc.) |
| Schema validation | schema/pipeline_config_schema_v*.yaml | Defines what a valid YAML looks like - enforces structure before anything runs |
| Pipeline orchestrator | pipeline_builder.ipynb | Maintains the entry point notebook that wires everything together |
| Runtime components | runtime/ (executor, resolver, readers, writers, transformers, DQ engine) | Adds new source types, column modes, DQ engines without users changing their YAMLs |

### What she does NOT own
- Individual pipeline YAML configs (those belong to the data teams)
- Business DDLs and transform notebooks (those belong to the data teams)
- Source data quality (she provides the DQ engine; teams define the rules)

### Her design principles

1. Users describe WHAT, framework handles HOW - A user says "source_type: cloud_files" and the framework knows to use Auto Loader. They never import cloudFiles themselves.

2. Metadata is the single source of truth - Once a YAML is parsed into metadata tables, the executor reads from those tables, not from the YAML directly. This means you can query pipeline_config to see all pipelines, quality_rule_config to see all DQ rules across the org.

3. Idempotent MERGE, not INSERT - Re-running the parser doesn't create duplicate rows. Every metadata row has a deterministic ID (SHA-256 hash of its natural key). MERGE ON that key.

4. Fail loud, fail early - Schema validation catches YAML mistakes before metadata is touched. DQ rules catch data issues before gold is written. The audit table records everything.

5. Extensible registries - New source types, transformers, DQ engines, and target writers are registered via dictionaries. Adding "source_type: kafka" means writing one reader function and registering it - zero changes to the executor.

### Her day-to-day tasks

- Schema evolution: When a new field is needed (like task_id or execution_order), she updates the DDL, the parser, and the resolver. Users' existing YAMLs keep working (defaults handle backward compat).
- Debugging user pipelines: Queries pipeline_audit for failures, checks quality_rule_config for stale rules, verifies metadata integrity.
- Performance: Monitors long-running tasks, optimizes the DQ engine, considers partitioning strategies.
- Onboarding: Reviews new team's first YAML PR, validates their DDLs match the schema, ensures execution_order doesn't conflict.

### What she says during a sprint review
"This sprint we added execution_order to handle cross-pipeline dependencies, refactored pipeline_id vs task_id so multi-task YAMLs share a parent identity, and fixed the resolver to use task_id for child table lookups. None of this required any user-facing YAML changes - existing configs just work."

---

## Summary: Who Touches What

| Asset | Veteran (Sarah) | New Hire (James) | Framework Engineer (Maria) |
| --- | --- | --- | --- |
| YAML configs | Creates and edits | Reads first, edits later | Reviews PRs, never owns |
| DDL files | Writes for her tables | Copies patterns | Writes metadata DDLs only |
| Transform notebooks | Writes complex ones | Observes first | Doesn't write (user-owned) |
| Framework source | Never touches | Never touches | Owns entirely |
| Metadata tables | Never queries directly | Views in Catalog Explorer | Designs, migrates, debugs |
| pipeline_builder | Runs it | Runs it | Maintains it |
| Landing volumes | Manages her data | Explores existing data | Sets up permissions |

---

## The Elevator Pitch (for all three)

"iDEE lets you build a full Bronze to Silver to Gold pipeline by writing a YAML file and a DDL.
No Spark imports, no notebook spaghetti, no manual scheduling.
You declare your source, target, column mappings, and quality rules -
the framework handles reading, transforming, validating, writing, and auditing.
If you need something custom, write a notebook and point the YAML at it.
Everything else is handled."

---

## Appendix: The Three Column Modes (Quick Reference)

| Mode | When to use | What happens |
| --- | --- | --- |
| star | Bronze ingestion (raw, as-is) | SELECT * from source + add ETL audit columns |
| explicit | Silver/Gold with known schema | Cast and rename columns per the YAML columns list |
| notebook | Complex transforms (JOINs, dedup, enrichment) | Framework calls your notebook, you write to a staging table, framework handles the rest |

---

## Appendix: Why DDLs Even With SELECT *?

"If star mode just passes everything through, why not auto-create the table?"

Three reasons:
1. Schema as a contract - If the source CSV changes shape, the write fails loudly instead of silently accepting garbage.
2. Proper typing - CSVs are all strings. DDLs enforce DATE, DECIMAL, BIGINT at the table level.
3. Governance metadata - COMMENT, TBLPROPERTIES, audit columns are defined upfront.

One-liner for the demo: "DDLs are cheap insurance - they cost you 30 seconds to write but catch schema drift before it poisons your Gold layer."

---

## Appendix: Cross-Pipeline Dependencies

Pipelines can depend on each other's Gold tables. Example:
- customer pipeline (execution_order: 1) produces gold_customer
- contracts pipeline (execution_order: 2) reads gold_customer to enrich gold_contract

The framework sorts plans by execution_order before execution. Lower numbers run first.
Default is 50, so any YAML without execution_order runs after explicitly ordered ones.
