---
title: "How to SQL in BigQuery"
description: "An attempt to simplify BigQuery SQL queries to the bare essentials"
toc: true
draft: false
date: '2022-11-07'
categories:
- sql
- GCP
comments:
  giscus:
    repo: stantonius/home
---



## Objective

1. Simplify SQL to its barebones essentials that were important for my SQL refresh and are relevent to BigQuery.

## Background

I have avoided learning SQL because I have never needed to use it extensively before. This has recently changed - below ar emy study notes of concepts I needed to recall in order to actually contribute to the team.

## Sample Data

Because most of the work I am doing is based in BigQuery, we will use a public dataset in BigQuery called [london bicycles](https://console.cloud.google.com/marketplace/product/greater-london-authority/london-bicycles)

## Concepts

### Data Discovery

Commands to get a sense of the data before you start running queries.

#### Get all tables in dataset

```SQL
SELECT
  *
FROM
  `bigquery-public-data.london_bicycles.INFORMATION_SCHEMA.TABLES`
```

> The use of `INFORMATION_SCHEMA`  is a set of read-only views for the dataset metadata. In MySQL you typically see it lowercase as `informatioin_schema`

#### Get all columns from table

```SQL
SELECT
  *
FROM
  `bigquery-public-data.london_bicycles.INFORMATION_SCHEMA.COLUMNS`
WHERE
  table_name = "cycle_stations"
```


#### Count the number of rows in the dataset. 

```SQL
SELECT
  COUNT(rental_id) AS row_count
FROM
  `bigquery-public-data.london_bicycles.cycle_hire`;
```

| row_count |
| --------- |
| 24369201  |

> Doing calculations in SQL/BigQuery is **fast** - we were able to count 24+ million rows of data in 418ms. 

You could also use wildcards:

```SQL
SELECT
  COUNT(*)
FROM
  `bigquery-public-data.london_bicycles.cycle_hire`
```

#### Limiting response size

```SQL
SELECT
  *
FROM
  `bigquery-public-data.london_bicycles.cycle_stations`
LIMIT
  10
```

Very helpful when playing around with the data - only return a few rows, and the queries are very fast.

### Joins

\* *There are so many tutorials, blogs, and images that describe this concept so I won't go into much detail here*

Add field(s) from one table to another table. For example, say we want to get every station's `latitude` and `longitude` for each ride in order to (eventually) calculate the distance travelled.

```SQL
SELECT
  hires.start_station_name,
  stations.latitude as start_latitude,
  stations.longitude as start_longitude
FROM
  `bigquery-public-data.london_bicycles.cycle_hire` AS hires
JOIN
  `bigquery-public-data.london_bicycles.cycle_stations` AS stations
ON
  CAST(hires.start_station_logical_terminal as string) = stations.terminal_name
```


### Use Aliasing

You can rename any column, table in SQL by appending  `as` command. You should use these whereever possible as it improves readability. In fact, you don't even need the `AS` keyword:

```SQL
-- ...
FROM
  `bigquery-public-data.london_bicycles.cycle_hire` hires  -- table aliased to hires without AS
JOIN
  `bigquery-public-data.london_bicycles.cycle_stations` AS stations  -- table aliased to stations
-- ...
```

### Having

I did not know what the field `start_station_logical_terminal` meant, but I suspected it was just a terminal ID. Therefore I wanted to calculate the *number of distinct `start_station_logical_terminal` for each `start_station_name` greater than 1*. The issue is that you **cannot use `WHERE` following an aggregation function**. Therefore we need to use the keyword **`HAVING`**

```SQL
SELECT
  start_station_name,
  start_station_logical_terminal,
  COUNT(DISTINCT start_station_logical_terminal) AS count_logical_terminal
FROM
  `bigquery-public-data.london_bicycles.cycle_hire` AS hires
GROUP BY
  start_station_name,
  start_station_logical_terminal
HAVING
  count_logical_terminal > 1;
```


### Aggregations

Notice the difference between these two techniques: `GROUP BY` **reduces the number of rows** whereas `PARTITION BY` keeps the same number of rows but adds an aggregated value

#### Group By

```SQL
SELECT
  start_station_name, COUNT(start_station_name) as total_num_rides
FROM
  `bigquery-public-data.london_bicycles.cycle_hire`
GROUP BY start_station_name
LIMIT 1000
```

| start_station_name                  | total_num_rides |
| ----------------------------------- | --------------- |
| Parson's Green , Parson's Green     | 35856           |
| Empire Square, The Borough          | 32228           |
| Kensington Olympia Station, Olympia | 15876           |
| Bury Place, Holborn                 | 29607           |
| Russell Square Station, Bloomsbury  | 48007           |

#### Partition By

```SQL
SELECT
  start_station_name, start_date, COUNT(start_station_name) OVER (PARTITION BY start_station_name) as total_num_rides
FROM
  `bigquery-public-data.london_bicycles.cycle_hire`
LIMIT 1000
```

| start_station_name       | start_date                     | total_num_rides |
| ------------------------ | ------------------------------ | --------------- |
| Coram Street, Bloomsbury | 2016-09-05 20:49:00.000000 UTC | 9205            |
| Coram Street, Bloomsbury | 2016-09-04 16:02:00.000000 UTC | 9205            |
| Coram Street, Bloomsbury | 2016-09-03 16:00:00.000000 UTC | 9205            |
| Coram Street, Bloomsbury | 2016-09-02 14:39:00.000000 UTC | 9205            |

The rows stayed the same - we just added a summary calculation on the end of each row

##### Window Functions

The above is an example of a **window function** - it is designated by the **`OVER`** keyword. The general syntax of a window function is:

```SQL
window_function (expression)
OVER(
	[PARTITION BY partition_clause]
	[ORDER BY order_clause]
)
```

The `PARTITION BY` and `ORDER BY` clauses are optional, but one or the other needs to exist. A good use case for window functions is demonstrated [here](https://www.youtube.com/watch?v=xFeOVIIRyvQ), where they calculate a *running total partitioned by date*

### Subqueries

I have grouped 3 different types of queries under this one umbrella because they all allow you to:

1. Create an initial query
2. Query the results of the first query

The major difference between them is their persistence

#### Subquery & CTE

Don't exist longer than the duration of the query; they are **stored in memory**.

These are extremely similar and perform about the same. For straightfoward subqueries, using a subquery or a CTE doesn't make much of a difference (its a matter of personal preference). You might consider using a CTE [if your subquery involves multiple subqueries](https://medium.com/analytics-vidhya/lets-talk-about-sql-part-7-242364486a0f) - a CTE is generally easier to read and understand for complex subqueries.

Personally I prefer CTEs as they are easier to understand the logic and the upfront creation of aliases. For example, see below for a CTE

```SQL
WITH
  hires AS (   -- first CTE/subquery
  SELECT
    *
  FROM
    `bigquery-public-data.london_bicycles.cycle_hire`
  LIMIT
    100 ),
  stations AS (    -- second CTE/subquery
  SELECT
    *
  FROM
    `bigquery-public-data.london_bicycles.cycle_stations` ),
  final AS (    -- final CTE/subquery
  SELECT
    *
  FROM
    hires
  JOIN
    stations
  ON
    CAST(hires.start_station_logical_terminal AS string) = stations.terminal_name )
SELECT    -- a select statement must be the final argument 
  *
FROM
  final
```

Notice a few things:

* The `WITH` keyword indicates this is a CTE
* There are 3 CTEs/subqueries that are **indented under `WITH`**
* The CTE ends with a `SELECT` statement
* The alilases for each subquery are stated up front, which I find easier to read

#### Temp Tables

Exist for the **duration of the session**; they actually are stored to disk in the same way other tables are. However they are "deleted" when the session ends.

Therefore these make sense when you need to reference a subquery in multiple places.

### Functions

#### Delivered Functions

There are a boatload of functions that are available in SQL and of course in BigQuery - `COUNT` and `SUM` are examples of these functions, but there are also string functions (that modify text) and many others.

Instead of going through them here, I find the [BigQuery docs](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators) are very good at explaining all of the things you can do directly in SQL

> Why are these important? Because we should be doing as much data wrangling as possible in SQL (as a general rule). A diverse set of functions allow us to do this directly in SQL and not Python

#### Custom Functions and Procedures

Coming soon