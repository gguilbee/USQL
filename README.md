# USQL
Dynatrace USQL (User Session Query Language) Documentation

## Table of contents
- [Introduction](#Introduction)
- [Timeframes](#Timeframes)
- [Keywords and conventions](#Keywords-and-conventions)
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

## Keywords and conventions

The following keywords are defined as part of the query language:
```
AND, AS, ASC, BETWEEN, BY, DESC, DISTINCT, FALSE, FILTER, FROM, FUNNEL, GROUP, IN, IS, JSON, LIMIT, NOT, NULL, OR, ORDER, SELECT, STARTSWITH, TRUE, WHERE
```

The following functions are defined as part of the query language:
```
SUM, MAX, MIN, AVG, MEDIAN, COUNT, YEAR, MONTH, DAY, HOUR, MINUTE, DATETIME, TOP, PERCENTILE
```

Keywords, functions and column names are case insensitive. String-matches in WHERE conditions are done case sensitive.
