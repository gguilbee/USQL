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
### SELECT &lt;columns&gt;
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
### FROM &lt;table&gt;
Only one table can be specified. Tables for RUM data are currently usersession and useraction.

#### Example
```
SELECT country, city, browserfamily FROM usersession
SELECT name, starttime, endtime, duration FROM useraction ORDER BY duration DESC
```
### WHERE &lt;condition&gt;
Multiple conditions can be combined using boolean logic, the right-hand side of conditions can only be a value, so no comparison between two fields is possible.
"field" always refers to a field, so do not use functions or aliases here.
```
condition: (condition AND condition) | (condition OR condition) | field IN(...) | field IS <value> | field IS NULL | field = <value> | field > <value> | field < <value> | field <> <value> | field NOT <value> | field BETWEEN <value> AND <value> | ...
```
#### Example
```
SELECT country, city, browserfamily FROM usersession WHERE country = 'Albania' AND screenWidth > 1000
SELECT top(country, 20), top(city, 20), TOP(duration, 10), avg(duration) AS average FROM usersession WHERE duration BETWEEN 1000 AND 2000
```

### GROUP BY &lt;grouping&gt;
Whenever fields are aggregated, a corresponding GROUP BY needs to be specified to indicate how the aggregation should be performed.

  `grouping: <column>, ...`
Limitation: Currently, functions are not allowed in the `GROUP BY` clause. For functions like `AVG`, this is obvious; for date functions this functionality might be added in the future. For the moment, e.g. if you want to group by month, you have to specify an alias.

#### Example
```
SELECT city, count(*) FROM usersession GROUP BY city
SELECT MONTH(starttime) as month, count(*) FROM usersession GROUP BY month
```

### LIMIT &lt;limit&gt;
Allow to limit the number of results that are returned, e. g. this can be used for only selecting the top 10 results when it is combined with ordering.

Some upper limit will be applied by the framework always in order to prevent overloading the system.

If the `LIMIT` clause is missing, a default limit is applied (for the API, this is 50 at the moment) - Therefore, `LIMIT` can also be used to increase the number of results.

#### Example
```
SELECT city, starttime FROM usersession ORDER BY starttime DESC LIMIT 10
```
### ORDER BY &lt;ordering&gt;
Allows to order the results by columns. Either ascending or descending. If not specified, the order is ascending.

Ordering is by default done b frequency; i.e. the top 5 cities are the most frequently occurring ones. Specifying a field in the "order by" clause will add a sort by value (currently supported for strings, dates and numbers; not for enums).

Ordering by enums or by function values like `AVG`, `SUM`, etc. will just order the returned result, but you are never guaranteed to get the top items for that case. This means if you request the top 5 results by `AVG(duration)`, requesting 10 instead may also add results on the top and not only at the bottom.

*Limitation: You may only specify up to 5 ordering statements/columns.*
```
ordering: <column> ASC | <column> DESC | <column>, ...
```
#### Example
```
SELECT useraction.name, starttime FROM usersession ORDER BY starttime DESC
```
Ordering by counts can be achieved by adding the DISTINCT keyword or a GROUP BY clause
#### Example
```
SELECT DISTINCT city, COUNT(*) FROM usersession ORDER BY COUNT(*) DESC
```
is equivalent to
```
SELECT city, COUNT(*) FROM usersession GROUP BY city ORDER BY COUNT(*) DESC
```
Ordering on functions in the select list can be done by stating the column-name only or the alias defined in the SELECT:
```
SELECT avg(duration) AS average, count(*) as number, day(startTime) as startDay
FROM usersession where duration < 2000
GROUP BY startTime
```
or
```
SELECT avg(duration) AS average, count(*) as number, day(startTime) as startDay
FROM usersession where duration < 2000
GROUP BY startTime
```
We will try to support stating `"GROUP BY day(startTime)"` as well.

## Functions

Functions extract and modify data. Each function will return its values as one column. The name of this column can be specified with the "AS alias" syntax.
For more information on the concepts of functions, visit the "technical details" section.

### MIN(field)
Queries the minimum value of a numeric or date field.
#### Example
```
SELECT MIN(duration), MAX(duration), AVG(duration), MEDIAN(duration) FROM usersession
```
### MAX(field)
Queries the maximum value of a numeric or date field.

#### Example
```
SELECT MIN(duration), MAX(duration), AVG(duration), MEDIAN(duration) FROM usersession
```
### AVG(field)
Queries the average value of a numeric or date field. May be NaN if the field is always null.
#### Example
```
SELECT MIN(duration), MAX(duration), AVG(duration), MEDIAN(duration) FROM usersession
```
### MEDIAN(field)
Queries the median value of a numeric or date field.
#### Example
```
SELECT MIN(duration), MAX(duration), AVG(duration), MEDIAN(duration) FROM usersession
```
### PERCENTILE(field, percentileValue)
Queries the percentile value of a numeric or date field.
#### Example
```
SELECT MIN(duration), MAX(duration), AVG(duration), PERCENTILE(duration, 80.0) FROM usersession
```
### SUM(field)
Computes the sum of a numerical field.
#### Example
```
SELECT TOP(name, 20), SUM(duration) FROM useraction GROUP BY name
```
### COUNT(field), COUNT(*), COUNT(DISTINCT field)
Counts how many rows match.
`COUNT(*)`: counts the number of matching items.
`COUNT(<field>)`: counts the number of matching items where <field> is not null.
`COUNT(DISTINCT <field>)`: counts the number of different values for <field> within the selected items.

Queries that uses `COUNT(DISTINCT <field>)` on extremely high cardinality fields (dateTime fields like usersession.startTime, usersession.endTime, useraction.networkTime) will be rejected from execution.

###### List of fields which are considered extremely high cardinality

usersession: startTime, endTime, replayEnd, clientTimeOffset. duration, replayStart
useraction:  domContentLoadedTime, startTime, firstPartyBusyTime, documentInteractiveTime, navigationStart, totalBlockingTime, largestContentfulPaint, visuallyCompleteTime, 
             cdnBusyTime, endTime, domCompleteTime, networkTime, loadEventStart, serverTime, firstInputDelay, responseStart, thirdPartyBusyTime, duration, loadEventEnd, 
             responseEnd, frontendTime, requestStart 
userevent:   startTime
usererror:   startTime

#### Example
```
SELECT country, COUNT(*), COUNT(city), COUNT(DISTINCT city) FROM usersession GROUP BY country
```
