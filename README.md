# adf-pipeline-backup-denis_proj-

# End-to-End Enterprise Medallion Data Lakehouse & BI Warehouse

An end-to-end multi-source Data Lakehouse and relational Data Warehouse built natively in Azure Data Factory (ADF). This project simulates a realistic enterprise scenario ("Denis") where raw data is scattered across heterogeneous systems and must be ingested, cleansed, and modeled into a denormalized Star Schema for downstream Power BI reporting using a strict Bronze-Silver-Gold Medallion Architecture.


## 📌 Project Overview
The objective was to centralize multi-year sales facts and dimension tables from four distinct storage paradigms without relying on external Spark clusters (like Azure Databricks). The entire ETL/ELT pipeline—including data ingestion, regex cleansing, surrogate key generation, deduplication, multi-stream joins, and financial metric calculations—was engineered natively within **Azure Data Factory** using Copy Activities, Execute Pipeline orchestrators, and Mapping Data Flows.

## 🛠️ Architecture & Tech Stack
* **Orchestration & ETL:** Azure Data Factory (ADF), Mapping Data Flows, Integration Runtimes (Azure IR & Self-Hosted IR)
* **Cloud Storage (Data Lake):** Azure Data Lake Storage Gen2 (Bronze, Silver, Gold Layers), Azure Blob Storage
* **Source Systems:** 
  * **On-Premises MySQL 8.0** (Connected via Self-Hosted Integration Runtime)
  * **Azure Cosmos DB for MongoDB** (NoSQL JSON Documents)
  * **Azure Blob Storage** (Raw Dimension CSVs)
  * **ADLS Gen2** (Multi-year Sales Fact CSVs)
* **Target Data Warehouse:** Azure SQL Database

## ⚙️ Medallion Architecture Implementation

### 🥉 Bronze Layer (Raw Ingestion)
* **Pipeline:** `Raw2Bronze_pl`
* **Process:** Orchestrates 4 child pipelines (`Mongodb2Bronze_pl`, `OnPrem2Bronze_pl`, `Rawdim2Bronze_pl`, `RawFact2Bronze_pl`) in parallel.
* **Resiliency:** Implemented wildcard file paths (`*.csv`) with recursive folder reads to dynamically ingest all yearly sales files, ensuring pipeline stability even when source files are added or removed.

### 🥈 Silver Layer (Cleansing & Standardization)
* **Pipeline:** `Bronze2Silver_pl`
* **Process:** Executes 5 dimension Mapping Data Flows concurrently, triggering the Fact Data Flow upon success.
* **Transformations (Data Quality):**
  * **Deduplication:** Applied `Aggregate` transformations to remove duplicate product records coming from the On-Prem MySQL database.
  * **Data Cleansing:** Used `regexReplace()` to strip dirty string prefixes (e.g., `"ID - "`) from `SalesRepID` and `CategoryKey`.
  * **Column Splitting & Parsing:** Used `split()` to separate composite `Location` strings (`Country;City`) and Sales Rep Full Names, and `toDate()` to convert unstandardized string dates into proper Date types.
  * **Key Generation:** Generated a unique surrogate key (`GeoKey`) for the Geography dimension.

### 🥇 Gold Layer (Denormalization & Aggregation)
* **Pipeline:** `Silver2gold_pl`
* **Process:** Executes the master Mapping Data Flow (`Denis_df`).
* **Transformations (Business Logic):**
  * **Multi-Stream Joins:** Performed sequential Inner Joins across all 6 Silver streams (Sales, Product, Category, SubCategory, SalesRep, Geography) to construct a single denormalized analytical table.
  * **Derived Metrics:** Calculated `TotalRevenue` (`Units * RetailPrice`), `TotalCost` (`StandardCost * Units`), and `GrossProfit` (`TotalRevenue - TotalCost`).
  * **Dual-Sink Delivery:** Loaded the final 20-column analytical dataset simultaneously into ADLS Gen2 Gold (`Denis_G.csv`) and Azure SQL Database (`Denis_GTb`) for Power BI consumption.

## 🚀 Key Topics Learned & Problems Solved
1. **Siloed Multi-Source Data:** Successfully consolidated relational (MySQL), NoSQL (Cosmos DB), and flat-file storage into a governed Medallion Lakehouse.
2. **ADF Mapping Data Flows:** Gained hands-on experience building complex code-free transformations natively in ADF rather than relying on PySpark/Databricks.
3. **Self-Hosted Integration Runtime (SHIR):** Configured secure hybrid network connections between on-premises databases and Azure cloud services.
4. **Parent-Child Orchestration:** Designed scalable ADF pipelines using `Execute Pipeline` activities for dependency management and parallel execution.
5. **Storage Optimization Trade-offs:** Leveraged `Single Partition` output to generate clean, deterministic CSV filenames for a small-scale project, while understanding the necessity of `Use current partitioning` (Parquet/Delta) for multi-terabyte production workloads.

## 📂 Repository Structure
```
medallion-proj/
├── data/                       # Sample schemas and dummy data extracts (if applicable)
├── docs/
│   └── Enterprise_Medallion_Data_Warehouse_ADF_Documentation.pdf  # Full step-by-step technical documentation
├── README.md                   # Project overview
└── Architecture.png            # High-level architecture diagram
```
