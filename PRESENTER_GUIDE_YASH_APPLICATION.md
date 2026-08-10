# Presenter Guide — Yash · The Application

**You present:** `idee_dbx_app/` — the configs, the DDL, the transform notebook.
**You go second.** Italo installs the framework and proves it works. You then
become a data engineer who has never seen that code, and onboard a dataset.
**Your time:** about 7 minutes.

---

## The one sentence to open with

> "I'm now a data engineer. I've never opened the framework folder and I'm not
> going to. I have a dataset to onboard, and one YAML file to fill out."

Say it deliberately. The role switch *is* the demo — it's what proves the
separation Italo just described is real rather than aspirational.

---

## Before the room joins — checklist

| Check | How |
|---|---|
| Widgets set | catalog, `metadata_schema`, `app_schema`, `config_file`, `dry_run` |
| Both CSVs in the Volume | `/Volumes/<catalog>/idee_app/landing/customer/incoming/` |
| The transform notebook's path matches your workspace | `customers_combined_landing_to_bronze_config.yaml` has `notebook_path` hardcoded to one user — **check this before you present** |
| Run all three configs once, alone | Especially notebook mode; it calls another notebook |
| Reset the app tables | So row counts start clean |

**One trap:** the notebook-mode config **cannot dry-run**. The notebook has to
execute before there's anything to inspect. Set `dry_run = false` for that one or
you'll get an error mid-demo. It's a clear error, but not the moment for it.

---

# Part 1 — Onboard a dataset from scratch (5 min)

**Notebook:** `idee_dbx_app/notebooks/run_sample_pipeline.py`

## Open with the folder

Before touching the notebook, show what a team actually owns:

```
idee_dbx_app/
├── config/      3 filled YAMLs — one per pipeline
├── ddl/         our bronze and silver tables
├── data/        sample source files
├── notebooks/   run_sample_pipeline + one custom transform
└── docs/
```

> "No framework code here. My notebook imports `idee_dbx` from Italo's folder
> and calls it. If I ever had to edit his code to onboard a dataset, the design
> would have failed."

## Step 0 — Where the config came from

> "I copied the template out of the framework's `templates/` folder. Important
> detail: copy the **template**, never another filled config — a filled one
> carries the previous dataset's table names, and that's easy to miss and lands
> your data in someone else's table."

## The config — walk the six sections

Open `customers_landing_to_bronze_config.yaml`. Don't read it line by line; make
these four points:

| Section | Say |
|---|---|
| `pipeline:` | name, description, owner email, team, tags — "this is who to call when it breaks at 2am" |
| `source:` | "where data comes from. Note `connection_ref` is a Unity Catalog connection name — never a credential." |
| `target:` | "where it goes. One source, one target, per pipeline." |
| `dataset:` | "how to process it. The one field that shapes everything is `column_mode`." |

Then the field that carries the design:

> "`column_mode` is one enum: `star`, `explicit`, or `notebook`. It replaced
> three separate booleans I originally wrote. Three booleans encode eight
> states — only three mean anything, and my own code had to silently pick a
> winner among the contradictions. An enum makes those broken states impossible
> to write."

That's a good moment: you're describing your own code being improved, which
lands better than defending it.

## Point at the commented-out options

> "Every alternative is still in the file, commented out, labelled IMPLEMENTED
> or NOT YET. Scaling to `INCREMENTAL` later is an uncomment, not a rewrite. And
> if I uncomment something unbuilt, validation still passes but warns me — I
> find out at validate time, not halfway through a 2am run."

## Step 2 — Validate

```
Config      : .../customers_landing_to_bronze_config.yaml
Version     : 0.1
Schema      : v0.1
Result      : PASSED
```

> "It reports every problem at once, each naming the exact field — so one pass
> tells me everything to fix instead of playing whack-a-mole."

**Worth breaking one live if you have time.** Change `load_type` to `INCREMENTAL`
without a watermark column and re-run the cell. The error names
`dataset.incremental_column`. Then put it back.

## Step 3 → 4 — Parse, review, load

```
Planned metadata actions (6 total):
  pipeline_config        1 row
  source_config          1 row
  target_config          1 row
  dataset_config         1 row
  column_config          0 rows
  quality_rule_config    2 rows
```

> "Zero column rows, because this is `star` mode — every source column passes
> through, so there's nothing to map."

Show the generated SQL before loading:

> "Last checkpoint before anything is written. I can read exactly what's about
> to happen."

## Step 6 — Run it

This is the payoff. Note how it's called:

```python
execute_pipeline(spark, catalog=CATALOG, pipeline_name=plan.pipeline_name, ...)
```

> "A pipeline **name**, not a file path. The engine reads the metadata I just
> loaded. That's what makes scheduling possible later — a scheduler knows a
> name, not a YAML."

## Step 7 — The four columns nobody configured

```sql
SELECT * FROM medtech_md_scion_dev.idee_app.bronze_customer
```

Point at `_ingested_at`, `_run_id`, `_pipeline_id`, `_source_file`:

> "I didn't configure these — the framework adds them. `_run_id` joins straight
> to `pipeline_audit`, so any row can be traced to the run that produced it."

## Step 8 — Quarantine and audit

> "The sample CSV has planted defects — a malformed email, a three-character
> country code where two are expected. That's on purpose: a demo where
> everything passes proves nothing."

Show `quarantine_records` — the failing row stored as JSON with the rule name
that rejected it — and this run's row in `pipeline_audit`.

## Now switch configs — the two moments that matter

**Config 2: `customers_bronze_to_silver_config.yaml`**

> "Different config, `explicit` mode. And look at the source: `source_type` is
> `delta_table`. It reads the table the first pipeline *wrote*. Same framework,
> same engine, identical code path — the only thing that changed is the YAML.
> That's the layer-agnostic claim Italo described, working."

Five columns renamed and cast. Then:

> "And `exclude_etl_columns: true`. Bronze already has `_ingested_at`. A plain
> `SELECT *` would copy it forward and Silver would claim it was loaded when
> *Bronze* was. The framework strips them on read and writes fresh ones."

Prove it:

```sql
SELECT MAX(_ingested_at) FROM medtech_md_scion_dev.idee_app.bronze_customer;
SELECT MAX(_ingested_at) FROM medtech_md_scion_dev.idee_app.silver_customer;
```

Silver is later. That's the audit trail staying honest.

**Config 3: `customers_combined_landing_to_bronze_config.yaml`** — your bit.

> "Two source files describing the same thing with different column names —
> `CUSTOMER_ID` in one, `CUST_ID` in the other. Standardising and unioning them
> is a join-and-union problem. Putting that in YAML would mean inventing a query
> language inside a config file. So the SQL goes in a notebook."

Set `dry_run = false`. Run it. Then open `transform_customer_combined.py`.

---

# Part 2 — Explain your parts (2 min)

## The notebook-mode contract — the part you discovered

This is your strongest technical moment. It's a real finding, not a design
preference:

> "The framework passes my notebook a `staging_table` parameter. I write my
> result there, and the framework picks it up — adds the audit columns, applies
> the DQ rules, writes the target, then drops the staging table.
>
> The obvious design is a temp view. It doesn't work. `dbutils.notebook.run`
> executes in a **separate Spark session** on serverless, so a temp view created
> inside my notebook is invisible to the caller. A real table is the only thing
> that crosses that boundary. That took real debugging to find, and there's now a
> comment in two places telling the next person not to 'simplify' it back."

## The three rules a custom notebook must follow

> "One: write to the `staging_table` you're given. Two: don't add the `_` audit
> columns — the framework owns those, and adding them here would duplicate them.
> Three: don't write to the target table yourself, because that would bypass the
> quality rules."

## Why it never falls back silently

Worth raising unprompted — it shows deliberate design:

> "We were asked whether the framework should fall back to the `columns:` block
> if my notebook produces nothing. We decided no. `explicit` and `notebook` don't
> produce *similar* data, they produce completely different datasets. A silent
> fallback would write the wrong shape while the audit log claimed the notebook
> ran. On a scheduled run that's days of bad data and metadata that lies.
>
> So it fails loudly — and if you genuinely want the fallback, you declare it:
> `on_empty_notebook: use_columns`. Then it's a choice someone made, not the
> framework guessing."

## Your DQ aliases

> "Rules are written against `{customer_id}`, not a literal column name. The
> engine resolves the alias to whichever candidate column actually exists. So the
> same rule works whether the source called it `CUSTOMER_ID` or `CUST_ID` — which
> matters a lot when you're unioning two systems. And if none of the candidates
> exist, it fails rather than substituting a missing column, because a rule
> evaluating against a column that isn't there looks like it passed."

## Your DDL

Two files: `01_create_bronze_customer.sql`, `02_create_silver_customer.sql`.

> "App DDL, in the app schema. Bronze keeps the source's uppercase column names
> and everything as STRING on purpose — casting happens in Silver, driven by
> config. Silver's columns mirror the config's `columns:` block exactly."

If asked why `bronze_customer_combined` has no DDL:

> "Notebook mode — the notebook decides the output shape, so the framework
> creates the table on first write. Writing DDL by hand would mean keeping two
> definitions of the same shape in sync."

## Be straight about scope

> "`FULL` loads only. `INCREMENTAL`, `CDC` and `SNAPSHOT` validate but aren't
> implemented — they're commented out in my configs with NOT YET labels. And no
> jobs or scheduling: `schedule` is recorded in metadata, nothing acts on it yet.
> The notebook runs the ETL; it doesn't create a Databricks Job."

---

# Likely questions

**"Why three config files? I thought it was one file?"**
One file per *pipeline*. Three pipelines, three files — each is one source to one
target. A user still fills out exactly one file for the thing they're onboarding.

**"Could you have done the union in the YAML?"**
Only by inventing a query language inside a config file. Joins, unions and window
functions belong in SQL. The config points at the SQL; it doesn't try to become
it.

**"What if I need a source type that doesn't exist?"**
That's a framework change, not mine — Italo registers a reader. Right now asking
for `jdbc` gives a clear error naming what's available, rather than failing
oddly.

**"How long does onboarding actually take?"**
Copy the template, fill in six sections, validate, dry run, run. The framework
part is minutes. The real time goes into knowing your data — which columns, which
types, which rules — and that's the part that should take thought.

**"What if I get the config wrong?"**
Validation catches it before anything runs and names the field. And if I'd chosen
something unbuilt, it warns me at validate time rather than failing mid-run.

**"Is the notebook's code stored in the metadata?"**
No — just the name and path. The notebook is versioned in Bitbucket. Storing the
content would mean two copies drifting apart.

---

# How the two halves come together

## The handoff you're receiving

Italo hands you a framework that's installed and proven: 217 tests green, eight
metadata tables created and empty, the validator demonstrably rejecting bad
input. Your first line acknowledges it and switches role:

> "Thanks — so that's installed. I'm now a data engineer who's never seen any of
> that code."

## What you use, and never touch

| You use | You never touch |
|---|---|
| The template in `templates/` | anything in `idee_dbx_framework/` |
| The 8 metadata tables | the validator or parser |
| `src/idee_dbx/` via import | the execution engine |
| The shared `quarantine_records` table | the metadata DDL |

Your notebook is about 260 lines, and every one of them is either a widget, an
import, or a call into the framework.

**Worth saying, because it's the honest history:**

> "My first version of this engine was 910 lines living inside *this* notebook.
> It worked — it ran on a cluster. But it was in the wrong folder: if another
> team adopted iDEE they'd have copied the app folder and got their own private
> fork of the engine. Two forks drift, and my bug fix never reaches them. So it
> moved into the framework, where it's now got tests around it."

That's a strong thing to say out loud. It shows the split is a real decision with
a cost, not decoration.

## The three things your half proves

1. **Layer-agnostic** — config 2's `source_type` is `delta_table`. Different hop,
   identical code, zero framework changes.
2. **All three column modes execute** — `star`, `explicit`, `notebook`, one
   config each.
3. **A user never touches framework code** — one YAML, press play.

## Your closing line

> "Adding the next dataset is: copy the template, fill it in, validate, run. No
> new pipeline, no new notebook, no framework change. That's the whole point."

Then hand back to Italo for the close.

## If you're asked what's next

- **Silver → Gold** via notebook mode. The engine already supports it; it needs a
  config, not code.
- **`INCREMENTAL`** and a merge writer — new components against the existing
  contract.
- **Jobs and scheduling**, and asset bundles once the Bitbucket repos exist.
- **And first**: this code hasn't run on a cluster before today. An earlier
  version did. Today is what closes that gap.
