# 05 Testing

Stage 3. For each row in `ref.test_scope` active at the run's `period_end`, the test's logic reads its `src` tables for that vendor on that contract. It writes one `result_summary` row per finding, and one or more rows in its `result_detail_{test}` table that point at the `src` rows behind the finding.

```mermaid
erDiagram
  REF_RUN ||--o{ RES_RESULT_SUMMARY : "produces"
  REF_TEST_DEFINITION ||--o{ RES_RESULT_SUMMARY : "generates"
  REF_CONTRACT_VENDOR ||--o{ RES_RESULT_SUMMARY : "found on"
  REF_FINDING_CATEGORY |o--o{ RES_RESULT_SUMMARY : "auto-tagged as"
  RES_RESULT_SUMMARY ||--|{ RES_RESULT_DETAIL : "evidenced by"
  REF_RUN ||--o{ RES_RESULT_DETAIL : "produces"
  SRC_CLAIM_FILE |o--o{ RES_RESULT_DETAIL : "evidence"
  SRC_TIMESHEET |o--o{ RES_RESULT_DETAIL : "evidence"
  SRC_PERSON |o--o{ RES_RESULT_DETAIL : "evidence"
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
    string description
  }
  REF_TEST_DEFINITION["ref.test_definition"]
  REF_CONTRACT_VENDOR["ref.contract_vendor"]
  REF_FINDING_CATEGORY["ref.finding_category"]
  RES_RESULT_DETAIL["results.result_detail_{test}"] {
    int detail_id PK
    int run_id FK
    int result_id FK
    int claim_line_id FK "if the test reads claims"
    int timesheet_id FK "if the test reads timesheets"
    int person_id FK "if the test reads people"
    string test_columns "test-specific columns"
  }
  SRC_CLAIM_FILE["src.claim_file"]
  SRC_TIMESHEET["src.timesheet"]
  SRC_PERSON["src.person"]
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

Every finding has at least one detail row. Each detail table has a foreign key column for every `src` table its test reads: a claim-versus-timesheet test carries `claim_line_id` and `timesheet_id`, and a test that only reads claims carries `claim_line_id`. A finding about two duplicate claim lines has two detail rows. A detail table carries only the keys for the tables its test reads; the three shown are the full set for the current sources.

Results are calculated only. `category_id` is set when the test logic can auto-tag a finding and left null otherwise. Reviewers never edit `results`; their decisions go to `validation`.

[Back to the README](../README.md)
