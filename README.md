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
Data Quality Audit: Prior to data cleansing, we conducted a comprehensive audit of 84 source fields. Key findings are as follows:
  1. Critical Missing Data: Analysis revealed that certain fields had extremely high missing rates and will be excluded from subsequent analysis:
      100% missing: FireplacesTotal, ElementarySchoolDistrict, CoveredSpaces, etc.
      Low Missing Values/High Value: ClosePrice (only 4 rows missing), LivingArea, PostalCode. These fields will be prioritized for cleaning.
  2. Data Redundancy: We identified duplicate columns (e.g., Latitude and Latitude.1). In the pipeline, we retained the original fields and removed the suffix columns to optimize performance.
