# IDX Real Estate Market Analysis: Data Pipeline & Exploration
This repository contains an end-to-end analysis workflow for CRMLS (California Regional MLS) data. The project aims to clean raw transaction data using Python and ultimately create an intelligent real estate market dashboard in Tableau. 
## Development Environment and Tools

Language: Python 3.x

Libraries: Pandas (data processing), Glob (file handling), NumPy (numerical computing)

Visualization: Tableau Desktop Public Edition 

### Data Pipeline Phase (Weeks 1–4)
**Phase 1:** 

Data Aggregation: We will merge monthly CSV files from January 2024 to March 2026 into two core master tables. 

Sold Dataset: Processes closed sales records; the key metric is ClosePrice. Listing Dataset: Processes listing records; the key metrics are ListPrice and StandardStatus. 

**Phase 2:** Data Engineering & Scalability Audit

In this phase, we conducted a rigorous pre-cleansing audit and optimized the data architecture to ensure the pipeline can handle over 1.2M+ transaction records without memory exhaustion.

1.Memory & Storage Optimization

By implementing Schema Pre-definition (converting high-cardinality strings to Categorical and mapping flags to Boolean), we achieved significant performance gains:
| Dataset|Initial Memory|	Optimized Memory|	Reduction (%)|	Impact|
| :--- | ---: | ---: |---: | ---: |
|SOLD	|1578.45 MB|1290.74 MB|	18.2%	Enabled smooth local processing|
|LISTING|2106.82 MB|	1773.71 MB|	15.8%	Reduced I/O overhead|

2. Statistical Variance & Feature Pruning
Our Skewness Audit identified several "Zero Variance" columns. These fields were pruned to reduce data noise and focus analytical resources on high-variance drivers:

| DPruning Category|Target Fields|	Reasoning|	
| --- | --- | --- |
|Universal Constants	|FireplaceYN, NewConstructionYN|100% 'False' across 1.2M+ rows; zero analytical value. |
|Dataset Specific|MlsStatus, OriginatingSystemName, ViewYN |Constant values (e.g., 'Closed', 'CRMLS') in sub-datasets. |
| Mirror Redundancy|LivingArea.1, ListPrice.1, Latitude.1, etc.|100% data overlap with original fields via `.equals()` validation.   |
|Null Disposal| ElementarySchoolDistrict, CoveredSpaces  | 100% missing values; identified as "Ghost Fields"|



**Phase 3:**

Prior to data cleansing, we conducted a comprehensive audit of the 84 source fields to ensure analytical integrity and optimize pipeline performance.

1. Data Volume Overview
   
The consolidated master tables represent market activity from January 2024 to March 2026:

Sold Dataset: 492,876 total records.

Listing Dataset: 756,095 total records.

2. Critical Missing Data (100% Nulls)
   
Through a full-missing value scan, we identified 5 fields that contain zero valid data across both datasets. These "Ghost Fields" have been earmarked for immediate removal to reduce memory overhead:
Common 100% Missing Fields: **FireplacesTotal**, **AboveGradeFinishedArea**, **ElementarySchoolDistrict**, **MiddleOrJuniorSchoolDistrict**, **CoveredSpaces**.

3. High-Value Field Integrity
   
Conversely, we verified the integrity of core metrics required for market analysis. The following fields showed exceptionally high completion rates:

**ClosePrice**: Only 4 rows missing in the Sold dataset (0.0008% missing rate), validating its use as a primary KPI.

**LivingArea** / **PostalCode**: Highly complete, ensuring accuracy for Price-per-SqFt and geographic trend analysis.


4. Redundancy & Conflict Resolution
During the data auditing phase, we performed a comprehensive Column-to-Column Collision Test on the 84 source fields to address systematic mirroring issues arising from the API export process.

      a. Audit Discovery & Differential Analysis
         Using the Python .equals() algorithm for row-by-row validation, we identified a significant structural divergence between the two master tables:
         
         SOLD Dataset: Demonstrated high structural integrity with zero 100% matching mirror fields.
         
         LISTING Dataset: Exhibited a consistent Suffix Mirroring (.1) pattern, with 11 pairs of fields identified as 100% redundant.

      b. Redundancy Inventory
         To optimize the schema, we retained the original fields and flagged the following .1 suffixed columns for pruning:
         
         Core Transaction KPIs:
         
         LivingArea (vs LivingArea.1)
         
         ListPrice (vs ListPrice.1)
         
         DaysOnMarket (vs DaysOnMarket.1)
         
         CloseDate (vs CloseDate.1)
         
         Geographic & Location Data:
         
         Latitude (vs Latitude.1)
         
         Longitude (vs Longitude.1)
         
         UnparsedAddress (vs UnparsedAddress.1)
         
         PropertyType (vs PropertyType.1)
         
         Personnel & Office Metadata:
         
         ListAgentFirstName (vs ListAgentFirstName.1)
         
         ListAgentLastName (vs ListAgentLastName.1)
         
         BuyerOfficeName (vs BuyerOfficeName.1)
         
      c. Technical Decision & Implementation
         
         Strategy Selection: Drop Mirrors. Since the redundant columns provided no incremental information (no complementary null values), complex merging logic was rejected in favor of a clean prune.
         
         Execution: The logic_cleaning.py pipeline automatically identifies and drops all columns ending in .1 using regex pattern matching.
         
         Impact: This reduction decreased the field count by approximately 13%, significantly lowering memory overhead and eliminating dimension ambiguity during downstream Tableau visualization.
         

### Data Engineering, Scalability & Audi Phase (Week 5)
to-do list: Data Partitioning, Schema Validation, Feature Engineering at Scale)

1. Data Partitioning：Hive-style geospatially partitioned storage

   a. City is used as the primary partitioning field for physical storage. This enables the system to perform partition pruning when conducting region-specific analyses (e.g., analyzing housing price trends in Irvine only), allowing it to skip irrelevant folders directly.
   
   b. This design significantly reduces disk I/O overhead. When reading data for a specific city, the scan volume is reduced from 100% of the total data to less than 1%, significantly improving the response speed of downstream Tableau reports and Python analysis scripts.
   
   c. The use of industry-standard Hive-style naming (e.g., City=Amador%20City) ensures high compatibility and scalability of the data across different computing environments, such as Spark or cloud data lakes.

After migrating to the partitioned Parquet architecture, I conducted a performance benchmark to validate the efficiency of the new system:

|Query Target:| Geographic subset City = 'Los Angeles'|
|---|---|
|Result Set:| 13,607 records|
|Latency: |0.2909 seconds (vs. several seconds with full CSV load)|

The implementation of Partition Pruning effectively decoupled storage from compute, allowing for high-speed localized market analysis without taxing system memory. 
   
2. Data Quality Audit & Findings

Before formally persisting the data in Parquet format, I built a four-dimensional audit framework (Schema, Integrity, Uniqueness, Logic). By applying this “security gate,” I successfully identified and intercepted anomalous data that had remained in the data cleansing pipeline:

| Audit Dimension        | Status   | Issues Identified                                                                 | Engineering Impact                                                                                  |
|----------------------|----------|-----------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|
| Integrity            | ❌ FAIL  | 276 missing values found in the `City` field                                      | Breaks geographic partitioning logic, resulting in `__HIVE_DEFAULT_PARTITION__` dirty partitions     |
| Uniqueness           | ❌ FAIL  | 275 duplicate `ListingId` records detected                                        | Inflates aggregated metrics such as total sales volume and average price                             |
| Domain Logic         | ❌ FAIL  | 91 records violate physical constraints (Price ≤ 0 or LivingArea < 100 sqft)      | Indicates upstream data entry errors; must be filtered out before loading into production systems    |

I will update the ETL pipeline to implement the “Check-then-Store” pattern:
a. Enhance the logic of `drop_duplicates(subset=[‘ListingId’])`.

b. Add a mandatory filter: df.dropna(subset=[‘City’]).

c. Introduce a “Quarantine Zone” mechanism: Export these 600+ anomalous records to dirty_data_audit.csv for further investigation, ensuring that data entering the Parquet repository achieves a 100% Quality Pass.

3. Data Patching & Refresh

a. During the data persistence process, I implemented a schema validation mechanism. This step serves not only to verify the format but also to establish a set of defensive data quality gates.
By writing custom audit scripts, I identified hidden logical anomalies in the generated repository. To ensure the integrity of the production repository, I adopted the Dead Letter Queue (DLQ) approach to physically isolate the anomaly records.

| Dataset  | Quarantined Records | Root Cause                                                                 | Final Output (Validated) |
|----------|---------------------|----------------------------------------------------------------------------|--------------------------|
| SOLD     | 641                 | Includes duplicate IDs, missing `City` values, and invalid data (Price ≤ 0 or LivingArea < 100 sqft) | 367,213                  |
| LISTING  | 1,041               | Primarily missing geographic partition field (`City`) and data entry logic errors                  | 539,637                  |

b. During the process, I encountered an OSError: Couldn't deserialize Thrift error indicating underlying data corruption. I implemented the most efficient **disaster recovery** solution:

(1) Reverted to the Source of Truth (the pre-cleaned CSV master table).
(2) Used shutil to force-delete the corrupted old partition directories, resolving metadata conflicts.
(3) Integrated the latest validation logic and re-executed the Hive-style partitioned write

c. Key Highlights

(a) I began deploying a closed-loop pipeline that follows the “validate first, then load” principle to prevent dirty data from contaminating the production environment.

(b) All intercepted data is exported to Quarantine_Audit_Report.csv, providing an audit trail for subsequent data lineage tracking.

(c) The refactored Apache Parquet partitioned database maintains 100% data purity while achieving ultra-fast geographic dimension lookups in 0.29 seconds.
  
         
