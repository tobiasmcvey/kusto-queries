
Start learning at https://portal.loganalytics.io/demo

**Learning resources**

* [Get started with Azure monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/log-query/get-started-portal)
* https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/data-explorer/write-queries.md
* https://4pp1n51ght5.com/2017/08/23/using-azure-log-analytics-to-calculate-user-engagement-metrics/
* https://4pp1n51ght5.com/2017/02/08/calculating-stickiness-using-appinsights-analytics/

## Searching 

Search across all datasets
```
search "event name" | take 10
```
Note we use the `take` command to limit our search to 10 search results. This speeds up our querying substantially. Kusto queries can take a long time to execute if the datasets are large. To avoid this, use the take command before running queries on a full dataset. 

The timeout can take anything from 10 seconds up to 30 minutes. You can cancel your query if you don't want to wait, or allow the query to run and open a new query in a new tab if you need it.

Searching in a single dataset
```
Perf | search "Contoso"
```
Searching in multiple datasets
```
search in (Perf, Event, Alert) "Contoso" | take 10
```
Searching in specific columns for a match
```
Perf
| search CounterName=="Available MBytes" | take 10
```
Searching for a term anywhere in text
```
Perf
| search "*Bytes*" | take 10
```
Searching for columns that begins with
```
Perf
| search * startswith "Bytes" | take 10
```
Searching for columns that ends with
```
Perf
| search * endswith "Bytes" | take 10
```
Search starts with specific term, anything in between and ends with a specific term
```
Perf
| search "Free*bytes" | take 10
```

**Combining searches**

Multiple terms
```
Perf
| search "Free*bytes" and ("C:" or "D:") | take 10
```
Using regular expressions
```
Perf
| search InstanceName matches regex "[A-Z]:" | take 10
```

## Time and timerange
Dynamic timerange
```
| where timestamp between (datetime(2019-01-01T00:01:24.615Z)..now())
```

## Where 

```
Perf
| where TimeGenerated >= ago(1h)
```
Note the time range for the query is automatically set in the query when we use time operators in our where clause

Using the `and` statement
```
Perf
| where TimeGenerated >= ago(1h) and CounterName == "Bytes Received/sec" | take 10
```

Using the `or` statement
```
Perf
| where TimeGenerated >= ago(1h) and (CounterName == "Bytes Received/sec" or CounterName == "% Processor Time") | take 10
```

Stacking multiple where clauses
```
Perf
| where TimeGenerated >= ago(1h) 
| where (CounterName == "Bytes Received/sec" 
        or
        CounterName == "% Processor Time") 
| where CounterValue > 0 
| take 10
```

Other uses for where - simulating search with where
```
Perf
| where * has "Bytes" | take 10
```

**Positional matching** with where

Matches words beginning with Bytes
```
Perf
| where * hasprefix "Bytes" | take 10
```

Matches words ending with Bytes
```
Perf
| where * hassuffix "Bytes" | take 10
```

Using `contains`
```
Perf
| where * contains "Bytes" | take 10
```

Using regular expressions with where
```
Perf
| where InstanceName matches regex "[A-Z]:" | take 10
```

## Take and Limit command
Take retrieves a random number of rows, f.ex
```
Perf
| take 10
```
Takes 10 rows

Limit is the equivalent
```
Perf
| limit 10
```

## Count operator 
Count rows in a table
```
Perf
| count
```
Counts all rows in the Perf dataset

Count can be used with filtering to count rows in a selected table
```
Perf
| where TimeGenerated >= ago(1h)
        and CounterName == "Bytes received/sec"
        and CounterValue > 0
| count
```

## Summarize

Aggregation function for summarizing data in a table, can include `by`
```
Perf
| summarize count()  by CounterName
```

Summarizing multiple columns
```
Perf
| summarize count()  by ObjectName, CounterName
```
Note it generates a third column called `count_`. The name is automatically generated.

If we want to supply a more suitable name as a variable, we can name it
```
Perf
| summarize PerfCount=count()
            by ObjectName, CounterName
```

With Summarize, use other aggregation functions
```
Perf
| where CounterName == "% Free Space"
| summarize NumberOfEntries=count()
            , AverageFreeSpace=avg(CounterValue)
            by CounterName
```

Using `Bin` to create logical groups
```
Perf
| summarize NumberOfEntries=count()
            by bin(TimeGenerated, 1d)
```

Using other values for binning
```
Perf
| where CounterName == "% Free Space"
| summarize NumberOfRowsAtThisPercentLevel=count()
            by bin(CounterValue, 10)
```


## Extend

Extend allows you to create calculated columns to add to your tables
```
Perf
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
```

You can also create multiple columns
```
Perf
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
        , FreeKB = CounterValue * 1000
```

Repeating a column
```
Perf
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000
        , FreeMB = CounterValue 
        , FreeKB = CounterValue * 1000
```

Create new string values

We can create a new column with string values
```
Perf
| where TimeGenerated >= ago(10m)
| extend ObjectCounter = strcat(ObjectName, " - ", CounterName) 
```
We use `strcat` to concatenate strings

## Project command

Project allows us to select which columns we want in our table
```
Perf
| project ObjectName
        , CounterName 
        , InstanceName 
        , CounterValue 
        , TimeGenerated 
```

Project and Extend are very useful when creating tables with our specific data.
```
Perf
| project ObjectName
        , CounterName 
        , InstanceName 
        , CounterValue 
        , TimeGenerated
| extend FreeGB = CounterValue / 1000
        , FreeMB = CounterValue 
        , FreeKB = CounterValue * 1000
```

If we want to omit a specific column we must calculate our values first, then project afterwards
```
Perf
| extend FreeGB = CounterValue / 1000
        , FreeMB = CounterValue 
        , FreeKB = CounterValue * 1000
| project ObjectName
        , CounterName 
        , InstanceName 
        , TimeGenerated
        , FreeGB 
        , FreeMB 
        , FreeKB 

```

OR we can just project our data and calculate it on the fly
```
Perf
| where CounterName == "Free Megabytes"
| project ObjectName
        , CounterName 
        , InstanceName 
        , TimeGenerated
        , FreeGB = CounterValue / 1000
        , FreeMB = CounterValue 
        , FreeKB = CounterValue * 1000

```

`Project-away` lets you remove selected columns from the output
```
Perf
| where CounterName == "Free Megabytes"
| project-away TenantId
            , SourceSystem 
            , CounterPath 
            , MG
```

`Project-rename` lets you rename a specific column
```
Perf
| where TimeGenerated > ago(1h)
| project-rename myRenamedComputer = Computer 
```

## Distinct






## **Common KPIs**

DAU to MAU activity ratio - Daily Active Users to Monthly Active Users
```
pageViews | union *
| where timestamp > ago(90d)
| evaluate activity_engagement(user_Id, timestamp, 1d, 28d)
| project timestamp, Dau_Mau=activity_ratio*100 
| where timestamp > ago(62d) // remove tail with partial data
| render timechart 
```

New Users
```
customEvents
| evaluate new_activity_metrics(session_Id, timestamp, startofday(ago(7d)), startofday(now()), 1d)
```

New Users, Returning and Churned
https://docs.microsoft.com/en-us/azure/kusto/query/new-activity-metrics-plugin
```
T | evaluate new_activity_metrics(id, datetime_column, startofday(ago(30d)), startofday(now()), 1d, dim1, dim2, dim3)
```

Stepping into a JSON object - example: Summarise events
```
customEvents
| where customDimensions != ""
| extend customDimensions.Properties.event 
| where customDimensions contains "Event name"
| take 10
```

Stepping into a JSON object - example: Counting users with an event
```
customEvents
| where timestamp > ago(30d)
| where customDimensions != ""
| extend customDimensions.Properties.event 
| where customDimensions contains "Event name"
| summarize dcount(user_Id) by bin(timestamp, 1d)
| take 10
```

### From Tutorial With Demo Dataset

Summarize average counter values for a specific machine by a specific metric for a given continuous variable, binning values
```
Perf
| where TimeGenerated > ago(30d)
| where Computer == "sqlserver-1.contoso.com"
| where CounterName == "Available MBytes"
| summarize avg(CounterValue) by bin(TimeGenerated, 1h)
```

Search for specific text - WORKS but not much data
```
customEvents
| search "2019"
```

Unique Users, New Users, Returning Users, Lost Users
Can be used in Workbooks with Parameter fields - not in Log Analytics
```
let timeRange = {TimeRange};
let monthDefinition = {Metric};
let hlls = union customEvents, pageViews
| where timestamp >= startofmonth(now() - timeRange - 2 * monthDefinition)
| where name in ({Activities}) or '*' in ({Activities})
{OtherFilters}
| summarize Hlls = hll(user_Id) by bin(timestamp, 1d)
| project DaysToMerge = timestamp, Hlls;
let churnSeriesWithHllsToInclude = materialize(range d from 0d to timeRange step 1d
| extend Day = startofday(now() - d)
| extend R = range(0d, monthDefinition - 1d, 1d)
| mvexpand R
| extend ThisMonth = Day - totimespan(R)
| extend LastMonth = Day - monthDefinition - totimespan(R)
| project Day, ThisMonth, LastMonth);
churnSeriesWithHllsToInclude
| extend DaysToMerge = ThisMonth
| join kind= inner (hlls) on DaysToMerge 
| project Day, ThisMonthHlls = Hlls
| union (
churnSeriesWithHllsToInclude
| extend DaysToMerge = LastMonth
| join kind= inner (hlls) on DaysToMerge
| project Day, LastMonthHlls = Hlls)
| summarize ThisMonth = hll_merge(ThisMonthHlls), LastMonth = hll_merge(LastMonthHlls) by Day
| evaluate dcount_intersect(ThisMonth, LastMonth)
| extend NewUsers = s0 - s1
| extend ChurnedUsers = -1 * (dcount_hll(LastMonth) - s1) // Last Months Users - Returning Users
| project Day, ["Active Users"] = s1 + NewUsers, ["Returning Users"] = s1, ["Lost Users"] = ChurnedUsers, ["New Users"] = NewUsers
```

