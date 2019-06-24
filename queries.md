
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

Retrieves a list of deduplicated values 
```
Perf
| distinct ObjectName, CounterName
```

Only show unique error events
```
Event
| where EventLevelName == "Error"
| distinct Source
```

Using top for the first 20 rows
```
Perf
| top 20 by TimeGenerated desc
```
Sort rows ascending with `asc` or descending with `desc`

Combining everything so far
```
Perf
| where CounterName == "Free Megabytes"     // Get the free Megabytes
        and TimeGenerated >= ago(1h)        // ... within the last hour
| project Computer                          // For each return the Computer Name
        , TimeGenerated                     // ...and when the counter was generated
        , CounterName                       // ...and the Counter Name
        , FreeMegaBytes=CounterValue        // ...and rename the counter value
| distinct Computer                         // Now weed out the duplicate rows as
        , TimeGenerated
        , CounterName                       // ...the perf table will have
        , FreeMegaBytes                     // ...multiple entries during the day
| top 25 by FreeMegaBytes asc               // Filter to most critical ones
```

# Scalar operators

Scalar operators allow us to format and transform data, and logical operators for "if then" logic.

**Print** retrieves an output that is defined or calculated
```
print "Hello world"
```
This command is useful for debugging a calculation stepwise

You can also name the output
```
print something="something"
```

Get the time in UTC. This is the standard in all Azure logs. We can use it for any changes to date and time data.
```
print now()
```

Print time one hour ago
```
print ago(1) // 1h 1m 1d 1s 1ms 1microsend 1tick
```

Print time in the future
```
print ago(-1d) // print time tomorrow, -365d gives us a year in the future
```

This lets us retrieve values for a specific timerange when combined with other clauses such as where and other operators.

**Sort** lets us sort the columns as desired
```
Perf
| where TimeGenerated > ago(15m)
| where CounterName == "Avg. Disk sec/Read"
        and InstanceName == "C:"
| project Computer 
        , TimeGenerated 
        , ObjectName 
        , CounterName 
        , InstanceName 
        , CounterValue 
| sort by Computer
        , TimeGenerated
```

Sort by ascending
```
Perf
| where TimeGenerated > ago(15m)
| where CounterName == "Avg. Disk sec/Read"
        and InstanceName == "C:"
| project Computer 
        , TimeGenerated 
        , ObjectName 
        , CounterName 
        , InstanceName 
        , CounterValue 
| sort by Computer asc
        , TimeGenerated asc
```

You can also mix sorting clauses
```
Perf
| where TimeGenerated > ago(15m)
| where CounterName == "Avg. Disk sec/Read"
        and InstanceName == "C:"
| project Computer 
        , TimeGenerated 
        , ObjectName 
        , CounterName 
        , InstanceName 
        , CounterValue 
| sort by Computer asc
        , TimeGenerated
```

`order by` is an alias for `sort by`

## Extract
Extract will match a string based on a regular expression pattern, and retrieves only the part that matches.

```
Perf
| where ObjectName == "LogicalDisk"
        and InstanceName matches regex "[A-Z]:" 
| project Computer 
        , CounterName 
        , extract("[A-Z]:", 0, InstanceName)
```

Extract only the disk name without the colon
```
Perf
| where ObjectName == "LogicalDisk"
        and InstanceName matches regex "[A-Z]:" 
| project Computer 
        , CounterName 
        , extract("([A-Z]):", 1, InstanceName)
```

## Parse
Parse takes a text string and extracts part of it into a column name using markers

`Parse` runs until it has parsed the entire dataset or reached the final match we've specified.

Parse is very useful when you have large blobs of text you want to turn into standard components

```
Event
| where RenderedDescription startswith "Event code:"
| parse RenderedDescription with "Event code: " myEventCode
                                " Event message: " myEventMessage
                                " Event time: " myEventTime
                                " Event time (UTC): " myEventTimeUTC
                                " Event ID: " myEventID
                                " Event sequence: " myEventSequence
                                " Event occurrence: " *
| project myEventCode, myEventMessage, myEventTime, myEventTimeUTC, myEventID, myEventSequence 
```

## datetime arithmetic

Convert a string into a datetime for our query - for year to date, using `datetime`
```
Perf
| where CounterName == "Avg. Disk sec/Read"
| where CounterValue > 0
| take 100 
| extend HowLongAgo=( now() - TimeGenerated )
        , TimeSinceStartofYear=( TimeGenerated - datetime(2018-01-01) )
| project Computer 
        , CounterName 
        , CounterValue 
        , TimeGenerated 
        , HowLongAgo 
        , TimeSinceStartofYear 

```

Converting a datetime, f.ex into hours can be done with simple arithmetic of division
```
Perf
| where CounterName == "Avg. Disk sec/Read"
| where CounterValue > 0
| take 100 
| extend HowLongAgo=( now() - TimeGenerated )
        , TimeSinceStartOfYear=( TimeGenerated - datetime(2018-01-01) )
| extend TimeSinceStartOfYearInHours=( TimeSinceStartOfYear / 1h)
| project Computer 
        , CounterName 
        , CounterValue 
        , TimeGenerated 
        , HowLongAgo 
        , TimeSinceStartOfYear 
        , TimeSinceStartOfYearInHours 

```

Simple datetime calculations over columns
```
Usage
| extend Duration=( EndTime - StartTime )
| project Computer 
        , StartTime 
        , EndTime 
        , Duration
```

Combining summarize with datetime functions by a specific timeperiod
```
Event
| where TimeGenerated >= ago(7d)
| extend DayGenerated = startofday(TimeGenerated) 
| project Source 
        , DayGenerated 
| summarize EventCount=count() 
        by DayGenerated
        , Source
```
Retrieves number of events per source and for the last 7 days.

```
Event
| where TimeGenerated >= ago(365d)
| extend MonthGenerated = startofmonth(TimeGenerated) 
| project Source 
        , MonthGenerated 
| summarize EventCount=count() 
        by MonthGenerated
        , Source
| sort by MonthGenerated desc
        , Source asc
```
Retrieves number of events per source by month for the last 365 days.

You can also use `startofweek` and `startofyear` for similar operations.

There are also corresponding end of functions, f.ex `endofday` `endofweek`, `endofmonth` and `endofyear`
```
Event
| where TimeGenerated >= ago(7d)
| extend DayGenerated = endofday(TimeGenerated)
| project Source 
        , DayGenerated 
| summarize EventCount=count() 
        by DayGenerated
        , Source
| sort by DayGenerated desc
        , Source asc
```

## Between commands
Allows us to specify a range of values, be it numeric or datetime, to retrieve.

```
Perf
| where CounterName == "% Free Space"
| where CounterValue between( 70.0 .. 100.0 )
```

Likewise for dates
```
Perf
| where CounterName == "% Free Space"
| where TimeGenerated between( datetime(2019-04-01) .. datetime(2019-04-03)  )
| take 10
```

Gathering data for start of and end of specific dates
```
Perf
| where CounterName == "% Free Space"
| where TimeGenerated between( startofday(datetime(2019-04-01)) .. endofday(datetime(2019-04-03))  )
| take 10
```

There is also a "not between" operator `!between`, which lets us fetch values not within a range - only those outside it.
```
Perf
| where CounterName == "% Free Space"
| where CounterValue !between ( 0.0 .. 69.9999 )
| take 10
```

## Todynamic 

Takes json stored in a string and lets you retrieve its individual values

Convert json to a variable using `todynamic`, then step into the json array and project the values

Use the key as column names

```
SecurityAlert
| where TimeGenerated > ago(365d)
| extend Extprops=todynamic(ExtendedProperties) 
| project AlertName 
        , TimeGenerated 
        , Extprops["Alert Start Time (UTC)"]
        , Extprops["Source"]
        , Extprops["Non-Existent Users"]
        , Extprops["Existing Users"]
        , Extprops["Failed Attempts"]
        , Extprops["Successful Logins"]
        , Extprops["Successful User Logons"]
        , Extprops["Account Logon Ids"]
        , Extprops["Failed User Logons"]
        , Extprops["End Time UTC"]
        , Extprops["ActionTaken"]
        , Extprops["resourceType"]
        , Extprops["ServiceId"]
        , Extprops["ReportingSystem"]
        , Extprops["OccuringDatacenter"]

```

You can use column-renaming to structure the output better

```
SecurityAlert
| where TimeGenerated > ago(365d)
| extend Extprops=todynamic(ExtendedProperties) 
| project AlertName 
        , TimeGenerated 
        , AlertStartTime = Extprops["Alert Start Time (UTC)"]
        , Source = Extprops["Source"]
        , NonExistentUsers = Extprops["Non-Existent Users"]
        , ExistingUsers = Extprops["Existing Users"]
        , FailedAttempts = Extprops["Failed Attempts"]
        , SuccessfulLogins = Extprops["Successful Logins"]
        , SuccessfulUserLogons = Extprops["Successful User Logons"]
        , AccountLogonIds = Extprops["Account Logon Ids"]
        , FailedUserLogons= Extprops["Failed User Logons"]
        , EndTimeUtc = Extprops["End Time UTC"]
        , ActionTaken = Extprops["ActionTaken"]
        , ResourceType = Extprops["resourceType"]
        , ServiceId = Extprops["ServiceId"]
        , ReportingSystem = Extprops["ReportingSystem"]
        , OccuringDataCenter = Extprops["OccuringDatacenter"]

```

If the JSON keys do not have spaces you can also use property notation

```
SecurityAlert
| where TimeGenerated > ago(365d)
| extend Extprops=todynamic(ExtendedProperties) 
| project AlertName 
        , TimeGenerated 
        , AlertStartTime = Extprops["Alert Start Time (UTC)"]
        , Source = Extprops.Source
        , NonExistentUsers = Extprops["Non-Existent Users"]
        , ExistingUsers = Extprops["Existing Users"]
        , FailedAttempts = Extprops["Failed Attempts"]
        , SuccessfulLogins = Extprops["Successful Logins"]
        , SuccessfulUserLogons = Extprops["Successful User Logons"]
        , AccountLogonIds = Extprops["Account Logon Ids"]
        , FailedUserLogons= Extprops["Failed User Logons"]
        , EndTimeUtc = Extprops["End Time UTC"]
        , ActionTaken = Extprops.ActionTaken
        , ResourceType = Extprops.resourceType
        , ServiceId = Extprops.ServiceId
        , ReportingSystem = Extprops.ReportingSystem
        , OccuringDataCenter = Extprops.OccuringDatacenter
```

Multilevel notation is also supported, f.ex `Extprops.Level1.Level2`

## format_datetime
Allows you to return specific date formats

```
Perf
| take 100 
| project CounterName 
        , CounterValue 
        , TimeGenerated
        , format_datetime(TimeGenerated, "y-M-d")
        , format_datetime(TimeGenerated, "yyyy-MM-dd")
        , format_datetime(TimeGenerated, "MM/dd/yyyy")
        , format_datetime(TimeGenerated, "MM/dd/yyyy hh:mm:ss tt")
        , format_datetime(TimeGenerated, "MM/dd/yyyy HH:MM:ss")
        , format_datetime(TimeGenerated, "MM/dd/yyyy HH:mm:ss.ffff")
```




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

Retention for cohorts
```
// Finally, calculate the desired metric for each cohort. In this sample we calculate distinct users but you can change
// this to any other metric that would measure the engagement of the cohort members.
| extend 
    r0 = DistinctUsers(startDate, startDate+7d),
    r1 = DistinctUsers(startDate, startDate+14d),
    r2 = DistinctUsers(startDate, startDate+21d),
    r3 = DistinctUsers(startDate, startDate+28d),
    r4 = DistinctUsers(startDate, startDate+35d)
| union (week | where Cohort == startDate + 7d 
| extend 
    r0 = DistinctUsers(startDate+7d, startDate+14d),
    r1 = DistinctUsers(startDate+7d, startDate+21d),
    r2 = DistinctUsers(startDate+7d, startDate+28d),
    r3 = DistinctUsers(startDate+7d, startDate+35d) )
| union (week | where Cohort == startDate + 14d 
| extend 
    r0 = DistinctUsers(startDate+14d, startDate+21d),
    r1 = DistinctUsers(startDate+14d, startDate+28d),
    r2 = DistinctUsers(startDate+14d, startDate+35d) )
| union (week | where Cohort == startDate + 21d 
| extend 
    r0 = DistinctUsers(startDate+21d, startDate+28d),
    r1 = DistinctUsers(startDate+21d, startDate+35d) ) 
| union (week | where Cohort == startDate + 28d 
| extend 
    r0 = DistinctUsers (startDate+28d, startDate+35d) )
// Calculate the retention percentage for each cohort by weeks
| project Cohort, r0, r1, r2, r3, r4,
          p0 = r0/r0*100,
          p1 = todouble(r1)/todouble (r0)*100,
          p2 = todouble(r2)/todouble(r0)*100,
          p3 = todouble(r3)/todouble(r0)*100,
          p4 = todouble(r4)/todouble(r0)*100 
| sort by Cohort asc
```
