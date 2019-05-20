
**Learning resources**

* [Get started with Azure monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/log-query/get-started-portal)
* https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/data-explorer/write-queries.md
* https://4pp1n51ght5.com/2017/08/23/using-azure-log-analytics-to-calculate-user-engagement-metrics/
* https://4pp1n51ght5.com/2017/02/08/calculating-stickiness-using-appinsights-analytics/

Searching 
```
search "event name" | take 10
```

Dynamic timerange
```
| where timestamp between (datetime(2019-01-01T00:01:24.615Z)..now())
```

**Common KPIs**

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

