# Bellabeat Case Study
## by Conor McGuire
A case study on the wellness company Bellabeat as part of the Coursera Google Data Analytics course.

# Introduction
In this case study, I play the part of a junior data analyst for Bellabeat, a high-tech manufacturer of health-focused products for women. The company's cofounder believes that analyzing smart device fitness data could lead to growth opportunities for the company. I've been asked to focus on one of Bellabeat's products and analyze smart device data to gain insight into how customers use smart devices. I will then present the insights to the Bellabeat executive team along with recommendations to guide the marketing strategy.

<details>
<summary><h2>Phase 1: Ask</h2></summary>
To guide myself in this analysis, I will ask myself:
1. What is the problem I'm trying to solve?
2. How can my insights drive business decisions?

## Business Task
Analyze Fitbit fitness tracker data to identify trends and use these insights to recommend a marketing strategy for Bellabeat's Leaf product.

## Stakeholders
- Bellabeat executive team (Urška Sršen, Sando Mur).
- Bellabeat marketing team.

## Key Questions
- What are the trends in smart device usage?
- How do these trends apply to Bellabeat's Leaf users?
- How can Bellabeat leverage these trends to shape the marketing strategy for the Leaf?

</details>

<details>
<summary><h2>Phase 2: Prepare</h2></summary>
Urška Sršen, the CCO of Bellabeat, has pointed to a specific dataset she thinks will be useful for answering the business question: [FitBit Fitness Tracker Data (Kaggle)](https://www.kaggle.com/datasets/arashnic/fitbit)

### Data Organization
The dataset is split into separate CSV files for activity, sleep, heart rate, calories and step count. It contains data on 30 users between March and May of 2016.

### Limitations
- Only 30 users - This sample size is not large enough to be representative of the population.
- No demographic data - Factors like gender would be relevant as Bellabeat's health products are designed for women.
- Only two months of data - Examining data from a longer time period would give more insight into how trends are developing in fitness tracker usage.

### Datasets Used
For this analysis, I used the datasets for daily activity, sleep, weight, heartrate and step count as I believe these are most relevant to answering the business question.

</details>

<details>
<summary><h2>Phase 3: Process</h2></summary>
### Tools Used
The process phase involves cleaning and transforming the data. For this I'll be using SQL. I chose SQL as this is a large dataset with multiple related tables. This makes SQL a more appropriate tool than spreadsheets in this instance. I will be using BigQuery as my SQL environment.

To begin, I imported the data from the CSV files into BigQuery so I could analyze it with SQL. Big Query did not automatically recognize the "Time" column of some of the datasets, and imported it as a string instead. I used the following query to parse the string as a corrected DATETIME value, then add the new values to a cleaned table. The same query was repeated on all datsets where "Time" was not automatically recognized as a datetime field in BigQuery.

```
CREATE TABLE IF NOT EXISTS `bellabeat-analysis-502600.fitbit_fitness_tracker.heartrates_cleaned` AS
  SELECT
    Id,
    PARSE_DATETIME('%m/%d/%Y %I:%M:%S %p', Time) AS Time,
    Value
  FROM
    `bellabeat-analysis-502600.fitbit_fitness_tracker.heartrates_1`
```

Next I ran the following query to get a brief overview of each table:
```
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT Id) AS unique_ids,
  MIN(ActivityDate) AS earliest_date,
  MAX(ActivityDate) AS latest_date
FROM
  `bellabeat-analysis-502600.fitbit_fitness_tracker.daily_activity_merged`
```

From this query I was able to recognise that not all tables contained the same number of unique user ids. For example, daily activity table had 33 unique ids while the weight log table only had 8.


</details>
