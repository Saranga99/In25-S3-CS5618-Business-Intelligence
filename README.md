# 📘 OULAD Medallion Architecture on Databricks

*A Complete Guide to Building a Bronze → Silver → Gold Lakehouse Using the Open University Learning Analytics Dataset*

---

## 📚 Table of Contents

1. [Introduction](#introduction)
2. [Dataset Overview (OULAD)](#dataset-overview-oulad)
3. [Architecture Overview](#architecture-overview)
4. [Naming Conventions](#naming-conventions)
5. [Step-by-Step Setup](#step-by-step-setup)

   * [1. Create Catalog](#1-create-catalog)
   * [2. Create Schemas (Bronze, Silver, Gold)](#2-create-schemas-bronze-silver-gold)
   * [3. Create Volume & Upload CSVs](#3-create-volume--upload-csvs)
   * [4. Bronze Layer – Raw Tables](#4-bronze-layer--raw-tables)
   * [5. Silver Layer – Cleaned Tables](#5-silver-layer--cleaned-tables)
   * [6. Gold Layer – Star Schema](#6-gold-layer--star-schema)
6. [Final Lakehouse Structure](#final-lakehouse-structure)
7. [Query Examples](#query-examples)
8. [Next Steps](#next-steps)

---

# 📘 Introduction

This project demonstrates how to implement a complete **Medallion Architecture** (Bronze → Silver → Gold) on **Databricks Lakehouse** using the **Open University Learning Analytics Dataset (OULAD)**.

You will learn how to:

* Create catalogs, schemas, volumes
* Ingest CSVs into Bronze
* Clean & standardize data into Silver
* Build a BI-ready star schema in Gold
* Prepare your pipeline for Looker Studio, Power BI, and ML workflows

---

# 📊 Dataset Overview (OULAD)

The **Open University Learning Analytics Dataset (OULAD)** consists of seven CSV files containing:

| Filename                  | Description                           |
| ------------------------- | ------------------------------------- |
| `studentInfo.csv`         | Student demographics & final outcomes |
| `courses.csv`             | Course metadata                       |
| `studentRegistration.csv` | Start/stop registration dates         |
| `assessments.csv`         | Assessment metadata                   |
| `studentAssessment.csv`   | Exam/assignment submissions           |
| `vle.csv`                 | List of learning materials            |
| `studentVle.csv`          | Student interactions & clicks on VLE  |

---

# 🏛 Architecture Overview

```text
                         ┌───────────────────────┐
                         │  analytics_oulad      │  (Catalog)
                         └──────────┬────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
┌───────────────┐         ┌────────────────┐          ┌─────────────────┐
│ bronze_raw    │         │ silver_clean   │          │ gold_star       │
│ (Raw ingest)  │         │ (Cleaned data) │          │ (Star Schema)   │
└───────┬───────┘         └─────────┬──────┘          └────────┬────────┘
        │                            │                          │
        ▼                            ▼                          ▼
Raw CSVs → Delta Tables → Typed, Cleaned → Dims + Facts (BI-ready)
```

---

# 🏷 Naming Conventions

### Catalog

```
analytics_oulad
```

### Schemas

```
bronze_raw
silver_clean
gold_star
```

### Volume

```
raw_volume
```

### Tables

**Bronze (raw):**

```
student_info_raw
courses_raw
student_registration_raw
assessments_raw
student_assessment_raw
student_vle_raw
vle_raw
```

**Silver (clean/typed):**

```
student_info_base
courses_base
student_registration_base
assessments_base
student_assessment_base
student_vle_base
vle_base
```

**Gold (facts & dimensions):**

```
dim_student
dim_course
dim_assessment
fact_assessment_score
fact_vle_interactions
```

### Column Naming

```
lowercase_snake_case
```

---

# 📝 Step-by-Step Setup

---

## 1. Create Catalog

In **SQL Editor** or a Notebook:

```sql
CREATE CATALOG IF NOT EXISTS analytics_oulad;
USE CATALOG analytics_oulad;
```

Databricks automatically creates:

* `default` (writable)
* `information_schema` (read-only metadata)

---

## 2. Create Schemas (Bronze, Silver, Gold)

```sql
CREATE SCHEMA IF NOT EXISTS bronze_raw;
CREATE SCHEMA IF NOT EXISTS silver_clean;
CREATE SCHEMA IF NOT EXISTS gold_star;
```

---

## 3. Create Volume & Upload CSVs

```sql
CREATE VOLUME IF NOT EXISTS bronze_raw.raw_volume;
```

Upload all 7 OULAD CSV files into:

```
/Volumes/analytics_oulad/bronze_raw/raw_volume/
```

---

## 4. Bronze Layer – Raw Tables

```python
spark.sql("USE CATALOG analytics_oulad")
spark.sql("USE SCHEMA bronze_raw")

base_path = "/Volumes/analytics_oulad/bronze_raw/raw_volume/"

files = {
    "student_info_raw": "studentInfo.csv",
    "courses_raw": "courses.csv",
    "student_registration_raw": "studentRegistration.csv",
    "assessments_raw": "assessments.csv",
    "student_assessment_raw": "studentAssessment.csv",
    "student_vle_raw": "studentVle.csv",
    "vle_raw": "vle.csv"
}

for table_name, file_name in files.items():
    df = spark.read.option("header", True).csv(base_path + file_name)
    df.write.format("delta").mode("overwrite").saveAsTable(f"analytics_oulad.bronze_raw.{table_name}")

print("Bronze ingestion complete.")
```

---

## 5. Silver Layer – Cleaned Tables

```python
spark.sql("USE CATALOG analytics_oulad")
spark.sql("USE SCHEMA silver_clean")

# Student Info
spark.sql("""
CREATE OR REPLACE TABLE silver_clean.student_info_base AS
SELECT
    CAST(id_student AS INT) AS student_id,
    code_module,
    code_presentation,
    gender,
    region,
    highest_education,
    imd_band,
    age_band,
    disability,
    final_result
FROM bronze_raw.student_info_raw
""")

# Courses
spark.sql("""
CREATE OR REPLACE TABLE silver_clean.courses_base AS
SELECT
    code_module,
    code_presentation,
    CAST(module_presentation_length AS INT) AS presentation_length
FROM bronze_raw.courses_raw
""")

# Registration
spark.sql("""
CREATE OR REPLACE TABLE silver_clean.student_registration_base AS
SELECT
    CAST(id_student AS INT) AS student_id,
    code_module,
    code_presentation,
    CAST(date_registration AS INT) AS date_registration,
    CAST(date_unregistration AS INT) AS date_unregistration
FROM bronze_raw.student_registration_raw
""")

# Assessments
spark.sql("""
CREATE OR REPLACE TABLE silver_clean.assessments_base AS
SELECT
    CAST(id_assessment AS INT) AS assessment_id,
    code_module,
    code_presentation,
    assessment_type,
    CAST(date AS INT) AS assessment_date,
    CAST(weight AS INT) AS weight
FROM bronze_raw.assessments_raw
""")

# Student Assessment
spark.sql("""
CREATE OR REPLACE TABLE silver_clean.student_assessment_base AS
SELECT
    CAST(id_assessment AS INT) AS assessment_id,
    CAST(id_student AS INT) AS student_id,
    CAST(date_submitted AS INT) AS date_submitted,
    CAST(is_banked AS INT) AS is_banked,
    CAST(score AS DOUBLE) AS score
FROM bronze_raw.student_assessment_raw
""")

# VLE
spark.sql("""
CREATE OR REPLACE TABLE silver_clean.vle_base AS
SELECT
    id_site,
    code_module,
    code_presentation,
    activity_type,
    week_from,
    week_to
FROM bronze_raw.vle_raw
""")

# Student VLE
spark.sql("""
CREATE OR REPLACE TABLE silver_clean.student_vle_base AS
SELECT
    CAST(id_student AS INT) AS student_id,
    id_site,
    CAST(date AS INT) AS date,
    CAST(sum_click AS INT) AS clicks
FROM bronze_raw.student_vle_raw
""")
```

---

## 6. Gold Layer – Star Schema

### Dimensions

```sql
CREATE OR REPLACE TABLE gold_star.dim_student AS
SELECT DISTINCT
    student_id,
    gender,
    region,
    highest_education,
    imd_band,
    age_band,
    disability
FROM silver_clean.student_info_base;
```

```sql
CREATE OR REPLACE TABLE gold_star.dim_course AS
SELECT DISTINCT
    code_module,
    code_presentation,
    presentation_length
FROM silver_clean.courses_base;
```

```sql
CREATE OR REPLACE TABLE gold_star.dim_assessment AS
SELECT
    assessment_id,
    code_module,
    code_presentation,
    assessment_type,
    assessment_date,
    weight
FROM silver_clean.assessments_base;
```

---

### Fact Tables

#### `fact_assessment_score`

```sql
CREATE OR REPLACE TABLE gold_star.fact_assessment_score AS
SELECT
    sa.student_id,
    sa.assessment_id,
    sa.date_submitted,
    sa.score,
    sa.is_banked,
    a.code_module,
    a.code_presentation,
    a.assessment_type,
    a.weight
FROM silver_clean.student_assessment_base sa
LEFT JOIN silver_clean.assessments_base a
    ON sa.assessment_id = a.assessment_id;
```

#### `fact_vle_interactions`

```sql
CREATE OR REPLACE TABLE gold_star.fact_vle_interactions AS
SELECT
    sv.student_id,
    sv.id_site,
    sv.date,
    sv.clicks,
    v.code_module,
    v.code_presentation,
    v.activity_type
FROM silver_clean.student_vle_base sv
LEFT JOIN silver_clean.vle_base v
    ON sv.id_site = v.id_site;
```

---

# 📦 Final Lakehouse Structure

```text
analytics_oulad
 ├─ bronze_raw
 │    ├─ student_info_raw
 │    ├─ courses_raw
 │    ├─ student_registration_raw
 │    ├─ assessments_raw
 │    ├─ student_assessment_raw
 │    ├─ student_vle_raw
 │    └─ vle_raw
 │
 ├─ silver_clean
 │    ├─ student_info_base
 │    ├─ courses_base
 │    ├─ student_registration_base
 │    ├─ assessments_base
 │    ├─ student_assessment_base
 │    ├─ student_vle_base
 │    └─ vle_base
 │
 └─ gold_star
      ├─ dim_student
      ├─ dim_course
      ├─ dim_assessment
      ├─ fact_assessment_score
      └─ fact_vle_interactions
```

---

# 🧪 Query Examples

### Number of students per course

```sql
SELECT code_module, code_presentation, COUNT(*) AS total_students
FROM gold_star.dim_student
GROUP BY code_module, code_presentation;
```

### Average assessment score per module

```sql
SELECT code_module, AVG(score) AS avg_score
FROM gold_star.fact_assessment_score
GROUP BY code_module;
```

### Total VLE clicks per student

```sql
SELECT student_id, SUM(clicks) AS total_clicks
FROM gold_star.fact_vle_interactions
GROUP BY student_id;
```

---

# 🚀 Next Steps

* Connect **Gold Layer** to Looker Studio / Power BI
* Build student success prediction models
* Add Databricks workflows for scheduled ETL
* Implement Unity Catalog access control
* Add Auto Loader for streaming ingestion
