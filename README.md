# Chicago Crime Data Warehouse and Dashboard

This project was built to practice real-world data engineering skills by designing and implementing a Data Warehouse using tools that are actively in demand in the industry — Azure Data Factory, Databricks, Unity Catalog and Power BI

## 📚 Project Overview

This project starts from real business questions about crime in Chicago, not from the tools. Using **Chicago Crime Data** and **Boundaries - Community Areas** sourced via the **Socrata Open Data API (SODA)** on the Chicago Data Portal, the pipeline ingests and transforms the data into a Data Warehouse, which is then used to answer those business questions through a Power BI dashboard.

## 📂 Table of Contents

-  [`01_Business Context`](./docs/01_business_context.md) - **MAY BE CHANGE LATER**
-  [`02_Source Data Dictionary`](./docs/02_sources_data_dictionary.md) - **END POINT NOT SURE**
-  [`03_Star Schema Design`](./docs/03_star_schema_design.md) - **NOT DONE YET**
-  [`04_Data Architecture`](./docs/04_data_architecture.md)
-  [`05_Design Decisions`](./docs/05_design_decisions.md)
-  [`06_Component Breakdown`](./docs/06_component_breakdown.md)

## 🕹️ Technology Used

|

## ▶️ How to implement the project

### 1. Setup Environment
1. create **Resource Group** - `rg-chicago-crime-dw` as the container for all project resources.
2. create **Storage Account with ADLS Gen2** - `chicagocrimeadlsgen2` with Hierarchical namespace enabled, Shared key access disabled, Hot access tier, TLS 1.2 minimum
3. create **Containers** inside `chicagocrimedadlsgen2` - `bronze`, `silver`, `gold`
4. create **Key Vault** - `kv-chicago-crime-dw` with RBAC permission model; granted self **Key Vault Secrets Officer** role; stored the Socrata App Token as secret `socrata-app-token`
5. store **Socrata App Token** - registered at data.cityofchicago.org, generated a free App Token (dataset: ijzp-q8t2), stored it in Key Vault
6. create **Databricks Workspace** - `dbw-chicago-crime-dw`, Premium tier, Hybrid/Classic workspace type
7. create **Entra ID Admin User** - `dbadmin` as a cloud-native organizational account with Global Administrator role, required because the Azure sign-up identity (a personal/Gmail-derived account) cannot authenticate to the Databricks Account Console
8. verify **Unity Catalog Metastore** - whether a metastore already existed in the target region (Databricks auto-provisions one per region on workspace creation); assigned it to `dbw-chicago-crime-dw`
9. create **Access Connector** - `ac-chicago-crime-dw` and granted it **Storage Blob Data Contributor** on `chicagocrimeadlsgen2`
10. create **Storage Credential** - `cred-chicago-crime-dw` in Unity Catalog, referencing the Access Connector's resource ID; granted **CREATE EXTERNAL LOCATION** privilege to the working account
11. create **External Location** - `ext-loc-bronze`, `ext-loc-silver`, `ext-loc-gold`, each mapped to its respective container via `abfss://<container>@chicagocrimeadlsgen2.dfs.core.windows.net/`
12. create **Catalog & Schema** - create catalog `chicago_crime`; created schemas `bronze`, `silver` and `gold`, each with an explicit storage location pointing to its matching External Location

### 2. Bronze Layer : ทำตาม 'script/bronze' **NOT DONT YET**
