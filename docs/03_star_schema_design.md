# ⭐️ Star Schema Design (Gold Layer) - Kimball's 4-Step Dimensional Design Process

This document designs the Gold Layer Star Schema following Ralph Kimball's 4-step process.

![Star Schema](./star_schema.png)

## Step 1 - Select the Business Process

**Business process chosen**: Incident Reporting & Investigation Tracking

**Important**: Boundaries - Community Areas is not a new business process - it's just a reference/geographic dimension that adds a spatial dimension to the existing business process

## Step 2 - Declare the Grain

**Grain statement**: One row in `fact_crime` represents one reported offense recorded by CPD (Chicago Police Department), identified by the natural key `id`. A single investigative case (`case_number`) may bundle multiple offense records

**Fact Table Type**: A transaction fact table maintained via MERGE/UPSERT because existing cases can be updated retroactively (`updated_on`)

## Step 3 - Identify the Dimensions

|Dimension|Business question it answers|
|---|---|
|`dim_date`|Time patterns (weekday/weekend, monthly, quarterly trend)|
|`dim_time`|Hour-of-day crime patterns, patrol shift timing|
|`dim_location`|Spatial patterns at the beat/district/ward/community-area level|
|`dim_block`|Street-level location detail (direction, street type/name)|
|`dim_crime_type`|Which crime types occur most often, and where|
|`dim_arrest_status`|How arrest rate changes over time, delayed arrests, and which incidents are domestic-related (`domestic_flag`)|
|`dim_location_description`|Which kinds of place (street, residence, apartment, ...) crimes happen in|

### 3.1 `dim_date`

|Column|Type|Source|Notes|
|---|---|---|---|
|`date_key`|int|derived from `full_date`|Surrogate key, format `YYYYMMDD` (year first so the integer sorts chronologically), generated from `full_date` (**USE AS A FK FOR JOIN ONLY!!**)|
|`full_date`|date|`incident_date` from `silver.crimes` (the date part of `date`)|`SELECT explode(sequence(DATE'2001-01-01', DATE'2040-12-31', INTERVAL 1 DAY)) AS full_date`. Starts in 2001 because incidents from 2001 can be in the warehouse (see `01_business_context.md`)|
|`date_description`|string|derived from `full_date`|Human-readable full date, e.g. "1 January 2024"|
|`day`|tinyint|derived from `full_date`||
|`month`|tinyint|derived from `full_date`||
|`month_name`|string|mapped from `month`||
|`year`|smallint|derived from `full_date`|`YEAR(full_date)` — must be `smallint`, not `tinyint` (range -128..127 cannot hold a calendar year)|
|`month_year`|string|derived from `month` + `year`|e.g. "2024-01", convenience column for monthly trend charts|
|`quarter`|tinyint|derived from `month`|`CEIL(month/3)`|
|`weekday`|string|derived from `full_date`|e.g. "Monday"..."Sunday"|
|`day_in_week`|tinyint|derived from `full_date`|1-7, numeric companion to `weekday` so a chart axis sorts in calendar order instead of alphabetical order|
|`is_weekend`|boolean|derived from `weekday`|`true` if Saturday or Sunday|
|`is_first_day_of_week`|boolean|derived from `full_date`||
|`is_last_day_of_week`|boolean|derived from `full_date`||
|`is_first_day_of_month`|boolean|derived from `full_date`||
|`is_last_day_of_month`|boolean|derived from `full_date`||

### 3.2 `dim_time`

|Column|Type|Source|Notes|
|---|---|---|---|
|`time_key`|tinyint|`incident_hour` from `silver.crimes`|Surrogate key = the hour value itself (0-23), a smart key in the same spirit as `date_key` (**USE AS A FK FOR JOIN ONLY!!**)|
|`hour_24`|tinyint|derived|Hour in 24-hour format (0-23), kept separate from `time_key` for readability|
|`hour_12`|tinyint|derived|Hour in 12-hour format (1-12)|
|`am_pm`|string|derived|"AM" / "PM"|
|`hour_label`|string|derived|Display-ready label, e.g. "14:00"|
|`day_part`|string|derived|"Night (00-05)", "Morning (06-11)", "Afternoon (12-17)", "Evening (18-23)" — a general 4-way split, **not** an official CPD patrol watch schedule (no authoritative source for CPD's actual shift-hour boundaries was found; adjust these buckets if one turns up later)|

### 3.3 `dim_location`

|Column|Type|Source|Notes|
|---|---|---|---|
|`location_key`|int|derived|Surrogate key|
|`beat`|string|`beat` from `silver.crimes`||
|`district`|string|`district` from `silver.crimes`|CPD encodes the district number as the first two digits of the beat code, so a beat normally belongs to exactly one district. **Exception:** old records of the districts closed in 2012 keep the old beat code but carry the new district (beat `13xx` with district `012`, `23xx` with `019`, `21xx` with `001`/`002`/`009`). Silver currently quarantines these rows (see `05_design_decisions.md`), so they are not in the dimension|
|`district_name`|string|CPD's published district list|e.g. "1st District" = "Central" — looked up from CPD's own district reference, not present anywhere in the crime dataset itself|
|`ward`|string|`ward` from `silver.crimes` (an `int` in Silver, cast to string here)||
|`community_area`|string|`community_area` from `silver.crimes` (an `int` in Silver, cast to string here)||
|`community_area_name`|string|`community` from `Boundaries - Community Areas`|Not present anywhere in the crime data itself — added via a lookup join on `community_area`|

> `latitude`/`longitude`/`x_coordinate`/`y_coordinate` are not here because raw coordinates are close to unique per incident — putting
> them in a dimension would make `dim_location` balloon to nearly one row per fact row, defeating the point of having a dimension at all.
> They live in `fact_crime` instead, as Non-additive Facts.

### 3.4 `dim_block`

|Column|Type|Source|Notes|
|---|---|---|---|
|`block_key`|int|derived|Surrogate key|
|`block`|string|`block` from `silver.crimes`|Raw anonymized block-level address, e.g. "025XX W ADDISON ST"|
|`block_hundred`|string|`block_hundred` from `silver.crimes`|e.g. "025XX"|
|`direction`|string|`direction` from `silver.crimes`, mapped to a word in Gold|Silver stores the letter (`N`, `S`, `E`, `W`); Gold maps it to "North", "South", "East", "West"|
|`street_name`|string|`street_name` from `silver.crimes`||
|`street_type`|string|`street_type` from `silver.crimes`, mapped to its full name in Gold|Silver stores the standard USPS abbreviation (`AVE`, `ST`, ...); Gold maps it to "Avenue", "Street", ... with the same USPS list (`STREET_TYPE_NAMES` in `transform_crimes`)|

### 3.5 `dim_crime_type`

|Column|Type|Source|Notes|
|---|---|---|---|
|`crime_type_key`|int|derived|Surrogate key|
|`iucr`|string|`iucr` from `silver.crimes`|Illinois's own 434-code classification; `primary_type`/`description` are both derived from this. Silver already checks every `iucr` and its `primary_type` against `silver.ref_iucr`|
|`primary_type`|string|`primary_type` from `silver.crimes`|Uses the spelling of `silver.ref_iucr` (e.g. `CRIMINAL SEXUAL ASSAULT`)|
|`description`|string|`description` from `Chicago Crime 2001 - Present`||
|`fbi_code`|string|`fbi_code` from `silver.crimes` (untouched by Silver)|FBI/NIBRS-aligned classification, coarser than `iucr` (likely many-`iucr`-to-one-`fbi_code`); kept as a bare code with no description column — no official city/CPD reference table mapping `fbi_code` to a description could be found, so adding one would mean relying on an unverifiable external source|

### 3.6 `dim_arrest_status`

|Column|Type|Source|Notes|
|---|---|---|---|
|`arrest_status_key`|int|derived|Surrogate key|
|`case_number`|string|`case_number` from `silver.crimes`|Kept here (not just on the fact table) so the full status-change history for a case can be queried directly by `case_number`|
|`arrest_flag`|string|`arrest` from `silver.crimes` (a boolean in Silver)|Text values: "Arrest Made", "No Arrest" — per Kimball's "Flags and Indicators as Textual Dimension Attributes" technique, not a raw boolean/0-1|
|`domestic_flag`|string|`domestic` from `silver.crimes` (a boolean in Silver)|Text values: "Domestic", "Not Domestic"|
|`effective_date`|timestamp|derived from `updated_on`|Set to the `updated_on` value at the load cycle when this version of the row was created|
|`end_date`|timestamp|derived|Set to the next version's `effective_date` when a newer version is inserted; `null` while the row is still current|
|`is_current`|boolean|derived|Maintained by the SCD 2 MERGE logic; exactly one row per `case_number` has `is_current = true` at any time|
|`version_source`|string|derived|`BACKFILL` for records from initial load, `DETECTED_CHANGE` for an actual detected change and `NEW_CASE` for the first version of a case that appears in a later incremental run|

> **Why SCD Type 2**: arrest status can change retroactively, so history must be preserved to calculate "average time-to-arrest",
> which is a direct business question.

### 3.7 `dim_location_description`

|Column|Type|Source|Notes|
|---|---|---|---|
|`location_description_key`|int|derived|Surrogate key|
|`location_description`|string|`location_description` from `silver.crimes`||

## Step 4 - Identify the Facts

**Fact Table**: `fact_crime`

|Column|Type|Fact type|Source|Notes|
|---|---|---|---|---|
|`id`|bigint|-|`id` from `silver.crimes`|Primary Key; kept as-is from source — no separate surrogate key needed for the fact table. `bigint` because Silver stores it as `bigint`|
|`case_number`|string|Degenerate Dimension|`case_number` from `silver.crimes`|Also appears in `dim_arrest_status`, but for a different purpose — this one is a stable, grouping/tracing identifier at the fact's own grain|
|`incident_date_key`|int (FK)|-|Derived|Named by business role, not just "date_key" (Role-Playing Dimension naming) → `dim_date.date_key`|
|`incident_time_key`|tinyint (FK)|-|Derived|→ `dim_time.time_key`|
|`location_key`|int (FK)|-|Derived|→ `dim_location.location_key`|
|`block_key`|int (FK)|-|Derived|→ `dim_block.block_key`|
|`crime_type_key`|int (FK)|-|Derived|→ `dim_crime_type.crime_type_key`|
|`arrest_status_key`|int (FK)|-|Derived|→ `dim_arrest_status.arrest_status_key`; always points to the row where `is_current=true` for that case|
|`location_description_key`|int (FK)|-|Derived|→ `dim_location_description.location_description_key`|
|`x_coordinate`|double|Non-additive Fact|`x_coordinate` from `silver.crimes`|State Plane coordinate; can be NULL. A row with a coordinate of `0` or outside the allowed range is quarantined in Silver, so the fact never contains `0`|
|`y_coordinate`|double|Non-additive Fact|`y_coordinate` from `silver.crimes`|Same as above|
|`latitude`|double|Non-additive Fact|`latitude` from `silver.crimes`|Too continuous/non-aggregatable to be a Degenerate Dimension or a dimension attribute — can never be meaningfully summed|
|`longitude`|double|Non-additive Fact|`longitude` from `silver.crimes`|Same as above|
|`crime_count`|tinyint|Additive Fact|Derived|Constant literal 1|
|`updated_on`|timestamp|-|`updated_on` from `silver.crimes`|The version of the record: the Gold MERGE only replaces a row when the new `updated_on` is not older. It is the watermark of the Bronze load (the Silver load uses `_ingested_at` of Bronze). Not linked to `dim_date` — it's ETL/technical plumbing, not a business date|


### ‼️ Special Table: `ref_community_area_boundaries` (from `Boundaries - Community Areas` only)

|Column|Type|Source|Notes|
|---|---|---|---|
|`community_area_code`|int|`area_numbe`|Canonical join key; the duplicate `area_num_1` field is dropped in Silver|
|`community_area_name`|string|`community`||
|`boundary_geometry`|string|`the_geom`|Kept as a raw string, not parsed into a native geometry type|
|`shape_area_sqft`|double|`shape_area`||
|`shape_perimeter_ft`|double|`shape_len`||

> This table exists purely to avoid that waste; it plays no role in the normal analytical join path (crime counts,
> arrest rates, time trends never touch it) — its only consumer is the BI/map-rendering layer.
