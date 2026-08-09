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
- Most of the datasets contain data for 33 users. Some datasets only contain data on even fewer users, e.g. the weight data was only present for 8 of the 33, and heartrate for only 14 of the 33. Due to the limited coverage these datasets will be excluded from my analysis.
- The data does not contain the user's gender. Gender would be relevant as Bellabeat's health products are designed for women.
- Only two months of data - Examining data from a longer time period would give more insight into how trends are developing in fitness tracker usage.

### Datasets Used
For this analysis, I used the datasets for daily activity, sleep, weight, heartrate and step count as I believe these are most relevant to answering the business question.

</details>

<details>
<summary><h2>Phase 3: Process</h2></summary>

### Tools Used
The process phase involves cleaning and transforming the data. For this I'll be using SQL. I chose SQL as this is a large dataset with multiple related tables, some of which have millions of rows. This makes SQL a more appropriate tool than spreadsheets in this instance. I will be using BigQuery as my SQL environment.

<details>
<summary><h3>Data Cleaning</h3></summary>
  
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
SELECT id, datetime, COUNT(*) AS row_count
FROM `fitbit_fitness_tracker.sleep`
GROUP BY id, datetime
HAVING COUNT(*) > 1;
```
Once again, I adapted and ran this query on each table in my database.
For the sleep table, I found 3 duplicate rows. At first I expected this would indicate two separate sleep sessions recorded on the same day, i.e. a nap during the day and a normal sleep session at night. To check, I ran the  following query.
```
--Select rows with the duplicate id+date pairs as identified in the previous query
SELECT *
FROM `fitbit_fitness_tracker.sleep`
WHERE (id, datetime) IN (
  (4388161847, DATE '2016-05-05'),
  (4702921684, DATE '2016-05-07'),
  (8378563200, DATE '2016-04-25')
)
ORDER BY id, datetime;
```
From these results, I can see that this was not what I expected. They were not separate sleep sessions on the same day, rather one sleep session recorded more than once. This makes sense as the sleep table comes from the sleepDay csv file from the dataset, which has already aggregated the sleep records by minute. This means it is safe for me to remove the duplicate rows without losing real data. I ran the following query to do this.
```
CREATE OR REPLACE TABLE `fitbit_fitness_tracker.sleep` AS
SELECT DISTINCT
  id,
  DATE(datetime) AS date,
  sleep_records,
  sleep_minutes,
  time_in_bed
FROM `fitbit_fitness_tracker.sleep`;
```

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
<img width="581" height="81" alt="image" src="https://github.com/user-attachments/assets/b086556e-86ab-4325-b489-526d501aacda" />

 
However, for some rows there is a large difference that cannot be excused as rounding errors. These indicate data quality issues. In these cases, the distance breakdowns may be misrepresented or missing. For cases such as the example below, I will take the total_distance as the correct value.
<img width="645" height="79" alt="image" src="https://github.com/user-attachments/assets/af980467-0236-4cb7-a03e-c9e39a34ee43" />


#### Check Data Coverage
I wanted to check how many distinct users were in each table, as a way to check if all users were represented in all tables.
```
-- How many distinct users have activity data?
SELECT COUNT(DISTINCT id) FROM `fitbit_fitness_tracker.activity`; --Result: 33

-- How many distinct users actually have weight data?
SELECT COUNT(DISTINCT id) FROM `fitbit_fitness_tracker.weight`; --Result: 8

-- How many distinct users have heartrate data?
SELECT COUNT(DISTINCT id) FROM `fitbit_fitness_tracker.heartrate`; --Result: 14

-- How many distinct users have sleep data?
SELECT COUNT(DISTINCT id) FROM `fitbit_fitness_tracker.sleep`; --Result: 24

-- How many distinct users have step data?
SELECT COUNT(DISTINCT id) FROM `fitbit_fitness_tracker.steps`; --Result: 33

-- How many distinct users have intensity data?
SELECT COUNT(DISTINCT id) FROM `fitbit_fitness_tracker.intensity`; --Result: 33
```
As we can see from the results, the weight and heartrate tables are missing a significant portion of the 33 users that are present in most other tables. The only other table that was short was sleep, however with 24/33 users being represented, I think it is acceptable to keep and use that data. I will however drop the heartrate and weight data from any further analysis. This limitation has been noted in the ask phase.
</details>

<details>
<summary><h3>Data Tranformation</h3></summary>

### Create Derived Metrics
Next, I want to use the data to derive some metrics I can use to inform the analysis. Since the Bellabeat Leaf product is the focus of this analysis, I want to inspect metrics relevant to sleep, stress and activity.

#### Sleep Efficiency
The first metric I want to create is sleep efficiency. This is defined as
```
sleep_minutes / time_in_bed
```
I used the following query to create the sleep efficiency column in the sleep table and populate it.
 ```
ALTER TABLE `fitbit_fitness_tracker.sleep`
ADD COLUMN IF NOT EXISTS sleep_efficiency FLOAT64; 
--Sleep efficiency defined as a float as the result will be a decimal representing a percentage value, e.g. 0.875 = 87.5% of time spent in bed was spent sleeping.

UPDATE `fitbit_fitness_tracker.sleep`
SET sleep_efficiency = ROUND(sleep_minutes / time_in_bed, 3) 
--Round for readability
WHERE time_in_bed > 0;
```
#### Activity Table Derived Metrics
Next up I want to create some derived metrics in the activity table that will help with the analysis later. Specifically, I think being able to compare weekdays vs weekends could be useful to gain insight into activity levels. I also want to create a column to inspect the total activity minutes per day, and to classify activity level by step count.
```
--Add derived columns to the activity table
ALTER TABLE `fitbit_fitness_tracker.activity`
ADD COLUMN IF NOT EXISTS total_active_minutes INT64,
ADD COLUMN IF NOT EXISTS activity_level STRING,
ADD COLUMN IF NOT EXISTS day_type STRING;

--Populate total active minutes
UPDATE `fitbit_fitness_tracker.activity`
SET total_active_minutes = very_active_minutes + fairly_active_minutes + lightly_active_minutes
WHERE TRUE;

--Classify each day by step count
--Step counts based on Tudor-Locke & Bassett (2004) https://pubmed.ncbi.nlm.nih.gov/14715035/
UPDATE `fitbit_fitness_tracker.activity`
SET activity_level = CASE
  WHEN steps < 5000 THEN 'Sedentary'
  WHEN steps < 7500 THEN 'Lightly Active'
  WHEN steps < 10000 THEN 'Fairly Active'
  ELSE 'Very Active'
END
WHERE TRUE;

--Flag weekday vs weekend
--EXTRACT(DAYOFWEEK) returns 1 for Sunday and 7 for Saturday
UPDATE `fitbit_fitness_tracker.activity`
SET day_type = CASE
  WHEN EXTRACT(DAYOFWEEK FROM date) IN (1,7) THEN 'Weekend'
  ELSE 'Weekday'
END
WHERE TRUE;
```
```
--Count how many days were logged per user during the date range of the dataset as a measure of user engagement with fitness tracking.
CREATE OR REPLACE TABLE `fitbit_fitness_tracker.user_engagement` AS
SELECT
  id,
  COUNT(DISTINCT date) AS days_logged,
  ROUND(
    COUNT(DISTINCT date) / 
    (SELECT COUNT(DISTINCT date) FROM `fitbit_fitness_tracker.activity`),
  2) AS days_logged_percentage
FROM `fitbit_fitness_tracker.activity`
GROUP BY id;
```
#### Summary Key Activity and Sleep Metrics
Finally I decided to create a daily_summary table that contained the key data from the activity and sleep tables.
```
--Create a summary table combining activity and sleep data. I expect this to be the most relevant to my analysis, so by creating a joined table now, I don't have to write the join again for every query in the analysis phase.
CREATE OR REPLACE TABLE `fitbit_fitness_tracker.daily_summary` AS
SELECT
  a.id,
  a.date,
  a.steps,
  a.calories,
  a.total_active_minutes,
  a.activity_level,
  a.day_type,
  s.sleep_minutes,
  s.time_in_bed,
  s.sleep_efficiency
FROM `fitbit_fitness_tracker.activity` a
LEFT JOIN `fitbit_fitness_tracker.sleep` s
  ON a.id = s.id AND a.date = s.date;
```
</details>

</details>

</details>

<details>
<summary><h2>Phase 4: Analyze</h2></summary>

#### User Engagement
To begin my analysis, I wanted to look further into the user_engagement table I created in the previous phase.
First, I wanted to understand the overall level of engagement across all users in the data.
```
-- Engagement distribution
SELECT
  id,
  days_logged,
  days_logged_percentage,
  CASE
    WHEN days_logged_percentage >= 0.75 THEN 'High'
    WHEN days_logged_percentage >= 0.4 THEN 'Medium'
    ELSE 'Low'
  END AS engagement_tier
FROM `fitbit_fitness_tracker.user_engagement`
ORDER BY days_logged_percentage DESC;

-- Count how many users per bucket (summary of above query)
SELECT
  CASE
    WHEN days_logged_percentage >= 0.75 THEN 'High'
    WHEN days_logged_percentage >= 0.4 THEN 'Medium'
    ELSE 'Low'
  END AS engagement_tier,
  COUNT(*) AS num_users
FROM `fitbit_fitness_tracker.user_engagement`
GROUP BY engagement_tier
ORDER BY engagement_tier;
```
 <img width="504" height="164" alt="image" src="https://github.com/user-attachments/assets/3b5e00c3-27c7-43a3-9028-89b0604b25ec" />

From these results, I could see that 23 of the 33 total users had a high engagement with the fitness tracker (Above 75%). 9 users had a medium level of engagement with the fitness tracker. This bucket was defined as being between 40% and 75%. However, by looking at the results of the first query, nobody in the medium bucket had engagement below 55%. There was only 1 user that fell into the low engagement category (Below 40%). From the results we can see that this user only logged 3 days over the course of the month covered by the data. I was curious about this user, so I ran another query to look further into their logging history.
```
-- Check whether the low-engagement user's activity is clustered at the start (early dropout) or spread out
SELECT date, steps, activity_level
FROM `fitbit_fitness_tracker.daily_summary`
WHERE id = 4057192912
ORDER BY date;
```
From these results we can see that the low engagement user only logged on the 12th, 13th and 15th of April. These dates are right at the beginning of the date range for this dataset. This indicates that this user dropped out of the tracking early, rather than indicating sporadic usage over a long term, such as logging once per week. This could be worth flagging as a user retention issue, however with only one user exhibiting this behaviour, it is not enough to generalize.

#### Activity Patterns
Next, I wanted to analyze the activity patterns of users using the daily_summary table I created in the process phase.
```
-- Number of days for each activity level
SELECT
  activity_level,
  COUNT(*) AS num_days,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct_of_days
FROM `fitbit_fitness_tracker.daily_summary`
GROUP BY activity_level
ORDER BY num_days DESC;

-- Weekday vs weekend
SELECT
  day_type,
  ROUND(AVG(steps), 0) AS avg_steps,
  ROUND(AVG(total_active_minutes), 1) AS avg_active_minutes,
  COUNT(*) AS num_days
FROM `fitbit_fitness_tracker.daily_summary`
GROUP BY day_type;

```
Here are the results of the first query:
 <img width="698" height="209" alt="image" src="https://github.com/user-attachments/assets/a17e83d1-c0dc-4c16-b065-2d3995755255" />

As we can see, the categories with the highest percentage of days are Very Active and Sedentary, the two extremes of the scale. This suggests that users mostly have days where they do lots of activity or very little. The “in between” categories of Lightly Active and Fairly Active have far less representation.

Here are the results of the second query:
 <img width="892" height="129" alt="image" src="https://github.com/user-attachments/assets/1e28c735-7df5-4e35-b732-af276e708c60" />

From these results, we can see that weekends and weekdays have almost identical average step counts and average active minutes across all users. This was quite a surprise to me, as before running the query I expected users to be more active on the weekends when most people are off work and have time for exercise.

#### Activity-Sleep Correlation
I want to investigate the correlation between activity levels and sleep. My assumption is that users who are more active will have better sleep efficiency.
```
SELECT
  CORR(total_active_minutes, sleep_efficiency) AS corr_activeminutes_efficiency,
  CORR(steps, sleep_efficiency) AS corr_steps_efficiency,
  CORR(total_active_minutes, sleep_minutes) AS corr_activeminutes_sleepminutes,
  COUNT(*) AS n
FROM `fitbit_fitness_tracker.daily_summary`
WHERE sleep_efficiency IS NOT NULL;

SELECT
  activity_level,
  ROUND(AVG(sleep_efficiency), 3) AS avg_sleep_efficiency,
  ROUND(AVG(sleep_minutes), 0) AS avg_sleep_minutes,
  COUNT(*) AS num_days
FROM `fitbit_fitness_tracker.daily_summary`
WHERE sleep_efficiency IS NOT NULL
GROUP BY activity_level
ORDER BY avg_sleep_efficiency DESC;
```
Here are the results of the first query:
 <img width="940" height="65" alt="image" src="https://github.com/user-attachments/assets/dd1fde58-4c49-4216-a893-3d4407285c9f" />

Once again, the results go against my expectations. Here we see that all correlations are mostly negligible (~0.04, ~0.11, ~0.07). This indicates that activity level doesn’t have a correlation with sleep based on our dataset. Note: n = 410 because the query specifies “WHERE sleep_efficiency IS NOT NULL”. Since the daily_summary table summarizes data from the activity table’ s 33 users with the sleep table’s 24 users, there are null rows for any user who has activity data but no sleep data.

Here are the results of the second query:
<img width="894" height="214" alt="image" src="https://github.com/user-attachments/assets/86f00987-ba30-4e0b-9654-8fad2f95944b" />

And as above, the results do not show any improvement in sleep efficiency on a day where a user is more active. In fact, the “Very Active” days have the lowest sleep efficiency, at 0.902. However, even this is an insignificant difference compared to the other activity levels. With these results I’m confident that activity level does not predict sleep efficiency.

#### Engagement-Activity Relationship
Finally, I want to check if users with high engagement are more active. I would assume this is the case as someone who is enthusiastic about fitness will be more active and also more consistent with logging and tracking their activity. 
```
-- Does engagement level relate to activity level? i.e. do highly-engaged
-- users also tend to be more physically active, or is engagement
-- (logging) independent of actual activity?
SELECT
  e.engagement_tier,
  ROUND(AVG(d.steps), 0) AS avg_steps,
  ROUND(AVG(d.total_active_minutes), 1) AS avg_active_minutes,
  COUNT(*) AS num_days
FROM `fitbit_fitness_tracker.daily_summary` d
JOIN (
  SELECT id,
    CASE
      WHEN days_logged_percentage >= 0.75 THEN 'High'
      WHEN days_logged_percentage >= 0.4 THEN 'Medium'
      ELSE 'Low'
    END AS engagement_tier
  FROM `fitbit_fitness_tracker.user_engagement`
) e ON d.id = e.id
GROUP BY e.engagement_tier
ORDER BY avg_steps DESC;
```
Here are the results:
<img width="889" height="164" alt="image" src="https://github.com/user-attachments/assets/0252c2ea-84c4-4646-913e-ec05c4d8c746" />

From these results we can see that the users with high engagement with logging their activity also have the highest average step count and active minutes count. Medium and low engagement users have noticeably lower activity levels. This shows that user engagement and activity are linked. It is important to note that causation cannot be determined from this data alone. Perhaps highly active users are more motivated to log consistently, or maybe consistent logging may encourage more activity. Either way, this result shows us that user engagement is a metric worth focusing on.
</details>

<details>
<summary><h2>Phase 5: Share</h2></summary>

To communicate the findings from the Analyze phase to a non-technical audience, I created a set of visualizations in Tableau Desktop, connecting directly to my cleaned and transformed BigQuery tables. Given the audience for this case study is the Bellabeat executive team, I focused each visualization on a single, clearly stated finding, using minimal chart types and a consistent visual style (a single accent color palette, direct data labels, and plain-language titles) rather than dense, multi-metric dashboards.

I produced five visualizations, each corresponding to one finding from the Analyze phase:

### Engagement Tiers
This visualization shows the distribution of users across High, Medium, and Low engagement tiers, based on the percentage of days each user logged data. The large majority of users fall into the High tier, indicating that consistent tracker use is common once the user has a tracking device.

<img width="500" height="400" alt="Engagement" src="https://github.com/user-attachments/assets/c4c94e5c-bdca-4db6-baf4-58979844225a" />


### Activity Level Distribution
This visualization shows the number of user-days at each activity level (Sedentary through Very Active). The distribution is bimodal, with users more often having distinctly active or distinctly inactive days rather than a smooth middle ground.

 <img width="500" height="400" alt="Activity" src="https://github.com/user-attachments/assets/426bbe44-ef27-401c-b84d-bf17344dea77" />


### Weekday vs. Weekend Activity
This visualization compares average steps and active minutes between weekdays and weekends. Both measures are nearly identical across the week, contradicting my expectation that activity would rise on the weekends.

 <img width="500" height="400" alt="Weekday vs Weekend" src="https://github.com/user-attachments/assets/198eb348-cfe9-4c54-ae69-54f29ef230e7" />


### Sleep Efficiency by Activity Level 
This visualization compares average sleep efficiency across activity level groups. Efficiency is nearly constant regardless of activity level, indicating no meaningful relationship between physical activity and sleep quality in this dataset.

<img width="500" height="400" alt="Sleep" src="https://github.com/user-attachments/assets/6e1be908-51c6-4274-bd3d-e899f6b5e3d4" />


### Engagement vs. Activity
In my opinion this is the most important finding. This visualization compares average steps and active minutes across engagement tiers. Both metrics increase substantially with engagement tier, suggesting that consistent device usage and physical activity are closely linked. However, causation cannot be determined from this data alone.

 <img width="500" height="400" alt="Engagement vs Activity" src="https://github.com/user-attachments/assets/d78e6588-bc4e-465d-8637-eb17611b29fa" />

</details>
