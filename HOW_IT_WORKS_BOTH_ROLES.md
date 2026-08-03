# iDEE for Databricks — How It Works, From Both Sides

**For:** Ojoma Ali
**From:** Italo Duran & Yash Yagnik
**Framework version:** 0.1.0 · **Config schema version:** 0.1

---

## Why this document exists

You asked to see Databricks rather than a slide deck. This is the written
companion to that — the narration for what we will open on screen, so you can
follow the same path yourself afterwards without either of us on the call.

It walks the framework from the two perspectives that matter, because they are
genuinely different jobs:

- **Part 1 — the framework engineer.** Sets the framework up in an environment.
  Does this once. Never touches anyone's dataset.
- **Part 2 — the application engineer.** Uses the framework to onboard their
  data. Does this once per dataset. Never touches framework code.

Everything referenced here exists in the workspace under
`/Workspace/Users/<user>/idee_for_databricks`.

---

## Before either part: what is where

Two folders under one parent. The split is the thing everything else follows
from.

```
idee_for_databricks/
│
├── idee_dbx_framework/          built once · shared by every application team
│   ├── templates/               the blank YAML that users copy
│   ├── schema/                  versioned validation contract
│   ├── src/idee_dbx/            validator · parser · loader · CLI
│   ├── ddl/metadata/            the five metadata tables
│   ├── tests/                   95 PyTest cases + fixtures
│   ├── notebooks/               end-to-end validation
│   └── docs/                    guides · naming standard · changelog
│
└── idee_dbx_app/                one of these per application team
    ├── config/                  filled YAML — one per dataset
    ├── ddl/                     bronze · silver · quarantine
    ├── data/                    sample source files
    ├── notebooks/               how this team calls the framework
    ├── docs/                    guide for data engineers
    └── output/                  validation and parse output
```

### Reading the framework folder

Two different kinds of folder live inside the framework, and they look identical
from the outside. Knowing which is which makes the rest of the structure
obvious.

**`src/idee_dbx/` is a Python package.** The folder name is not decoration — it
is literally the import name. `import idee_dbx` works *because* the folder is
called `idee_dbx`. Rename it and every import in the project breaks. The `src/`
wrapper around it is a packaging convention: it exists so that once the
framework is published as a wheel, Python cannot accidentally import the working
copy instead of the installed one. Until then it is a container with one job.

**Everything else is a plain bucket of files.** `schema/`, `templates/`,
`ddl/`, `tests/`, `notebooks/`, `docs/` are ordinary folders whose names are
free choices. Renaming one only means updating the paths that point at it.

So the framework is really four groups:

| Group | Folder | What it is |
|---|---|---|
| The engine | `src/idee_dbx/` | the only code that runs — validator, parser, loader, CLI |
| What the engine reads | `schema/`, `templates/`, `ddl/` | contracts, the blank config, table definitions |
| What checks the engine | `tests/` | 95 PyTest cases + fixtures |
| What humans open | `notebooks/`, `docs/` | the demo, the guides |

One folder is the engine. Everything else is fuel, tooling, or documentation.

### Two schemas

A different namespace from the folders, and not to be confused with them:

| Schema | Holds | Written by |
|---|---|---|
| `idee_metaschema` | metadata tables only | the framework |
| `idee_app` | bronze, silver, quarantine | the application team's pipelines |

The rule for deciding where a new thing goes: **if another application team
would also need it, it belongs in the framework folder.** If it only makes sense
for this team's data, it belongs in the app folder.

---

# Part 1 — The framework engineer

**Job:** stand the framework up in an environment.
**Frequency:** once per environment.
**Touches:** the framework folder only.

The application folder is not part of this. It belongs to whoever consumes the
framework, and a framework engineer setting things up has no reason to open it.

## Step 1 — Copy the framework folder into the workspace

Only `idee_dbx_framework`. Nothing else is required to stand the framework up.

## Step 2 — Set the paths and the catalog

One variable block at the top of the validation notebook:

```python
PROJECT_ROOT     = "/Workspace/Users/<your.email>@its.jnj.com/idee_for_databricks"
FRAMEWORK        = f"{PROJECT_ROOT}/idee_dbx_framework"
APP              = f"{PROJECT_ROOT}/idee_dbx_app"

CATALOG          = "medtech_md_scion_dev"
METADATA_SCHEMA  = "idee_metaschema"
APP_SCHEMA       = "idee_app"
```

That block is the only thing edited by hand, anywhere in setup. Catalog and
schema names are parameters everywhere else — in the Python, in the SQL, in the
notebooks. Nothing is hardcoded, so the same artifacts move to QA and prod
without a single edit.

## Step 3 — Run the end-to-end validation notebook

`idee_dbx_framework/notebooks/iDEE Framework End to End Validation`

This is the artifact to open on screen. It performs the rest of setup itself and
proves the framework is healthy while doing it. What each phase does:

| Phase | What happens | What you should see |
|---|---|---|
| 1 | Confirm folder layout | Framework path resolves |
| 2 | Install `pytest`, `pyyaml`, restart Python | Clean install |
| 3 | Run the unit tests | **95 passed**, exit code 0 |
| 5 | Feed the validator five deliberately broken configs | Five `FAILED` results, each naming the exact field |
| 5b | Validate the blank template | `FAILED` — an unfilled template must never pass |
| 6 | Preflight, then metadata DDL | Both schemas confirmed, five tables created |
| 7 | Dry run against the sample config | Ten planned actions, MERGE statements printed, **nothing written** |
| 8 | Live load | Metadata rows appear |
| 9 | Run the load again | **Counts unchanged** |

Three of those are worth pausing on.

**Phase 5 — the validator refusing bad input.** Tests passing is necessary but
not persuasive. Watching the validator reject an `INCREMENTAL` load with no
watermark column, a two-part table name, and a mismatched `source_id` — each
with a readable message naming the field — is what shows it will protect people
who are not us.

**Phase 6 — preflight, not creation.** The framework **verifies** that
`idee_metaschema` and `idee_app` exist. It does not create them. Those are
provisioned by the platform team. If someone mistypes a schema name, the run
stops with a clear message rather than silently creating a second schema and
scattering tables across two places.

**Phase 9 — idempotency.** The load runs a second time and the row counts do not
move: still 1 / 1 / 5 / 3. The parser emits `MERGE`, not `INSERT`, and derives
deterministic rule IDs rather than random UUIDs. A framework built the other way
would quietly double its metadata every re-run, and nobody would notice until an
audit. This matters more once configs are in CI/CD and re-runs are routine.

## Step 4 — Confirm and hand over

`99_verify_metadata_tables.sql` returns five tables at zero rows. The framework
is ready, and **no application team repeats any of the above.**

---

# Part 2 — The application engineer

**Job:** onboard a dataset.
**Frequency:** once per dataset.
**Touches:** the app folder only.

No framework code is copied, edited, or forked. The app notebook imports
`idee_dbx` from the framework folder and calls it.

## Step 1 — Copy the template

```
idee_dbx_framework/templates/idee_pipeline_config_template.yaml
        ↓  copy
idee_dbx_app/config/<object>_<load_type>_config.yaml
```

Copy the **template**, never another filled config. A filled config carries a
previous dataset's `object_id` and table names, and those are easy to miss and
land your data in someone else's table.

Once filled, it is no longer a template and does not carry `template` in its
name.

## Step 2 — Fill it out

Four sections, each mapping to one metadata table:

| Section | Describes | Produces |
|---|---|---|
| `source:` | where data comes from, how to connect | 1 row in `source_config` |
| `dataset:` | which dataset, how it loads | 1 row in `object_config` |
| `columns:` | what each column looks like after cleaning | 1 row per column |
| `quality:` | what makes a record valid | rules |

Every field is commented in the file with its meaning, allowed values, and
whether it is required. Three rules worth stating:

- **Do not rename keys.** Each YAML key is named exactly after its target table
  column, and the parser maps them with no translation layer. Renaming a key
  breaks the parse — deliberately, because it keeps the config honest.
- **No credentials in `connection_ref`.** It takes a Unity Catalog connection
  name or a secret scope reference.
- **IDs are permanent.** `source_id` and `object_id` are never reused. Audit
  history keys off them.

## Step 3 — Validate

```python
from idee_dbx.config_validator import validate_config_file
print(validate_config_file(CONFIG_PATH, schema_dir=f"{FRAMEWORK}/schema").report())
```

```
Config      : .../customer_full_refresh_config.yaml
Version     : 0.1
Schema      : v0.1
Result      : PASSED
```

The validator reports **every** problem at once rather than stopping at the
first, so one pass tells you everything to fix. It also checks things that are
easy to get wrong and hard to spot: that `dataset.source_id` matches
`source.source_id`, that column names and sequence numbers are unique, that
table names are three-part, and that `INCREMENTAL` has a watermark column.

## Step 4 — Create this team's tables

`idee_dbx_app/ddl/` — bronze, silver, quarantine — run against `idee_app`.

This DDL is application-specific and belongs to this team. Bronze deliberately
keeps source column names and STRING types; renaming and casting happen on the
way to Silver, driven by `column_config`.

## Step 5 — Dry run

`idee_dbx_app/notebooks/run_sample_pipeline` with dry run on.

```
Planned metadata actions (10 total):
  source_config          1 row
  object_config          1 row
  column_config          5 rows
  quality_rule_config    3 rows

Mode : DRY RUN (nothing written)
```

The counts should match the config — five `columns:` entries, three `rules:`
entries. If they do not, the config is not saying what its author thinks it
says. The MERGE statements print in full. This is the last checkpoint before
anything is written.

## Step 6 — Load

Dry run off, re-run. The dataset is now registered in `idee_metaschema`.

---

# What repeats, and what does not

This is the part most people get wrong when they first meet a metadata-driven
framework, so it is worth being explicit.

| Activity | How often | Who |
|---|---|---|
| Framework setup | once per environment | framework engineer |
| Onboarding a dataset | once per dataset | application engineer |
| Loading the data | every scheduled run | nobody — it is a job |

**Onboarding is not repeated to refresh data.** Filling out a config registers
the dataset. After that, a scheduled job reloads it against the same metadata,
and the config is never touched again. You would only revisit the config to
change something structural — a new column, a different load type, a new quality
rule.

That third row is design intent, not something we have built. See below.

---

# Scope — stated plainly

**What v0.1 does:** loads *metadata* about a pipeline.

**What v0.1 does not do:** move data. There is no ingestion engine yet.

We built the config path first and proved it end to end before putting anything
on top of it. That is deliberate, and it is the whole reason this is one
complete unit of work rather than several half-built ones.

Also not built, and why:

| Not built | Waiting on |
|---|---|
| Bronze/Silver ingestion | this skeleton being signed off |
| Scheduled loading | the ingestion engine |
| DQ runtime enforcement | the ingestion engine. All three DQ engines validate and parse correctly; nothing executes rules against records yet |
| Append, incremental, CDC at run time | full refresh proven first |
| Asset bundles, CI/CD | Bitbucket repos and AD groups |

The folder structure is repo-shaped already, so the move to Bitbucket should be
a copy rather than a redesign.

---

# Open questions

Carried forward, still worth your call:

1. **Parser output form.** Should the parser emit insert statements, structured
   objects, or write directly to the metadata tables? We built all three paths —
   `parse` returns a plan, `--show-sql` prints the DML, `load` writes. Which is
   the intended default?
2. **The metadata table set.** Four config tables plus one runtime audit table.
   Worth confirming before the ingestion engine is built on top of it.
3. **Logging.** You mentioned logs should live somewhere separate — not in the
   framework code, and probably not alongside the DDL and config either. We have
   an `output/` folder as a placeholder, but nothing writes to it yet and
   `pipeline_audit` is created but unpopulated. We would like your steer on the
   shape before building it.
4. **PyTest coverage expectations** once SonarQube is introduced.

---

# What to open on screen

If you want to follow this yourself, in this order:

1. `idee_dbx_framework/notebooks/iDEE Framework End to End Validation` — the
   framework proving itself. Phases 5, 8, and 9 are the ones worth watching.
2. `idee_dbx_app/config/customer_full_refresh_config.yaml` — what an
   application engineer actually fills out.
3. `idee_dbx_app/notebooks/run_sample_pipeline` — the same framework, called by
   a consuming team, against their own config and their own schema.

The contrast between 1 and 3 is the point: same three stages, different caller,
and no framework code copied into the app folder to make it work.
