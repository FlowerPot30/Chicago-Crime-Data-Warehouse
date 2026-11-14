# ⭐️ Star Schema Design (Gold Layer) - Kimball's 4-Step Dimensional Design Process

This document designs the Gold Layer Star Schema following Ralph Kimball's 4-step process)

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
|`dim_arrest_status`|How arrest rate changes over time, delayed arrests|
|`dim_incident_flag`|Domestic-related incidents, incident location setting|

### 3.1 `dim_date`

|Column|Type|Source|Notes|
|---|---|---|---|
|`date_key`|int|derived from `full_date`|Surrogate key, format `YYYYMMDD` (year first so the integer sorts chronologically), generated from `full_date` (**USE AS A FK FOR JOIN ONLY!!**)|
|`full_date`|date|`date` from `Chicago Crime 2001 - Present`|`SELECT explode(sequence(DATE'2001-01-01', DATE'2040-12-31', INTERVAL 1 DAY)) AS full_date`|
|`date_description`|string|derived from `full_date`|Human-readable full date, e.g. "Monday, January 1, 2024"|
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
|`time_key`|tinyint|derived|Surrogate key = the hour value itself (0-23), a smart key in the same spirit as `date_key` (**USE AS A FK FOR JOIN ONLY!!**)|
|`hour_24`|tinyint|derived|Hour in 24-hour format (0-23), kept separate from `time_key` for readability|
|`hour_12`|tinyint|derived|Hour in 12-hour format (1-12)|
|`am_pm`|string|derived|"AM" / "PM"|
|`hour_label`|string|derived|Display-ready label, e.g. "14:00"|
|`day_part`|string|derived|"Night (00-05)", "Morning (06-11)", "Afternoon (12-17)", "Evening (18-23)" — a general 4-way split, **not** an official CPD patrol watch schedule (no authoritative source for CPD's actual shift-hour boundaries was found; adjust these buckets if one turns up later)|

### 3.3 `dim_location`

|Column|Type|Source|Notes|
|---|---|---|---|
|`location_key`|int|derived|Surrogate key|
|`beat`|string|`beat` from `Chicago Crime 2001 - Present`||
|`district`|string|`district` from `Chicago Crime 2001 - Present`|`beat` is a strict subset of `district` (CPD encodes the district number as a numeric prefix inside the beat code), so one beat always belongs to exactly one district|
|`district_name`|string|CPD's published district list|e.g. "1st District" = "Central" — looked up from CPD's own district reference, not present anywhere in the crime dataset itself|
|`ward`|string|`ward` from `Chicago Crime 2001 - Present`||
|`community_area`|string|`community_area` from `Chicago Crime 2001 - Present`||
|`community_area_name`|string|`community` from `Boundaries - Community Areas`|Not present anywhere in the crime data itself — added via a lookup join on `community_area`|

> `latitude`/`longitude`/`x_coordinate`/`y_coordinate` are not here because raw coordinates are close to unique per incident — putting
> them in a dimension would make `dim_location` balloon to nearly one row per fact row, defeating the point of having a dimension at all.
> They live in `fact_crime` instead, as Non-additive Facts.

### 3.4 `dim_block`

|Column|Type|Source|Notes|
|---|---|---|---|
|`block_key`|int|derived|Surrogate key|
|`block`|string|`block` from `Chicago Crime 2001 - Present`|Raw anonymized block-level address, e.g. "025XX W ADDISON ST"|
|`block_hundred`|string|decomposed from `block` in Silver||
|`direction`|string|decomposed from `block` in Silver|e.g. "West", "North"|
|`street_name`|string|decomposed from `block` in Silver||
|`street_type`|string|decomposed from `block` in Silver|e.g. "Avenue", "Street"|

> **Where the decomposition happens**: `block` is split in the **Silver** layer (not in Gold) so that every part can be validated column by
> column as a data quality check (e.g. `direction` must be a valid compass value). The raw `block` is kept alongside the parts. If a
> `block` does not match the expected pattern, the parts that cannot be extracted become `NULL` and the number of such rows is
> reported — the row itself is never dropped. Gold only takes the distinct combinations to build `dim_block`.

> **Why a new dimension instead of a Degenerate Dimension, or merging into `dim_location`**: `block` on its own (one high-cardinality
> identifier, no descriptive attributes) would be a textbook Degenerate Dimension. Once decomposed into real sub-attributes
> (`direction`, `street_type`, `street_name`) it stops being "just an identifier" and becomes a true dimension. Merging those
> sub-attributes into `dim_location` was rejected for the same reason `dim_location_type` was originally split out: they're
> orthogonal to `district`/`ward`/`beat`/`community_area` (a street segment can sit in any geographic area and vice versa), so
> merging would balloon `dim_location`'s row count for no analytical benefit.
> asks for street-level detail, but the fact table already carries `block_key`

### 3.5 `dim_crime_type`

|Column|Type|Source|Notes|
|---|---|---|---|
|`crime_type_key`|int|derived|Surrogate key|
|`iucr`|string|`iucr` from `Chicago Crime 2001 - Present`|Illinois's own 434-code classification; `primary_type`/`description` are both derived from this|
|`primary_type`|string|`primary_type` from `Chicago Crime 2001 - Present`||
|`description`|string|`description` from `Chicago Crime 2001 - Present`||
|`fbi_code`|string|`fbi_code` from `Chicago Crime 2001 - Present`|FBI/NIBRS-aligned classification, coarser than `iucr` (likely many-`iucr`-to-one-`fbi_code`); the descriptions and index/violent-property groupings come from CPD's official "Crime Categories" table (see the three columns below and the note under this table)|
|`fbi_code_description`|string|CPD "Crime Categories" table (static reference)|e.g. `01A` → "HOMICIDE – 1ST OR 2ND DEGREE (INDEX)"; kept exactly as published by CPD|
|`fbi_index_type`|string|CPD "Crime Categories" table (static reference)|`INDEX` / `NON-INDEX`; the table also uses `MCC`, `TRF`, `WRT` and `NON-CRIMINAL` as their own values for the non-FBI codes|
|`fbi_crime_category`|string|CPD "Crime Categories" table (static reference)|`VIOLENT INDEX` / `PROPERTY INDEX` / `NON-INDEX` (or `MCC` / `TRF` / `WRT` / `NON-CRIMINAL`); lets BI users split violent vs property crime without memorising codes|

> **Source of the FBI code descriptions**: CPD's "Publicly Accessible Data" page (https://www.chicagopolice.org/?p=40515), section
> "Crime Categories" (checked 2026-11-14), lists each code with Description, Index and Violent/Property columns. This replaces the
> earlier note that no official reference could be found. The table also contains four codes that are **not** FBI/NIBRS codes —
> `00` (non-criminal), `MCC`, `TRF`, `WRT` — so a format check on `fbi_code` must allow them (a pattern like "2 digits + optional A/B"
> alone would wrongly reject them). The mapping is kept as a static reference shared by Silver (validation: a code that is not in the
> list = WARN) and Gold (these three descriptive columns), so both layers use the same list.
>
> **IUCR validation**: Silver validates `iucr` against a static copy of CPD's IUCR reference (Chicago Data Portal dataset `c7ck-438e`,
> which also keeps retired codes for historical reference). An unknown code = WARN, not FAIL, because CPD can add new codes after the
> static copy was taken.

### 3.6 `dim_arrest_status`

|Column|Type|Source|Notes|
|---|---|---|---|
|`arrest_status_key`|int|derived|Surrogate key|
|`case_number`|string|`case_number` from `Chicago Crime 2001 - Present`|Kept here (not just on the fact table) so the full status-change history for a case can be queried directly by `case_number`|
|`arrest_flag`|string|`arrest` from `Chicago Crime 2001 - Present`|Text values: "Arrest Made", "No Arrest" — per Kimball's "Flags and Indicators as Textual Dimension Attributes" technique, not a raw boolean/0-1|
|`domestic_flag`|string|`domestic` from `Chicago Crime 2001 - Present`|Text values: "Domestic", "No Domestic"|
|`effective_date`|timestamp|derived from `updated_on`|Set to the `updated_on` value at the load cycle when this version of the row was created|
|`end_date`|timestamp|derived|Set to the next version's `effective_date` when a newer version is inserted; `null` while the row is still current|
|`is_current`|boolean|derived|Maintained by the SCD 2 MERGE logic; exactly one row per `case_number` has `is_current = true` at any time|
|`version_source`|string|derived|`BACKFILL` for records from initial load, `DETECTED_CHANGE` for an actual detected change|

> **Why SCD Type 2**: arrest status can change retroactively, so history must be preserved to calculate "average time-to-arrest",
> which is a direct business question.
>
> **Why `arrest_flag` itself didn't move to `dim_incident_flag` too**: unlike `domestic_flag`/`location_description`, `arrest_flag`
> genuinely needs change history (SCD Type 2). Junk dimensions are flat/static by nature — they aren't built to track history —
> so arrest status has to stay in its own dedicated SCD2 dimension.

### 3.7 `dim_location_description`

|Column|Type|Source|Notes|
|---|---|---|---|
|`location_description_key`|int|derived|Surrogate key|
|`location_description`|string|`location_description` from `Chicago Crime 2001 - Present`||

## Step 4 - Identify the Facts

**Fact Table**: `fact_crime`

|Column|Type|Fact type|Source|Notes|
|---|---|---|---|---|
|`id`|bigint|-|`id` from `Chicago Crime 2001 - Present`|Primary Key; kept as-is from source — no separate surrogate key needed for the fact table. Typed `bigint` to match the Silver layer, where `id` is cast from string and is the key of the Silver MERGE|
|`case_number`|string|Degenerate Dimension|`case_number` from `Chicago Crime 2001 - Present`|Also appears in `dim_arrest_status`, but for a different purpose — this one is a stable, grouping/tracing identifier at the fact's own grain, independent of the mutable `arrest_status_key` FK (which gets rewritten every time the arrest status changes)|
|`incident_date_key`|int (FK)|-|Derived from `incident_date` (Silver column decomposed from `date`)|Named by business role, not just "date_key" (Role-Playing Dimension naming) → `dim_date.date_key`|
|`incident_time_key`|tinyint (FK)|-|Derived from `incident_hour` (Silver column decomposed from `date`)|→ `dim_time.time_key`|
|`location_key`|int (FK)|-|Derived|→ `dim_location.location_key`|
|`block_key`|int (FK)|-|Derived|→ `dim_block.block_key`|
|`crime_type_key`|int (FK)|-|Derived|→ `dim_crime_type.crime_type_key`|
|`arrest_status_key`|int (FK)|-|Derived|→ `dim_arrest_status.arrest_status_key`; always points to the row where `is_current=true` for that case. **1st**: close the existing current row — set `end_date` to the new `effective_date` and `is_current=false`. **2nd**: insert the new current row with `is_current=true`. **3rd**: update `fact_crime.arrest_status_key` for that `case_number` to the newly inserted row's key|
|`location_description_key`|int (FK)|-|Derived|→ `dim_location_description.dim_location_description_key`|
|`x_coordinate`|double|Non-additive Fact|`x_coordinate` from `Chicago Crime 2001 - Present`|State Plane coordinate; `0` is a known geocoding-failure sentinel value in the source — **Silver converts it to `NULL`** (unknown) so it cannot distort ranges, averages or maps; the count and examples of converted rows are reported before the conversion|
|`y_coordinate`|double|Non-additive Fact|`y_coordinate` from `Chicago Crime 2001 - Present`|Same as above|
|`latitude`|double|Non-additive Fact|`latitude` from `Chicago Crime 2001 - Present`|Too continuous/non-aggregatable to be a Degenerate Dimension or a dimension attribute — can never be meaningfully summed. Silver sets values outside Chicago's bounding box to `NULL` (bounds set from data profiling) and reports them first|
|`longitude`|double|Non-additive Fact|`longitude` from `Chicago Crime 2001 - Present`|Same as above|
|`crime_count`|tinyint|Additive Fact|Derived|Constant literal 1|
|`updated_on`|timestamp|-|`updated_on` from source|The watermark of the **Bronze** incremental load. The **Silver** incremental watermark is `_ingested_at` (the time a row landed in Bronze), not `updated_on`, so rows re-ingested by a `manual_backfill` (which can carry an old `updated_on`) are still picked up. From Silver onwards `updated_on` decides which version of an `id` is the latest and drives `dim_arrest_status.effective_date`. Not linked to `dim_date` — it's ETL/technical plumbing, not a business date|


### ‼️ Special Table: `ref_community_area_boundaries` (from `Boundaries - Community Areas` only)

|Column|Type|Source|Notes|
|---|---|---|---|
|`community_area_code`|int|`area_numbe`|Canonical join key; the duplicate `area_num_1` field is not ingested at all (excluded already in Bronze)|
|`community_area_name`|string|`community`||
|`boundary_geometry`|string|`the_geom`|Kept as a raw string, not parsed into a native geometry type|
|`shape_area_sqft`|double|`shape_area`||
|`shape_perimeter_ft`|double|`shape_len`||

> **Silver source**: the Silver table `chicago_crime.silver.community_area_boundaries` already renames and casts these columns
> (`area_numbe` → `community_area_code` int, `community` → `community_area_name`, `the_geom` → `boundary_geometry`, `shape_area` →
> `shape_area_sqft` double, `shape_len` → `shape_perimeter_ft` double) and checks that `boundary_geometry` is parseable GeoJSON text;
> Gold copies it as-is under the `ref_` name. Silver also validates `fact_crime`'s `community_area` against its `community_area_code` values.

> This table exists purely to avoid that waste; it plays no role in the normal analytical join path (crime counts,
> arrest rates, time trends never touch it) — its only consumer is the BI/map-rendering layer.
