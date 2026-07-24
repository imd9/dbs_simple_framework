# ELT Config Layer - Beginner Guide

## Why this guide exists
You asked for a beginner explanation of each Lego box in the Config/Metadata layer, plus clarification on:
- Why we would design DDLs if we do not know every user's source data
- What "YAML schema for metadata objects" means

This guide explains all of that in simple terms.

---

## 1) What does "Lego block" mean in a data framework?
A Lego block is a reusable component with one clear responsibility.

A component is a good Lego block when it is:
- Modular: it does one job
- Replaceable: can be swapped without rewriting everything
- Reusable: can be used by many teams/datasets
- Contract-based: accepts/produces a known shape of metadata

The goal is to avoid hardcoded, one-off pipelines.

---

## 2) The Config/Metadata layer flow in plain English
Think of the flow as:
1. Team requests onboarding for a new source/dataset
2. Team selects connector type
3. Team fills metadata objects
4. Framework validates metadata quality/completeness
5. Approved metadata is published as one canonical runtime spec
6. Runtime spec is stored and made available to the execution engine

This means config drives execution, not custom code per source.

---

## 3) Explain each Lego box (beginner version)

### Box A - Source Onboarding Request
What it is:
- The start of the process when someone wants to ingest data.

What it does:
- Captures basic request context (who, what data, business purpose).

Purpose:
- Creates a formal starting point instead of ad-hoc ingestion.

Why it is a Lego block:
- Reusable intake step for every new dataset.

---

### Box B - Choose Native Connector Type
What it is:
- Selection of how Databricks will access the source.

What it does:
- Identifies connector path (Auto Loader, JDBC, Lakeflow Connect, COPY INTO, etc.).

Purpose:
- Makes source access explicit and standardized.

Why it is a Lego block:
- Connector choice is independent of business logic and can be swapped.

---

### Box C - connector_profile
What it is:
- Metadata object describing connection method and auth references.

What it does:
- Stores connector type, endpoint/host reference, secret references, and connector options.

Purpose:
- Separates connectivity setup from dataset-specific settings.

Why it is a Lego block:
- Many datasets can reuse one connector profile.

---

### Box D - dataset_contract
What it is:
- Canonical metadata record for one dataset.

What it does:
- Declares source object, load type, target bronze destination, schema mode, active status.

Purpose:
- Provides one stable contract for runtime.

Why it is a Lego block:
- Everything downstream can run from this one standard shape.

---

### Box E - ingestion_policy
What it is:
- Metadata object for ingestion behavior.

What it does:
- Defines full/incremental/cdc behavior, watermark columns, checkpoint policy, retries.

Purpose:
- Keeps ingestion behavior configurable and consistent.

Why it is a Lego block:
- You can change load behavior without changing pipeline code.

---

### Box F - quality_profile
What it is:
- Metadata object for data quality rulesets.

What it does:
- Points to rule packs (for example iDEE/GE rules) and expected thresholds.

Purpose:
- Decouples rule authoring from pipeline mechanics.

Why it is a Lego block:
- DQ engine can evolve without rewriting ingestion metadata.

---

### Box G - transform_profile
What it is:
- Metadata object for downstream transformations.

What it does:
- Defines standard transforms or references to Silver/Gold logic profiles.

Purpose:
- Keeps transform behavior declarative where possible.

Why it is a Lego block:
- Reusable transform patterns for many datasets.

---

### Box H - governance_profile
What it is:
- Metadata object for ownership, compliance, and access controls.

What it does:
- Stores domain, owning team, steward, sensitivity/classification, approval status, SLA.

Purpose:
- Embeds governance into onboarding from day one.

Why it is a Lego block:
- Governance is reusable policy metadata, not hardcoded per pipeline.

---

### Box I - Validate Metadata Contract
What it is:
- Validation gate before activation.

What it does:
- Checks required fields, valid enums, reference integrity, and policy completeness.

Purpose:
- Prevents broken/incomplete metadata from reaching runtime.

Why it is a Lego block:
- Same checks apply to all datasets; one validation component serves all.

---

### Box J - Approved? (Decision)
What it is:
- Workflow gate.

What it does:
- Routes records to either correction path or publication path.

Purpose:
- Enforces quality and governance before execution.

Why it is a Lego block:
- Standard approval control usable across teams.

---

### Box K - Publish Canonical Dataset Spec
What it is:
- Final, normalized runtime metadata output.

What it does:
- Produces one canonical specification used by pipeline engine.

Purpose:
- Ensures runtime only reads one unified contract, regardless of input source format.

Why it is a Lego block:
- Adapters can change, but runtime contract remains stable.

---

### Box L - Register in Delta Metadata Registry
What it is:
- Durable storage of active metadata records in Delta tables.

What it does:
- Tracks current and historical versions of metadata objects.

Purpose:
- Supports auditing, discoverability, and reproducibility.

Why it is a Lego block:
- Standard metadata registry works for any project/domain.

---

### Box M - Export Runtime Spec for Pipeline Engine
What it is:
- Final handoff artifact from config layer to execution layer.

What it does:
- Delivers validated canonical spec to ingestion/transform jobs.

Purpose:
- Creates clear boundary: config layer designs intent, execution layer runs intent.

Why it is a Lego block:
- Reusable interface between control plane and data plane.

---

## 4) Native connector boxes (what they mean)

### Auto Loader for files
- Best for cloud file ingestion with incremental discovery and schema evolution.

### JDBC / SQL sources
- Pull from relational systems (Oracle, SQL Server, etc.) using query/table access.

### Lakeflow Connect sources
- Managed connectors for specific platforms where available in Databricks.

### COPY INTO / external locations
- Useful for file-driven ingestion patterns with controlled loads.

All of these are connection methods. They should be represented in metadata, not hardcoded.

---

## 5) Your DDL question (important)
Question:
"Wouldn't users draft DDLs? Why would we design DDLs if we do not know their source data?"

Correct answer:
- Yes, users know their source data best.
- But there are two different DDL categories:

Category A - Framework metadata DDL (we design this)
- These are tables like connector_profile, dataset_contract, governance_profile.
- They define how the framework stores control metadata.
- This is platform architecture and should be standardized by framework owners.

Category B - Source/business table DDL (users/domain teams own this)
- These are source schemas, business attributes, domain-specific targets.
- Teams provide these details through metadata records and contracts.

So we are not guessing user source schema.
We are standardizing the metadata control plane so all teams can onboard consistently.

---

## 6) What is "YAML schema for the 6 metadata objects"?
A YAML schema is a structured template and rule definition for config files.

It answers:
- Which fields are required
- Which values are allowed
- Data types for each field
- Validation rules and defaults

It is not source data itself.
It is a contract for how metadata must be written.

Simple example idea:
- connector_profile must include connector_type and auth_ref
- dataset_contract must include dataset_id, source_object, load_type, target_table
- governance_profile must include owner_team and classification

Why use YAML schema:
- Easier onboarding
- Better validation
- Less runtime failure
- More consistent metadata across teams

---

## 7) Practical split of responsibilities
Framework team owns:
- Metadata object design
- Metadata DDLs
- Validation logic
- Canonical runtime contract

Domain/user teams own:
- Their source mappings
- Their business attributes
- Their DQ/transform needs
- Their onboarding records in the framework metadata

---

## 8) One-line explanation you can reuse
"The Config/Metadata layer is the control plane: it standardizes how teams describe data sources and policies, validates that metadata, and publishes one canonical runtime spec that reusable pipeline Lego blocks execute."

---

## 9) Suggested next step (for implementation)
Start with only three metadata objects first (MVP):
1. connector_profile
2. dataset_contract
3. governance_profile

Then add:
4. ingestion_policy
5. quality_profile
6. transform_profile

This keeps phase 1 simple while preserving the full target architecture.
