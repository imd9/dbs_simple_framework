# Databricks Metadata-Driven ELT Framework

## Design and Build Plan — Presentation Document

**Framework**: Metadata-Driven ELT Framework (MDEF)
**Platform**: Databricks Lakeflow
**Proof of Concept**: SCION Data Platform
**Date**: July 2026
**Author**: Data Engineering Team

> *A general-purpose, reusable ELT platform for any Databricks project or catalog.
> SCION just happens to be our first proof of concept.*

---

## 1. What We Are Building

The MDEF is a single reusable data engineering engine that processes any number of datasets
from any source system. All behavior is controlled through six metadata tables.
No separate pipeline per table. No separate notebook per source. One framework. Configuration drives everything.

### The Problem Today

| Without This Framework | With This Framework |
|---|---|
| One pipeline per table | One reusable engine |
| Separate notebook per source | Configuration rows per dataset |
| DQ logic scattered everywhere | Centralized rule registry |
| Audit logic rebuilt each time | Standard audit contract |
| Onboarding a new dataset = engineering sprint | Onboarding a new dataset = INSERT rows |

### The Core Principle

```
Metadata  =  The instruction book
Code      =  The worker that follows instructions
Lakeflow  =  The execution and orchestration platform
```

### General-Purpose Design

This framework is not tied to any catalog, team, or source system.
Any Databricks project can adopt it:

1. Deploy the framework via Declarative Asset Bundle
2. Create the six metadata tables in the project catalog
3. Populate metadata rows for source objects
4. Run the Lakeflow Job

**SCION Data Platform** is the first project to adopt this framework as a Proof of Concept.
The architectural decisions made during the SCION POC will inform the framework's evolution.

---

## 2. The LEGO Design Philosophy

The framework is built as interchangeable LEGO bricks.
Each brick is self-contained, well-defined, and replaceable.
Metadata controls which bricks activate and how they connect.
Swapping one brick does not require redesigning the rest.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  SOURCE      │  │  INGESTION   │  │  METADATA    │  │ORCHESTRATION │
│  LEGO        │  │  LEGO        │  │  LEGO        │  │  LEGO        │
│              │  │              │  │              │  │              │
│ Oracle       │  │ Auto Loader  │  │ source_config│  │ Lakeflow Jobs│
│ SAP          │  │ JDBC         │  │ object_config│  │ Schedule     │
│ CSV/JSON     │  │ LF Connect   │  │ column_config│  │ Retry        │
│ Kafka        │  │ Kafka Conn.  │  │ quality_rules│  │ Monitor      │
│ REST API     │  │ API Conn.    │  │ gold_config  │  │ Alert        │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│TRANSFORMATION│  │  QUALITY     │  │  AUDIT       │  │ CONSUMPTION  │
│  LEGO        │  │  LEGO        │  │  LEGO        │  │  LEGO        │
│              │  │              │  │              │  │              │
│ Bronze stage │  │ Rule engine  │  │pipeline_audit│  │ Power BI     │
│ Silver stage │  │ SQL booleans │  │ Metrics      │  │ SQL queries  │
│ Gold models  │  │ Quarantine   │  │ Run history  │  │ ML pipelines │
│ CDC / SCD2   │  │ Warn/Drop    │  │ Alerting     │  │ AI apps      │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

---

## 3. The Six Metadata Tables

A simple framework starts with six tables.
Together they are the complete instruction book.

### 3.1 source_config

One row per source system. Answers: *Where does data come from?*

| Column | Type | Purpose |
|---|---|---|
| source_id | STRING | Unique identifier |
| source_name | STRING | Friendly name |
| source_type | STRING | cloud_files / jdbc / lakeflow_connect / kafka / api |
| ingestion_method | STRING | autoloader / jdbc / lakeflow_connect / kafka / api |
| landing_required | BOOLEAN | true = files need landing volume before Bronze |
| connection_ref | STRING | Secret scope reference or UC connection name |
| base_path | STRING | Root path for file-based sources |
| active | BOOLEAN | Enable or disable this source |

### 3.2 object_config

One row per dataset. Answers: *Which datasets to process and how?*

| Column | Type | Purpose |
|---|---|---|
| object_id | STRING | Unique identifier |
| source_id | STRING | FK to source_config |
| object_name | STRING | Logical name (e.g. customers) |
| source_path | STRING | Full source path or table name |
| file_format | STRING | csv / json / parquet |
| load_type | STRING | FULL / INCREMENTAL / CDC / SNAPSHOT |
| primary_key | STRING | Business key column(s), comma-separated |
| incremental_column | STRING | Watermark column for incremental loads |
| bronze_table | STRING | Target Bronze table (catalog.schema.table) |
| silver_table | STRING | Target Silver table |
| active | BOOLEAN | Enable or disable this object |
| execution_order | INT | Optional sequence number |
| parallel_group | STRING | Optional parallel execution group |

### 3.3 column_config

One row per column per dataset. Answers: *What should each dataset look like after cleaning?*

| Column | Type | Purpose |
|---|---|---|
| object_id | STRING | FK to object_config |
| source_column | STRING | Column name in source |
| target_column | STRING | Standardized target name |
| target_type | STRING | Spark SQL type: STRING, BIGINT, TIMESTAMP... |
| nullable | BOOLEAN | Allow null values |
| primary_key_indicator | BOOLEAN | Part of the business key |
| sequence | INT | Output column order (1-based) |
| trim_indicator | BOOLEAN | Strip leading/trailing whitespace |
| pii_classification | STRING | PII / SENSITIVE / PUBLIC |
| active | BOOLEAN | Include or exclude this column |

### 3.4 quality_rule_config

One row per DQ rule per dataset. Answers: *What makes a record valid?*

| Column | Type | Purpose |
|---|---|---|
| rule_id | STRING | Unique identifier |
| object_id | STRING | FK to object_config |
| rule_name | STRING | Human-readable rule name |
| rule_expression | STRING | Spark SQL boolean (TRUE = row passes) |
| action | STRING | warn / drop / quarantine / fail |
| severity | STRING | informational / warning / error / critical |
| active | BOOLEAN | Enable or disable this rule |

Rule expression examples:
- `customer_id IS NOT NULL`
- `email IS NULL OR email LIKE '%@%'`
- `order_amount >= 0`
- `TRY_CAST(created_date AS TIMESTAMP) IS NOT NULL`

### 3.5 gold_model_config

One row per Gold business model. Answers: *What business aggregations should run?*

| Column | Type | Purpose |
|---|---|---|
| gold_model_id | STRING | Unique identifier |
| target_table | STRING | Fully qualified Gold table name |
| definition_path | STRING | Path to SQL or Python model file |
| refresh_type | STRING | incremental / materialized / full |
| dependency_list | STRING | Comma-separated Silver table dependencies |
| active | BOOLEAN | Enable or disable this model |

Gold models store business logic in version-controlled SQL or Python files.
The metadata only points to them; it does not embed the business logic.

### 3.6 pipeline_audit

One row per (run, object, layer). Answers: *What happened, when, and with how many records?*

| Column | Type | Purpose |
|---|---|---|
| run_id | STRING | Framework execution identifier |
| object_id | STRING | Dataset being processed |
| layer | STRING | landing / bronze / silver / gold |
| start_time | TIMESTAMP | Execution start |
| end_time | TIMESTAMP | Execution end |
| rows_read | BIGINT | Input record count |
| rows_written | BIGINT | Output record count |
| rows_rejected | BIGINT | Quarantine record count |
| status | STRING | STARTED / SUCCESS / FAILED / SKIPPED |
| error_message | STRING | Failure details |
| source_watermark | STRING | Last watermark value processed |

---

## 4. Metadata Relationships

```
source_config
  One row per source system
       │
       │  1 : many
       ▼
object_config
  One row per dataset / table / file feed
       │
       ├──── 1 : many ────► column_config
       │                      One row per column
       │
       ├──── 1 : many ────► quality_rule_config
       │                      One row per DQ rule
       │
       └──── execution ────► pipeline_audit
                              One row per (run, object, layer)

gold_model_config
  One row per Gold model
  References Silver tables in dependency_list
```

**Example hierarchy:**

```
Source: sales_files
│
├── Object: customers  (object_id = 101)
│   ├── column: customer_id       (PK, BIGINT)
│   ├── column: customer_name     (STRING, trim)
│   ├── column: email             (STRING, PII)
│   ├── column: created_date      (TIMESTAMP)
│   ├── rule:   customer_id IS NOT NULL             → drop
│   └── rule:   email IS NULL OR email LIKE '%@%'  → quarantine
│
├── Object: orders  (object_id = 102)
│   ├── column: order_id          (PK, BIGINT)
│   ├── column: customer_id       (BIGINT)
│   ├── column: order_amount      (DOUBLE)
│   └── rule:   order_amount >= 0                  → fail
│
└── Object: products  (object_id = 103)
    ├── column: product_id        (PK, BIGINT)
    └── column: product_name      (STRING, trim)

Gold Model: daily_sales
  dependencies: silver.orders, silver.products
  definition:   sql/gold/daily_sales.sql
```

---

## 5. End-to-End Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  1. SOURCE SYSTEMS                                           │
│  Oracle │ SAP │ CSV/JSON │ Parquet │ Kafka │ API │ Salesforce│
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  2. METADATA CONFIGURATION  ◄── Populated by DE team        │
│  source_config  ·  object_config  ·  column_config          │
│  quality_rule_config  ·  gold_model_config                  │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  3. LAKEFLOW JOB  (Orchestration LEGO)                       │
│  Task 1: Initialize run (run_id, STARTED audit)              │
│  Task 2: Validate metadata (pre-flight checks)               │
│  Task 3: Run Bronze + Silver pipeline                        │
│  Task 4: Run Gold models                                     │
│  Task 5: Reconcile counts + audit                            │
│  Task 6: Notify on success or failure                        │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
                  ┌───── landing_required? ─────┐
                  │                             │
             YES (Files)                   NO (DB/Stream)
                  │                             │
                  ▼                             ▼
        UC Volume / ADLS               JDBC / Lakeflow Connect
        Vendor Drop Zone               Kafka / API Connector
                  │                             │
                  └──────────────┬──────────────┘
                                 │
                                 ▼
                     AUTO LOADER  (cloudFiles)
                     Incremental · Checkpointed · Schema-evolving
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────┐
│  4. BRONZE TABLES  (Raw LEGO)                                │
│  Raw Delta · Append-only · Near-source fidelity             │
│  + _ingestion_timestamp  _source_file  _object_id  _run_id  │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  5. SILVER  (Transformation LEGO + Quality LEGO)             │
│  column_config  → rename, cast to target_type, trim         │
│  quality_rule_config → evaluate SQL boolean expressions      │
│  CDC / SCD2 when load_type = CDC                             │
└───────────────────────┬──────────────────────┬───────────────┘
                        │                      │
                        ▼                      ▼
             VALID SILVER RECORDS       QUARANTINE TABLES
             Clean · Typed · Deduped    Rejected rows + violation tags
                        │               _failed_rule · _failure_reason
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  6. GOLD MODELS  (Business LEGO)                             │
│  gold_model_config points to SQL or Python files             │
│  Facts · Dimensions · KPIs · Aggregations                   │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  7. CONSUMPTION                                              │
│  Power BI · SQL Analytics · ML Feature Pipelines · AI Apps  │
└──────────────────────────────────────────────────────────────┘

Audit spans every layer:
pipeline_audit ──► Metrics dashboards ──► Alerts ──► Notifications
```

---

## 6. Execution Flow — Step by Step

### Step A: Initialize Execution
- Generate a unique `run_id` (e.g. `RUN-20260714-080001`)
- Record start timestamp
- Confirm destination catalogs and schemas are accessible
- Write STARTED audit row to `pipeline_audit`

### Step B: Validate Metadata
- Every active object has an active source
- Incremental objects have an `incremental_column`
- CDC objects have a `primary_key` and sequence column
- Primary key columns exist in `column_config`
- Quality rule expressions are syntactically valid
- Gold model dependencies exist as Silver tables
- No two active columns map to the same target name
- Secrets are referenced, not stored as plaintext

**Fail early. Never move data against invalid configuration.**

### Step C: Discover Active Objects
- `SELECT * FROM object_config WHERE active = true ORDER BY execution_order`
- Each returned row is a unit of work
- Rows sharing a `parallel_group` can run concurrently

### Step D: Ingest to Bronze
For each active object:
- Read `source_config.landing_required`
- If `true`: read files from UC Volume using Auto Loader
- If `false`: read directly from source via JDBC or connector
- Append to Bronze Delta table
- Add: `_ingestion_timestamp`, `_source_file`, `_object_id`, `_run_id`
- Write BRONZE audit row: rows read, rows written, status

### Step E: Standardize to Silver
For each Bronze table:
- Load `column_config` for this object
- Rename source columns → target column names
- Cast to target types using `TRY_CAST` (ANSI-safe, no failures)
- Trim whitespace where `trim_indicator = true`
- Deduplicate on `primary_key`
- Evaluate quality rules, split valid/quarantine
- Write SILVER audit row

### Step F: Build Gold Models
- Load `gold_model_config WHERE active = true`
- Resolve execution order from `dependency_list`
- Run each model's SQL or Python definition file
- Write GOLD audit row: rows written, duration, status

---

## 7. Load Strategies

| Load Type | When to Use | Required Fields |
|---|---|---|
| FULL | Small reference tables, complete snapshots | None |
| INCREMENTAL | Tables with a reliable timestamp or ID | `incremental_column` |
| CDC | Change data capture from transactional sources | `primary_key` + sequence column |
| SNAPSHOT | Detect changes by comparing current vs previous | `primary_key` |

---

## 8. Lakeflow Jobs vs Lakeflow Spark Declarative Pipelines

The framework uses both Databricks products in complementary roles.

| Responsibility | Lakeflow Jobs | Lakeflow SDP |
|---|---|---|
| Procedural task sequencing | ✅ | — |
| Scheduling and retries | ✅ | — |
| Notifications and alerting | ✅ | — |
| Parallel task coordination | ✅ | — |
| Declarative dataset dependencies | — | ✅ |
| Streaming tables | — | ✅ |
| Materialized views | — | ✅ |
| Built-in DQ expectations | — | ✅ |
| Auto CDC and SCD | — | ✅ |

Lakeflow Jobs handles the procedural outer loop.
Lakeflow SDP handles declared dataset relationships and incremental processing inside the pipeline.

---

## 9. Directory Structure

```
databricks-metadata-framework/
│
├── databricks.yml                          ← Declarative Asset Bundle root
│
├── resources/
│   ├── lakeflow_pipeline.yml               ← SDP pipeline declaration
│   └── lakeflow_job.yml                    ← Lakeflow Job: tasks, schedule, compute
│
├── src/
│   │
│   ├── pipeline/
│   │   ├── pipeline.py                     ← Entry point: reads metadata, calls generators
│   │   ├── bronze_generator.py             ← Ingests source to Bronze via metadata config
│   │   ├── silver_generator.py             ← Applies column_config + quality rules to Silver
│   │   └── quality_engine.py              ← Evaluates quality_rule_config expressions
│   │
│   ├── orchestration/
│   │   ├── initialize_run.py               ← Generates run_id, writes STARTED audit
│   │   ├── validate_metadata.py            ← Pre-flight checks across all config tables
│   │   ├── audit_reconciliation.py         ← Writes final audit rows with metrics
│   │   └── notify.py                       ← Alerts on failure or data anomalies
│   │
│   ├── metadata/
│   │   ├── metadata_reader.py              ← get_active_objects(), get_column_rules(), etc.
│   │   └── schemas.py                      ← IngestionSpec dataclass (neutral contract)
│   │
│   └── gold/
│       └── daily_sales.sql                 ← Example Gold model SQL definition
│
├── sql/
│   ├── migrations/
│   │   ├── V001__create_source_config.sql
│   │   ├── V002__create_object_config.sql
│   │   ├── V003__create_column_config.sql
│   │   ├── V004__create_quality_rule_config.sql
│   │   ├── V005__create_gold_model_config.sql
│   │   └── V006__create_pipeline_audit.sql
│   │
│   └── seeds/
│       ├── DEV__seed_source_config.sql
│       ├── DEV__seed_object_config.sql
│       ├── DEV__seed_column_config.sql
│       ├── QA__seed_source_config.sql
│       └── PROD__seed_source_config.sql
│
└── tests/
    ├── test_metadata_validation.py
    ├── test_bronze_generator.py
    ├── test_silver_generator.py
    ├── test_quality_engine.py
    └── test_gold_models.py
```

---

## 10. File Descriptions

### `databricks.yml`
Declarative Asset Bundle root. Defines project name, workspace targets (dev, QA, prod),
and artifact paths. The entire framework deploys with `databricks bundle deploy`.

### `resources/lakeflow_pipeline.yml`
Declares the Lakeflow Spark Declarative Pipeline. Defines name, catalog target,
libraries (pipeline.py), and mode (development vs production).

### `resources/lakeflow_job.yml`
Defines the Lakeflow Job with all six tasks in execution order:
initialize → validate → pipeline → gold → reconcile → notify.
Includes schedule, retry policy, compute config, and notification settings.

### `src/pipeline/pipeline.py`
Main entry point for pipeline execution. Reads source_config and object_config.
Calls bronze_generator and silver_generator for each active object.
Passes the populated IngestionSpec to each generator function.

### `src/pipeline/bronze_generator.py`
Creates or updates Bronze Delta tables.
- File sources: uses Auto Loader (cloudFiles) — incremental, checkpointed, schema-evolving
- DB sources: uses JDBC or the appropriate connector
Adds standard ingestion metadata columns.
Handles FULL and INCREMENTAL append modes.

### `src/pipeline/silver_generator.py`
Reads Bronze data for a given object.
Applies column_config: rename, cast with TRY_CAST, trim whitespace.
Deduplicates on primary_key.
Calls quality_engine to split valid/quarantine.
Handles CDC and SCD2 when load_type = CDC.
Writes to Silver Delta table.

### `src/pipeline/quality_engine.py`
Evaluates quality_rule_config rows as Spark SQL boolean expressions.
Splits the DataFrame into a valid set and a quarantine set.
Tags quarantine rows with: _failed_rule, _failure_reason, _run_id, _object_id.
Supports warn, drop, quarantine, and fail actions.

### `src/orchestration/initialize_run.py`
Generates a unique run_id at the start of every Lakeflow Job execution.
Stores start timestamp. Confirms catalog and schema accessibility.
Writes the initial STARTED record to pipeline_audit.

### `src/orchestration/validate_metadata.py`
Pre-flight validation across all config tables.
Checks FK integrity, required fields, rule syntax, and configuration consistency.
Raises a descriptive error if any critical check fails.
Prevents bad configuration from reaching the data layer.

### `src/orchestration/audit_reconciliation.py`
Called at the end of each object's processing.
Calculates rows read, rows written, rows rejected, and duration.
Writes the completed audit row to pipeline_audit with final status.
Also executes the run-level reconciliation step.

### `src/orchestration/notify.py`
Sends alerts based on run outcomes.
Triggers when: a critical object fails, rejected rows exceed threshold,
freshness SLA is breached, or schema drift introduces an unexpected column.

### `src/metadata/metadata_reader.py`
Central metadata access layer.
Provides: get_active_objects(), get_column_rules(object_id),
get_quality_rules(object_id), get_gold_models().
Abstracts all SQL queries from the rest of the framework.
Caches results per run to avoid repeated queries.

### `src/metadata/schemas.py`
Defines the IngestionSpec dataclass.
This is the neutral, generation-agnostic internal contract
that flows through all stage functions.
The metadata reader populates IngestionSpec objects.
The generators consume them.
This contract is what makes the framework portable across metadata generations.

### `sql/migrations/V00N__*.sql`
Versioned DDL scripts for each metadata table.
Run in sequence to create the framework schema in any environment.
Each script uses CREATE TABLE IF NOT EXISTS (idempotent).

### `sql/seeds/ENV__seed_*.sql`
Environment-specific seed data.
DEV: lightweight sample datasets for development and testing.
QA: production-like shapes without real data.
PROD: approved bootstrap entries only.

### `tests/test_*.py`
pytest-compatible unit and integration tests.
Cover metadata validation logic, column mapping transformations,
quality rule evaluation, and Gold model output shapes.
Run in CI before any bundle deployment.

---

## 11. Adding a New Dataset

To add a new source object after initial deployment:

```sql
-- 1. Add to object_config
INSERT INTO framework.object_config VALUES
(104, 1, 'suppliers', '/Volumes/raw/sales/suppliers',
 'csv', 'incremental', 'supplier_id', 'updated_timestamp',
 'bronze.suppliers', 'silver.suppliers', TRUE, 40, 'sales');

-- 2. Add column definitions
INSERT INTO framework.column_config VALUES
(104, 'SUPPLIER_ID',   'supplier_id',   'BIGINT', FALSE, TRUE,  1, FALSE, NULL,   TRUE),
(104, 'SUPPLIER_NAME', 'supplier_name', 'STRING', FALSE, FALSE, 2, TRUE,  NULL,   TRUE),
(104, 'COUNTRY',       'country',       'STRING', TRUE,  FALSE, 3, TRUE,  NULL,   TRUE);

-- 3. Add quality rules
INSERT INTO framework.quality_rule_config VALUES
(10, 104, 'supplier_id_required', 'supplier_id IS NOT NULL', 'drop', 'error', TRUE);
```

The next job run automatically discovers and processes the new object.
**No code changes. No new notebook. No new job task. Configuration only.**

---

## 12. SCION as the Proof of Concept

SCION Data Platform (Surgery Commercial Innovation on Next-Generation Data Platform)
is the first project to adopt this framework.

### What SCION contributes to the POC
- Multiple source types: file-based (MBOX), database (Oracle / Teradata), enterprise (SAP)
- Pre-existing IngestionSpec abstraction and stage functions
- Working Auto Loader Bronze implementation
- Active development team with real surgery commercial datasets

### What the POC validates
- Six-table metadata model is sufficient to drive a full Bronze-Silver-Gold pipeline
- The IngestionSpec contract bridges existing code to the new metadata model
- The LEGO architecture scales from one dataset to dozens
- Lakeflow Jobs can coordinate the reusable stage engine

### What the POC produces for the framework
- Proven migration pattern from legacy metadata models to MDEF
- Reference implementation for new teams adopting the framework
- Template seed data covering file and database source patterns

---

## 13. Next Steps

### Phase 1 — Metadata Foundation (Immediate)
1. Create the six metadata tables via versioned SQL migration scripts
2. Populate DEV seed data for one file source and one database source
3. Implement `metadata_reader.py` and `schemas.py` (IngestionSpec)
4. Validate that `get_active_objects()` returns correct IngestionSpec objects

### Phase 2 — Stage Implementation (Short-term)
5. Implement `bronze_generator.py` using Auto Loader against metadata config
6. Implement `silver_generator.py` using column_config mappings
7. Implement `quality_engine.py` using quality_rule_config rules
8. Wire `pipeline_audit` writes via `audit_reconciliation.py`

### Phase 4 — Orchestration (Medium-term)
9. Add execution_order and parallel_group support to the runner
10. Wire `lakeflow_job.yml` with all six tasks
11. Deploy to QA, run against production-like data shapes

### Phase 5 — Execution Engine Decision (Design)
12. Decide: stay on plain PySpark jobs (Option A) or move pipeline execution
    to native Lakeflow Spark Declarative Pipelines (Option B)
13. Document the decision with the design team

### Phase 6 — Developer Experience
14. Finalize provisioning guide for new teams
15. Publish seed templates and onboarding checklist

---

## Summary

```
One framework.
Six metadata tables.
Any source. Any catalog. Any team.

Onboarding a new dataset = INSERT rows.
Not a new pipeline.
```

This is the design. SCION proves it works.
