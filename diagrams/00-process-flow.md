# 00 Process flow

How a run moves data through the five stages. Each box is a schema. `ref` persists across runs; `raw`, `src`, `results` and `validation` hold the current run only and are cleared once it is archived. For the data model growing stage by stage, see [lifecycle/](lifecycle/).

```mermaid
flowchart LR
  files[/"Client files and extracts"/]

  subgraph REF["ref (persists across runs)"]
    direction TB
    r_run["run"]
    r_cv["contract, vendor, contract_vendor"]
    r_test["test_definition, test_scope, test_source_map"]
    r_val["finding_category, validation_rule"]
    r_reg["source_table_registry"]
  end

  subgraph RAW["1. Raw data: raw"]
    raw_t["raw.claim_file, raw.timesheet, raw.person"]
  end

  subgraph SRC["2. Modelled data: src"]
    src_t["src.claim_file, src.timesheet, src.person"]
    src_l["src.{entity}_lineage"]
  end

  subgraph RES["3. Testing: results"]
    res_t["result_summary, result_detail_{test}"]
  end

  subgraph VAL["4. Validation: validation"]
    val_t["result_validation"]
  end

  subgraph ARC["5. Archive: archive"]
    arc_t["archive.raw_*, src_*, results_*, validation_*"]
    arc_v["archive.v_* reporting views"]
  end

  pbi(["Power BI"])

  files -->|load as delivered| raw_t
  raw_t -->|clean, type, key| src_t
  src_l -.->|links each modelled row to its raw rows| raw_t
  src_t --- src_l
  src_t -->|run tests in scope| res_t
  res_t -.->|detail rows link to the src rows behind each finding| src_t
  res_t -->|pre-fill and review| val_t
  raw_t & src_t & res_t & val_t -->|append when finalised, then clear| arc_t
  arc_t --> arc_v
  arc_v --> pbi

  r_reg -.->|registers tables| raw_t
  r_cv -.->|scopes| src_t
  r_test -.->|which tests, for whom| res_t
  r_val -.->|standing rules| val_t
  r_run -.->|run_id on every row| arc_t
```

| Stage | Reads | Writes | Run status |
| --- | --- | --- | --- |
| 1. Raw data | Client files | `raw` | `loading` |
| 2. Modelled data | `raw`, `ref` | `src`, including lineage | `loading` |
| 3. Testing | `src`, `ref.test_scope` | `results` | `testing` |
| 4. Validation | `results`, `ref.validation_rule` | `validation` | `validating` |
| 5. Archive | `raw`, `src`, `results`, `validation` | `archive` | `finalised` |

The reporting views union the live tables with their archive copies, so Power BI sees the in-flight run and every finalised one.

[Back to the README](../README.md)
