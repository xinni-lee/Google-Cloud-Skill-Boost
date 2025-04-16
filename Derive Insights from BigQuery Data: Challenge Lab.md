## Derive Insights from Data with BigQuery: Challenge Lab
Dataset for analysis: bigquery-public-data.covid19_open_data.covid19_open_data
<br/><br/>

### Query 1: Total Confirmed Cases
What was the total count of confirmed cases on April 15, 2020?
```sql
SELECT SUM(cumulative_confirmed) AS total_cases_worldwide
FROM `bigquery-public-data.covid19_open_data.covid19_open_data` 
WHERE date = '2020-06-15'
```


### Query 2: Worst affected areas
How many states in the US had more than 250 deaths on April 15, 2020?
```sql
WITH death_by_states AS (
  SELECT subregion1_name AS state, SUM(cumulative_deceased) AS death_count
  FROM `bigquery-public-data.covid19_open_data.covid19_open_data` 
  WHERE date = '2020-04-15' AND country_name = 'United States of America' AND subregion1_name IS NOT NULL
  GROUP BY subregion1_name
)

SELECT COUNT(*) as count_of_states
FROM death_by_states
WHERE death_count > 250;
```

### Query 3: Identify hotspots
List all the states in the United States of America that had more than 2000 confirmed cases on April 15, 2020?
```sql
SELECT subregion1_name AS state, SUM(cumulative_confirmed) AS total_confirmed_cases
FROM `bigquery-public-data.covid19_open_data.covid19_open_data` 
WHERE date = '2020-04-15' AND country_name = 'United States of America' AND subregion1_name IS NOT NULL
GROUP BY subregion1_name
HAVING total_confirmed_cases > 2000
ORDER BY total_confirmed_cases DESC;
```

### Query 4: Fatality ratio
What was the case-fatality ratio in Italy for the month of April 2020?<br/>
Case-fatality ratio here is defined as (total deaths / total confirmed cases) * 100.
```sql
SELECT
  SUM(cumulative_confirmed) AS total_confirmed_cases,
  SUM(cumulative_deceased) AS total_deaths,
  (SUM(cumulative_deceased) / SUM(cumulative_confirmed)*100) AS case_fatality_ratio
FROM `bigquery-public-data.covid19_open_data.covid19_open_data` 
WHERE date BETWEEN '2020-04-01' AND '2020-04-30' 
      AND country_name = 'Italy';
```

### Query 5: Identifying specific day
On what day did the total number of deaths cross 14000 in Italy?
```sql
SELECT date
FROM `bigquery-public-data.covid19_open_data.covid19_open_data` 
WHERE country_name = 'Italy' AND cumulative_deceased > 14000
ORDER BY date
LIMIT 1;
```

### Query 6: Finding days with zero net new cases
Update the query to identify the number of days in India between 21, Feb 2020 and 13, March 2020 when there were zero increases in the number of confirmed cases. 
```sql
WITH india_cases_by_date AS (
  SELECT
    date,
    SUM(cumulative_confirmed) AS cases
  FROM
    `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE
    country_name="India"
    AND date between '2020-02-21' and '2020-03-13'
  GROUP BY
    date
  ORDER BY
    date ASC
 )

, india_previous_day_comparison AS
(SELECT
  date,
  cases,
  LAG(cases) OVER(ORDER BY date) AS previous_day,
  cases - LAG(cases) OVER(ORDER BY date) AS net_new_cases
FROM india_cases_by_date
)

SELECT COUNT(date)
FROM india_previous_day_comparison
WHERE net_new_cases = 0;
```

### Query 7: Doubling rate
Write a query to find out the dates on which the confirmed cases increased by more than 20% compared to the previous day (indicating doubling rate of ~ 7 days) in the US between the dates March 22, 2020 and April 20, 2020.
```sql
WITH us_cases_by_date AS (
  SELECT
    date,
    SUM(cumulative_confirmed) AS cases
  FROM
    `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE
    country_name="United States of America"
    AND date between '2020-03-22' and '2020-04-20'
  GROUP BY
    date
  ORDER BY
    date ASC
 )
, us_previous_day_comparison AS (
SELECT
  date,
  cases,
  LAG(cases) OVER(ORDER BY date) AS previous_day,
  cases - LAG(cases) OVER(ORDER BY date) AS net_new_cases,
  (cases - LAG(cases) OVER(ORDER BY date))*100/LAG(cases) OVER(ORDER BY date) AS percentage_increase
FROM us_cases_by_date
)

SELECT 
  date AS Date, 
  cases AS Confirmed_Cases_On_Day, 
  previous_day AS Confirmed_Cases_Previous_Day,
  percentage_increase AS Percentage_Increase_In_Cases
FROM us_previous_day_comparison
WHERE percentage_increase > 20
ORDER BY Date;
```

### Query 8: Recovery rate
Build a query to list the recovery rates of countries arranged in descending order (limit to 20) upto the date May 10, 2020.<br/>
Restrict the query to only those countries having more than 50K confirmed cases.
```sql
WITH cases_by_country AS (
  SELECT
    country_name AS country,
    SUM(cumulative_confirmed) AS cases,
    SUM(cumulative_recovered) AS recovered_cases
  FROM
    `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE
    date = '2020-05-10' 
  GROUP BY
    country_name
 )
, recovered_rate AS (
  SELECT
    country, cases, recovered_cases,
    (recovered_cases/cases*100) AS recovery_rate
FROM cases_by_country
)

SELECT country, cases AS confirmed_cases, recovered_cases, recovery_rate
FROM recovered_rate
WHERE cases > 50000
ORDER BY recovery_rate DESC
LIMIT 20;
```

### Query 9: CDGR - Cumulative daily growth rate
Update the following query to calculate the CDGR on April 15, 2020 (Cumulative Daily Growth Rate) for France since the day the first case was reported.The first case was reported on Jan 24, 2020.<br/>
The CDGR is calculated as: ((last_day_cases/first_day_cases)^1/days_diff)-1)<br/>
<br/>
Where:<br/>
•	last_day_cases is the number of confirmed cases on May 10, 2020<br/>
•	first_day_cases is the number of confirmed cases on Jan 24, 2020<br/>
•	days_diff is the number of days between Jan 24 - May 10, 2020<br/>

```sql
WITH france_cases AS (
  SELECT
    date,
    SUM(cumulative_confirmed) AS total_cases
  FROM
    `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE
    country_name="France"
    AND date IN ('2020-01-24',
      '2020-04-15')
  GROUP BY
    date
  ORDER BY
    date)
, summary as (
SELECT
  total_cases AS first_day_cases,
  LEAD(total_cases) OVER(ORDER BY date) AS last_day_cases,
  DATE_DIFF(LEAD(date) OVER(ORDER BY date), date, DAY) AS days_diff
  FROM france_cases
  LIMIT 1
)

SELECT
  first_day_cases,
  last_day_cases,
  days_diff,
  POW(last_day_cases/first_day_cases, (1/days_diff)) - 1 AS cdgr
FROM summary;
```

### Query 10: Create a Looker Studio report
Create a Looker Studio report that plots the following for the United States:<br/>
•	Number of Confirmed Cases<br/>
•	Number of Deaths<br/>
•	Date range : 2020-03-15 to 2020-04-27<br/>
<br/>
Brief steps: Looker Studio (incognito) > Create > Data Source > (sign in) > BigQuery > Authorize > Custom Query (choose Billing Project ID):<br/>
```sql
SELECT date, SUM(cumulative_confirmed) AS country_cases,
    SUM(cumulative_deceased) AS country_deaths
FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE date BETWEEN '2020-03-15' AND '2020-04-27'
  AND country_name = 'United States of America'
GROUP BY date;
```
