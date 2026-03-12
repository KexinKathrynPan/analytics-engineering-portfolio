## Revenue-to-Target Gap Analytics Pipeline

### Overview

###### This project builds....

### Architecture
![Pipeline Architecture](gap_report_architecture.png.png)

### SQL Data Preparation
###### This SQL Layer standardized and transformed CRM and financial data into reusable reporting datasets for Power BI transformation and dashboarding

![SQL Layer Preparation]

#### Stages Explanation
###### **1. Source extraction and standardization**
######  - extracted active pipeline records
######  - standardized date and deal stage fields
######  - filtered non-reportable records

###### **2. Enrichment and mapping**
######  - joined deals to business-unit mappings
######  - categorized deals conditionally into business pipelines
######  - created a unified pipeline deal dataset

###### **3. Aggregation for reporting**
######  - summarized data by month, pipeline category, and business unit
######  - prepared YTD-ready reporting outputs

####  Representative SQL Examples
######  Example 1
######  Example 2
######  Example 3


### Power Query Transformations

### Power BI Semantic Model

### Dashboard Visuals 


