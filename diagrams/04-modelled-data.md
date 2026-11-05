# 04 Modelled data

Stage 2. Raw data is cleaned, typed and keyed into the modelled `src` tables that tests read from. Every row carries `run_id` and the client's natural key, and every modelled row is linked to the raw rows it was built from. Solid lines are enforced foreign keys. Dashed lines are logical links that are not enforced: the raw end of the lineage, and everything in `archive`.

```mermaid
erDiagram
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
  REF_CONTRACT ||--o{ REF_CONTRACT_VENDOR : "engages"

  REF_RUN["ref.run"]
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
  REF_VENDOR["ref.vendor"]
  REF_CONTRACT_VENDOR["ref.contract_vendor"]
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
  RAW_SOURCE["raw.{source}"]
  REF_SOURCE_TABLE_REGISTRY["ref.source_table_registry"]
  REF_CONTRACT["ref.contract"]
```

| Table | Grain |
| --- | --- |
| `src.person` | One row per person per run |
| `src.timesheet` | One row per `person_id`, `contract_vendor_id` and `work_date` |
| `src.claim_file` | One row per claim line; header fields repeat on each line |
| `src.{entity}_lineage` | One row per modelled row and raw row it was built from |

Each table is unique on `run_id` + its natural key. Claim lines match timesheets on `person_id` + `contract_vendor_id` + `work_date` within a run. A claim line naming someone not in `src.person` is kept with a null `person_id`, because that is a finding, not a load error. `src.person.vendor_id` and a timesheet's `contract_vendor_id` can disagree; testing for that is a finding, not a constraint.

**Lineage.** Modelling isn't always one raw row to one modelled row: duplicates merge, and one raw row can feed several tables (a timesheet export can also supply people). So each `src` table has a lineage bridge with one row per raw row behind each modelled row. `raw_table_id` says which raw table, from the registry, and `raw_row_id` says which row. The link to the raw row isn't an enforced key, because one column points into whichever raw table `raw_table_id` names; `raw_row_id` still identifies exactly one raw row, because all raw tables share one sequence.

[Back to the README](../README.md)
