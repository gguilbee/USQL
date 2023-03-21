# USQL
Dynatrace USQL (User Session Query Language) Documentation

## Table of contents
- [Introduction](#Introduction)
- [Timeframes](#Timeframes)
- [Keywords and Conventions](#Keywords-and-Conventions)
- Powerup List:
    - [Disclaimer](#Disclaimer)
    - [Tooltips](#Tooltips)
    - [Colorize](#Colorize)
    
## Introduction

Dynatrace captures detailed user session data each time a user interacts with your monitored application. This data includes all user actions and high level performance data. Using either the Dynatrace API or Dynatrace User Sessions Query Language (USQL), you can easily run powerful queries, segmentations, and aggregations on this captured data. To assist you, this topic provides detail about keywords and functions, syntax, working with Real User Monitoring tables, automated queries, and more.

## Timeframes

The data in Elasticsearch should always be accessed with a timeframe as it can become very costly to access large timeframes due to the amount of potential single matches to queries. Therefore there always needs to be a timeframe selected. This fits into the Dynatrace WebUI and it's global timeframe selector, which always provides a timeframe selection.

Therefore we designed the query language to always receive the timeframe as separate input to the framework outside of the actual query and therefore usually the timeframe is not part of the query itself.

You can however use the time-fields like starttime and endtime for selecting and for functions, e.g. HOUR(starttime) to find out when during the day most user sessions are starting.

## Keywords and Conventions

The following keywords are defined as part of the query language:
```
AND, AS, ASC, BETWEEN, BY, DESC, DISTINCT, FALSE, FILTER, FROM, FUNNEL, GROUP, IN, IS, JSON, LIMIT, NOT, NULL, OR, ORDER, SELECT, STARTSWITH, TRUE, WHERE
```

The following functions are defined as part of the query language:
```
SUM, MAX, MIN, AVG, MEDIAN, COUNT, YEAR, MONTH, DAY, HOUR, MINUTE, DATETIME, TOP, PERCENTILE
```

Keywords, functions and column names are case insensitive. String-matches in WHERE conditions are done case sensitive.

## Query Language Syntax

A query is built out of the following parts
```  
SELECT <columns> FROM <table> WHERE <condition> GROUP BY <grouping> ORDER BY <ordering>
```
The only mandatory elements are `SELECT <columns>` and `FROM <table>`

#### Example
```
SELECT ip, browserType, userId, city, AVG(userActionCount) AS "Average user action count", AVG(duration) AS "Average duration", count(*) AS "Sessions", SUM(totalErrorCount) AS "Errors" 
FROM usersession 
WHERE ip between '52.179.11.1' and '52.179.11.255'
```
### SELECT "columns"
Selects one or more columns from the given data table or aggregation functions from the set of supported functions.
```
columns: [DISTINCT|FUNNEL] <column>, <column>, ... | function(<parameter>) | <column> AS <alias> | JSON
```
The modifiers `DISTINCT` and `FUNNEL` change the way the columns are interpreted and what values will be allowed for the columns.
Be aware that the `FUNNEL` modifier also requires additional parenthesis.

#### Example
```
SELECT country, city, browserfamily FROM usersession
SELECT DISTINCT country, city, useractioncount FROM usersession
SELECT country, city, avg(duration) AS average FROM usersession GROUP BY country, city
SELECT FUNNEL (useraction.name = "AppStart",useraction.name = "searchJourney", useraction.name = "bookJourney") FROM usersession
```
### DISTINCT
The `DISTINCT` modifier will add implicit grouping.

For distinct queries, you cannot use `SELECT *`,  or the `JSON` keyword.

Grouping functions, date/time functions and field names in the columns will be added to the grouping, metric functions will be added ungrouped.

#### Example
```
SELECT DISTINCT country, city, COUNT(*) FROM usersession
```
is the same as
```
SELECT country, city, COUNT(*) FROM usersession GROUP BY country, city
```

### FUNNEL
The FUNNEL modifier allows to use a predefined funnel format for the query. It changes the syntax of the query to
```
SELECT FUNNEL (<condition> AS <alias>, <condition>, ...) FROM <table> WHERE <condition>
```
For funnel queries, you cannot use `SELECT *`, functions, or the keywords like `JSON` and also no `GROUP BY, ORDER BY or LIMIT` statements

*Limitation: Currently, `GROUP BY, ORDER BY or LIMIT` statements are not allowed in the funnels. These functionality might be added in the future.* 

#### Example
```
SELECT FUNNEL (useraction.name = "AppStart",useraction.name = "searchJourney", useraction.name = "bookJourney") FROM usersession
```
is the same as running 3 count(*) queries
```
SELECT COUNT(*) FROM usersession where useraction.name = "AppStart"
SELECT COUNT(*) FROM usersession where useraction.name = "AppStart" AND useraction.name = "searchJourney"
SELECT COUNT(*) FROM usersession where useraction.name = "AppStart" AND useraction.name = "searchJourney" AND useraction.name = "bookJourney"
```
### FROM "table"
Only one table can be specified. Tables for RUM data are currently usersession and useraction.

#### Example
```
SELECT country, city, browserfamily FROM usersession
SELECT name, starttime, endtime, duration FROM useraction ORDER BY duration DESC
```

WHERE <condition>
Multiple conditions can be combined using boolean logic, the right-hand side of conditions can only be a value, so no comparison between two fields is possible.
"field" always refers to a field, so do not use functions or aliases here.



condition: (condition AND condition) | (condition OR condition) | field IN(...) | field IS <value> | field IS NULL | field = <value> | field > <value> | field < <value> | field <> <value> | field NOT <value> | field BETWEEN <value> AND <value> | ...
Example
SELECT country, city, browserfamily FROM usersession WHERE country = 'Albania' AND screenWidth > 1000
SELECT top(country, 20), top(city, 20), TOP(duration, 10), avg(duration) AS average FROM usersession WHERE duration BETWEEN 1000 AND 2000
