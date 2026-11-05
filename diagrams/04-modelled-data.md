# 04 Modelled data

Stage 2. Raw data is cleaned, typed and keyed into the modelled `src` tables that tests read from. Every row carries `run_id` and the client's natural key.

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
  REF_CONTRACT["ref.contract"]
```

| Table | Grain |
| --- | --- |
| `src.person` | One row per person per run |
| `src.timesheet` | One row per `person_id`, `contract_vendor_id` and `work_date` |
| `src.claim_file` | One row per claim line; header fields repeat on each line |

Each table is unique on `run_id` + its natural key. Claim lines match timesheets on `person_id` + `contract_vendor_id` + `work_date` within a run. A claim line naming someone not in `src.person` is kept with a null `person_id`, because that is a finding, not a load error. `src.person.vendor_id` and a timesheet's `contract_vendor_id` can disagree; testing for that is a finding, not a constraint.

Each `src` table is registered in `ref.source_table_registry`, which catalogues tables rather than rows, so it has no key to these tables.

[Back to the README](../README.md)
