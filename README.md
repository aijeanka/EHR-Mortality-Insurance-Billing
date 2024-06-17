# HIDS6002: Patient Mortality, Morbidity, Insurance and Surprise Billing

**Author**: Aizhan Uteubayeva  
**Published**: September 18, 2023

## Project Overview

This project involves analyzing synthetic healthcare data to understand morbidity, mortality, insurance spending, uninsured rates, and the occurrence of surprise billing in the United States. The data analysis is conducted using SQL queries on a PostgreSQL database.

## Table of Contents
- [Installation](#installation)
- [Database Connection](#database-connection)
- [Analysis](#analysis)
  - [Morbidity and Mortality](#1-morbidity-and-mortality)
  - [Insurance Spending](#2-insurance-spending)
  - [Lack of Insurance](#3-lack-of-insurance)
  - [Surprise Billing](#4-surprise-billing)
- [References](#references)

## Installation

Ensure you have R installed along with the necessary libraries:

```R
install.packages("tidyverse")
install.packages("RPostgres")
install.packages("connections")
install.packages("keyring")
```

Load the libraries in your R script:

```R
library(tidyverse)
library(RPostgres)
library(connections)
library(keyring)
```

## Database Connection

Connect to the Synthea database using the following code:

```R
con <- connection_open(RPostgres::Postgres(),
          dbname = "syntheticMGUH",
          host = "34.85.177.140",
          user = "hids502_student",
          password = key_get(service = "syntheticmguh", 
                             username = "hids502_student"))
```

## Analysis

### 1. Morbidity and Mortality

**Mortality**: The top 3 causes of mortality were determined using the following SQL query:

```SQL
SELECT value, COUNT(*) AS mortality
FROM observations
WHERE code = '69453-9'
GROUP BY value
ORDER BY mortality DESC
LIMIT 3;
```

**Morbidity**: The top 3 causes of morbidity were determined using the following SQL query:

```SQL
SELECT description, COUNT(*) AS morbidity
FROM conditions
GROUP BY description 
ORDER BY morbidity  DESC
LIMIT 3;
```

### 2. Insurance Spending

The breakdown of spending between Medicaid, Medicare, and other payers was analyzed using this SQL query:

```SQL
SELECT payer_name, SUM(payer_coverage) AS total_spent
FROM
(SELECT 
CASE WHEN p.name in ('Medicare', 'Medicaid') THEN p.name ELSE 'Others' END AS payer_name, e.payer_coverage
FROM encounters e
JOIN payers p ON e.payer = p.id
WHERE e.start >= '2022-01-01 00:00:00' AND  e.stop <= '2022-12-30 23:59:59'
) T
GROUP BY payer_name;
```

### 3. Lack of Insurance

To estimate the percentage of uninsured Americans over time, the following SQL query was used:

```SQL
WITH total_patients_2010 AS (SELECT COUNT(id) AS total_patients FROM patients),
total_patients_2023 AS (SELECT COUNT(id) AS total_patients FROM patients WHERE deathdate is  NULL),

uninsured_patients_2010 AS (SELECT COUNT(DISTINCT e.patient) AS uninsured_americans
FROM encounters e
JOIN payers p ON e.payer = p.id
WHERE (e.start >= '2010-01-01 00:00:00' AND  e.stop <= '2010-12-30 23:59:59') 
AND p.name = 'NO_INSURANCE'),

uninsured_patients_2023 AS (SELECT COUNT(DISTINCT e.patient) AS uninsured_americans
FROM encounters e
JOIN payers p ON e.payer = p.id
WHERE (e.start >= '2023-01-01 00:00:00' AND e.stop <= '2023-12-30 23:59:59') 
AND p.name = 'NO_INSURANCE')

SELECT 
t3.uninsured_americans AS uninsured_2010 , t1.total_patients  AS total_patients_2010,
t4.uninsured_americans AS uninsured_2023 , t2.total_patients AS total_patients_2023
FROM 
total_patients_2010 t1,
total_patients_2023 t2,
uninsured_patients_2010 t3,
uninsured_patients_2023 t4;
```

### 4. Surprise Billing

To identify instances of surprise billing, the following SQL query was used:

```SQL
SELECT COUNT(DISTINCT patient)
FROM
payer_transitions pt
JOIN payers p ON pt.payer = p.id
WHERE p.name = 'NO_INSURANCE'
AND patient IN 
(SELECT DISTINCT patient
FROM 
encounters e 
JOIN payers p ON e.payer = p.id
WHERE p.name != 'NO_INSURANCE');
```

## References

- Cohen, R.A., 2022. Health insurance coverage: Early release of estimates from the National Health Interview Survey, 2022.
- [FastStats](https://www.cdc.gov/nchs/fastats/leading-causes-of-death.htm), 2023.
- [National Health Index | Blue Cross Blue Shield](https://www.bcbs.com/the-health-of-america/health-index/national-health-index), n.d.
- [NHE Fact Sheet | CMS](https://www.cms.gov/data-research/statistics-trends-and-reports/national-health-expenditure-data/nhe-fact-sheet), n.d.
- Pollitz, K., et al., 2020. US Statistics on Surprise Medical Billing. JAMA 323, 498.
- [Products - NHIS Early Release - Health Insurance - 2010](https://www.cdc.gov/nchs/data/nhis/earlyrelease/insur201106.htm), 2019.
