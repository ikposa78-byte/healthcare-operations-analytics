<img width="1222" height="821" alt="healthcare_dashboard" src="https://github.com/user-attachments/assets/c1d5f83c-1cca-4b1b-a731-958f9605fa9e" />
# Healthcare Operations & Utilization Analysis

##  Author
* **Name:** Ubonanam Ebana
* **Role:** Data Analyst
* **Tools Used:** Google BigQuery (SQL), Tableau
* **GCP Project ID:** `serene-bastion-480402-k8`
* **Dataset:** `healthcare_analysis`

---

##  Executive Summary
In the healthcare sector, operational efficiency and cost transparency directly impact patient care quality and institutional financial health. This project establishes an end-to-end ELT (Extract, Load, Transform) pipeline within Google BigQuery to process raw, unstructured hospital data into clean, analysis-ready reporting layers. 

By engineering deterministic tracking mechanics and advanced analytical views, this project provides hospital administrators with critical insights into patient length of stay (LOS), billing distributions, clinical department performance, and high-utilization patient cohorts.

## Executive Dashboard
To view the fully interactive business intelligence dashboard, visit the live project here: [Tableau Public - Healthcare Operations & Utilization Dashboard](https://public.tableau.com/views/HealthcareOperationsUtilizationDashboard/OperationsDashboard?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

![Healthcare Operations Dashboard](<img width="1222" height="821" alt="healthcare_dashboard" src="https://github.com/user-attachments/assets/ec6332aa-0170-4e00-b6a1-803b2a10cb4d" />
)

###  Key Deliverables
1. **Production Transformation Pipeline:** Standardized raw strings, parsed dates, calculated core operational metrics, and implemented cryptographic patient hashing.
2. **Dynamic BI Reporting Layer:** Engineered 4 optimized SQL views leveraging advanced window functions to power operational and financial dashboards.

---

##  Data Architecture & Pipeline

The pipeline follows an ELT architecture inside Google BigQuery, isolating raw storage from the business logic layer.

```text
[Raw Landing Zone]                 [Data Mart Layer]              [BI Layer]
healthcare_data_raw  ──(SQL ELT)──>  healthcare_data_clean  ───>  Analytical Views  ───>  Tableau
```


### 1. Data Transformation & Cleaning (`scripts/01_data_transformation.sql`)

Before running any analytics, the raw healthcare dataset (`healthcare_data_raw`) required substantial data engineering to handle inconsistent formatting, type casting, and data integrity gaps. 

###  Key Transformation Steps:
* **Deterministic Hashing (`patient_id`):** The raw dataset did not include unique patient identifiers. Generating random UUIDs means every script refresh breaks data lineage. To fix this, I implemented a deterministic hashing algorithm using `FARM_FINGERPRINT()` on a concatenated string of natural keys (`Gender`, `Age`, and `Date of Admission`). This ensures returning patients consistently map to the same ID across execution cycles.
* **Datetime Optimization (`admission_date` & `discharge_date`):** Converted raw timestamp/string strings into strict standardized ISO `DATE` formats to enable optimized time-series tracking.
* **Calculated Metrics (`length_of_stay`):** Engineered a new operational metric using `DATE_DIFF` to calculate the exact number of days a patient occupied a hospital bed.
* **Text Standardization (`medical_condition`):** Applied `UPPER(TRIM(...))` to eliminate leading/trailing whitespaces and case-sensitivity duplicates in diagnostic data (e.g., standardizing "Asthma ", "asthma", and "ASTHMA" into a single category).
* **Financial Schema Enforcement (`total_charges` & `age`):** Cast billing numbers into highly accurate `NUMERIC` types to avoid floating-point rounding errors in financial reporting, and standardized `Age` into `INT64`.

####  The SQL Pipeline Script:

```sql
-- =====================================================
-- Project: Healthcare Operations & Utilization Analysis
-- Author: Ubonanam Ebana
-- Tool: Google BigQuery
-- Project ID: serene-bastion-480402-k8
-- Dataset: healthcare_analysis
-- SECTION 1: CLEAN BASE TABLE (Data Transformation Pipeline)
-- =====================================================

CREATE OR REPLACE TABLE
  `serene-bastion-480402-k8.healthcare_analysis.healthcare_data_clean` AS
SELECT
  -- Cryptographic stable ID generation for data integrity
  FARM_FINGERPRINT(CONCAT(Gender, '_', CAST(Age AS STRING), '_', `Date of Admission`)) AS patient_id,
  CAST(`Date of Admission` AS DATE) AS admission_date,
  CAST(`Discharge Date` AS DATE) AS discharge_date,
  DATE_DIFF(
    CAST(`Discharge Date` AS DATE),
    CAST(`Date of Admission` AS DATE),
    DAY
  ) AS length_of_stay,
  UPPER(TRIM(`Medical Condition`)) AS medical_condition,
  CAST(`Billing Amount` AS NUMERIC) AS total_charges,
  `Gender` AS gender,
  CAST(`Age` AS INT64) AS age
FROM `serene-bastion-480402-k8.healthcare_analysis.healthcare_data_raw`;
```

---

##  Section 2: Analytical Views & Business Insights
The reporting layer utilizes **Advanced Analytical Window Functions** and strategic groupings to create highly optimized semantic views inside BigQuery. These views are specifically engineered to answer critical business questions and feed executive dashboards without running heavy calculations on the fly.

###  1. Clinical Performance Matrix (`v_condition_performance_matrix`)
* **Business Question Answered:** Which medical conditions consume the most hospital resources, and what is their relative financial contribution to our overall revenue?
* **Advanced Metric:** Uses a partition-free window function (`SUM() OVER()`) to dynamically calculate each medical condition's percentage contribution to total hospital billing.

```sql
CREATE OR REPLACE VIEW `serene-bastion-480402-k8.healthcare_analysis.v_condition_performance_matrix` AS
SELECT
  medical_condition,
  COUNT(*) AS total_admissions,
  ROUND(AVG(length_of_stay), 2) AS avg_length_of_stay,
  ROUND(SUM(total_charges), 2) AS total_billing_amount,
  -- Percent contribution to hospital revenue by condition
  ROUND(
    100 * SUM(total_charges) / SUM(SUM(total_charges)) OVER(), 
    2
  ) AS pct_of_total_revenue
FROM `serene-bastion-480402-k8.healthcare_analysis.healthcare_data_clean`
GROUP BY medical_condition;
```

---

###  2. Daily Admissions Trend (`v_daily_admissions`)
* **Business Question Answered:** What are our daily admission trends, and how can we better plan our hospital bed capacity?
* **Business Value:** Provides a clean time-series aggregation perfect for line charts to detect seasonal spikes, weekend drops, or sudden surges in patient volume.

```sql
CREATE OR REPLACE VIEW `serene-bastion-480402-k8.healthcare_analysis.v_daily_admissions` AS
SELECT
  admission_date,
  COUNT(*) AS daily_admissions
FROM `serene-bastion-480402-k8.healthcare_analysis.healthcare_data_clean`
GROUP BY admission_date;
```

---

### 3. Patient Financial Risk & Outlier Rankings (v_patient_financial_demographics)
* **Business Question Answered:** Who are our financial outliers, and which individual cases cost significantly more than the average for their specific condition?

* **Advanced Metric:** Employs PERCENT_RANK() OVER(PARTITION BY...) to rank every admission's cost dynamically within its own medical condition category. This allows financial auditors to immediately isolate the top 5% most expensive cases for internal review.

```sql
CREATE OR REPLACE VIEW `serene-bastion-480402-k8.healthcare_analysis.v_patient_financial_demographics` AS
SELECT
  patient_id,
  age,
  gender,
  medical_condition,
  total_charges,
  length_of_stay,
  -- Dynamically ranks admissions to isolate the top 5% most expensive cases
  PERCENT_RANK() OVER(PARTITION BY medical_condition ORDER BY total_charges DESC) AS cost_percentile_rank
FROM `serene-bastion-480402-k8.healthcare_analysis.healthcare_data_clean`;
```

---

### 4. High Operational Utilization Cohorts (v_high_utilization_patients)
* **Business Question Answered:** Who are our "super-utilizer" patients, and how can we better manage their care to reduce hospital strain?

* **Business Value:** Filters specifically for extreme cases—patients with over 20 cumulative days in care or greater than $50,000 in total charges. Identifying this cohort allows hospital administrators to assign dedicated caseworkers or preventive care programs to optimize resource allocation.

 ```sql
  CREATE OR REPLACE VIEW `serene-bastion-480402-k8.healthcare_analysis.v_high_utilization_patients` AS
SELECT
  patient_id,
  COUNT(*) AS total_visits,
  SUM(length_of_stay) AS total_days_in_care,
  ROUND(SUM(total_charges), 2) AS total_billing_amount
FROM `serene-bastion-480402-k8.healthcare_analysis.healthcare_data_clean`
GROUP BY patient_id
HAVING
  total_days_in_care > 20
  OR total_billing_amount > 50000;
```

---

---

##  Section 3: Insights Generation & Verification Queries
To extract actionable business intelligence from the engineered reporting layer, the following queries are executed to populate static executive briefs and verify dashboard data integrity.

###  Executive Query Script:

```sql
-- 1. Volume Analysis: Admissions by clinical condition to identify resource demand
SELECT *
FROM `serene-bastion-480402-k8.healthcare_analysis.v_admissions_by_condition`
ORDER BY total_admissions DESC;

-- 2. Operational Efficiency: Identifying conditions resulting in the longest bed occupancy
SELECT *
FROM `serene-bastion-480402-k8.healthcare_analysis.v_avg_los_by_condition`
ORDER BY avg_length_of_stay DESC;

-- 3. Revenue Drivers: Isolating the highest cost conditions affecting hospital billing
SELECT *
FROM `serene-bastion-480402-k8.healthcare_analysis.v_total_billing_by_condition`
ORDER BY total_billing_amount DESC;

-- 4. Trend Forecasting: Extracting daily admissions for capacity planning models
SELECT *
FROM `serene-bastion-480402-k8.healthcare_analysis.v_daily_admissions`
ORDER BY admission_date;

-- 5. Risk Stratification: Pinpointing high-utilization patients for targeted care management
SELECT *
FROM `serene-bastion-480402-k8.healthcare_analysis.v_high_utilization_patients`
ORDER BY total_billing_amount DESC;

-- 6. Cost Audit: Reviewing the top 10 most expensive single admissions for billing validation
SELECT *
FROM `serene-bastion-480402-k8.healthcare_analysis.v_top_costly_admissions`
LIMIT 10;
```

---

## Section 4: Data Visualization & Dashboard Architecture (Tableau)

The semantic reporting layer built inside Google BigQuery serves as the direct data source for an interactive **Tableau Executive Dashboard**. By processing aggregations and window functions directly within the cloud data warehouse (SQL), Tableau's rendering speed is maximized, eliminating the need for heavy, unoptimized calculated fields within the BI tool itself.

### Dashboard Layout & Component Mapping:

1. **Hospital Capacity Planning & Operations**
   * **Source View:** `v_daily_admissions`
   * **Visual Component:** A continuous **Line Chart** tracking daily admission trends over time, paired with a **7-day Moving Average Trendline** to smooth out daily variance and forecast upcoming weekend bed availability.
2. **Clinical Resource Matrix**
   * **Source View:** `v_condition_performance_matrix`
   * **Visual Component:** A horizontal **Bar-in-Bar Chart** or side-by-side bar chart showing `total_admissions` and `avg_length_of_stay` by medical condition, allowing administrators to identify high-volume, high-bed-occupancy departments instantly.
3. **Financial Contribution & Revenue Share**
   * **Source View:** `v_condition_performance_matrix`
   * **Visual Component:** A dynamic **Donut Chart** or **Treemap** representing `pct_of_total_revenue`, illustrating which clinical diagnoses are driving the hospital's financial health.
4. **Patient Risk Stratification & Audit Panel**
   * **Source View:** `v_high_utilization_patients` & `v_patient_financial_demographics`
   * **Visual Component:** A dual-axis **Scatter Plot** mapping `length_of_stay` against `total_charges`. High-utilization cohorts and outliers ranking in the top 5% (`cost_percentile_rank`) are highlighted in red, allowing case managers to click and drill down into specific patient histories for care management intervention.

---

##  How to Run This Project Locally
1. Clone this repository to your machine.
2. Load the raw data into a Google BigQuery dataset named `healthcare_analysis`.
3. Execute `scripts/01_data_transformation.sql` to build the clean production table.
4. Execute `scripts/02_analytical_views.sql` to generate the reporting infrastructure.
5. Connect your Tableau Desktop / Tableau Public instance to BigQuery and replicate the dashboard schema mapped above.

---

## Data Source & Attribution
The dataset used for this business intelligence pipeline is publicly available on Kaggle.
* **Source:** [Kaggle - Healthcare Dataset](https://www.kaggle.com/code/likhithagudimetla/healthcare-dataset/input)
* **Scope:** Contains records detailing patient demographics, medical conditions, admission dates, length of stay, and operational financial billing metadata used to model hospital utilization workflows.






