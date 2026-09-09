<img width="674" height="119" alt="image" src="https://github.com/user-attachments/assets/532e8c77-b3ba-4a6c-b29f-aeed809c5335" /># ⭐️ Star Schema Design (Gold Layer) - Kimball's 4-Step Dimensional Design Process

This document designs the Gold Layer Star Schema following Ralph Kimball's 4-step process (Select the Business Process -> 
Declare the Grain -> Identify the Dimensions -> Identify the Facts), so every design decision has a stated reason instead of just listing tables.

## Step 1 - Select the Business Process

**Business process chosen**: Incident Reporting & Investigation Tracking 

> Per [`docs/business_context.md`](docs/business_context.md), this business process is the source of every business question in the project 
(spatial/temporal crime patterns, arrest rate, resolution tag)

**Important**: Boundaries - Community Areas is not a new business process - it's just a reference/geographic dimension that adds a spatial dimension 
to the existing business process (it has no fact table of its own, and no recurring "event" that needs to be recorded)

## Step 2 - Declare the Grain

**Grain statement**: One row in `fact_crime` represents one crime case recorded by CPD (Chicago Police Department), identified by the natural key `case_number`

**Fact Table Type**: A transaction fact table maintained via MERGE/UPSERT because existing cases can be updated retroactively (`updated_on`)

## Step 3 - Identify the Dimensions

|Dimension|Business question it answers|SCD Type|
|---|---|---|
|`dim_date`|Time patterns (weekday/weekend, monthly, quarterly trend)|Type 0|
|`dim_location`|Spatial patterns at the beat/district/ward/location-type level|Type 1|
|`dim_crime_type`|Which crime types occur most often, and where|Type 1|
|`dim_arrest_status`|How arrest rate changes over time, delayed arrests|Type 2|

### 3.1 `dim_date`

|Column|Type|Source|Notes|
|---|---|---|---|
|`date_key`|int|derived from `full_date`|Surrogate key, format `DDMMYYYY`, generated from `full_date`|
|`full_date`|date|`date` from `Chicago Crime 2001 - Present`|Cast from the source floating timestamp|
|`year`|int|derived from `full_date`|`YEAR(full_date)`|
|`month`|int|derived from `full_date`||
|`quarter`|int|derived from `month`|`CEIL(month/3)`|
|`day`|int|derived from `full_date`||
|`weekday`|string|derived from `full_date`|eg. "Monday"..."Sunday"|
|`is_weekend`|boolean|derived from `weekday`| `true` if Saturday or Sunday|

### 3.2 `dim_location`

|Column|Type|Source|Notes|
|---|---|---|---|
|`location_key`|int|derived|Surrogate key|
|`district`|string|`district` from `Chicago Crime 2001 - Present`|Kept as text to preserve values like "008"|
|`ward`|int|`ward` from `Chicago Crime 2001 - Present`||
|`beat`|string|`beat` from `Chicago Crime 2001 - Present`||
|`community_area_code`|int|`community_area` from `Chicago Crime 2001 - Present`| cast to int|
|`community_area_name`|string|`community` from `Boundaries - Community Areas`|Not present anywhere in the crime data itself — added via a lookup join on `community_area_code`|
|`location_description`|string|`location_description` from `Chicago Crime 2001 - Present`|eg. "STREET", "RESIDENCE", "APARTMENT"|

> I didn't put `latitude` and `longitude` in `dim_location` because raw coordinates are close to unique per incident, so putting them in a dimension would make dim_location balloon to nearly one row per fact row, defeating the point of having a dimension at all. Hence I put them into `fact_crime` instead

### 3.3 `dim_crime_type`

|Column|Type|Source|Notes|
|---|---|---|---|
|`crime_type_key`|int|derived|Surrogate key|
|`iucr`|string|`iucr` from `Chicago Crime 2001 - Present`|
|`primary_type`|string`|`primary_type` from `Chicago Crime 2001 - Present`|
|`description`|string|`description` from `Chicago Crime 2001 - Present`|
|`fbi_code`|string|`fbi_code` from `Chicago Crime 2001 - Present`|

### 3.4 `dim_arrest_status`

|Column|Type|Source|Notes|
|---|---|---|---|
|`arrest_key`|bigint|derived|Surrogate key|
|`case_number`|string|`case_number` from `Chicago Crime 2001 - Present`|Kept here (not just on the fact table) so the full status-change history for a case can be queried directly by `case_number`|
|`arrest_flag`|boolean|`arrest` from `Chicago Crime 2001 - Present`||
|`domestic_flag`|boolean|`domestic` from `Chicago Crime 2001 - Present`||
|`effective_date`|timestamp|derived from `updated_on`|Set to the `updated_on` value at the load cycle when this version of the row was created|
|`end_date`|timestamp|derived|Set to the next version's `effective_date` when a newer version is inserted; `null` while the row is still current|
|`is_current`|boolean|derived|Maintained by the SCD 2 MERGE logic; exactly one row per `case_number` has `is_current = true` at any time|

>**Why SCD Type 2**: arrest status can change retroactively, so history must be preserved to calculate "average time-to-arrest", which is a direct business question.

### 3.5 `ref_community_area_boundaries` (from `Boundaries - Community Areas` only)

|Column|Type|Source|Notes|
|---|---|---|---|
|`community_area_code`|int|`area_numbe`|Canonical join key; the duplicate area_num_1 field is dropped in Silver|
|`community_area_name|string|`community`||
|`boundary_geometry`|string|`the_geom`|Kept as a raw string, not parsed into a native geometry type|
|`shape_area_sqft`|double|`shape_area`||
|`shape_perimeter_ft|double|`shape_len`||

>**Why this stays a reference table and not a full dimension**: Kimball's default guidance is to keep dimensions denormalized rather than snow flaking them - `community_area_code`/`community_area_name` are low-cardinality (77 values) and belong flat in `dim_location`, same as `district`/`ward`/`beat`. The one field that justifies a split is `boundary_geometry` — a large polygon (hundreds to thousands of coordinate points). If it were embedded in `dim_location`, it would be duplicated across every row sharing the same community area, and `dim_location` can realistically have far more rows than the 77 distinct boundary shapes that exist. This table exists purely to avoid that waste; it plays no role in the normal analytical join path (crime counts, arrest rates, time trends never touch it) — its only consumer is the BI/map-rendering layer.

## Step 4 - Identify the Facts

**Fact Table**: `fact_crime`

|Column|Type|Fact type|Source|Notes|
|---|---|---|---|---|
|`id`|string||`id` from `Chicago Crime 2001 - Present`|Primary Key|
|`case_number`|string|Degenerate Dimension|`case_number` from `Chicago Crime 2001 - Present`||
|`date_key`|int (FK)||Derived||
|`location_key`|int (FK)||Derived||
|`crime_type_key`|int (FK)||Derived||
|`arrest_key`|bigint (FK)||Derived||
|`latitude`|double||`latitude` from `Chicago Crime 2001 - Present`||
|`longitude`|double||`longitude` from `Chicago Crime 2001 - Present`||
|`crime_count`|int| Additive Fact|Derived|Constant literal 1|
|`arrest_flag`|boolean|Type 1 (overwritten via MERGE)|`arrest` from `Chicago Crime 2001 - Present`|Current status; answers snapshot questions like "arrest rate this quarter" - different job from `dim_arrest_status`, which stores history|
|`domestic_flag`|boolean **NOT DONE YET** 

1.ทำไมมี domestic_flag อยู่ทั้งใน dim และ fact 
2.ref_community_area_boundaries ใช้แค่ join เพื่อเอา community_area_name แค่นั้นใช่ไหม ไม่ได้เอาไว้ทำอย่างอื่นแล้วใช่ไหม
3.ฉันต้องสร้าง table ที่เก็บทั้ง multipolygon และ community_area_num ด้วยใช่ไหม ถ้าใช่ ฉันสามารถ extract 2 cols นี้มาจาก ref_community_area_boundaries ได้ไหม แล้วให้ไฟล์ format เป็น .geojson

