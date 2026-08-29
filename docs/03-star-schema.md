# Star Schema Design (Gold Layer)

## 1. Overview

The Gold layer is designed as a **Star Schema** to support analytical queries
(dashboards/BI), consisting of one fact table and four dimension tables.

## 2. Fact Table: `fact_crime`

| Column | Type | Notes |
|---|---|---|
| crime_key | bigint (surrogate key) | Primary Key |
| case_number | string | Natural key from source |
| date_key | int (FK → dim_date) | |
| location_key | int (FK → dim_location) | |
| crime_type_key | int (FK → dim_crime_type) | |
| arrest_flag | boolean | |
| domestic_flag | boolean | |
| updated_on | timestamp | Used as watermark + SCD tracking |

## 3. Dimension Tables

### `dim_date`
`date_key, full_date, year, month, day, weekday, is_weekend`

### `dim_location`
`location_key, district, ward, community_area, latitude, longitude, beat`

### `dim_crime_type`
`crime_type_key, primary_type, description, fbi_code`

### `dim_arrest_status` — **SCD Type 2**
`arrest_key, arrest_flag, domestic_flag, effective_date, end_date, is_current`

**Why SCD Type 2**: A case's arrest status can change after the fact (e.g., initially
no arrest, then a suspect is apprehended later). Keeping this as SCD Type 2 lets us:
- Preserve a full history of status changes per case (audit trail)
- Analyze trends such as "average time-to-arrest," which would be impossible with
  SCD Type 1 (overwrite), since the historical value would simply be lost

## 4. Example Merge Logic (Databricks SQL)

```sql
MERGE INTO gold.fact_crime AS target
USING silver.crime_updates AS source
ON target.case_number = source.case_number
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

> Note: for `dim_arrest_status` (SCD Type 2), the merge logic is more involved.
> It requires (1) closing the existing record by setting `end_date` and
> `is_current = false`, then (2) inserting a new record with `effective_date` set
> to the change date and `is_current = true`.

## 5. Why Star Schema over Snowflake Schema

- Simpler queries (fewer join levels), which is a better fit for BI tools like
  Power BI / Databricks SQL Dashboard
- Crime records don't have a dimension hierarchy complex enough to justify further
  normalization — the denormalized Star Schema is worth it for read performance
