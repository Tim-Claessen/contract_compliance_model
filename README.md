# Contract compliance model

## Purpose

A data model for testing vendor compliance against contracts on Azure SQL, with Power BI reporting. A client may hold several contracts, each with several vendors, and every vendor on every contract can need its own tests. The model starts with three sources (vendor claim files, timesheets and people) and is designed to take new sources and tests without redesign.

It is adapted from the [CCM design](https://github.com/Tim-Claessen/ccm_design), with the process and risk ontologies replaced by contracts and vendors.

## How it works

Each execution cycle is a **run**, and one run covers one period across every contract. A run moves through five stages:

```
Raw data → Modelled data → Testing → Validation → Archive → Report
                                         ↑                     |
                                         └─ Next run inherits standing validation rules from ref
```

1. **Raw data.** Whatever the client gives us is loaded as delivered into `raw`.
2. **Modelled data.** Raw data is cleaned, typed and keyed into the modelled `src` tables. This is the layer tests read from.
3. **Testing.** Test logic runs against `src` for each vendor on each contract in scope, and writes findings to `results`.
4. **Validation.** Every finding gets a disposition in `validation`. Where a standing rule in `ref` matches, the disposition is pre-filled. Reviewers accept it or change it.
5. **Archive.** Once the run is finalised, everything in `raw`, `src`, `results` and `validation` is appended to `archive`, and those schemas are cleared for the next run.

Each stage's procedure moves `ref.run.status` on when it finishes: modelling sets `testing`, testing sets `validating`, and the archive step sets `finalised`. Power BI then reports across every run.

## Diagrams

Mermaid diagrams live in [`diagrams/`](diagrams/), one Markdown file each, so they render on GitHub and in most Markdown viewers.

| Diagram | Shows |
| --- | --- |
| [00 Lifecycle](diagrams/00-lifecycle.md) | How data flows through the five stages and which schema each sits in |
| [01 Data model](diagrams/01-data-model.md) | Every table in every schema, with all relationships |
| [02 Reference](diagrams/02-reference.md) | The `ref` schema: contracts, vendors, tests, categories, rules, runs |
| [03 Raw data](diagrams/03-raw-data.md) | Stage 1: the shape every raw table shares |
| [04 Modelled data](diagrams/04-modelled-data.md) | Stage 2: `src.person`, `src.timesheet`, `src.claim_file` |
| [05 Testing](diagrams/05-testing.md) | Stage 3: how tests are scoped and what they produce |
| [06 Validation](diagrams/06-validation.md) | Stage 4: per-run dispositions and carry-forward |
| [07 Archive](diagrams/07-archive.md) | Stage 5: the archive copies and the reporting views |

Mermaid ER diagrams cannot group tables into schema boxes, so each table is labelled with its schema (`src.timesheet`). The lifecycle diagram shows the schemas as boxes.

---

## Schema architecture

Six schemas separate data by lifecycle and access control.

| Schema | Holds | Lifecycle |
| --- | --- | --- |
| `ref` | Contracts, vendors, test definitions and scope, finding categories, standing validation rules, source registry, run log | Persists across runs |
| `raw` | Client data as delivered | Current run only |
| `src` | Modelled source data | Current run only |
| `results` | Calculated findings | Current run only |
| `validation` | Validated findings | Current run only |
| `archive` | Append-only copies of `raw`, `src`, `results` and `validation`, plus reporting views | Grows over time |

Only one run is in flight at a time, because one run covers every contract.

### `ref`: reference and configuration (persists across runs)

Everything that defines _what_ the platform does and _who_ it is done for. Controlled access: only leads should modify it.

**Changes are new rows, not edits.** Archived results point at `ref` rows, so changing what a row means rewrites history. Anything whose meaning changes over time (`contract_vendor`, `test_scope`, `validation_rule`) carries effective dates. To change it, end-date the old row and add a new one. Cosmetic fixes, such as correcting a vendor's name, can be edited in place.

### `raw`: raw source (current run only)

Raw data is whatever the client gives us: claim files, timesheet exports, HR extracts, in whatever shape they arrive. It is loaded into `raw` as delivered, with every column kept as text, so a bad value never stops a load. Each raw table adds three load columns: `run_id`, `source_file` and `loaded_at`.

`raw` is not the focus of this model. The diagrams show it only as a generic shape, because its tables mirror each client's files and change whenever the files do. Its job is to prove what we received. Files that sit upstream of the database (in blob storage or a shared drive) follow their own retention process. Anything loaded into the database is archived.

### `src`: modelled source (current run only)

The main focus. Raw data is cleaned, typed, de-duplicated and keyed into modelled tables that tests read from. Each row carries `run_id` and the client's natural key. Mapping from `raw` to `src` is done by load procedures, one per source.

### `results`: calculated results (current run only)

What the tests found. `result_summary` is generic, with one row per finding. Each test also writes a `result_detail_{test}` table with its own columns for drill-through. Results are calculated only: reviewers never edit them.

### `validation`: validated results (current run only)

One `result_validation` row per finding, holding the final category, disposition and explanation. Keeping reviewer judgement here, apart from `results`, means a run's results can be recalculated without losing review work, and the two are easy to tell apart in an audit.

### `archive`: history (append-only)

Holds a copy of every `raw`, `src`, `results` and `validation` table, named `{schema}_{table}` (for example `archive.src_timesheet`). Each has the same columns as its live table plus `archived_at`. A separate schema with its own permissions: the team can read it, and only the archive step can write to it, so nobody clears history by mistake. Its foreign key columns are not enforced, because the live rows they pointed at are cleared.

### Keys

Every table has a surrogate primary key, separate from the natural key, and the natural key is kept. Surrogate keys come from a SQL Server `SEQUENCE` per table, not `IDENTITY`. `TRUNCATE` resets an identity column, so identity keys would repeat across runs and collide in the archive. Sequence keys are unique across all runs, so archive tables keep a single-column primary key, and Power BI relationships stay one column wide.

### Reporting

Power BI reads views, never the live tables directly. Each view is the live table `UNION ALL` its archive copy (for example `archive.v_result_summary`), so reports show the in-flight run alongside every finalised one and don't go blank when the live schemas are cleared. Refresh Power BI after a run is finalised, not during the archive step.

### Run completion process

The archive step runs in one transaction, rolls back on any error, and refuses to run twice.

```sql
SET XACT_ABORT ON;  -- any error, including THROW, rolls back the whole transaction
BEGIN TRANSACTION;

-- 1. Guard: the run is validating, every finding has a disposition, and it isn't archived yet
IF EXISTS (SELECT 1 FROM ref.run WITH (UPDLOCK) WHERE run_id = @run_id AND status <> 'validating')
   OR EXISTS (SELECT 1 FROM results.result_summary s
              WHERE s.run_id = @run_id
                AND NOT EXISTS (SELECT 1 FROM validation.result_validation v
                                WHERE v.result_id = s.result_id AND v.disposition IS NOT NULL))
   OR EXISTS (SELECT 1 FROM archive.results_result_summary WHERE run_id = @run_id)
    THROW 50001, 'Run is not ready to archive, or is already archived.', 1;

-- 2. Append each live table to its archive copy, with an explicit column list
INSERT INTO archive.src_timesheet (timesheet_id, run_id, timesheet_key, person_id, contract_vendor_id,
                                   work_date, hours, approval_status, approved_by, archived_at)
SELECT timesheet_id, run_id, timesheet_key, person_id, contract_vendor_id,
       work_date, hours, approval_status, approved_by, SYSUTCDATETIME()
FROM src.timesheet
WHERE run_id = @run_id;

-- 3. Repeat for every table in ref.source_table_registry
--    (raw, src, results and validation, including each result_detail_{test})

-- 4. Finalise the run
UPDATE ref.run SET status = 'finalised' WHERE run_id = @run_id;

-- 5. Clear the live tables, children before parents
DELETE FROM validation.result_validation;
DELETE FROM results.result_detail_claim_without_timesheet;  -- each detail table
DELETE FROM results.result_summary;
DELETE FROM src.timesheet;
DELETE FROM src.claim_file;
DELETE FROM src.person;
-- and each raw table

COMMIT;
```

The live tables use `DELETE`, not `TRUNCATE`, because SQL Server won't truncate a table that a foreign key points at, and the live tables keep their foreign keys. Step 3 is a loop over the registry with dynamic SQL, so a new table or test only needs registering to be archived.

List columns explicitly rather than `SELECT *`, so a column added to a live table fails loudly instead of landing in the wrong archive column. When a table changes, change its archive copy in the same release.

---

## Entity groups

### 1. Execution (`ref` schema)

| Entity | Schema | Role |
| --- | --- | --- |
| `run` | `ref` | Registry of all runs, one per period across every contract. Status drives the lifecycle: `loading` → `testing` → `validating` → `finalised`, enforced by a check constraint. A filtered unique index (on status, where status is not `finalised`) stops a second run starting while one is in flight. |

### 2. Contracts and vendors (`ref` schema)

| Entity | Schema | Role |
| --- | --- | --- |
| `contract` | `ref` | One row per contract under test. `contract_number` is the natural key. |
| `vendor` | `ref` | One row per vendor. `vendor_number` is the natural key. |
| `contract_vendor` | `ref` | A vendor engaged on a contract, with start and end dates. This is the unit that tests, results and validation rules are scoped to. Most vendors sit on one contract, but a vendor can sit on several. |

**Contract terms are not modelled yet.** Rate cards, approved roles, hour caps and contract values would sit in `ref` against `contract_vendor`, with effective dates, once a test needs them.

### 3. Source data

| Entity | Schema | Role |
| --- | --- | --- |
| `source_table_registry` | `ref` | Catalogue of every per-run table (`schema_name` is `raw`, `src`, `results` or `validation`). Register a table before loading it, and a test's detail table when the test is added. The archive step uses it to know what to copy. |
| `raw.{source}` | `raw` / `archive` | Client data as delivered. One table per file type. |
| `person` | `src` / `archive` | A vendor's worker. Grain: one row per person per run. `person_key` is the client's natural key. `vendor_id` is the vendor they work for. |
| `timesheet` | `src` / `archive` | Hours worked. Grain: one row per `person_id`, `contract_vendor_id` and `work_date`. |
| `claim_file` | `src` / `archive` | What the vendor claimed. Grain: one row per claim line, with claim header fields (`claim_number`, `submitted_date`) repeated on each line. `amount` is as submitted, not recalculated. |

Each `src` table is unique on `run_id` + its natural key.

**Claim to timesheet.** Claim lines match timesheets on `person_id` + `contract_vendor_id` + `work_date` within a run. Surrogate IDs only join within a run. Across runs, use `person_key`. More than one claim line can land on one timesheet day, and that is often a finding in itself. A claim line naming someone not in `src.person` is kept with a null `person_id`, so the claim still reaches testing.

**Person and vendor.** A person belongs to one vendor, but their timesheets and claim lines carry a `contract_vendor_id`. The two can disagree, and that is a test (a person claimed under a vendor they don't work for), not a constraint.

### 4. Test definitions (`ref` schema)

| Entity | Schema | Role |
| --- | --- | --- |
| `test_definition` | `ref` | What each test checks, its logic reference, its detail table and its owner. |
| `test_source_map` | `ref` | Bridge table. A test can read from several source tables. |
| `test_scope` | `ref` | Which tests run for which vendor on which contract, with active dates. A test runs in a run when its scope is active at the run's `period_end`. Each pairing is chosen separately, so tests can overlap across contracts or differ by vendor. |

### 5. Results (`results` schema)

| Entity | Schema | Role |
| --- | --- | --- |
| `result_summary` | `results` / `archive` | Generic. One row per finding per run, linked to the test and the vendor on the contract. `category_id` is set when test logic can auto-tag the finding and is null otherwise. Power BI dashboards report from this layer. |
| `result_detail_{test}` | `results` / `archive` | One table per test with test-specific columns. Links to `result_summary` via `result_id` and carries `run_id`. Power BI drill-through lands here. Not validated. |
| `finding_category` | `ref` | Taxonomy of finding types. Grows organically: the first findings arrive uncategorised, reviewers create categories, and later runs auto-tag where logic permits. |

### 6. Validation

| Entity | Schema | Role |
| --- | --- | --- |
| `validation_rule` | `ref` | Standing rule: for test X, on vendor-contract Y, with category Z, the default disposition and explanation. Has `effective_from` and `effective_to`. Only one rule per test + vendor-contract + category can be in effect on any date (checked when a rule is added). |
| `result_validation` | `validation` / `archive` | One row per finding per run: final category, disposition, explanation, reviewer, and the rule it came from (null when a reviewer set it by hand). Category and disposition stay null until set by a rule or a reviewer. Disposition values are fixed by a check constraint, with the list agreed with the team. |

**Carry-forward logic.** When a run moves to `validating`, every finding gets a `result_validation` row. The system looks up `validation_rule` by the finding's `test_id` + `contract_vendor_id` + `category_id`, effective at the run's `period_end`. A match pre-fills the category, disposition and explanation, and records `validation_rule_id`. Reviewers accept it or change it.

Findings with no category never match a rule, so a new test's first run is reviewed entirely by hand. Rules build up from there.

**Simplicity for the team.** Validation attaches to the summary layer, not to individual detail tables. The team validates findings at one level, and detail is there for drill-through only.

## Naming conventions

### Schemas

`ref`, `raw`, `src`, `results`, `validation`, `archive`.

### Source tables: `raw.{source}` and `src.{entity}`

| Layer | Examples |
| --- | --- |
| Raw | `raw.claim_file`, `raw.timesheet`, `raw.person` |
| Modelled | `src.claim_file`, `src.timesheet`, `src.person` |

### Result detail tables: `results.result_detail_{test}`

Examples: `results.result_detail_claim_without_timesheet`, `results.result_detail_claimed_hours_exceed_timesheet`, `results.result_detail_person_wrong_vendor`.

### Archive tables: `archive.{schema}_{table}`

Examples: `archive.raw_timesheet`, `archive.src_timesheet`, `archive.results_result_summary`, `archive.validation_result_validation`. Reporting views: `archive.v_{table}`.

### General rules

- All lowercase, underscores as separators.
- Entity and test names should be self-documenting in SSMS.
- Every table has a surrogate primary key from a sequence, separate from the natural key. Keep the natural key too.
