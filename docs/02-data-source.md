# Data Source

## 1. Source

**Chicago Crime Data** published via the **Socrata Open Data API (SODA)**

| Detail | Value |
|---|---|
| Endpoint | `data.cityofchicago.org` |
| Dataset | Crimes - 2001 to Present |
| Cost | Free, no API key required |
| Recommendation | Request a free App Token to avoid rate limiting on frequent pulls |
| Record limit per request | 50,000 rows — pagination must be handled |

## 2. Why This Dataset

- It's a real, continuously updated public dataset, which makes it a great fit for
  practicing incremental load
- It has separate columns for "date the incident occurred" and "date the record was
  last updated," which makes it possible to design a realistic watermark strategy
  (not just append-only)
- The data volume (a few thousand rows/day) fits comfortably within a portfolio-scale
  environment without needing a large cluster

## 3. Watermark Strategy

**Uses the `updated_on` column as the watermark (not `date`)**

Reasoning:
- `date` is the date the crime occurred, which is **fixed** once the record is first written
- `updated_on` is the date the record was most recently modified — this matters because
  some cases get updated retroactively, e.g., the `arrest` status changes from false → true
  once a suspect is later apprehended
- If `date` were used as the watermark, updates to old cases that were just modified
  would be **missed**, leaving the fact table with stale arrest/domestic values that no
  longer match reality

## 4. Example Incremental Pull Flow

```
1. ADF Lookup Activity reads the last_watermark value from a control table
2. Call the Socrata API with a filter: WHERE updated_on > last_watermark
3. Write the results to ADLS Gen2 (Bronze layer) as append-only
4. After the pipeline succeeds, update the control table with the latest watermark pulled
```

## 5. Implementation Caveats to Handle

- **Pagination**: Socrata caps requests at 50,000 records — need to loop until all data
  is pulled (handled via an Until Activity in ADF, or delegated to Python/Databricks)
- **Rate limiting**: use an App Token to reduce the chance of being throttled
- **Duplicates**: the same record can be pulled twice if watermark windows overlap —
  must be deduplicated in the Silver layer using `case_number`
