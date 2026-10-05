# Merkle Data Engineer Case Study

This repository contains the solution to the Merkle Data Engineer case study. It builds a three-layer Delta Lake pipeline in Databricks using PySpark, producing the `layer3.top_item` data mart for yearly item-view analysis.

## Architecture

```text
S3 CSV
  → Layer 1: raw Delta
  → Layer 2: cleaned / flattened Delta
  → Layer 3: top_item
```

- **Layer 1 — Raw:** `item.csv` and `event.csv` become `layer1.item` and `layer1.event`. Source attributes remain STRING, with no business transformations.
- **Layer 2 — Clean:** `layer2.item` and `layer2.event` use appropriate data types. The event JSON payload is parsed, and vertically stored parameters are pivoted into one logical row per `event_id`. The event table is partitioned by `event_year`.
- **Layer 3 — Mart:** `layer3.top_item` contains one row per item/year with views, including yearly view count, item rank, and most-used platform. The small mart is not physically partitioned.

## Source Data

The notebook reads two public S3 CSV files:

- `item.csv`: `s3a://merkle-de-interview-case-study/de/item.csv`
- `event.csv`: `s3a://merkle-de-interview-case-study/de/event.csv`

`event.csv` stores event parameters vertically, so several source rows may belong to one logical `event_id`.

## Data Mart

The grain of `layer3.top_item` is one row per `(item_id, year)` with at least one `view_item` event.

| Column | Description |
| --- | --- |
| `item_id` | Item identifier. |
| `item_name` | Item name from the cleaned item dataset. |
| `year` | Calendar year of the views. |
| `total_views` | Number of logical item-view events in the year. |
| `item_rank` | Item rank within the year by view count. |
| `most_used_platform` | Platform with the most views for the item/year. |

`RANK()` is applied within each year by descending `total_views`. Tied items share the same rank, and gaps may occur. The most-used platform is selected by view count; platform ties are resolved alphabetically for deterministic output.

## Validation

Fail-fast assertions check:

- All Layer 1 columns are STRING.
- Each `event_id` has consistent event attributes and non-null core fields.
- The same event parameter cannot have conflicting values.
- Item keys are unique and non-null.
- Viewed items exist in the item dataset, with non-null item/year/platform fields.
- Mart item/year keys are unique and non-null.
- Overall and yearly mart view counts reconcile with logical `view_item` events in Layer 2.

The executed solution produced:

| Check | Result |
| --- | ---: |
| Item source rows | 2,198 |
| Event source parameter rows | 853,640 |
| Logical events | 428,364 |
| `view_item` events | 262,786 |
| `top_item` rows | 13,093 |
| Duplicate item/year rows | 0 |
| Null mart business keys | 0 |
| Mart/source view totals | 262,786 / 262,786 |

## Repository Structure

```text
notebooks/
  data_engineer_case_study.ipynb
OPEN_QUESTIONS.md
README.md
```

- [Notebook](notebooks/data_engineer_case_study.ipynb): source exploration, three-layer implementation, validations, and final result preview.
- `OPEN_QUESTIONS.md`: production considerations and dbt-core assessment.
- `README.md`: project overview and execution instructions.

## Running the Solution

The notebook is intended to run top-to-bottom in Databricks Free Edition or another Databricks environment with PySpark and Delta support.

1. Import or clone the repository into Databricks.
2. Open `notebooks/data_engineer_case_study.ipynb`.
3. Attach available compute.
4. Run all cells from top to bottom.

The inputs are public S3 URLs; no local input files are required. The notebook creates the required schemas and tables in the current catalog, where the execution identity needs creation permissions. Each run overwrites the output tables, and the mart is written only after its critical validations pass.

## Open Questions

[OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) covers production industrialization, dbt-core architecture, its advantages and disadvantages, and the implementation estimate.

## Technology

Python · PySpark / Spark SQL · Databricks · Delta Lake
