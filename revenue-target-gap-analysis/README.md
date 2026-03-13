## Revenue-to-Target Gap Analytics Pipeline

### Overview

###### This project implements a Revenue-to-Target Gap Analysis pipeline that integrates operational pipeline data and financial reporting data to monitor business performance against annual targets. Data from multiple sources is standardized through a SQL transformation layer, further reshaped in Power BI using Power Query, and modeled in a semantic layer to support analytical calculations. The final dashboard visualizes actual revenue, pipeline contributions, and the remaining gap to target across business units, enabling quick identification of performance trends and potential shortfalls.

### Architecture
![Pipeline Architecture](gap_report_architecture.png.png)

### SQL Data Preparation
###### This SQL Layer standardized and transformed CRM and financial data into reusable reporting datasets for Power BI transformation and dashboarding

![SQL Layer Preparation](sql_layer_diagram.png)

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
###### The SQL examples below are simplified and anonymized versions of transformations used in the project. Table names, fields, and business labels were modified to protect confidential information while preserving the transformation logic.
######  Example 1 transformed operational CRM stages into analytics-friendly categories for reporting
```
WITH revenue_pipeline AS (
    SELECT
        opp.id AS opportunity_id,
        acct.name AS company_name,
        client.name AS client_name,
        COALESCE(eng.amount_usd, eng.amount, 0) AS engagement_amount,

        -- weighted revenue metric
        CAST(opp.probability AS DECIMAL) / 100
            * COALESCE(eng.amount_usd, eng.amount, 0)
            AS weighted_revenue,

        opp.stage_name AS opportunity_stage,
        opp.engagement_stage,
        opp.close_date AS expected_start_date,
        opp.invoice_date
    FROM opportunities opp
    LEFT JOIN engagements eng
        ON opp.id = eng.opportunity_id
    LEFT JOIN accounts acct
        ON opp.account_id = acct.id
    LEFT JOIN accounts client
        ON opp.client_id = client.id

    WHERE opp.is_deleted = 0
),

classified_pipeline AS (
    SELECT *,
        CASE
            WHEN opportunity_stage NOT IN ('Closed Won','Closed Lost')
                THEN 'pipeline'

            WHEN opportunity_stage = 'Closed Won'
                AND engagement_stage NOT LIKE 'Closed%'
                THEN 'engagement'

            WHEN opportunity_stage IN ('Closed Lost','On Hold')
                THEN 'lost'

            WHEN engagement_stage LIKE 'Closed%'
                THEN 'invoiced'

            ELSE 'other'
        END AS category
    FROM revenue_pipeline
)

SELECT *
FROM classified_pipeline;
```
######  Example 2 build a reporting-ready monthly dataset across category and business unit, including zero-filled combinations for complete reporting coverage
```
WITH reporting_months AS (
    SELECT CONVERT(date, v.report_month) AS report_month
    FROM (VALUES
      ('2025-09-01'),
      ('2025-10-01'),
      ('2025-11-01'),
      ('2025-12-01')
    ) v(report_month)
),
metric_category AS (
    SELECT *
    FROM (VALUES
       ('actual'),
       ('recurring'),
       ('pipeline')
    ) v(category_id)
),
combined_metrics AS (
    SELECT
        report_month,
        category_id,
        business_unit_id,
        SUM(amount) AS amount
    FROM unified_revenue_events
    GROUP BY
        report_month,
        category_id,
        business_unit_id
)
SELECT
    m.report_month,
    c.category_id,
    b.business_unit_id,
    ISNULL(cm.amount, 0) AS amount
FROM reporting_months m
CROSS JOIN metric_category c
CROSS JOIN dim_business_unit b
LEFT JOIN combined_metrics cm
    ON m.report_month = cm.report_month
   AND c.category_id = cm.category_id
   AND b.business_unit_id = cm.business_unit_id;
```
######  Example 3 prepares recurring revenue for forecasting by adjusting renewal dates and aggregating expected renewal value at a monthly reporting grain
```
WITH recurring_revenue AS (
    SELECT
        opportunity_id,
        DATEADD(month, 2, renewal_date) AS projected_renewal_date,
        renewal_amount,
        business_unit_id
    FROM source_renewals
    WHERE
        renewal_date IS NOT NULL
        AND renewal_booked_date IS NULL
)

SELECT
    DATEFROMPARTS(
        YEAR(projected_renewal_date),
        MONTH(projected_renewal_date),
        1
    ) AS report_month,
    business_unit_id,
    SUM(renewal_amount) AS projected_revenue
FROM recurring_revenue
GROUP BY
    DATEFROMPARTS(
        YEAR(projected_renewal_date),
        MONTH(projected_renewal_date),
        1
    ),
    business_unit_id;
```

### Power Query Transformations
###### Additional transformations were performed using Power Query(M). This layer reshaped the SQL outputs into a reporting-ready dataset used in the Power BI semantic model.
###### Key responsibilities of this layer included: 
######  - integrating multiple datasets (financial actuals, pipeline data, and budget targets)
######  - standardizing business-unit labels using mapping tables
######  - creating grouped reporting categories
######  - reshaping data using pivot and unpivot operations
######  - appending datasets across multiple reporting periods
######  - preparing the final dataset used for gap analysis

### Power BI Semantic Model
###### The processed dataset was loaded into the Power BI semantic model, where additional calculations and measures were defined.
###### This model supports:
###### - revenue vs target comparison
###### - gap analysis over time
###### - category-level performance analysis

### Dashboard Visuals 
##### Example visualization from the Revenue-to-Target Gap Analysis dashboard.
![Dashboard](dashboard_example.png)
##### Stacked bars show actual revenue and pipeline contribution by business unit compared against the annual target, highlighting the remaining gap

## Data Privacy
##### All data structures and queries in this repository are simplified examples based on real-world experience. No confidential or proprietary data from Hilco is included.
