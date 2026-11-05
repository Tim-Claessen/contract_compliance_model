# 03 Raw data

Stage 1. Client data is loaded into `raw` as delivered, every column as text. Raw tables mirror each client's files, so only their shared shape is shown: one table per file type (`raw.claim_file`, `raw.timesheet`, `raw.person`), each with the same load columns.

```mermaid
erDiagram
  REF_RUN ||--o{ RAW_SOURCE : "loads"

  REF_RUN["ref.run"] {
    int run_id PK
    date period_start
    date period_end
    date run_date
    string status "loading | testing | validating | finalised"
  }
  RAW_SOURCE["raw.{source}"] {
    int raw_row_id PK
    int run_id FK
    string source_file
    datetime loaded_at
    string source_columns "as delivered, all text"
  }
```

Each raw table is registered in `ref.source_table_registry` with `schema_name = raw`, which is how the archive step knows to copy it.

[Back to the README](../README.md)
