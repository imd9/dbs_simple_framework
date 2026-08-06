# How iDEE Works — The Complete Flow

**Framework v0.1 · config schema v0.1**

This explains what each folder does, how the two halves fit together, what the
work looks like for each of the two roles, and what to show in a demo.

---

## 1. The short version

```
        ┌──────────────────────────┐        ┌──────────────────────────┐
        │   idee_dbx_framework     │        │      idee_dbx_app        │
        │                          │        │                          │
        │  built once              │◀───────│  one per team            │
        │  shared by every team    │ imports│  filled configs + DDL    │
        └──────────────────────────┘        └──────────────────────────┘
                     │                                   │
                     │  provides the engine              │  provides the answers
                     ▼                                   ▼
        ┌────────────────────────────────────────────────────────────┐
        │  YAML → validate → parse → metadata tables → RUN → data    │
        └────────────────────────────────────────────────────────────┘
```

The framework is the machine. The app folder is the instructions you feed it.
Nobody edits the machine to add a dataset.

---

## 2. What each folder actually holds

### `idee_dbx_framework` — built once, shared by everyone

| Folder | What it is |
|---|---|
| `templates/` | The blank YAML users copy |
| `schema/` | The validation contract, versioned |
| `src/idee_dbx/` | Validator, parser, metadata loader, CLI |
| `src/idee_dbx/runtime/` | The execution engine: readers, transformers, DQ, writers, executor |
| `ddl/metadata/` | The eight metadata tables |
| `ddl/runtime/` | The shared quarantine table |
| `tests/` | 187 PyTest cases |
| `notebooks/` | End-to-end validation |
| `docs/` | Guides, naming standard, design decisions, changelog |

Only one folder here contains code that runs: `src/idee_dbx/`. Everything else
is what the code reads, what checks the code, or what humans open.

### `idee_dbx_app` — one per application team

| Folder | What it is |
|---|---|
| `config/` | Filled YAML — one file per pipeline |
| `ddl/` | This team's bronze and silver tables |
| `data/` | Sample source files |
| `notebooks/` | How this team calls the framework |
| `docs/` | Guide for data engineers |
| `output/` | Validation and parse output |

**No framework logic lives here.** The app notebook imports `idee_dbx` and calls
it. The test for whether something belongs on this side: *would another team
need it too?* If yes, it belongs in the framework.

---

## 3. The two roles — and whether they are the same person

**In production they are different people, usually different teams:**

| | Framework engineer | Application engineer |
|---|---|---|
| Does what | Stands the framework up | Onboards a dataset |
| How often | Once per environment | Once per dataset |
| Touches | The framework folder | The app folder |
| Needs to know | Python, the metadata model | How to fill out a YAML file |

**For the demo, one person wears both hats** — you set the framework up, then
you switch roles and onboard a dataset. That is fine and worth narrating out
loud, because the separation is the point: *"I am now going to stop being the
framework engineer and start being a data engineer who has never seen this
code."*

---

## 4. Role 1 — Setting up the framework

**Once per environment. Framework folder only.**

The app folder is not involved. It belongs to whoever consumes the framework.

### Step 1 — Copy the framework folder into the workspace

`idee_dbx_framework`. That is the whole install.

### Step 2 — Set the paths

One variable block at the top of the validation notebook:

```python
PROJECT_ROOT    = "/Workspace/Users/<you>@its.jnj.com/idee_for_databricks"
FRAMEWORK       = f"{PROJECT_ROOT}/idee_dbx_framework"
APP             = f"{PROJECT_ROOT}/idee_dbx_app"
CATALOG         = "medtech_md_scion_dev"
METADATA_SCHEMA = "idee_metaschema"
APP_SCHEMA      = "idee_app"
```

That block is the only thing edited by hand in the entire setup. Catalog and
schema are parameters everywhere else, so the same artifacts promote to QA and
prod untouched.

### Step 3 — Run the end-to-end validation notebook

`idee_dbx_framework/notebooks/iDEE Framework End to End Validation`

It does the rest of the install and proves the framework is healthy while doing
it:

| Phase | What happens |
|---|---|
| 1 | Confirm the folder layout resolves |
| 2 | Install `pytest` and `pyyaml`, restart Python, re-set paths |
| 3 | Run the 187 unit tests |
| 4 | Validate both shipped configs |
| 5 | Feed the validator six broken configs — each rejected by name |
| 5b | Confirm the blank template is rejected |
| 6 | Preflight: verify the schemas exist. **Creates nothing** |
| 6b | Create the eight metadata tables |
| 6b2 | Create the shared quarantine table |
| 7 | Dry run — parse and preview the SQL, write nothing |
| 8 | Load metadata for both pipelines |
| 9 | Re-load; metadata row counts must not double |
| 10–12 | Run both pipelines: landing → bronze, then bronze → silver |
| 13 | Inspect quarantine and the run audit |
| 14 | Re-run the data load; row counts must not double |

Phase 6 is worth pausing on: the framework **verifies** that `idee_metaschema`
and `idee_app` exist and refuses to create them. A mistyped schema name should
stop the run, not silently produce a second schema with tables scattered across
both.

### Step 4 — Hand over

Eight metadata tables at zero rows. **No application team repeats any of this.**

---

## 5. Role 2 — Onboarding a dataset

**Once per dataset. App folder only. No framework code is touched.**

### Step 1 — Copy the template

```
idee_dbx_framework/templates/idee_pipeline_config_template.yaml
        ↓  copy
idee_dbx_app/config/<pipeline_name>_config.yaml
```

Copy the **template**, never an existing filled config. A filled config carries
another dataset's `object_name` and table names, and those are easy to miss.

### Step 2 — Fill it out

Six sections:

| Section | Answers |
|---|---|
| `pipeline:` | What is this called and who owns it |
| `source:` | Where does the data come FROM |
| `target:` | Where does it go TO |
| `dataset:` | How should it be processed |
| `columns:` | Which columns, renamed and cast (explicit mode only) |
| `quality:` | What makes a record valid |

The one field that shapes everything else is `column_mode`:

| Mode | Behaviour | Typical hop |
|---|---|---|
| `star` | every source column passes through | landing → bronze |
| `explicit` | rename and cast; unlisted columns dropped | bronze → silver |
| `notebook` | reference a notebook holding complex SQL | silver → gold |

Every alternative for every option is already in the file, commented out and
labelled `IMPLEMENTED` or `NOT YET`. Scaling up is an uncomment, not a rewrite.

### Step 3 — Run the notebook

`idee_dbx_app/notebooks/run_sample_pipeline`

| Step | What happens |
|---|---|
| 1 | Create this team's bronze and silver tables |
| 2 | Validate the config — every error at once, each naming its field |
| 3 | Parse into metadata actions |
| 3b | Review the generated SQL before anything is written |
| 4 | Load the metadata |
| 5 | Confirm what landed |
| 6 | **Run the pipeline** — data moves |
| 7 | Look at the data, including the four framework-added columns |
| 8 | Check quarantine and the run audit |

Leave `dry_run = true` for the first pass. It reads, transforms and reports
without writing anything.

---

## 6. What actually happens when a pipeline runs

Five stages. The same five for every hop.

```
  resolve  →  read  →  transform  →  apply_dq  →  write
     │          │          │            │           │
     │          │          │            │           └─ overwrite (FULL)
     │          │          │            └─ warn / drop / quarantine / fail
     │          │          └─ select, rename, cast, trim, + ETL audit columns
     │          └─ cloud_files or delta_table
     └─ read the metadata for this pipeline_name
                                              (+ audit row before and after)
```

**Resolve** takes a pipeline *name*, not a file path. The engine reads the
metadata tables. That is what makes scheduling possible later — a scheduler
knows a name.

**Transform** owns the ETL audit columns (`_ingested_at`, `_run_id`,
`_pipeline_id`, `_source_file`), so they can never be added twice or forgotten.
In `star` mode it also *excludes* any it finds on the way in — otherwise a
table-to-table hop copies the upstream load timestamp forward and the target
claims it was loaded when the source was.

**DQ runs before the write**, because you cannot quarantine a row you have
already written. Every rule evaluates in one pass, and the strictest action a
row trips decides its fate: a row failing both a `warn` and a `fail` rule is
treated as `fail`.

**Write** is `overwrite` for `FULL` — atomic in Delta, so a concurrent reader
sees the old version or the new one, never an empty table.

**Audit** is written by the executor, not the components: a STARTED row before
reading, an outcome row after. One place, and it still works when a component
raises.

---

## 7. Why one engine covers every hop

Nothing in the config names a medallion layer. There is no `bronze_table`
field. A pipeline has a source and a target; which layer each represents lives
in the names you chose.

The two shipped configs are the proof:

| | landing → bronze | bronze → silver |
|---|---|---|
| `source_type` | `cloud_files` | `delta_table` |
| `column_mode` | `star` | `explicit` |
| Engine used | the same | the same |
| Code changed | none | none |

Silver → gold will be the same story: `delta_table` source, `notebook` column
mode, same executor.

---

## 8. What the demo looks like

Around fifteen minutes. Drive from Databricks, not from slides.

### Part 1 — as the framework engineer (5 min)

1. **Show the two folders.** Give the rule: *if another team would need it too,
   it goes in the framework.*
2. **Run the unit tests.** Land on 187 passing and say what they cover.
3. **Show the validator refusing bad input** (Phase 5). Six broken configs, six
   errors naming the exact field. This lands better than green dots, because it
   shows the framework protecting people who are not you.
4. **Show the blank template being rejected** (Phase 5b). Small detail,
   signals care.
5. **Preflight** (Phase 6): the framework verifies the schemas exist and
   refuses to create them.

### Part 2 — as the data engineer (7 min)

Say the switch out loud: *"I'm now a data engineer who has never seen that
code."*

6. **Open the filled config.** One file. Walk the six sections.
7. **Dry run.** Here is exactly what would be written, before anything is.
8. **Run it.** Data lands in bronze. Point at the four columns nobody
   configured.
9. **Show quarantine.** The sample CSV has planted defects, so rules visibly
   fire rather than everything passing silently.
10. **Run the second hop.** Bronze → silver, `delta_table` source, `explicit`
    mode — *same engine, only the config differs.*

### Part 3 — the closer (3 min)

11. **Run it again.** Row counts unchanged. Explain idempotency in one
    sentence: MERGE for metadata, overwrite for data, deterministic IDs
    throughout. Most metadata frameworks quietly double their rows on a re-run.
12. **State the scope plainly** (see below).

### The sentence to lead with

> To onboard a dataset, a data engineer fills out one YAML file. No new
> pipeline, no new notebook, no code change.

---

## 9. Scope — say this out loud

**Where the framework starts.** It starts at **landing**. Files are assumed to
already be in the landing volume, put there by a vendor push, an existing job,
or Lakeflow Connect. Getting them there is outside the framework — there is no
source→landing step.

**What runs today:**

| Area | Built | Not yet |
|---|---|---|
| Sources | `cloud_files`, `delta_table` | `jdbc`, `kafka`, `api`, `lakeflow_connect` |
| Targets | `delta_table` | `s3`, `adls`, `jdbc` |
| Load types | `FULL` | `INCREMENTAL`, `CDC`, `SNAPSHOT` |
| Column modes | `star`, `explicit` | `notebook` |
| DQ engines | `inline` | `native`, `great_expectations` |
| Layers | landing → bronze → silver | silver → gold |

Everything unbuilt raises a specific error naming what is missing, and is
already present in the config commented out. Selecting one validates with a
warning rather than failing halfway through a run.

**Also deferred:** asset bundles and CI/CD, waiting on Bitbucket repos and AD
groups; scheduling, which is recorded in metadata but not yet wired to
Databricks.

Naming these yourself is what makes the rest credible. Ojoma will find the
boundary in ninety seconds either way.

---

## 10. Adding the next dataset

1. Copy the template into `config/`.
2. Name the file after the pipeline:
   `<application>_<object>_<source>_to_<target>_config.yaml`.
3. Fill it in. New `pipeline_name` — never reuse one.
4. Add the target DDL to `ddl/`.
5. Validate, dry run, run.

No framework change. That is the whole point.
