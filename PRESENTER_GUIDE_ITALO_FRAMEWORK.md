# Presenter Guide — Italo · The Framework

**You present:** `idee_dbx_framework/` — the engine, the contract, the tests.
**You go first.** You install the framework and prove it works. Then you hand to
Yash, who becomes a data engineer using it.
**Your time:** about 8 minutes. Yash needs 7.

---

## The one sentence to open with

> "I'm going to install the framework and prove it works. Then Yash is going to
> forget he ever saw this code, and onboard a dataset as a user."

That framing *is* the demo. The separation between the two halves is the whole
architectural point, and saying it out loud at the start makes everything after
it land.

---

## Before the room joins — checklist

| Check | How |
|---|---|
| `PROJECT_ROOT` set in the notebook | It appears **twice** — once at the top, once after `restartPython()`, which clears every variable. Both must be right. |
| Both schemas exist | `idee_metaschema` and `idee_app`. The framework refuses to create them. |
| Volume has both CSVs | `/Volumes/<catalog>/idee_app/landing/customer/incoming/` |
| Run the whole notebook once, alone | Phase 3 takes a minute; you don't want to discover a path problem live. |
| Then reset | Drop the 8 metadata tables so Phase 6c genuinely shows zeros. |

**Stop at Phase 9.** Phases 10–14 run pipelines, which is Yash's section. Say
"the notebook also self-tests the execution engine, but I'll let Yash show that
part properly" and hand over.

---

# Part 1 — Install from scratch (5 min)

**Notebook:** `idee_dbx_framework/notebooks/run_framework_validation.py`

## Phase 1 — Copy one folder, set one variable

Open with what a new engineer actually does:

> "To install this framework, an engineer copies **one folder** and edits **one
> variable block**. That's it. Notice the app folder isn't involved — that
> belongs to the team consuming the framework, not to setting it up."

```python
PROJECT_ROOT    = "/Workspace/Users/<you>@its.jnj.com/idee_for_databricks"
CATALOG         = "medtech_md_scion_dev"
METADATA_SCHEMA = "idee_metaschema"
APP_SCHEMA      = "idee_app"
```

**The point to make:** nothing else in the framework has a hardcoded catalog or
schema. SQL uses `${catalog}`, Python takes arguments, notebooks use widgets. So
the same artifacts promote to QA and prod untouched — no copy-and-modify, which
is where environment drift starts.

## Phase 2 — Dependencies

`pytest` and `pyyaml`, then `restartPython()`. Mention that the restart clears
every variable, which is why the paths block appears twice. It's the most common
thing that trips people up.

## Phase 3 — Run 217 unit tests

This is your credibility moment. Land on green, then say what it covers rather
than letting a wall of dots speak for itself:

> "217 tests. They cover all three column modes, config versioning, required
> fields, type checking, enums, conditional rules, cross-field integrity,
> deterministic IDs, SQL rendering, and — this one matters — that the pieces of
> the project still agree with each other."

**If someone asks for a specific example, use this one:**

> "One test checks that `bool` is not accepted where an integer is expected. In
> Python `True` genuinely *is* an `int`, so a naive type check lets
> `sequence: true` through. That's the kind of thing you find by writing tests,
> not by reading."

## Phase 5 — Show it rejecting bad input

**Do not skip this. It's more persuasive than the green tests.**

Six configs broken on purpose. Six failures, each naming the exact field:

- illegal `load_type`
- illegal `column_mode`
- `INCREMENTAL` with no watermark column
- a two-part table name
- `primary_key` written as a string instead of a list
- both `target_table` and `target_path` supplied

> "Green tests prove it works for me. This proves it protects someone who isn't
> me. When another team adopts this, the validator is the only thing standing
> between a typo and a broken table."

## Phase 5b — The blank template is rejected

Small, and it signals care:

> "Every placeholder in the template — `"enter pipeline name here"` — is
> perfectly legal YAML. So the template *passed validation* when I first wrote
> the tests. Someone could have deployed an unfilled template. There's now a
> placeholder check and a test asserting the template must fail."

## Phase 6 — Preflight: verify, never create

> "The framework checks that `idee_metaschema` and `idee_app` exist. It refuses
> to create them. If someone mistypes a schema name, the run stops with a clear
> message instead of quietly creating a second schema and scattering tables
> across both. Failing loudly beats a silent partial success."

Then 6b creates the 8 metadata tables, 6b2 the shared quarantine table, 6c shows
8 tables at 0 rows.

## Phase 7 — Dry run

Validated, parsed, MERGE statements printed, **nothing written**.

> "Parse and load are separate stages on purpose. Parsing builds a plan without
> executing it — which is what makes a dry run mean something. You see exactly
> what would be written before anything is."

## Phase 8 → 9 — Load, then load again

Load metadata for all three pipelines. Then **re-run and show the counts don't
move.**

> "Same counts. The parser emits `MERGE`, not `INSERT`, and every ID is a
> deterministic hash rather than a UUID — so re-running updates the same rows.
> A framework built the other way doubles its metadata on every re-run, and
> nobody notices until an audit."

Hand over here.

---

# Part 2 — Explain the framework (3 min)

## The folder, in four groups

> "Seven folders, but really four ideas."

| Group | Folder | What it is |
|---|---|---|
| The engine | `src/idee_dbx/` | the only code that runs |
| What it reads | `schema/`, `templates/`, `ddl/` | the contract, the blank YAML, the tables |
| What checks it | `tests/` | 217 tests |
| What humans open | `notebooks/`, `docs/` | the demo, the guides |

If asked about `src/idee_dbx/`: the inner folder name **is** the Python import
name — `import idee_dbx` works because the folder is called that. `src/` is a
packaging convention so that once this ships as a wheel, Python can't
accidentally import the working copy instead of the installed one.

## The eight metadata tables

```
pipeline_config      one row per pipeline (= per YAML file)
source_config        where data is read FROM
target_config        where data is written TO
dataset_config       what to process and how
column_config        explicit column mapping
quality_rule_config  the DQ rules
config_parse_audit   which config was read, when, did it work
pipeline_audit       which data moved, when, how much, did it work
```

**On the two audit tables** — Ojoma asked for this specifically:

> "Two, at different grains. One records reading a *config*; the other records
> moving *data*. They fail for different reasons and get read by different
> people. Forcing them into one table would leave half the columns null on every
> row."

Be honest: `config_parse_audit` is created but nothing writes to it yet.

## The design decision worth the most airtime

**Nothing in the schema names a medallion layer.**

> "There is no `bronze_table` field. A pipeline has a source and a target. Which
> layer each represents lives only in the names the user chose. That's why one
> engine runs landing→bronze, bronze→silver, and silver→gold. If we'd baked the
> layers in, we'd need a code path per layer, and adding one would mean changing
> the framework."

Yash's section proves this — his second config reads from a *table* instead of
files, through identical code.

## Identity: why IDs aren't in the config

> "Users name things; the system identifies them. `pipeline_id` is a SHA-256
> hash of the pipeline name **plus the config file name** — both, deliberately.
> Copy a YAML to a new filename to test something and you get a *new* pipeline,
> not a silent overwrite of the original."

And the one that surprises people:

> "`source_id` hashes *where the source points*, not who uses it. So two
> pipelines reading the same S3 prefix automatically share one source row. That's
> what makes a source reusable — no registry file needed."

You can show this live:

```sql
SELECT source_id, COUNT(*) FROM medtech_md_scion_dev.idee_metaschema.source_config
GROUP BY source_id;
```

Fewer source rows than pipelines. That's the reuse working.

## The execution engine

```
resolve → read → transform → apply_dq → write     (+ audit around it)
```

Six components behind one interface in `contracts.py`. Two things worth saying:

> "`contracts.py` contains no logic — only the shapes. It exists because Yash
> and I were building components in parallel. Without an agreed interface, two
> correct components still don't fit, and you find out at integration time,
> which is the most expensive moment."

> "Components register themselves in a dictionary and the executor looks them up
> from the metadata. Adding a source type means registering a function — never
> editing the executor."

## Scope — say it before anyone asks

> "It starts at **landing**, not at source. Nothing here pulls from source
> systems; files are assumed already in the volume. `FULL` loads only —
> `INCREMENTAL`, `CDC` and `SNAPSHOT` validate but aren't implemented, and the
> engine says so rather than guessing. No jobs or scheduling: `schedule` is
> recorded in metadata and nothing acts on it yet."

Then the honest one:

> "And this specific code hasn't run on a cluster before today. An earlier
> version of the engine did, inline in a notebook — we restructured it into a
> tested package, which is what you're seeing. The tests cover the decision
> logic; the Spark calls are what today proves."

That last line is worth saying. Ojoma will find the boundary in ninety seconds
either way, and naming it yourself is what makes the rest credible.

---

# Likely questions

**"Why not just write a pipeline per dataset?"**
Fifty datasets means fifty notebooks to review, test and maintain, and dataset
fifty costs the same as dataset one. Here the marginal cost is a YAML file.

**"What stops someone breaking it with a bad config?"**
Phase 5. The validator reports every problem at once, each naming its field, and
refuses ambiguity rather than guessing.

**"How do you know a re-run is safe?"**
Phase 9, and Yash shows the data equivalent. MERGE plus deterministic IDs for
metadata; FULL overwrite for data.

**"What happens when you add a new source type?"**
Register a function in `SOURCE_READERS`. The executor doesn't change. Ask for one
that doesn't exist and you get `ComponentNotRegistered` naming it and listing
what is available.

**"Can another team use this?"**
That's why the engine is in the framework folder and not the app folder. They
copy the app folder pattern and get their own configs — not their own fork of
the engine.

**"Is this tied to SCION?"**
No. SCION is the first proof of concept. Nothing in the framework names it.

---

# How the two halves come together

## The handoff line

> "That's the framework installed and proven. Eight metadata tables, empty and
> ready. Yash — over to you as someone who's never seen any of this code."

## What you handed him

| You provided | He uses it |
|---|---|
| The blank template in `templates/` | copies it to make his config |
| The 8 metadata tables | his config becomes rows in them |
| `src/idee_dbx/` | his notebook imports and calls it |
| The shared `quarantine_records` table | his failing rows land there |
| The validator | catches his mistakes before anything runs |

## What he never touches

Any file under `idee_dbx_framework/`. His notebook is ~260 lines of *calling*
the framework. If he ever needed to edit framework code to onboard a dataset,
the design would have failed.

**The rule, if anyone asks where a new thing belongs:**

> "Would another team need it too? Then it's framework. Only makes sense for
> this team's data? Then it's app."

## The three claims his half proves

1. **Layer-agnostic** — his second config's `source_type` is `delta_table`, not
   `cloud_files`. Different hop, identical code.
2. **All three column modes work** — `star`, `explicit`, `notebook`, one config
   each.
3. **A user never touches framework code** — he fills out one YAML and presses
   play.

## Close together, after his part

Suggested closing line for you:

> "One YAML file per pipeline. The framework validates it, registers it, and
> runs it. Adding the fiftieth dataset costs the same as adding the second —
> and neither one requires touching this code."
