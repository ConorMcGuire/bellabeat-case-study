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

<details>
<summary><h2>Phase 3: Process</h2></summary>

### Tools Used
The process phase involves cleaning and transforming the data. For this I'll be using SQL. I chose SQL as this is a large dataset with multiple related tables, some of which have millions of rows. This makes SQL a more appropriate tool than spreadsheets in this instance. I will be using BigQuery as my SQL environment.

#### Data Cleaning
To begin, I imported the data from the CSV files into BigQuery so I could analyze it with SQL. Big Query did not automatically recognize the date/time columns of some of the datasets and imported it as a string instead. I used the following query to parse the string as a corrected DATETIME value, then add the corrected values to a new column. 
```
--Add a new empty column called date
ALTER TABLE `fitbit_fitness_tracker.heartrate`
ADD COLUMN IF NOT EXISTS datetime DATETIME;

--Parse each date from a string to a datetime and insert into new column
UPDATE `fitbit_fitness_tracker.heartrate`
SET datetime = PARSE_DATETIME('%m/%d/%Y %I:%M:%S %p', time)
WHERE TRUE; --Update requires where clause, but we want to include all rows

--Check that the new column has been populated correctly by comparing to the old string date column
SELECT
  time,
  datetime
FROM
  `fitbit_fitness_tracker.heartrate`
LIMIT 10;
```
After manually confirming that the new column had been populated correctly, I dropped the original string format column.
```
ALTER TABLE fitbit_fitness_tracker.heartrate
DROP COLUMN time;

```
The same query was repeated on all datsets where the date or time was not automatically recognized as a datetime field in BigQuery.

#### Check for Nulls
Next, I checked if the heartrate table contained null values using the following query.
```
SELECT
  COUNT(*) AS total_rows, --Total rows in table
--For each row, sum how many nulls are present
  SUM(CASE WHEN id IS NULL THEN 1 ELSE 0 END) AS null_ids, 
  SUM(CASE WHEN datetime IS NULL THEN 1 ELSE 0 END) AS null_times,
  SUM(CASE WHEN value IS NULL THEN 1 ELSE 0 END) AS null_values,

FROM `fitbit_fitness_tracker.heartrate`;
```
I performed the same check on each of the activity, heartrate, intensity, sleep, steps and weight tables.

### Check for Duplicates
Then, I checked if any tables contained duplicate rows using the following query.
```
SELECT
  id,
  datetime,
  COUNT(*) AS duplicate_count
FROM `fitbit_fitness_tracker.heartrate`
GROUP BY id, datetime
HAVING COUNT(*) > 1;
```
Once again, I adapted and ran this query on each table in my database.

#### Check for Outliers and Invalid Data
I checked the activity table for outliers and invalid data using the following query.
```
--Check the min and max step counts and calorie counts
SELECT
  MIN(steps) AS min_steps,  --result 0
  MAX(steps) AS max_steps,  --result 36019
  MIN(calories) AS min_calories,  --result 0
  MAX(calories) AS max_calories   --result 4900
FROM `fitbit_fitness_tracker.activity`;

--Check the min and max heartrate values
SELECT
  MIN(value) AS min_heartrate, --result 36
  MAX(value) AS max_heartrate  --result 203
FROM `fitbit_fitness_tracker.heartrate`;

-- Check how many sleep counts fall outside 5-12 hours, typical range for adults
SELECT COUNT(*)
FROM `fitbit_fitness_tracker.sleep`
WHERE sleep_minutes < 300 OR sleep_minutes > 720; --result 54

--Check how many weights fall outside 40kg and 200kg, a typical range for adults
SELECT COUNT(*)
FROM `fitbit_fitness_tracker.weight`
WHERE weight_kg < 40 OR weight_kg > 200; --result 0
```
The result of these queries allow me to quickly examine that the data fall within the expected range. For example, we know that neither steps nor calories should be negative. The max step count was 36,019, which is a valid value. The max calories were 4900, which while high, is not high enough for me to consider it an outlier. The min and max heartrates are realistic numbers, between 30-220 bpm is typical. 54 sleep records being outside the 5-12 hour range is acceptable, as it is the typical range but not a strict limit. 
One result that stood out was that the minimum step count and minimum calories were zero. I decided to investigate further. I ran the following query to inspect some of the suspect rows.
```
SELECT
  id,
  date,
  steps,
  calories,
  very_active_minutes,
  fairly_active_minutes,
  lightly_active_minutes,
  sedentary_minutes
FROM `fitbit_fitness_tracker.activity`
WHERE steps = 0 OR calories = 0
LIMIT 10;
```
This result revealed that in rows where the steps were 0, sedentary minutes were 1440 (24 hours). 
The activity table only shows the total steps for a day. The steps table shows the number of steps per hour. I decided to use the steps per hour to recalculate the daily steps and check if the zero step rows persisted. To do this I used the following query.
```
--Create a new column in the activity table where the sum of hourly steps will be added
ALTER TABLE `fitbit_fitness_tracker.activity`
ADD COLUMN IF NOT EXISTS total_steps_from_hourly INT64;

--Update the new row to contain the sum of steps for each day for each user
UPDATE `fitbit_fitness_tracker.activity` a
SET total_steps_from_hourly = (
  SELECT SUM(steps)
  FROM `fitbit_fitness_tracker.steps` s
  WHERE s.Id = a.Id AND DATE(s.datetime) = DATE(a.date)
)
WHERE TRUE;

--Verify if the rows that had 0 steps in the orignal activity table have 0 steps when calculated from the hourly values
SELECT
  id,
  date,
  steps,
  total_steps_from_hourly
FROM `fitbit_fitness_tracker.activity`
WHERE steps = 0
LIMIT 10;
```
From the results of this query, I can see that the step counts are still zero. This means that the 0 in the activity table is not an error. The most likely explanation is that zero is a default value that is entered when no steps are recorded. I decided that these rows would not be useful to my analysis, so I removed them from the table using the following query.
```
--Delete rows with zero steps
DELETE FROM `fitbit_fitness_tracker.activity`
WHERE steps = 0 AND total_steps_from_hourly = 0;
```
Next, I checked the logical consistency of the total_distance column by adding each of the breakdown columns together and comparing the result.
```
SELECT
  --Round total distance and total from breakdown to 2 decimal places for readability
  ROUND(total_distance,2) AS total_distance,
  ROUND((very_active_distance +
  moderately_active_distance +
  light_active_distance +
  sedentary_active_distance), 2) AS total_from_breakdown,
  --Due to the high decimal accuracy of the individual distance by activity level columns, the total from breakdown may not be exactly equal. In this case I use ABS to get the difference between the two values to make sure they are within an acceptable margin of eachother.
  ABS(total_distance - 
  (very_active_distance +
  moderately_active_distance +
  light_active_distance +
  sedentary_active_distance)) AS difference
FROM
  `bellabeat-analysis-502600.fitbit_fitness_tracker.activity`
WHERE
  ABS(
    total_distance - 
    (very_active_distance +
    moderately_active_distance +
    light_active_distance +
    sedentary_active_distance)
  ) > 0.1 --I used 0.1 as the acceptable threshold of difference between the two values.
ORDER BY
  difference ASC;
```
From the results of this query, we can see that some rows are quite close. For small differences (close to 0.1) we can assume that these are due to floating-point precision errors (rounding errors in calculations), as per the example below. 
 
However, for some rows there is a large difference that cannot be excused as rounding errors. These indicate data quality issues. In these cases, the distance breakdowns may be misrepresented or missing. For cases such as the example below, I will take the total_distance as the correct value.
 
</details>
