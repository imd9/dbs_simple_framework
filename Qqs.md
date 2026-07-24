# Metadata-Driven Lakehouse ELT Framework
## Installation, Validation, Onboarding, and User Experience

## Purpose

This document explains how a new employee, developer, platform administrator, operations engineer, or manager would install, configure, test, validate, and use the metadata-driven Databricks Lakehouse ELT framework.

The framework should not be presented as a ZIP file containing unrelated notebooks. It should be treated as a source-controlled Databricks product that is:

- stored in Bitbucket;
- developed locally or in Databricks;
- deployed into a Databricks workspace;
- governed through Unity Catalog;
- configured through metadata;
- tested through automated validation and smoke tests;
- promoted through pull requests and CI/CD.

The framework supports:

```text
Source systems
    ↓
Optional landing or direct ingestion
    ↓
Bronze
    ↓
Silver
    ↓
Quarantine, when required
    ↓
Gold
    ↓
Power BI, SQL, ML, applications, and AI
```

---

# 1. Where is the framework installed?

The complete framework exists across three locations:

```text
┌──────────────────────────────────────────────────────┐
│ 1. BITBUCKET                                        │
│ Permanent source of truth                           │
│ Code, SQL, YAML, tests, templates, documentation    │
└──────────────────────────┬───────────────────────────┘
                           │ clone
                           ▼
┌──────────────────────────────────────────────────────┐
│ 2. LOCAL DEVELOPMENT ENVIRONMENT                    │
│ VS Code, Git, Databricks CLI, optional .venv         │
│ Used to edit, test, validate, and deploy             │
└──────────────────────────┬───────────────────────────┘
                           │ bundle deploy
                           ▼
┌──────────────────────────────────────────────────────┐
│ 3. DATABRICKS WORKSPACE                             │
│ Lakeflow Jobs, Lakeflow Pipelines, metadata tables,  │
│ Bronze, Silver, Gold, audit, and monitoring          │
└──────────────────────────────────────────────────────┘
```

| Component | Location |
|---|---|
| Source code | Bitbucket |
| Local working copy | VS Code or another IDE |
| Deployment definitions | `databricks.yml` and resource YAML |
| Jobs and pipelines | Databricks workspace |
| Metadata tables | Unity Catalog |
| Bronze, Silver, Gold | Unity Catalog |
| Optional landing files | ADLS Gen2 or Unity Catalog Volumes |
| Secrets | Managed identity, Unity Catalog connections, Key Vault, or another approved mechanism |
| Tests | Repository and Databricks test jobs |
| Documentation | `README.md` and `docs/` |

VS Code is the development environment. Databricks is the runtime environment.

---

# 2. Prerequisites for a new employee

## Required access

```text
Bitbucket repository access
Databricks workspace access
Unity Catalog development permissions
Approved compute or serverless access
Permission to deploy or run development resources
Access to approved source systems or sample data
Access to approved secret references
```

A new developer should be able to deploy and test in development, but should not automatically receive production write access.

## Recommended local tools

```text
Git
VS Code
Python
Databricks CLI
Approved Databricks authentication method
Optional Python virtual environment
```

The `.venv` is only for local dependency isolation and testing. Lakeflow does not run inside the local virtual environment.

---

# 3. Recommended installation journey

```text
STEP 1  Clone the repository
STEP 2  Create and activate a local Python environment
STEP 3  Install development dependencies
STEP 4  Authenticate to Databricks
STEP 5  Validate the Databricks bundle
STEP 6  Deploy framework resources to development
STEP 7  Run the framework bootstrap workflow
STEP 8  Run the installation acceptance test
STEP 9  Review the results in Databricks
```

Example:

```bash
git clone <approved-bitbucket-repository>
cd metadata-lakehouse-elt-framework

python -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt

databricks auth login
databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run framework_bootstrap_job -t dev
databricks bundle run installation_acceptance_test -t dev
```

Windows PowerShell activation:

```powershell
.venv\Scripts\Activate.ps1
```

---

# 4. Is the framework a ZIP folder?

It should not normally be distributed as a ZIP file.

The recommended enterprise model is:

```text
Bitbucket repository
        ↓
Approved branch or tagged release
        ↓
Bundle validation
        ↓
Deployment to Databricks
        ↓
Bootstrap and acceptance testing
```

A ZIP may exist as a release artifact, but it should not replace source control.

A ZIP alone does not provide:

- change history;
- branches;
- pull requests;
- approvals;
- release tags;
- traceability;
- automated deployment;
- rollback visibility.

---

# 5. Does the user run one file?

The user should not manually run one enormous Python file.

The framework should provide controlled workflows.

## Deployment

```bash
databricks bundle deploy -t dev
```

This creates or updates Databricks resources such as:

- Lakeflow Jobs;
- Lakeflow Spark Declarative Pipelines;
- source files;
- environment-specific resource settings.

## Bootstrap

```bash
databricks bundle run framework_bootstrap_job -t dev
```

The bootstrap workflow should:

```text
1. Validate environment configuration
2. Confirm Unity Catalog access
3. Create catalogs and schemas when permitted
4. Create metadata tables
5. Create audit and quarantine tables
6. Seed baseline or sample metadata
7. Confirm required pipeline definitions
8. Run installation health checks
```

Deployment and bootstrap remain separate because resource deployment and database initialization are different responsibilities.

---

# 6. Connector setup

Users should select a provided connector template rather than rewrite connector code.

```text
templates/
└── sources/
    ├── autoloader_source.yml
    ├── jdbc_source.yml
    ├── lakeflow_connect_source.yml
    ├── kafka_source.yml
    └── api_source.yml
```

Example Auto Loader source:

```yaml
source:
  source_name: vendor_sales_files
  source_type: cloud_files
  ingestion_method: autoloader
  landing_required: true
  landing_path: /Volumes/dev_raw/vendor_sales
  file_format: csv
  active: true
```

Example JDBC source:

```yaml
source:
  source_name: oracle_finance
  source_type: jdbc
  ingestion_method: jdbc
  landing_required: false
  connection_ref: oracle-finance-connection
  active: true
```

## Typical responsibility model

| Activity | Typical owner |
|---|---|
| Request source-system account | Source owner |
| Configure ADLS access | Cloud or platform team |
| Configure managed identity | Azure or platform team |
| Create Unity Catalog connection | Databricks administrator or platform engineer |
| Add `source_config` | Data engineer |
| Add `object_config` | Data engineer |
| Add column definitions and quality rules | Data engineer |
| Approve PII access | Governance or security |
| Deploy production configuration | CI/CD service principal |

The framework makes onboarding repeatable, but it does not bypass enterprise access controls.

---

# 7. DDL setup

The framework should include ordered SQL scripts:

```text
sql/
├── bootstrap/
│   ├── 001_create_catalogs.sql
│   ├── 002_create_schemas.sql
│   ├── 003_create_metadata_tables.sql
│   ├── 004_create_audit_tables.sql
│   ├── 005_create_quarantine_schema.sql
│   └── 006_apply_dev_permissions.sql
│
├── migrations/
│   ├── V001__initial_metadata_model.sql
│   ├── V002__add_landing_required.sql
│   └── V003__add_sequence_column.sql
│
└── samples/
    └── seed_sample_customer_pipeline.sql
```

The order is:

```text
Create catalog
    ↓
Create schema
    ↓
Create metadata tables
    ↓
Create audit and quarantine objects
    ↓
Apply permissions
    ↓
Seed sample configuration
```

Use idempotent statements when practical:

```sql
CREATE SCHEMA IF NOT EXISTS dev_framework.config;
```

Production changes should use controlled migration scripts.

---

# 8. Templates included with the framework

```text
templates/
├── source_onboarding/
│   ├── file_source/
│   ├── jdbc_source/
│   ├── kafka_source/
│   └── api_source/
│
├── metadata/
│   ├── source_config_template.csv
│   ├── object_config_template.csv
│   ├── column_config_template.csv
│   ├── quality_rule_template.csv
│   └── gold_model_template.yml
│
├── gold/
│   ├── materialized_view_template.sql
│   ├── fact_table_template.sql
│   └── dimension_template.sql
│
└── tests/
    ├── unit_test_template.py
    ├── quality_test_template.py
    └── smoke_test_template.py
```

A new employee should begin from a template rather than a blank file.

---

# 9. Documentation strategy

A single giant README is not enough.

```text
README.md
docs/
├── 01_getting_started.md
├── 02_prerequisites.md
├── 03_installation.md
├── 04_connector_setup.md
├── 05_metadata_configuration.md
├── 06_source_onboarding.md
├── 07_testing_and_validation.md
├── 08_deployment.md
├── 09_operations_runbook.md
├── 10_troubleshooting.md
└── 11_user_experience.md
```

The root `README.md` should explain:

```text
What the framework is
Who should use it
Prerequisites
How to install it
How to run the sample
How to verify success
Where the detailed documentation lives
```

---

# 10. How installation is verified

Verification should occur at several levels.

## Level 1: Static validation

```bash
databricks bundle validate -t dev
pytest
ruff check .
```

Optional:

```bash
mypy src
```

## Level 2: Deployment verification

Confirm that:

```text
Lakeflow Job exists
Lakeflow Pipeline exists
Permissions are applied
Framework schemas exist
Metadata tables exist
Audit table exists
Quarantine schema exists
```

Example:

```text
dev_framework.config
dev_framework.audit
dev_bronze.sales
dev_silver.sales
dev_gold.sales
dev_quarantine.sales
```

## Level 3: Metadata health check

Validate that:

```text
At least one active source exists
At least one active object exists
Each active object has column definitions
Incremental objects have incremental columns
CDC objects have primary keys and sequence columns
Connector type is supported
Landing configuration is valid
Target names are valid
No duplicate target-column mappings exist
```

## Level 4: Smoke-test dataset

Example test file:

```csv
CUSTOMER_ID,CUSTOMER_NAME,EMAIL,CREATED_DATE
1001, John Smith ,john@example.com,2026-07-01
1002,Mary Jones,invalid-email,2026-07-02
1003,Alex Brown,alex@example.com,2026-07-03
```

Expected result:

```text
Bronze rows: 3
Silver rows: 2
Quarantine rows: 1
Job status: SUCCESS
```

## Level 5: Acceptance criteria

| Check | Expected result |
|---|---|
| Bundle validation | Passed |
| Deployment | Successful |
| Bootstrap | Successful |
| Metadata tables | Created |
| Bronze | 3 rows |
| Silver | 2 rows |
| Quarantine | 1 row |
| Audit | SUCCESS |
| Lineage | Bronze to Silver visible |
| Rerun | No unintended duplicates |
| Quality rules | Expected violations recorded |

---

# 11. Installation acceptance test

The framework should provide one explicit acceptance workflow:

```bash
databricks bundle run installation_acceptance_test -t dev
```

Expected output:

```text
[PASS] Workspace reachable
[PASS] Unity Catalog accessible
[PASS] Required schemas exist
[PASS] Metadata tables exist
[PASS] Sample source readable
[PASS] Bronze table populated
[PASS] Silver transformation correct
[PASS] Invalid row quarantined
[PASS] Audit record created
[PASS] Pipeline rerun is idempotent

Installation result: PASSED
```

---

# 12. Definition of a healthy framework

The framework is healthy when:

```text
Deployment resources match source control
Metadata validation passes
Scheduled jobs are enabled
Latest critical runs succeeded
Data freshness is within SLA
No critical quality rules failed
Audit counts reconcile
No unapproved schema drift exists
Expected lineage is visible
Production changes were deployed through CI/CD
```

---

# 13. User experience by persona

## Platform administrator

```text
Provision workspace and Unity Catalog
Configure identities and storage
Create approved connections
Apply permissions
Approve deployment targets
Monitor platform controls
```

## Framework developer

```text
Clone repository
Create feature branch
Modify reusable framework code
Run unit tests
Deploy to personal development target
Open pull request
Support framework releases
```

## Source-onboarding data engineer

```text
1. Select a source template
2. Complete source metadata
3. Complete object metadata
4. Add column mappings
5. Add quality rules
6. Run metadata validation
7. Deploy to development
8. Run sample ingestion
9. Review Bronze, Silver, quarantine, and audit
10. Submit a pull request
```

A helper command may generate a source package:

```bash
python tools/new_source.py --template jdbc --name oracle_customers
```

Generated structure:

```text
onboarding/oracle_customers/
├── source.yml
├── objects.yml
├── columns.yml
├── quality_rules.yml
├── README.md
└── tests/
```

## Data analyst or Power BI developer

```text
Browse approved Gold schemas
Read descriptions
Inspect lineage
Query certified datasets
Build reports
```

## Support or operations engineer

```text
Open Lakeflow Job
Review failed task
Inspect audit record
Identify failed object
Review quarantine records
Repair or rerun failed tasks
Confirm freshness recovery
```

---

# 14. Ideal source-onboarding experience

```text
Choose source type
        ↓
Complete metadata template
        ↓
Run validation
        ↓
Deploy to development
        ↓
Execute sample load
        ↓
Review Bronze, Silver, quarantine, audit, and lineage
        ↓
Submit pull request
        ↓
Promote through CI/CD
```

---

# 15. Recommended deployment model

## Development

```text
Developer
   ↓
Clone Bitbucket repository
   ↓
Create feature branch
   ↓
Configure source from templates
   ↓
Run local tests
   ↓
Validate bundle
   ↓
Deploy to development
   ↓
Run smoke or integration test
   ↓
Submit pull request
```

## Promotion

```text
Pull request approved
   ↓
CI validates and tests
   ↓
Deploy to test
   ↓
Run integration and acceptance tests
   ↓
Approval gate
   ↓
Deploy to production with service principal
```

Production should be deployed through automation, not through manual workspace edits.

---

# 16. Recommended directory

```text
metadata-lakehouse-elt-framework/
│
├── README.md
├── CONTRIBUTING.md
├── CHANGELOG.md
├── SECURITY.md
├── databricks.yml
├── pyproject.toml
├── requirements-dev.txt
├── .gitignore
│
├── resources/
│   ├── jobs/
│   │   ├── framework_bootstrap_job.yml
│   │   ├── metadata_framework_job.yml
│   │   ├── metadata_validation_job.yml
│   │   └── installation_acceptance_test.yml
│   └── pipelines/
│       ├── bronze_silver_pipeline.yml
│       └── gold_pipeline.yml
│
├── sql/
│   ├── bootstrap/
│   ├── migrations/
│   └── samples/
│
├── src/
│   ├── framework_core/
│   ├── ingestion/
│   ├── bronze/
│   ├── silver/
│   ├── gold/
│   ├── audit/
│   ├── notifications/
│   └── common/
│
├── templates/
│   ├── source_onboarding/
│   ├── metadata/
│   ├── gold/
│   └── tests/
│
├── tools/
│   ├── new_source.py
│   ├── validate_metadata.py
│   ├── run_smoke_test.py
│   └── inspect_installation.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── acceptance/
│   └── fixtures/
│
└── docs/
    ├── 01_getting_started.md
    ├── 02_prerequisites.md
    ├── 03_installation.md
    ├── 04_connector_setup.md
    ├── 05_metadata_configuration.md
    ├── 06_source_onboarding.md
    ├── 07_testing_and_validation.md
    ├── 08_deployment.md
    ├── 09_operations_runbook.md
    ├── 10_troubleshooting.md
    └── 11_user_experience.md
```

---

# 17. Summary

> This framework is not a folder of manually installed notebooks. It is a source-controlled Databricks product. A new employee clones the repository, authenticates to an approved development workspace, validates and deploys the project through a Databricks bundle, runs an idempotent bootstrap workflow, and executes an automated acceptance test.
>
> Source onboarding is template-driven. The engineer selects a connector type, enters source, object, column, and quality metadata, validates it, deploys to development, and reviews Bronze, Silver, quarantine, audit, and lineage results.
>
> Production promotion occurs through pull requests and CI/CD rather than direct manual editing. Complex business logic remains version-controlled in SQL or Python, while common ingestion, schema mapping, and quality behavior are metadata-driven.

---

# 18. Final position

The framework should be treated as:

```text
A source-controlled
metadata-driven
Databricks Lakehouse ELT product
with repeatable installation,
template-driven source onboarding,
automated testing,
governed promotion,
and clear operational visibility.
```

The goal is not only reusable pipelines. The goal is a repeatable product experience:

```text
Install
Configure
Validate
Deploy
Test
Operate
Troubleshoot
Promote
```
