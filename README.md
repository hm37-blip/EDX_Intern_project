# IDX Real Estate Market Analysis: Data Pipeline & Exploration
This repository contains an end-to-end analysis workflow for CRMLS (California Regional MLS) data. The project aims to clean raw transaction data using Python and ultimately create an intelligent real estate market dashboard in Tableau. 
## Development Environment and Tools
Language: Python 3.x
Libraries: Pandas (data processing), Glob (file handling), NumPy (numerical computing)
Visualization: Tableau Desktop Public Edition 

### Data Pipeline Phase (Weeks 1–5)
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
| Pruning Category | Target Fields| Reasoning|
| :--- | ---: | ---: |
| Universal Constants | FireplaceYN, NewConstructionYN                            | 100% 'False' across 1.2M+ rows; zero analytical value.                   |
| Dataset Specific    | MlsStatus, OriginatingSystemName, ViewYN                  | Constant values (e.g., 'Closed', 'CRMLS') in sub-datasets.               |
| Mirror Redundancy   | LivingArea.1, ListPrice.1, Latitude.1, etc.               | 100% data overlap with original fields via `.equals()` validation.       |
| Null Disposal       | ElementarySchoolDistrict, CoveredSpaces                   | 100% missing values; identified as "Ghost Fields".                       |

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
         
         
         
         
