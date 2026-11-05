# Lifecycle 2. Raw data loaded

Client files land in `raw`, one table per file type, every row stamped with `run_id`. `ref.run` and `ref.source_table_registry` (which lists the raw tables to load) are the only `ref` tables in use. New tables are shown with columns; tables from earlier steps by name only.

```mermaid
erDiagram
  REF_CONTRACT ||--o{ REF_CONTRACT_VENDOR : "engages"
  REF_VENDOR ||--o{ REF_CONTRACT_VENDOR : "engaged on"
  REF_TEST_DEFINITION ||--o{ REF_TEST_SOURCE_MAP : "reads"
  REF_SOURCE_TABLE_REGISTRY ||--o{ REF_TEST_SOURCE_MAP : "read by"
  REF_TEST_DEFINITION ||--o{ REF_TEST_SCOPE : "scoped to"
  REF_CONTRACT_VENDOR ||--o{ REF_TEST_SCOPE : "tested by"
  REF_TEST_DEFINITION ||--o{ REF_VALIDATION_RULE : "for test"
  REF_CONTRACT_VENDOR ||--o{ REF_VALIDATION_RULE : "for vendor on contract"
  REF_FINDING_CATEGORY ||--o{ REF_VALIDATION_RULE : "for category"
  REF_RUN ||--o{ RAW_SOURCE : "loads"

  REF_RUN["ref.run"]
  REF_CONTRACT["ref.contract"]
  REF_VENDOR["ref.vendor"]
  REF_CONTRACT_VENDOR["ref.contract_vendor"]
  REF_SOURCE_TABLE_REGISTRY["ref.source_table_registry"]
  REF_TEST_DEFINITION["ref.test_definition"]
  REF_TEST_SOURCE_MAP["ref.test_source_map"]
  REF_TEST_SCOPE["ref.test_scope"]
  REF_FINDING_CATEGORY["ref.finding_category"]
  REF_VALIDATION_RULE["ref.validation_rule"]
  RAW_SOURCE["raw.{source}"] {
    int raw_row_id PK "one sequence across all raw tables"
    int run_id FK
    string source_file
    datetime loaded_at
    string source_columns "as delivered, all text"
  }
```

**New at this step:** `raw.{source}`.

**Run status:** `loading`.

[Previous: 1. Run created](01-run-created.md) · [Next: 3. Modelled data](03-modelled-data.md)

[Back to the README](../../README.md)
