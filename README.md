# AdTech Data Warehouse & ETL Pipeline (SSIS & SQL Server)

[![Platform](https://img.shields.io/badge/Platform-SQL%20Server-red.svg)](https://www.microsoft.com/sql-server)
[![ETL](https://img.shields.io/badge/ETL-SSIS-blue.svg)](https://learn.microsoft.com/en-us/sql/integration-services/sql-server-integration-services)
[![Architecture](https://img.shields.io/badge/Schema-Star%20Schema%20%7C%20SCD%20Type%202-green.svg)]()
[![Fact Type](https://img.shields.io/badge/Fact-Accumulating%20Snapshot-orange.svg)]()

An enterprise-grade **Extract, Transform, Load (ETL)** data pipeline and **Star Schema Data Warehouse** designed for the **Digital Advertising Technology (AdTech)** domain. Built with **Microsoft SQL Server Integration Services (SSIS)** and **T-SQL**, this project integrates cross-platform advertising events, tracks user demographic changes over time using **Slowly Changing Dimensions (SCD Type 2)**, and captures end-to-end transaction lifecycles using an **Accumulating Snapshot Fact Table**.

---

## Architecture Overview

The solution follows a multi-tier data warehouse architecture:

```
[ Heterogeneous Sources ]
 ├── AdTech_SourceDB (OLTP SQL Database)
 └── Flat Files (.CSV: Users, Ad Events, Event Completions)
              │
              ▼  (01_Load_Staging.dtsx)
[ Staging Area: AdTech_Staging ]
 ├── StgAdEvents
 ├── StgChangedUsers
 └── StgCompletionUpdates
              │
              ▼  (02_Load_DW.dtsx)
[ Enterprise Data Warehouse: AdTech_DW ]
 ├── DimCampaign
 ├── DimAd
 ├── DimUser (SCD Type 2)
 ├── DimDate & DimTime
 └── FactAdEvent (Accumulating Snapshot Fact Table)
              ▲
              │  (03_Update_Accumulating_Fact.dtsx)
[ Asynchronous Completion Updates ]
```

---

## Data Warehouse Schema (Star Schema)

The dimensional model is organized as a Star Schema centered on ad interaction and conversion events:

### Fact Table
* **`FactAdEvent`** (Accumulating Snapshot Fact Table):
  * **Keys**: `FactEventKey` (PK), `EventTransactionID` (Degenerate/Natural Key), `DateKey` (FK), `TimeKey` (FK), `UserKey` (FK), `CampaignKey` (FK), `AdKey` (FK).
  * **Metrics / Additive Measures**: `EventCount`, `ImpressionCount`, `ClickCount`, `LikeCount`, `CommentCount`, `ShareCount`, `PurchaseCount`, `AllocatedSpend`, `RevenueGenerated`.
  * **Lifecycle Tracking Columns**:
    * `accm_txn_create_time`: Timestamp of initial event occurrence.
    * `accm_txn_complete_time`: Timestamp when post-click conversion/processing finalized.
    * `txn_process_time_hours`: Elapsed lifecycle duration in hours:
      $$\text{txn\_process\_time\_hours} = \frac{\text{DATEDIFF(MINUTE, create\_time, complete\_time)}}{60.0}$$

### Dimension Tables
* **`DimUser` (Slowly Changing Dimension Type 2)**:
  * Tracks historical user demographic and preference shifts.
  * Columns: `UserKey` (Surrogate PK), `UserNK`, `UserGender`, `UserAge`, `AgeGroup`, `Country`, `Location`, `PrimaryInterest`, `AllInterests`, `ValidFrom`, `ValidTo`, `IsCurrent`.
* **`DimAd`**:
  * Attributes: `AdKey` (Surrogate PK), `AdNK`, `CampaignNK`, `AdPlatform` (Google, Meta, TikTok, etc.), `AdType` (Video, Carousel, Banner), `TargetGender`, `TargetAgeGroup`, `TargetInterests`.
* **`DimCampaign`**:
  * Attributes: `CampaignKey` (Surrogate PK), `CampaignNK`, `CampaignName`, `StartDateKey`, `EndDateKey`, `DurationDays`, `TotalBudget`.
* **`DimDate` & `DimTime`** (Conformed Dimensions):
  * Comprehensive temporal attributes enabling calendar and time-of-day analytics (`FullDate`, `DayName`, `MonthNumber`, `MonthName`, `QuarterNumber`, `YearNumber`, `IsWeekend`, `HourNumber`, `DayPart`).

---

## ETL Pipeline Implementation (SSIS)

The ETL process is implemented using **Visual Studio SQL Server Data Tools (SSDT)** across three modular SSIS packages:

### 1. `01_Load_Staging.dtsx` — Ingestion & Staging
* Extracts records from disparate data sources (Relational OLTP database + Delimited CSV files).
* Executes `TRUNCATE TABLE` on staging tables prior to loading to maintain pipeline idempotency.
* Ingests:
  * `ad_events_extended.csv` $\rightarrow$ `dbo.StgAdEvents`
  * `users_master.csv` & `users_updates.csv` $\rightarrow$ User staging
  * `event_completion_updates.csv` $\rightarrow$ `dbo.StgCompletionUpdates`

### 2. `02_Load_DW.dtsx` — Warehouse Loading & SCD Type 2
* Enforces strict referential dependency ordering through precedence constraints:
  1. `Truncate FactAdEvent` $\rightarrow$ `Truncate DimAd` $\rightarrow$ `Truncate DimCampaign` $\rightarrow$ `Truncate DimUser`
  2. Loads independent dimensions: `DimCampaign` and `DimAd`.
  3. **SCD Type 2 Engine for `DimUser`**:
     * Ingests baseline user demographics.
     * Uses **Conditional Split** and **Lookup** transformations to detect demographic/interest updates.
     * **Expires active records**: Sets `ValidTo = GETDATE()` and `IsCurrent = 0` for changed users.
     * **Inserts new record versions**: Sets `ValidFrom = GETDATE()`, `ValidTo = NULL`, and `IsCurrent = 1`.
  4. Loads `FactAdEvent` via surrogate key lookups against dimension tables.

### 3. `03_Update_Accumulating_Fact.dtsx` — Fact Accumulation
* Executes an asynchronous batch update task to finalize multi-day transaction completions:
  ```sql
  UPDATE f
  SET
      f.accm_txn_complete_time = s.accm_txn_complete_time,
      f.txn_process_time_hours =
          DATEDIFF(MINUTE, f.accm_txn_create_time, s.accm_txn_complete_time) / 60.0
  FROM dbo.FactAdEvent f
  INNER JOIN AdTech_Staging.dbo.StgCompletionUpdates s
      ON f.EventTransactionID = s.txn_id;
  ```

---

## Getting Started & Execution

### Prerequisites
* **Microsoft SQL Server** (2019/2022 recommended)
* **SQL Server Integration Services (SSIS)** installed on the database instance
* **Visual Studio** (2019/2022) with **SQL Server Data Tools (SSDT)** / Integration Services extension

### Execution Sequence
1. Open [`AdTech_DWBI_Assignment.sln`](file:///c:/Users/pasin/Downloads/Database%20Project/AdTech_DWBI_Assignment-master/AdTech_DWBI_Assignment.sln) in Visual Studio.
2. Verify connection manager configurations (`CM_DW`, `CM_Staging`, flat file paths) to match your environment.
3. Run packages in the following sequence:
   1. `01_Load_Staging.dtsx`
   2. `02_Load_DW.dtsx`
   3. `03_Update_Accumulating_Fact.dtsx`
