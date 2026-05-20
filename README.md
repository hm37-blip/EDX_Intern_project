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


### Infrastructure Validation & KPI Extraction (Week 6-7)  

1.Disaster Recovery & Storage Integrity Audit

Because the project was suspended for 12 days in the local development environment, severe metadata corruption occurred in the storage layer upon restart.

a. Incident: When attempting to load `Sold_Data_Partitioned`, the system throws an `OSError: Couldn't deserialize thrift`, and is unable to read the existing Parquet partitions.
b. Diagnostic: (1) I verified that the storage paths matched using PowerShell commands, thereby ruling out the “FileNotFound” error.
               (2) It was confirmed that the corruption occurred in the Thrift header of the Parquet file; it is suspected to be silent corruption caused by a cloud storage synchronization conflict or a write interruption.
               (3) Remediation: We used `shutil.rmtree` to force-delete the corrupted partition directory and leveraged the validation pipeline developed in Week 5 to trigger a 100% full partition write from the original CSV (Source of Truth).
               (4) Lessons Learned (Engineering Insights): When managing distributed or partitioned data, backing up the “Source of Truth” is critical.

2. KPI Extraction

After resolving the issue of corrupted underlying storage and completing a 100% data reconstruction, this project entered the data assetization phase. By building a cross-district aggregation pipeline, we performed in-depth indicator extraction for 564 key cities in California.

a. Engineering Implementation

(1) To address the instability of the local synchronization environment, we have implemented an integrated “clear-reconstruct-aggregate” pipeline to ensure that computational logic is based on 100% clean in-memory objects.
(2) A hard filtering threshold of `Sold_Count >= 30` was introduced during aggregation, effectively removing outliers with excessively low transaction volumes and ensuring the KPI’s business relevance.
(3) Calculate the price per square foot for each transaction and determine the median price for the area, using this as the key indicator for measuring land prices.
(4) Quantify the supply-demand imbalance in the region using the ratio of sales to listings.

b. Key Business Insights

Based on real-time aggregation of over 900,000 records, we identified several “outliers” in the California market

California Price Per Square Foot (PPSF) cap:
| City               | Median PPSF     | Median Sold Price | Sample Size (Sold) |
|--------------------|------------------|-------------------|--------------------|
| Corona Del Mar     | $1,599 / sqft    | $2,140,000        | 34                 |
| Manhattan Beach    | $1,270 / sqft    | $1,925,000        | 173                |
| Laguna Beach       | $1,243 / sqft    | $1,850,000        | 217                |

Market Heat (Absorption Rate) Rankings:

| Area              | Absorption Rate | Market Insight                                                                 |
|-------------------|-----------------|---------------------------------------------------------------------------------|
| Sun City          | 125.5%          | Extremely strong inventory absorption; sales volume exceeds current listings. |
| Lakeview Terrace  | 110.0%          | Highly constrained supply and demand; intense buyer competition.              |

Overall conclusion: The average absorption rate in the California market is 68.7%, indicating that the market as a whole remains very active, with coastal cities (such as Corona Del Mar) commanding a significant premium.

### Predictive Modeling - Market Cooling Inde (week 8)
This week, the project formally established its core analytical objective: **to identify areas within the California real estate market that are at risk of a sudden downturn**. By quantifying supply-demand imbalances and price pressures, we aim to pinpoint market inflection points—those at the tipping point of growth and on the verge of entering a downturn.

1. Phase 1: Experimental MCI Modeling
As an initial step to guide our investigation, I developed the Market Cooling Index (MCI) v1.0. This model uses "Signal Convergence" to flag cities where current growth might be unsustainable.

The MCI score is calculated using a weighted average of three critical dimensions:

（1）Liquidity Exhaustion (40%): Measured by Absorption Rate. Areas with < 20% absorption are flagged for low buyer velocity.

（2）Price Ceiling Pressure (30%): Measured by Median PPSF. Cities exceeding $1,000/sqft are entering the "affordability friction" zone.

（3）Inventory Pressure (30%): Measured by the LS Ratio (Listings-to-Sold). A ratio > 5:1 indicates a significant buildup of unsold inventory.

Preliminary Results：The script identified Los Altos (Score: 100) and Corona Del Mar (Score: 100) as the highest risk areas, characterized by extreme price points coupled with near-stagnant liquidity (<10% absorption).

2. Phase 2: Critical Reflection & Identified Limitations

It is crucial to note that this MCI v1.0 serves as a directional tool rather than an absolute prediction. We have identified the following limitations that will be addressed in future iterations:

Static Snapshot: The current model uses cross-sectional data and cannot distinguish between "seasonal slowing" and "structural decline."

Luxury Market Resilience: High-end markets (e.g., Malibu) often exhibit "low velocity but high price stability," which might skew risk scores.

### 📊 Week 8: Exploratory Data Analysis (EDA) on Time-Series Metrics

Following the deployment of `analyze_historical_trends.py`, a comprehensive historical scan was executed across **16,000+ Month-over-Month (MoM) city registries**. The empirical findings have provided definitive signals that reshape our predictive model assumptions.

#### **1. Midsummer Freeze (Temporal Analysis)**
* **Data Core**: Out of 1,673 historical cooling events, an overwhelming volume clustered in **August and September** (August 2024: 5.7%, August 2025: 5.4%, September 2024: 5.3%).
* **TPM Insight**: This disproves our initial hypothesis of winter-driven seasonal drop-offs. Instead, it uncovers a "Post-School Session Exhaustion." After the peak spring/summer buying frenzy for school districts, demand sharply dries up by August, forcing lagging sellers to slash prices simultaneously.

#### **2. The Four-Quadrant Market Sentiment (Volume-Price Matrix)**
The California housing ecosystem exhibits a highly symmetrical state distribution:
* 🟥 **Quadrant 1: Price Drop & Volume Spike (Panic Selling / Liquidation)** — **20.4%**
* 🟨 **Quadrant 2: Price Drop & Volume Drop (Market Freeze / Illiquidity)** — **19.6%**
* 🟩 **Quadrant 3: Price Rise & Volume Spike (Bull Market Euphoria)** — **20.8%**
* 🟦 **Quadrant 4: Price Rise & Volume Drop (Sellers' Strike / Thin Volatility)** — **19.5%**

Insight: The near-identical split implies that "Market Cooling" cannot be treated as a monolith. The model must differentiate between **Panic Liquidation** and **Structural Freezes** .

#### **3. High-Risk Vulnerability in Mid-Tier Hubs (City Analysis)**
* **Data Core**: Aggressive Tier-1 economic hubs like **Los Angeles (8 events, Median PPSF: \$690)** and **Oakland (8 events, Median PPSF: \$564)** emerged as chronic cooling leaders alongside premium suburbs like **Danville (9 events, Median PPSF: \$766)**.
* **TPM Insight**: This establishes the **"Mid-Tier Vulnerability Hypothesis."** While ultra-luxury sellers (\$1,500+/sqft) can afford to hold inventory and low-end markets have rigid organic demand, the \$500-\$700/sqft segment represents mid-class buyers who are highly reactive to macro rate hikes, causing frequent, cyclical localized cooling shocks.

Heuristic Weighting: The 4:3:3 weighting is based on industry logic but lacks empirical confirmation from historical crash data.

3. Future Roadmap: The Path to Validation

To transform this into a robust predictive engine, the project will move toward an Evidence-Based Loop:

(1) Historical Ground Truth: We will query the Parquet Lake for historical "market cooling" events (e.g., cities that experienced 3+ consecutive months of price drops) to serve as a training set.

(2) ML-Assisted Weighting: Implement a Logistic Regression or Random Forest model to determine the actual statistical weight each factor (Liquidity vs. Price) contributes to a subsequent price drop.

(3) Macro-Factor Integration: Incorporate external variables such as Federal Interest Rates, regional unemployment data, and net migration flows to strengthen the predictive power.

### 🤖 Phase 5: Supervised Machine Learning Pipeline design

To transition from heuristic rules  to an empirical, data-driven system without relying on volatile external parameters, I deployed a **Binary Risk Classifier (LightGBM/XGBoost)** as our core predictive engine.

* **Problem Formulation**: Defined as a supervised binary classification task. The model consumes rolling $N$-month internal market signals to predict the probability of a market correction (`Is_Cooling_Event = 1`) in month $T+1$.
* **Robust Feature Engineering**: 
  * **Rolling Window Aggregates**: Transformed static snapshot metrics into 3-month velocity indicators (`Volume_Trend_3M`, `Price_Volatility_3M`).
  * **Cyclical Seasonality Encoding**: Modeled the "Midsummer Freeze Paradox" uncovered in our EDA by encoding calendar months into cyclical features, allowing the model to organically adjust risk thresholds for August and September.
* **Strategic TPM Value**: This evaluation loop replaces our initial heuristic 4:3:3 weighting matrix with statistical **Feature Importance Weights**, driving an empirical understanding of whether volume contraction or price friction serves as the leading indicator for localized real estate satiation.
