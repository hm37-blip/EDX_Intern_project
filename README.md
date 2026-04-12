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

**Phase 2:**

Prior to data cleansing, we conducted a comprehensive audit of the 84 source fields to ensure analytical integrity and optimize pipeline performance.

1. Data Volume Overview
   
The consolidated master tables represent market activity from January 2024 to March 2026:

Sold Dataset: 492,876 total records.

Listing Dataset: 756,095 total records.

3. Critical Missing Data (100% Nulls)
   
Through a full-missing value scan, we identified 5 fields that contain zero valid data across both datasets. These "Ghost Fields" have been earmarked for immediate removal to reduce memory overhead:
Common 100% Missing Fields: **FireplacesTotal**, **AboveGradeFinishedArea**, **ElementarySchoolDistrict**, **MiddleOrJuniorSchoolDistrict**, **CoveredSpaces**.

4. High-Value Field Integrity
   
Conversely, we verified the integrity of core metrics required for market analysis. The following fields showed exceptionally high completion rates:

ClosePrice: Only 4 rows missing in the Sold dataset (0.0008% missing rate), validating its use as a primary KPI.

LivingArea / PostalCode: Highly complete, ensuring accuracy for Price-per-SqFt and geographic trend analysis.

5. Redundancy & Conflict Resolution

We identified systematic redundancy in the API export (e.g., Latitude vs. Latitude.1).

Technical Decision: In the cleaning pipeline, we retain the original field and drop the .1 suffix mirrors to maintain a clean data schema.
