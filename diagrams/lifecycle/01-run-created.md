# Lifecycle 1. Run created

A new row in `ref.run` with status `loading`. Everything else in `ref` is already set up from earlier runs (contracts, vendors, tests, scope, categories, rules), but no per-run table holds data yet, so the `ref` tables hang unused. New tables are shown with columns; tables from earlier steps by name only.

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

  REF_RUN["ref.run"] {
    int run_id PK
    date period_start
    date period_end
    date run_date
    string status "loading | testing | validating | finalised"
  }
  REF_CONTRACT["ref.contract"] {
    int contract_id PK
    string contract_number "natural key"
    string contract_name
    date start_date
    date end_date
    string status
  }
  REF_VENDOR["ref.vendor"] {
    int vendor_id PK
    string vendor_number "natural key"
    string vendor_name
    string abn
  }
  REF_CONTRACT_VENDOR["ref.contract_vendor"] {
    int contract_vendor_id PK
    int contract_id FK
    int vendor_id FK
    date start_date
    date end_date
  }
  REF_SOURCE_TABLE_REGISTRY["ref.source_table_registry"] {
    int source_table_id PK
    string schema_name "raw | src | results | validation"
    string table_name
    string description
  }
  REF_TEST_DEFINITION["ref.test_definition"] {
    int test_id PK
    string test_name
    string test_description
    string test_logic_ref "SP or script name"
    string detail_table "results.result_detail_{test}"
    string owner
  }
  REF_TEST_SOURCE_MAP["ref.test_source_map"] {
    int test_id FK
    int source_table_id FK
  }
  REF_TEST_SCOPE["ref.test_scope"] {
    int test_scope_id PK
    int test_id FK
    int contract_vendor_id FK
    date active_from
    date active_to
  }
  REF_FINDING_CATEGORY["ref.finding_category"] {
    int category_id PK
    string category_name
    string description
  }
  REF_VALIDATION_RULE["ref.validation_rule"] {
    int validation_rule_id PK
    int test_id FK
    int contract_vendor_id FK
    int category_id FK
    string disposition
    string explanation
    date effective_from
    date effective_to
  }
```

**New at this step:** no new tables, just a new `ref.run` row.

**Run status:** `loading`.

[Next: 2. Raw data loaded](02-raw-data.md)

[Back to the README](../../README.md)
