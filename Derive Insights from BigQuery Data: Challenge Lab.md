## Derive Insights from Data with BigQuery: Challenge Lab
Dataset for analysis: bigquery-public-data.covid19_open_data.covid19_open_data



### Query 1: Total Confirmed Cases
```sql
## What was the total count of confirmed cases on June 15, 2020
SELECT SUM(cumulative_confirmed) AS total_cases_worldwide
FROM `bigquery-public-data.covid19_open_data.covid19_open_data` 
WHERE date = '2020-06-15'
```


### Query 2: 
