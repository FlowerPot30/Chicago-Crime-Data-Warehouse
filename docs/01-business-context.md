# Business Context & Decision History

## 1. Project Goal

Build a Data Warehouse that ingests Chicago crime data **incrementally** (not a full reload
every run), so an analytics team (e.g., building a crime-trend dashboard) can query the
latest data efficiently without re-pulling the entire historical dataset each time.

## 2. Business Questions

Before deciding on any tool or architecture, the schema and pipeline in this project are
designed to answer the following business questions. Each question maps directly to
columns already defined in `fact_crime` and its dimension tables, so the design isn't
guesswork — it's driven by what the warehouse actually needs to answer:

1. **Crime trend over time** — How is the number of crimes trending by month/year, and
   is it increasing or decreasing? *(uses `dim_date`)*
2. **Geographic hotspots** — Which districts, wards, or community areas have the highest
   concentration of crime? *(uses `dim_location`)*
3. **Crime type breakdown** — What are the most common types of crime (`primary_type`),
   and how does that mix shift over time or by location? *(uses `dim_crime_type` +
   `dim_date`/`dim_location`)*
4. **Arrest rate / effectiveness** — What percentage of reported crimes result in an
   arrest, and how long does it typically take from report to arrest?
   *(uses `dim_arrest_status`, enabled specifically by SCD Type 2 tracking)*
5. **Domestic vs. non-domestic incidents** — How do domestic-related crimes differ in
   volume, type, or arrest rate compared to non-domestic ones? *(uses `domestic_flag`
   in `fact_crime` + `dim_arrest_status`)*

> These questions are also the justification for choosing **SCD Type 2** on
> `dim_arrest_status`: question #4 specifically requires tracking status *changes* over
> time, which a simple overwrite (SCD Type 1) cannot support.

## 3. Why Incremental Load instead of Real-time Streaming

This project started as a debate between **Real-time Streaming** and a
**Data Warehouse (Batch/Incremental)** approach, and concluded with **Incremental Load**
for the following reasons:

- Chicago Crime Data is published by the city in periodic batches, not as a naturally
  continuous stream. Forcing a real-time replay would simulate a scenario that doesn't
  match how the actual agency operates.
- Incremental Load still fully demonstrates the core Data Engineer skills we want to show:
  - Watermark strategy
  - MERGE INTO / Upsert
  - SCD Type 2
  - Star Schema design

## 4. Measurable Goals (for the portfolio)

- Demonstrate practical, hands-on understanding of the Medallion Architecture
  (Bronze/Silver/Gold), not just theory
- Demonstrate correct Incremental Load design using a watermark that truly reflects change
- Demonstrate handling of Slowly Changing Dimensions (SCD Type 2) for data that gets
  retroactively updated
- Demonstrate an end-to-end pipeline: ingestion → transformation → consumption layer
