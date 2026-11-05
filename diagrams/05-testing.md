# 05 Testing

Stage 3. For each row in `ref.test_scope` active at the run's `period_end`, the test's logic reads its `src` tables for that vendor on that contract and writes one `result_summary` row per finding, plus test-specific columns in its `result_detail_{test}` table.

```mermaid
erDiagram
  REF_RUN ||--o{ RES_RESULT_SUMMARY : "produces"
  REF_TEST_DEFINITION ||--o{ RES_RESULT_SUMMARY : "generates"
  REF_CONTRACT_VENDOR ||--o{ RES_RESULT_SUMMARY : "found on"
  REF_FINDING_CATEGORY |o--o{ RES_RESULT_SUMMARY : "auto-tagged as"
  RES_RESULT_SUMMARY ||--o{ RES_RESULT_DETAIL : "drill-through"
  REF_RUN ||--o{ RES_RESULT_DETAIL : "produces"
  REF_TEST_DEFINITION ||--o{ REF_TEST_SOURCE_MAP : "reads"
  REF_SOURCE_TABLE_REGISTRY ||--o{ REF_TEST_SOURCE_MAP : "read by"
  REF_TEST_DEFINITION ||--o{ REF_TEST_SCOPE : "scoped to"
  REF_CONTRACT_VENDOR ||--o{ REF_TEST_SCOPE : "tested by"

  REF_RUN["ref.run"]
  RES_RESULT_SUMMARY["results.result_summary"] {
    int result_id PK
    int run_id FK
    int test_id FK
    int contract_vendor_id FK
    int category_id FK "auto-tag, nullable"
    string record_key "source row the finding is about"
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
  REF_CONTRACT_VENDOR["ref.contract_vendor"]
  REF_FINDING_CATEGORY["ref.finding_category"]
  RES_RESULT_DETAIL["results.result_detail_{test}"] {
    int detail_id PK
    int run_id FK
    int result_id FK
    string test_columns "test-specific columns"
  }
  REF_TEST_SOURCE_MAP["ref.test_source_map"] {
    int test_id FK
    int source_table_id FK
  }
  REF_SOURCE_TABLE_REGISTRY["ref.source_table_registry"]
  REF_TEST_SCOPE["ref.test_scope"] {
    int test_scope_id PK
    int test_id FK
    int contract_vendor_id FK
    date active_from
    date active_to
  }
```

Results are calculated only. `category_id` is set when the test logic can auto-tag a finding and left null otherwise. Reviewers never edit `results`; their decisions go to `validation`.

[Back to the README](../README.md)
