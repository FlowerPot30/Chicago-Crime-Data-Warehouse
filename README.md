# Chicago Crime Data Warehouse

Data Engineering portfolio project: building an **Incremental Load** Data Warehouse from
**Chicago Crime Data**, using a **Medallion Architecture (Bronze → Silver → Gold)** on
**Azure + Databricks**.

> 📌 See detailed docs for each topic in [`docs/`](./docs)

---

## Why Incremental Load instead of Real-time Streaming?

Chicago Crime Data is published by the city in periodic batches, not as a naturally
continuous stream. Forcing a real-time replay would simulate a scenario that doesn't
match how the actual agency operates. An Incremental Load approach still demonstrates
the core Data Engineering skills we want to showcase:

- Watermark strategy
- MERGE INTO / Upsert
- Slowly Changing Dimension (SCD Type 2)
- Star Schema design

Full reasoning behind this decision: [`docs/01-business-context.md`](./docs/01-business-context.md)

---

## Architecture

```
Chicago Data Portal (Socrata API)
   → Azure Data Factory (orchestration: Copy Activity pulls data using watermark)
   → Azure Key Vault (stores API credentials / connection strings)
   → Azure Data Lake Storage Gen2 (Bronze / Silver / Gold)
   → Databricks Notebook (PySpark: Bronze → Silver → Gold)
   → Databricks Workflows (task orchestration)
   → Unity Catalog (governance)
   → Entra ID / Managed Identity (authentication between ADF ↔ Databricks)
   → GitHub Actions (deploy notebook/pipeline code)
   → Databricks SQL Dashboard / Power BI (consumption layer)
```

## Documentation

| Document | Content |
|---|---|
| [`docs/01-business-context.md`](./docs/01-business-context.md) | Scope decisions, decision history, tools that were deliberately excluded and why |
| [`docs/02-data-source.md`](./docs/02-data-source.md) | Chicago Crime Data / Socrata API details, watermark strategy |
| [`docs/03-star-schema.md`](./docs/03-star-schema.md) | Fact/Dimension table design, SCD Type 2, MERGE logic |

## Current Status

🚧 In planning/design phase (no notebooks/pipelines written yet)

## Implementation Plan

- [ ] Design the full Star Schema (all facts + dimensions, complete columns)
- [ ] Notebook 1: Bronze (append-only)
- [ ] Notebook 2: Silver (clean, dedupe, DQ checks)
- [ ] Notebook 3: Gold (MERGE INTO, SCD Type 2)
- [ ] Databricks Workflows connecting the 3 notebooks
- [ ] ADF: Linked Services + Pipeline
- [ ] Unity Catalog permissions
- [ ] GitHub Actions deploy pipeline
