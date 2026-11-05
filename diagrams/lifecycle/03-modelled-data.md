# Lifecycle 3. Modelled data

Raw rows are modelled into `src`, with a lineage row for every raw row behind each modelled row. `ref.vendor` and `ref.contract_vendor` (and through it `ref.contract`) fold in, and the registry now also identifies each lineage row's raw table. When modelling finishes, the run moves to `testing`. New tables are shown with columns; tables from earlier steps by name only.

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
  REF_RUN ||--o{ SRC_PERSON : "models"
  REF_RUN ||--o{ SRC_TIMESHEET : "models"
  REF_RUN ||--o{ SRC_CLAIM_FILE : "models"
  REF_VENDOR ||--o{ SRC_PERSON : "employs"
  SRC_PERSON ||--o{ SRC_TIMESHEET : "works"
  SRC_PERSON |o--o{ SRC_CLAIM_FILE : "claimed for"
  REF_CONTRACT_VENDOR ||--o{ SRC_TIMESHEET : "worked under"
  REF_CONTRACT_VENDOR ||--o{ SRC_CLAIM_FILE : "claimed under"
  SRC_PERSON ||--|{ SRC_PERSON_LINEAGE : "built from"
  SRC_TIMESHEET ||--|{ SRC_TIMESHEET_LINEAGE : "built from"
  SRC_CLAIM_FILE ||--|{ SRC_CLAIM_FILE_LINEAGE : "built from"
  REF_RUN ||--o{ SRC_PERSON_LINEAGE : "models"
  REF_RUN ||--o{ SRC_TIMESHEET_LINEAGE : "models"
  REF_RUN ||--o{ SRC_CLAIM_FILE_LINEAGE : "models"
  RAW_SOURCE ||..o{ SRC_PERSON_LINEAGE : "feeds"
  RAW_SOURCE ||..o{ SRC_TIMESHEET_LINEAGE : "feeds"
  RAW_SOURCE ||..o{ SRC_CLAIM_FILE_LINEAGE : "feeds"
  REF_SOURCE_TABLE_REGISTRY ||--o{ SRC_PERSON_LINEAGE : "raw table"
  REF_SOURCE_TABLE_REGISTRY ||--o{ SRC_TIMESHEET_LINEAGE : "raw table"
  REF_SOURCE_TABLE_REGISTRY ||--o{ SRC_CLAIM_FILE_LINEAGE : "raw table"

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
  RAW_SOURCE["raw.{source}"]
  SRC_PERSON["src.person"] {
    int person_id PK
    int run_id FK
    string person_key "natural key"
    int vendor_id FK "vendor they work for"
    string full_name
    string role
    date start_date
    date end_date
  }
  SRC_TIMESHEET["src.timesheet"] {
    int timesheet_id PK
    int run_id FK
    string timesheet_key "natural key"
    int person_id FK
    int contract_vendor_id FK
    date work_date
    decimal hours
    string approval_status
    string approved_by
  }
  SRC_CLAIM_FILE["src.claim_file"] {
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
  }
  SRC_PERSON_LINEAGE["src.person_lineage"] {
    int person_lineage_id PK
    int run_id FK
    int person_id FK "modelled row"
    int raw_table_id FK "raw table, from the registry"
    int raw_row_id "row in that raw table"
  }
  SRC_TIMESHEET_LINEAGE["src.timesheet_lineage"] {
    int timesheet_lineage_id PK
    int run_id FK
    int timesheet_id FK "modelled row"
    int raw_table_id FK "raw table, from the registry"
    int raw_row_id "row in that raw table"
  }
  SRC_CLAIM_FILE_LINEAGE["src.claim_file_lineage"] {
    int claim_file_lineage_id PK
    int run_id FK
    int claim_line_id FK "modelled row"
    int raw_table_id FK "raw table, from the registry"
    int raw_row_id "row in that raw table"
  }
```

**New at this step:** `src.person`, `src.timesheet`, `src.claim_file`, `src.person_lineage`, `src.timesheet_lineage`, `src.claim_file_lineage`.

**Run status:** `loading`, then `testing`.

[Previous: 2. Raw data loaded](02-raw-data.md) · [Next: 4. Testing](04-testing.md)

[Back to the README](../../README.md)
