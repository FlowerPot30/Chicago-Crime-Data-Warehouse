# Business Context & Decision History

## 1. Project Goal

Build a Data Warehouse that ingests Chicago crime data **incrementally** (not a full reload
every run), so an analytics team (e.g., building a crime-trend dashboard) can query the
latest data efficiently without re-pulling the entire historical dataset each time.

## 2. Why Incremental Load instead of Real-time Streaming

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

## 3. Measurable Goals (for the portfolio)

- Demonstrate practical, hands-on understanding of the Medallion Architecture
  (Bronze/Silver/Gold), not just theory
- Demonstrate correct Incremental Load design using a watermark that truly reflects change
- Demonstrate handling of Slowly Changing Dimensions (SCD Type 2) for data that gets
  retroactively updated
- Demonstrate an end-to-end pipeline: ingestion → transformation → consumption layer
