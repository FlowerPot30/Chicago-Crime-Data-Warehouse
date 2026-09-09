# ⭐️ Star Schema Design (Gold Layer) - Kimball's 4-Step Dimensional Design Process

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

|Dimens


