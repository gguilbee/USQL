# USQL - User Session Query Language
Dynatrace USQL (User Session Query Language) Documentation
+ https://www.dynatrace.com/support/help/shortlink/usql-info

## Introduction

Dynatrace captures detailed user session data each time a user interacts with your monitored application. This data includes all user actions and high level performance data. Using either the Dynatrace API or Dynatrace User Sessions Query Language (USQL), you can easily run powerful queries, segmentations, and aggregations on this captured data. To assist you, this topic provides detail about keywords and functions, syntax, working with Real User Monitoring tables, automated queries, and more.

## Table of contents
- [Introduction](#Introduction)
- [Timeframes](#Timeframes)
- [Keywords and Conventions](#Keywords-and-Conventions)
    - [SELECT &lt;columns&gt;](#select-columns)
    - [DISTINCT](#DISTINCT)
    - [FUNNEL](#FUNNEL)
    - [FROM &lt;table&gt;](#from-table)
    - [WHERE &lt;condition&gt;](#where-condition)
    - [GROUP BY &lt;condition&gt;](#group-by-grouping)
    - [LIMIT &lt;condition&gt;](#limit-limit)
    - [ORDER BY &lt;ordering&gt;](#order-by-ordering)
- [Functions](#functions)
    - [MIN(field)](#minfield)
    - [MAX(field)](#maxfield)
    - [AVG(field)](#avgfield)
    - [MEDIAN(field)](#medianfield)
    - [PERCENTILE(field, percentileValue)](#percentilefield-percentilevalue)
    - [SUM(field)](#sumfield)
    - [COUNT(field), COUNT(*), COUNT(DISTINCT field)](#countfield-count-countdistinct-field)
    - [TOP(field, n)](#topfield-n)
    - [YEAR(datefield), MONTH(datefield), DAY(datefield), HOUR(datefield), MINUTE(datefield)](#yeardatefield-monthdatefield-daydatefield-hourdatefield-minutedatefield)
    - [DATETIME(datefield [, format [, interval]])](#datetimedatefield--format--interval)
    - [CONDITION(function, condition)](#conditionfunction-condition)
    - [KEYS(customProperty)](#keycustomproperty)
    - [Advanced Function Syntax: FILTER clauses](#advanced-function-syntax-filter-clauses)

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
```
usersession: startTime, endTime, replayEnd, clientTimeOffset. duration, replayStart
useraction:  domContentLoadedTime, startTime, firstPartyBusyTime, documentInteractiveTime, navigationStart, totalBlockingTime, largestContentfulPaint, visuallyCompleteTime, 
             cdnBusyTime, endTime, domCompleteTime, networkTime, loadEventStart, serverTime, firstInputDelay, responseStart, thirdPartyBusyTime, duration, loadEventEnd, 
             responseEnd, frontendTime, requestStart 
userevent:   startTime
usererror:   startTime
```
#### Example
```
SELECT country, COUNT(*), COUNT(city), COUNT(DISTINCT city) FROM usersession GROUP BY country
```
### TOP(field, n)
Defines that the top <n> results should be returned. The default if <n> is not specified is "1" (i.e. the top value).
#### Example
```
SELECT TOP(name, 20), SUM(duration) FROM useraction GROUP BY name
```
If `TOP(<field>, n)` is selected and the results are grouped, but `<field>` is not part of the grouping, the top-n elements are returned as a list within a single field.

#### Example
```
SELECT TOP(country, 20), TOP(city, 3), COUNT(*) FROM usersession GROUP BY country
```
### YEAR(datefield), MONTH(datefield), DAY(datefield), HOUR(datefield), MINUTE(datefield)
Returns the given element extracted from a datefield.
```
YEAR: the 4-digit year
MONTH: the month-number between 1 and 12
DAY: the day of the month between 1 and 31
HOUR: the hour value between 0 and 23
MINUTE: the minute value between 0 and 59
```
You may use this to display data of single user session results (Example 1), but also to create a histogram (Example 2). Additionally, if you are just interested in e.g. the hours of the day when an application was used, but not in the counts, you can use these functions outside a grouping, just as you might use the `TOP` function (Example 3). Or you can apply these functions on other aggregation functions (like `MAX`, `MIN`, `AVG`) in order to convert the result. At the moment, date functions are the only ones that may take another function as parameter (Example 4).

#### Example
```
(1) SELECT starttime, DATETIME(starttime), YEAR(starttime), MONTH(starttime), DAY(starttime), HOUR(starttime), MINUTE(starttime) FROM usersession ORDER BY starttime DESC
(2) SELECT HOUR(starttime) as hourOfDay, COUNT(*) FROM usersession GROUP BY hourOfDay
(3) SELECT application, HOUR(starttime) AS hoursOfDay FROM useraction GROUP BY application
(4) SELECT application, DATETIME(MAX(starttime)) AS LastUsedTime FROM useraction GROUP BY application
```
### DATETIME(datefield [, format [, interval]])
Allows to format the selected datefield with the given format-string.

Default format is `yyyy-MM-dd HH:mm`

allowed letters within the format-string:
```
y: year-of-era
u: year
M: month
d: day of month
H: hour (0-23)
h: hour (1-12)
m: minute
s: second
E: day of week (Mon-Sun)
e: day of week (1-7,  1 for Sunday, 2 for Monday, etc.)
c: localized day of week (For Wednesday (English Locale): c → 4, ccc → Wed, cccc → Wednesday, German Locale: ccc → Mi, cccc → Mittwoch)
```
allowed values for the interval-string:
```
year|month|week|day|hour|minute|second
or, alternatively: a number followed by d (days), h (hours), m (minutes) or s (seconds); e.g.: 7d
```
#### Example
```
SELECT DATETIME(starttime, 'yyyy-MM') FROM usersession
SELECT DISTINCT DATETIME(starttime, 'HH:mm', '5m'), count(*) FROM usersession
```
Similar to the other date functions `(YEAR, MONTH, DAY, HOUR, MINUTE)` you can use these functions to format a result (even the result of another function like `MAX, MIN, AVG, CONDITION)` (Example 1), to create histograms (Example 2) or to retrieve a list of times where there are results; e.g. the days of a week when an application was used (Example 3).

#### Example
```
(1) SELECT application, DATETIME(MAX(starttime)) AS LastUsedTime FROM useraction GROUP BY application
(2) SELECT DATETIME(starttime, "HH") as hourOfDay, COUNT(*) FROM usersession GROUP BY hourOfDay
(3) SELECT application, DATETIME(starttime, "E") AS daysOfWeek FROM useraction GROUP BY application
(4) SELECT DATETIME(CONDITION(MAX(startTime), WHERE name = "index.jsp")) as d FROM useraction 
```

### CONDITION(function, condition) 
Allows you to combine multiple functions with different conditions.

allowed functions:
```
MIN()
MAX()
AVG()
SUM()
PERCENTILE()
MEDIAN()
COUNT()
```
condition :  similar to where clause
```
WHERE  (condition AND condition) | (condition OR condition) | field IN(...) | field IS <value> | field IS NULL | field = <value> | field > <value> | field < <value> | field <> <value> | field NOT <value> | field BETWEEN <value> AND <value> | ...
```
also allows `FILTER` clause to be used similar to the nested allowed functions

#### Example
```
SELECT CONDITION(count(usersessionId), where userActionCount > 2 AND useraction.name = "search.jsp") from usersession
SELECT CONDITION(SUM(usersession.duration), where name = "index.jsp") as c1, CONDITION(SUM(usersession.duration), where name = "search.jsp") as c2, CONDITION(SUM(usersession.duration), where name NOT is "index.jsp" AND name NOT is "search.jsp") as c3 from useraction where  (duration > 1000 OR usersession.userActionCount > 4)
SELECT CONDITION(SUM(usersession.duration), where name = "index.jsp") as c1 from useraction where  (duration > 1000 OR usersession.userActionCount > 4) order by c1
SELECT CONDITION(count(usersessionId), where userActionCount > 2 AND useraction.name = "search.jsp") FILTER > 1000, city from usersession group by city
SELECT DATETIME(CONDITION(MIN(startTime ), WHERE useraction.application = "RUM Default Application" ), "yyyy-MM-dd" )  FROM usersession 
```
### KEYS(customProperty) 
Returns the list of keys of custom property defined in the argument.
customProperty :  can be of one of the multi fields defined in USQL. For ex. stringProperties, longProperties, doubleProperties etc.
#### Examples:
```
SELECT KEYS(stringProperties) FROM usersession WHERE application = "rum" AND city = "Linz" 
SELECT KEYS(useraction.longProperties) FROM usersession WHERE application = "rum" AND city = "Linz" 

SELECT KEYS(stringProperties) FROM useraction WHERE application = "rum" AND city = "Linz" 
SELECT KEYS(usersession.stringProperties) FROM useraction WHERE application = "rum" AND city = "Linz" 
```
##### For fetching distinct keys use the following queries
```
SELECT DISTINCT KEYS(stringProperties) FROM useraction where useraction.application = "easytravel-ang.lab.dynatrace.org" ORDER BY keys(stringProperties) 
SELECT DISTINCT city, KEYS(stringproperties) FROM usersession
```
### Advanced Function Syntax: FILTER clauses
For functions that return numeric values, filters can be specified. Filters may be compared to a "HAVING" clause in SQL, or facets in other languages. These filters allow to select only certain results from the aggregation above.
If you want a general filter (e.g. duration >3000), you should use a condition in the WHERE clause instead-but if you want to aggregate by ISP and check the ones with slow network times, you can use filters.
 
Syntax:
Function FILTER operator value
Function: one of the following: `MIN, MAX, AVG, SUM, MEDIAN, PERCENTILE, COUNT, CONDITION`
`FILTER`: keyword
operator: one of the following: `=, !=, <>, <, >, <=, >=`
value: some numeric value

#### Example
```
SELECT isp, AVG(useraction.networkTime) FILTER > 500 AS delay FROM usersession GROUP BY isp ORDER BY delay DESC
SELECT CONDITION(count(usersessionId), where userActionCount > 2 AND useraction.name = "search.jsp") FILTER > 1000, city from usersession group by city
````
## Mathematical Operation

Currently we support simple forms of mathematical operations as part of queries like 

+ operations on Numbers 
+ operations on numeric  and dateTime fields (Since we store dateTime fields as long)
+ operations on some functions including YEAR, MONTH, DAY, HOUR, MINUTE 
#### Syntax:
`Number/NumericField/DateTimeField/Function OPERATOR Number/NumericField/DateTimeField/Function`

Function: one of the following: `YEAR, MONTH, DAY, HOUR, MINUTE`

Operator: one of the following: `+, -, *, /, , %, MOD`

*Limitation: Currently, GROUP BY, ORDER BY or operations on functions except mentioned above are not allowed . These functionality might be added in the future.*

#### Example
```
SELECT 7 + 80 * 100, duration + startTime, MONTH(startTime) - 1  FROM usersession
```
## Conditions
all conditions start with an identifier; i.e. a field name, and they are compared against some value. It is not possible to compare two fields with each other.
Be aware that quoted text is case sensitive!

### Basic Operators
Basic operators for comparison are `=, !=, <>, <, >, <=, >=, IS, IS NOT`. `(=` and `IS` are equivalent for checking equality; `!=` and `<>` and `IS NOT` are equivalent for non equals).
You can also compare against `NULL` to check whether a value is present or not.
#### Example
```
SELECT userId FROM usersession WHERE userActionCount > 3
```
### Ranges
Ranges are handled by keywords: `BETWEEN <lowerLimit> AND <upperLimit>`.

This would be equivalent to `"WHERE <field> > <lowerLimit> AND <field> < <upperLimit>"`

#### Example
```
SELECT DISTINCT ip FROM usersession WHERE ip BETWEEN "192.168.0.0" AND "192.168.255.255" 
```

### Sets
To have a shorter version of `"WHERE <field> = val1 OR <field> = val2 OR <field> = val3"`, you can use the `IN` keyword.

#### Example
```
SELECT userId FROM usersession WHERE city IN ("New York", "San Francisco")
```

### String conditions - STARTSWITH
It checks whether a String or enum field starts with some given text.

#### Example
```
SELECT city FROM usersession WHERE userId STARTSWITH "dynatrace_"
```
    
    
