# 07 Archive

Stage 5. When every finding has a disposition, the archive step appends each live table to its `archive` copy in one transaction, sets the run to `finalised` and clears the live tables. Archive copies have the live columns plus `archived_at`. Their `FK` columns are not enforced, because the live rows they pointed at are cleared. Sequence-based keys stay unique across runs, so archive keys never collide.

```mermaid
erDiagram
  REF_RUN ||--o{ ARC_RAW_SOURCE : "archived"
  REF_RUN ||--o{ ARC_SRC_PERSON : "archived"
  REF_RUN ||--o{ ARC_SRC_TIMESHEET : "archived"
  REF_RUN ||--o{ ARC_SRC_CLAIM_FILE : "archived"
  REF_RUN ||--o{ ARC_RES_RESULT_SUMMARY : "archived"
  REF_RUN ||--o{ ARC_RES_RESULT_DETAIL : "archived"
  REF_RUN ||--o{ ARC_VAL_RESULT_VALIDATION : "archived"
  ARC_RES_RESULT_SUMMARY ||--o{ ARC_RES_RESULT_DETAIL : "drill-through"
  ARC_RES_RESULT_SUMMARY ||--o| ARC_VAL_RESULT_VALIDATION : "validated by"

  REF_RUN["ref.run"]
  ARC_RAW_SOURCE["archive.raw_{source}"] {
    int raw_row_id PK
    int run_id FK
    string source_file
    datetime loaded_at
    string source_columns "as delivered, all text"
    datetime archived_at
  }
  ARC_SRC_PERSON["archive.src_person"] {
    int person_id PK
    int run_id FK
    string person_key "natural key"
    int vendor_id FK "vendor they work for"
    string full_name
    string role
    date start_date
    date end_date
    datetime archived_at
  }
  ARC_SRC_TIMESHEET["archive.src_timesheet"] {
    int timesheet_id PK
    int run_id FK
    string timesheet_key "natural key"
    int person_id FK
    int contract_vendor_id FK
    date work_date
    decimal hours
    string approval_status
    string approved_by
    datetime archived_at
  }
  ARC_SRC_CLAIM_FILE["archive.src_claim_file"] {
    int claim_line_id PK
    int run_id FK
    string claim_number
    string claim_line_key "natural key"
    int contract_vendor_id FK
    int person_id FK "null if not in src.person"
    date work_date
    decimal hours
    decimal rate
    decimal amount "as submitted"
    date submitted_date
    datetime archived_at
  }
  ARC_RES_RESULT_SUMMARY["archive.results_result_summary"] {
    int result_id PK
    int run_id FK
    int test_id FK
    int contract_vendor_id FK
    int category_id FK "auto-tag, nullable"
    string record_key "source row the finding is about"
    string description
    datetime archived_at
  }
  ARC_RES_RESULT_DETAIL["archive.results_result_detail_{test}"] {
    int detail_id PK
    int run_id FK
    int result_id FK
    string test_columns "test-specific columns"
    datetime archived_at
  }
  ARC_VAL_RESULT_VALIDATION["archive.validation_result_validation"] {
    int result_validation_id PK
    int run_id FK
    int result_id FK "one per finding"
    int validation_rule_id FK "null = set by hand"
    int category_id FK "final category, null until reviewed"
    string disposition
    string explanation
    string reviewed_by
    datetime reviewed_at
    datetime archived_at
  }
```

Power BI reads the reporting views, never the tables directly. Each view is the live table `UNION ALL` its archive copy:

| View | Unions |
| --- | --- |
| `archive.v_result_summary` | `results.result_summary` + `archive.results_result_summary` |
| `archive.v_result_detail_{test}` | `results.result_detail_{test}` + `archive.results_result_detail_{test}` |
| `archive.v_result_validation` | `validation.result_validation` + `archive.validation_result_validation` |
| `archive.v_{entity}` | `src.{entity}` + `archive.src_{entity}` |

The views sit in `archive` because they are mostly archive data, even though each also reads the live table.

[Back to the README](../README.md)
