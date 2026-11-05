# Risk & controls analytics platform — data model

## Purpose

A modular, process-agnostic data model for automated risk and controls testing on Azure SQL, with Power BI reporting. Designed to start with Accounts Payable and scale to other finance processes.

## How it works

Each execution cycle is a **run**. A run loads source data into `active`, executes test logic, and produces findings. Reviewers classify and validate findings. Once complete, active data is pushed to `archive` and the active schema is cleared for the next run. Validated dispositions carry forward automatically.

```
Load source → Execute tests → Classify → Validate → Archive → Report
     ↑                                                           |
     └──── Next run inherits validations from ref ───────────────┘
```

---

## Schema architecture

Three schemas enforce separation by lifecycle and access control.

### `ref` — reference / configuration (persists across runs)

Contains everything that defines _what_ the platform does: ontologies, test definitions, validations, finding categories, source table registry, and the run log itself. Controlled access — only leads should modify.

**Persists between runs:**

- Process and risk ontologies
- Test definitions and their mappings (risk, process, source)
- Finding categories
- Validations and overrides
- Source table registry
- Run log

### `active` — working data (current run only)

Contains the source snapshots, result summaries, and result detail tables for the run currently in progress. The team works here. Cleared after each run is completed and archived.

**Present only during a run:**

- Source snapshot tables (`src_{process}_{entity}`)
- Result summary
- Result detail tables (`result_detail_{process}_{test}`)

### `archive` — historical data (append-only)

Identical table structures to `active`. Receives data when a run is completed. Kept seperate to avoid accidental deletion.

**Grows over time:**

- All completed source snapshots
- All completed result summaries
- All completed result detail tables

### Run completion process

```sql
-- 1. Insert active data into archive
INSERT INTO archive.result_summary
SELECT * FROM active.result_summary WHERE run_id = @run_id;

-- 2. Repeat for source snapshots and each result detail table

-- 3. Update run status
UPDATE ref.run SET status = 'complete' WHERE run_id = @run_id;

-- 4. Truncate active tables
TRUNCATE TABLE active.result_summary;
-- etc.
```

## Entity groups

### 1. Execution (`ref` schema)

| Entity | Schema | Role                                                                                                                                                   |
| ------ | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `RUN`  | `ref`  | Registry of all runs. Status drives the lifecycle: `in_progress` → `validated` → `complete`. Lives in `ref` because it spans active and archived runs. |

### 2. Source data

| Entity                   | Schema               | Role                                                                                                                                              |
| ------------------------ | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SOURCE_TABLE_REGISTRY`  | `ref`                | Metadata catalogue of all source tables across processes. Each entry maps to a L3 process. When adding a new process, register tables here first. |
| `SOURCE_SNAPSHOT`        | `active` / `archive` | One row per source record per run. Keyed by `run_id` + `source_table_id` + `record_key`. Rolling window — each run loads the relevant period.     |
| `src_{process}_{entity}` | `active` / `archive` | The actual source data tables. One per entity per process. See naming conventions below.                                                          |

### 3. Ontologies

| Entity    | Schema | Role                                                                                                                                 |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `PROCESS` | `ref`  | Self-referencing three-level hierarchy (L1 → L2 → L3). Uses `parent_process_id` + `level` (1–3). Source tables and tests map to L3.  |
| `RISK`    | `ref`  | Self-referencing three-level hierarchy (L1 → L2 → L3). Uses `parent_risk_id` + `level` (1–3). Tests map to risks via a bridge table. |

### 4. Test definitions (`ref` schema)

| Entity             | Schema | Role                                                                                                                            |
| ------------------ | ------ | ------------------------------------------------------------------------------------------------------------------------------- |
| `TEST_DEFINITION`  | `ref`  | What each test checks, its logic reference, primary process, and owner.                                                         |
| `TEST_RISK_MAP`    | `ref`  | Bridge table. A test can map to multiple risks (e.g. a segregation of duties test maps to both fraud risk and compliance risk). |
| `TEST_PROCESS_MAP` | `ref`  | Bridge table. A test can touch multiple processes (e.g. PO-invoice match spans AP and Procurement).                             |
| `TEST_SOURCE_MAP`  | `ref`  | Bridge table. A test can read from multiple source tables.                                                                      |

**Primary process vs. mapped processes:** `primary_process_id` on `TEST_DEFINITION` determines which process "owns" the test for reporting and avoids double-counting. The bridge table captures all processes a test touches for traceability.

### 5. Results & findings

| Entity                           | Schema               | Role                                                                                                                                                                                                               |
| -------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `RESULT_SUMMARY`                 | `active` / `archive` | Process-agnostic. One row per finding per run. Links to test and (optionally) finding category. The `explanation` field is pre-populated from the matching validation. Power BI dashboards report from this layer. |
| `result_detail_{process}_{test}` | `active` / `archive` | One table per test with test-specific columns. Links to `RESULT_SUMMARY` via `result_id`. Power BI drill-through lands here. Does not participate in validation — validation happens at the summary level only.    |
| `FINDING_CATEGORY`               | `ref`                | Taxonomy of finding types. Grows organically: first run findings arrive uncategorised, reviewers create categories, subsequent runs auto-tag where logic permits.                                                  |

### 6. Validation (`ref` schema)

| Entity                | Schema | Role                                                                                                                                                 |
| --------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `VALIDATION`          | `ref`  | Standing rule: for test X + category Y, the default disposition and explanation. Carries forward across runs. Has `effective_from` for auditability. |
| `VALIDATION_OVERRIDE` | `ref`  | Run-level exception. Overrides the standing validation for a specific run. References both the validation and the run.                               |

**Carry-forward logic:** When a new run produces findings, the system looks up `VALIDATION` by `test_id` + `category_id`. If a match exists, the finding's `explanation` in `RESULT_SUMMARY` is pre-populated. Reviewers accept or override per run.

**Simplicity for the team:** Validation attaches to the summary layer, not to individual detail tables. The team validates findings at one level, and detail is there for drill-through only.

## Naming conventions

### Source tables: `src_{process}_{entity}`

| Process          | Examples                                                                         |
| ---------------- | -------------------------------------------------------------------------------- |
| Accounts Payable | `src_ap_invoices`, `src_ap_payments`, `src_ap_vendors`, `src_ap_purchase_orders` |
| Procurement      | `src_procurement_pos`, `src_procurement_contracts`, `src_procurement_suppliers`  |
| Payroll          | `src_payroll_employees`, `src_payroll_pay_runs`, `src_payroll_deductions`        |

### Result detail tables: `result_detail_{process}_{test}`

| Process          | Examples                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------- |
| Accounts Payable | `result_detail_ap_duplicate_invoice`, `result_detail_ap_po_match`, `result_detail_ap_payment_without_receipt` |
| Payroll          | `result_detail_payroll_ghost_employee`, `result_detail_payroll_overtime_threshold`                            |

### General rules

- All lowercase, underscores as separators.
- Process abbreviations: `ap`, `proc`, `payroll`, `gl`, etc. Keep consistent once chosen.
- Entity/test names should be self-documenting in SSMS.
- All tables require a primary key - this should be seperate from a natural key (if a natural key exists, this could be bifurcated as the PK but should be maintained)
