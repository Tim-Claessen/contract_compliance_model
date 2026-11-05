# 06 Validation

Stage 4. When a run moves to `validating`, every finding gets one `result_validation` row. If a `validation_rule` matches the finding's test, vendor on contract and auto-tagged category, effective at the run's `period_end`, it pre-fills the category, disposition and explanation. Reviewers accept or change each one.

```mermaid
erDiagram
  RES_RESULT_SUMMARY ||--o| VAL_RESULT_VALIDATION : "validated by"
  REF_VALIDATION_RULE |o--o{ VAL_RESULT_VALIDATION : "pre-fills"
  REF_FINDING_CATEGORY |o--o{ VAL_RESULT_VALIDATION : "classified as"
  REF_RUN ||--o{ VAL_RESULT_VALIDATION : "reviews"
  REF_TEST_DEFINITION ||--o{ REF_VALIDATION_RULE : "for test"
  REF_CONTRACT_VENDOR ||--o{ REF_VALIDATION_RULE : "for vendor on contract"
  REF_FINDING_CATEGORY ||--o{ REF_VALIDATION_RULE : "for category"

  RES_RESULT_SUMMARY["results.result_summary"]
  VAL_RESULT_VALIDATION["validation.result_validation"] {
    int result_validation_id PK
    int run_id FK
    int result_id FK "one per finding"
    int validation_rule_id FK "null = set by hand"
    int category_id FK "final category, null until reviewed"
    string disposition
    string explanation
    string reviewed_by
    datetime reviewed_at
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
  REF_FINDING_CATEGORY["ref.finding_category"]
  REF_RUN["ref.run"]
  REF_TEST_DEFINITION["ref.test_definition"]
  REF_CONTRACT_VENDOR["ref.contract_vendor"]
```

A finding with no category never matches a rule, so a new test's first run is reviewed by hand. `validation_rule_id` records where a disposition came from: null means a reviewer set it.

[Back to the README](../README.md)
