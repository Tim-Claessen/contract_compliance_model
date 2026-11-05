# Lifecycle: the data model, stage by stage

An additive view of the data model. Each diagram shows everything in use at that point in a run. The `ref` tables are all there from the start, unused, and fold in as each stage starts reading them. Per-run tables appear at the stage that populates them, shown with columns at that step and by name only after it.

Solid lines are enforced foreign keys. Dashed lines are logical links that are not enforced: the raw end of the lineage, and everything in `archive`.

| Step | New tables | Run status |
| --- | --- | --- |
| [1. Run created](01-run-created.md) | no new tables, just a new `ref.run` row | `loading` |
| [2. Raw data loaded](02-raw-data.md) | `raw.{source}` | `loading` |
| [3. Modelled data](03-modelled-data.md) | `src.person`, `src.timesheet`, `src.claim_file`, `src.person_lineage`, `src.timesheet_lineage`, `src.claim_file_lineage` | `loading`, then `testing` |
| [4. Testing](04-testing.md) | `results.result_summary`, `results.result_detail_{test}` | `testing`, then `validating` |
| [5. Validation](05-validation.md) | `validation.result_validation` | `validating` |
| [6. Archive](06-archive.md) | `archive.raw_{source}`, `archive.src_person`, `archive.src_timesheet`, `archive.src_claim_file`, `archive.src_{entity}_lineage`, `archive.results_result_summary`, `archive.results_result_detail_{test}`, `archive.validation_result_validation` | `finalised` |

For the five stages one at a time, without the build-up, see the stage diagrams in [diagrams/](../).

[Back to the README](../../README.md)
