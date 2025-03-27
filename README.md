# KustoQuery

Great! Tracking **App Uptime** using **Kusto Query Language (KQL)** depends on where your application logs its availability — typically through **Heartbeat logs**, **Availability Results**, **Custom Events**, or **Application Logs** in **Azure Monitor / Application Insights**.

Here’s a **detailed KQL** to calculate and visualize **App Uptime**, including availability percentage, downtime duration, and response health.

---

## ✅ Scenario 1: **Using `availabilityResults` Table (App Insights Ping Test)**

```kusto
availabilityResults
| where timestamp > ago(7d)
| extend Result = iff(success == true, "Success", "Failure")
| summarize
    TotalTests = count(),
    SuccessfulTests = countif(success == true),
    FailedTests = countif(success == false),
    UptimePct = todecimal(round(100 * countif(success == true) / count(), 2))
by bin(timestamp, 1h), name
| order by timestamp asc
```

### 📊 Output:
| timestamp     | name           | TotalTests | SuccessfulTests | FailedTests | UptimePct |
|---------------|----------------|------------|------------------|--------------|-----------|
| 2025-03-25 00:00 | MyApp-UptimeTest | 12         | 12               | 0            | 100       |

📌 `name` refers to the availability test name (e.g., ping or URL test)

---

## ✅ Scenario 2: **Using `heartbeat` Table (VM/Container Uptime)**

```kusto
Heartbeat
| where TimeGenerated > ago(7d)
| summarize HeartbeatsPerHour = count() by bin(TimeGenerated, 1h), Computer
| extend Uptime = iff(HeartbeatsPerHour >= 1, 1, 0)
| summarize
    TotalHours = count(),
    UptimeHours = sum(Uptime),
    DowntimeHours = TotalHours - UptimeHours,
    UptimePct = round(100.0 * UptimeHours / TotalHours, 2)
by Computer
```

### 📊 Output:
| Computer       | TotalHours | UptimeHours | DowntimeHours | UptimePct |
|----------------|------------|--------------|----------------|-----------|
| vm-app01       | 168        | 167          | 1              | 99.40     |

🧠 Assumes a heartbeat every hour. You can tune for other intervals like `5m`, `10m`, etc.

---

## ✅ Scenario 3: **Using Custom App Logs (e.g., App Logs a Health Ping)**

If your app logs a success/fail message (e.g., `/healthcheck` endpoint), you can track uptime this way:

```kusto
AppTraces
| where TimeGenerated > ago(7d)
| where Message has "HealthCheck"
| extend Status = iff(Message has "Healthy", "Success", "Failure")
| summarize
    TotalChecks = count(),
    Successes = countif(Status == "Success"),
    Failures = countif(Status == "Failure"),
    UptimePct = round(100.0 * Successes / TotalChecks, 2)
by bin(TimeGenerated, 1h)
```

---

## ✅ Bonus: Overall Uptime as a KPI (for today)
```kusto
availabilityResults
| where timestamp >= startofday(now())
| summarize
    Total = count(),
    Up = countif(success == true),
    Down = countif(success == false),
    UptimePct = round(100.0 * Up / Total, 2)
```

---

## 📈 Suggested Visuals

| KPI / Chart            | Use                      | Data Source           |
|------------------------|--------------------------|------------------------|
| Uptime % (KPI Card)    | Quick glance             | `availabilityResults` |
| Downtime duration      | Alerting                 | `availabilityResults` |
| Uptime trend (Line)    | Historical tracking      | `availabilityResults` |
| Uptime by Region/App   | Split by dimension       | `availabilityResults` or `Heartbeat` |
| Failures list (Table)  | Drill-down into issues   | `availabilityResults` |

---

Would you like a **dashboard-ready Workbook**, **Power BI version**, or help mapping this to **custom app health events**?
Great question! If you're in a constrained environment like **Microsoft Sentinel Workbook queries**, **Log Analytics alerts**, or **Power BI DirectQuery mode**, you might **not be allowed to use `let` statements**. In those cases, you can still achieve the same logic using **nested subqueries**, **common table expressions**, or just **inlined logic**.

Absolutely! Below is a detailed **Kusto Query Language (KQL)** script for each of the **key SignInLogs indicators** you can use in **Azure Monitor Workbooks**, **Sentinel**, or **Power BI**.

---

## ✅ 1. **Total Sign-ins Over Time**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| summarize TotalSignIns = count() by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```
📊 **Chart**: Line/Area  
🕐 **Use**: See daily or hourly usage trends.

---

## ✅ 2. **Sign-in Success vs Failure**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| summarize
    Success = countif(ResultType == 0),
    Failure = countif(ResultType != 0)
```
📊 **Chart**: Pie/Bar  
🛠️ **Use**: Measure overall sign-in health.

---

## ✅ 3. **Sign-ins by Application**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| summarize SignInCount = count() by AppDisplayName
| top 10 by SignInCount desc
```
📊 **Chart**: Horizontal Bar  
📎 **Use**: Identify most/least accessed apps.

---

## ✅ 4. **Sign-ins by User**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| summarize SignInCount = count() by UserPrincipalName
| top 10 by SignInCount desc
```
📊 **Chart**: Bar/Table  
👤 **Use**: See who’s most active.

---

## ✅ 5. **Sign-ins by Location (Country)**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| summarize SignInCount = count() by Location
| top 10 by SignInCount desc
```
📊 **Chart**: Map or Column Bar  
🗺️ **Use**: Spot unusual geographies.

---

## ✅ 6. **Sign-ins by Device or OS**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| extend OS = tostring(DeviceDetail.operatingSystem)
| summarize SignInCount = count() by OS
| top 10 by SignInCount desc
```
📊 **Chart**: Pie or Bar  
💻 **Use**: Track device adoption.

---

## ✅ 7. **MFA Status Summary**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| summarize Count = count() by AuthenticationRequirement
```
📊 **Chart**: Pie/Stacked Bar  
🔐 **Use**: Ensure MFA coverage.

---

## ✅ 8. **Sign-ins by Risk Level** (Entra ID P2 Required)
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| summarize Count = count() by RiskLevelDuringSignIn
```
📊 **Chart**: Bar  
⚠️ **Use**: Spot risky sign-ins.

---

## ✅ 9. **Top Failed Users**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| where ResultType != 0
| summarize Failures = count() by UserPrincipalName
| top 10 by Failures desc
```
📊 **Chart**: Table  
🚨 **Use**: Identify users with repeated login issues.

---

## ✅ 10. **Average Sign-in Duration (if available)**
```kusto
SignInLogs
| where TimeGenerated > ago(7d)
| where isnotempty(SignInDurationMs)
| summarize AvgSignInTimeMs = avg(todouble(SignInDurationMs)) by bin(TimeGenerated, 1h)
```
📊 **Chart**: Line  
⏱️ **Use**: Track sign-in performance trends.

---

## ⚙️ Bonus: Total Sign-ins Today
```kusto
SignInLogs
| where TimeGenerated >= startofday(now())
| summarize TotalToday = count()
```
📊 **Chart**: KPI Card  
📆 **Use**: Daily metric at a glance.

---

Would you like me to combine these into a **Power BI-ready query set** or help you deploy them into an **Azure Workbook with visuals**?

Here are a few **workarounds** to replace `let`:

---

### ✅ 1. **Inline Filters and Expressions**
Instead of:
```kusto
let startTime = ago(30d);
SignInLogs
| where TimeGenerated > startTime
```

Use:
```kusto
SignInLogs
| where TimeGenerated > ago(30d)
```

---

### ✅ 2. **Use Subqueries Instead of `let` Blocks**
Instead of:
```kusto
let logs = SignInLogs | where TimeGenerated > ago(30d);
logs | summarize count()
```

Use:
```kusto
(
    SignInLogs
    | where TimeGenerated > ago(30d)
)
| summarize count()
```

---

### ✅ 3. **Rewrite `let` Chains as `union` of Subqueries**

Here’s a version of your query **without any `let`**:

```kusto
// Hourly Summary
(
    SignInLogs
    | where TimeGenerated > ago(30d)
    | extend Hour = format_datetime(TimeGenerated, 'yyyy-MM-dd HH:00')
    | extend Result = iff(ResultType == 0, "Success", "Failure")
    | summarize TotalSignIns = count(),
                SuccessCount = countif(Result == "Success"),
                FailureCount = countif(Result == "Failure"),
                DistinctUsers = dcount(UserPrincipalName)
      by Hour
    | project TimePeriod = Hour, Level = "Hourly", TotalSignIns, SuccessCount, FailureCount, DistinctUsers
)
| union (
    // Daily Summary
    SignInLogs
    | where TimeGenerated > ago(30d)
    | extend Day = format_datetime(TimeGenerated, 'yyyy-MM-dd')
    | extend Result = iff(ResultType == 0, "Success", "Failure")
    | summarize TotalSignIns = count(),
                SuccessCount = countif(Result == "Success"),
                FailureCount = countif(Result == "Failure"),
                DistinctUsers = dcount(UserPrincipalName)
      by Day
    | project TimePeriod = Day, Level = "Daily", TotalSignIns, SuccessCount, FailureCount, DistinctUsers
)
| union (
    // Monthly Summary
    SignInLogs
    | where TimeGenerated > ago(30d)
    | extend Month = format_datetime(TimeGenerated, 'yyyy-MM')
    | extend Result = iff(ResultType == 0, "Success", "Failure")
    | summarize TotalSignIns = count(),
                SuccessCount = countif(Result == "Success"),
                FailureCount = countif(Result == "Failure"),
                DistinctUsers = dcount(UserPrincipalName)
      by Month
    | project TimePeriod = Month, Level = "Monthly", TotalSignIns, SuccessCount, FailureCount, DistinctUsers
)
| union (
    // Today's Summary
    SignInLogs
    | where TimeGenerated >= startofday(now())
    | summarize TotalSignIns = count(),
                SuccessCount = countif(ResultType == 0),
                FailureCount = countif(ResultType != 0),
                DistinctUsers = dcount(UserPrincipalName)
    | extend TimePeriod = format_datetime(startofday(now()), 'yyyy-MM-dd'), Level = "Today"
    | project TimePeriod, Level, TotalSignIns, SuccessCount, FailureCount, DistinctUsers
)
| order by TimePeriod desc
```

---

### 🛠 When to Use These Alternatives

- 🔄 **Power BI / Dashboards**: Use subqueries when `let` is blocked.
- 📉 **Workbooks / Sentinel Analytics Rules**: Subqueries are more compatible.
- 🧪 **Testing**: For environments where `let` syntax leads to failure.

---

Let me know your specific platform (Sentinel, Power BI, Workbook, etc.), and I can tailor this even further to ensure it works 100%.

Absolutely! Let's expand the Kusto query to give you more control and insight, breaking it down into **separate time-based summaries** for **hourly**, **daily**, and **monthly**, including details like **successful vs failed sign-ins**, **location**, and **app used**.

---

### 🔍 Expanded Kusto Query for `SignInLogs` with Hourly, Daily, and Monthly Summary

```kusto
// Define the time range
let startTime = ago(30d);
let endTime = now();

// Filter SignInLogs and calculate Hour, Day, Month
let logs = SignInLogs
| where TimeGenerated between (startTime .. endTime)
| extend
    Hour = format_datetime(TimeGenerated, 'yyyy-MM-dd HH:00'),
    Day = format_datetime(TimeGenerated, 'yyyy-MM-dd'),
    Month = format_datetime(TimeGenerated, 'yyyy-MM'),
    Result = iff(ResultType == 0, "Success", "Failure");

// Hourly Summary
let hourlySummary = logs
| summarize TotalSignIns = count(),
            SuccessCount = countif(Result == "Success"),
            FailureCount = countif(Result == "Failure"),
            DistinctUsers = dcount(UserPrincipalName)
  by Hour;

// Daily Summary
let dailySummary = logs
| summarize TotalSignIns = count(),
            SuccessCount = countif(Result == "Success"),
            FailureCount = countif(Result == "Failure"),
            DistinctUsers = dcount(UserPrincipalName)
  by Day;

// Monthly Summary
let monthlySummary = logs
| summarize TotalSignIns = count(),
            SuccessCount = countif(Result == "Success"),
            FailureCount = countif(Result == "Failure"),
            DistinctUsers = dcount(UserPrincipalName)
  by Month;

// Output each section separately
union 
  hourlySummary | project TimePeriod = Hour, Level = "Hourly", TotalSignIns, SuccessCount, FailureCount, DistinctUsers,
  dailySummary | project TimePeriod = Day, Level = "Daily", TotalSignIns, SuccessCount, FailureCount, DistinctUsers,
  monthlySummary | project TimePeriod = Month, Level = "Monthly", TotalSignIns, SuccessCount, FailureCount, DistinctUsers
| order by TimePeriod desc
```

---

### 📊 Output Columns Explained:
- `TimePeriod`: Time bucket (hour, day, or month).
- `Level`: Indicates whether it's an hourly, daily, or monthly summary.
- `TotalSignIns`: Total number of sign-in attempts.
- `SuccessCount`: Number of successful logins.
- `FailureCount`: Number of failed logins.
- `DistinctUsers`: Number of unique users.

---

### ✅ Optional Add-ons
You can enhance this query by grouping on additional dimensions:
- Group by `Location` to analyze geographic trends.
- Group by `AppDisplayName` to see which apps users are accessing.
- Add `DeviceDetail` or `ClientAppUsed` for device/client insights.

Let me know if you want one of those added!

Here’s a **Kusto Query** (KQL) that summarizes `SignInLogs` by **hour**, **day**, and **month**, showing the count of sign-ins:

```kusto
SignInLogs
| extend Hour = format_datetime(TimeGenerated, 'yyyy-MM-dd HH:00')
| extend Day = format_datetime(TimeGenerated, 'yyyy-MM-dd')
| extend Month = format_datetime(TimeGenerated, 'yyyy-MM')
| summarize CountPerHour = count() by Hour
| join kind=inner (
    SignInLogs
    | extend Day = format_datetime(TimeGenerated, 'yyyy-MM-dd')
    | summarize CountPerDay = count() by Day
) on Day
| join kind=inner (
    SignInLogs
    | extend Month = format_datetime(TimeGenerated, 'yyyy-MM')
    | summarize CountPerMonth = count() by Month
) on $left.Month == $right.Month
| project Hour, CountPerHour, Day, CountPerDay, Month, CountPerMonth
```

### Explanation:
- `format_datetime(...)`: Formats `TimeGenerated` into specific units.
- `summarize ... by`: Aggregates the count by each time bucket.
- `join`: Combines all the summaries on common time columns.

Let me know if you'd like:
- Filters (like `ResultType`, `UserPrincipalName`, or `Location`),
- A visual-ready format (like for Power BI),
- Or an hourly trend chart over a selected time frame.

You can incorporate an average percentage calculation by dividing the count of each status category by the total count for that hour and multiplying by 100. For example:

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and ResponseStatus_d != ""
| extend StatusCategory = case(
    ResponseStatus_d <= 399, "Green",
    ResponseStatus_d > 399 and ResponseStatus_d <= 499, "Yellow",
    ResponseStatus_d >= 500, "Red",
    "Unknown"
)
| summarize 
    Count = count(), 
    GreenCount = countif(StatusCategory == "Green"),
    YellowCount = countif(StatusCategory == "Yellow"),
    RedCount = countif(StatusCategory == "Red"),
    GreenPercentage = 100 * countif(StatusCategory == "Green") / count(),
    YellowPercentage = 100 * countif(StatusCategory == "Yellow") / count(),
    RedPercentage = 100 * countif(StatusCategory == "Red") / count()
    by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

### Explanation:
- **`countif()`**: Counts only the rows matching the condition for each category.
- **`GreenPercentage`, `YellowPercentage`, `RedPercentage`**: These fields represent the percentage of each category out of the total count for each hour.

This query provides both the raw counts and the relative percentages for each status category, allowing you to monitor not just the total occurrences but also the proportion of each type of status over time.

To visualize the indicators (green, yellow, red) in Azure Monitor’s Log Analytics, you can use the built-in charting capabilities. You don’t need to modify the KQL query structure significantly; instead, you just apply a rendering directive after your main query. For example:

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and ResponseStatus_d != ""
| extend StatusCategory = case(
    ResponseStatus_d <= 399, "Green",
    ResponseStatus_d > 399 and ResponseStatus_d <= 499, "Yellow",
    ResponseStatus_d >= 500, "Red",
    "Unknown"
)
| summarize Count = count() by bin(TimeGenerated, 1h), StatusCategory
| render columnchart with (title="Gateway Status Indicators", ytitle="Count", series=StatusCategory, kind=stacked)
```

### Key Additions:
- **`| render columnchart`**: Tells Log Analytics to create a column chart.  
- **`with (...)`**: Sets properties like the chart title, Y-axis title, and the grouping series (`StatusCategory`).  
- **`kind=stacked`**: Displays a stacked column chart so you can easily see the proportion of each status category at each time interval.

### Expected Output:
- **X-Axis:** Time intervals (hourly bins).
- **Y-Axis:** The count of each status category (Green, Yellow, Red).
- **Visualization:** A stacked column chart showing the distribution of status codes over time, with distinct colors (green, yellow, red) for each category.

This approach directly integrates a visual representation into your Log Analytics results, making it easier to track trends and identify periods of higher error rates.

To analyze status codes from Azure Gateway logs and categorize them into a simple “Green/Yellow/Red” status indicator based on whether the status code is above 399, you can use the following Kusto query:

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and ResponseStatus_d != ""
| extend StatusCategory = case(
    ResponseStatus_d <= 399, "Green",
    ResponseStatus_d > 399 and ResponseStatus_d <= 499, "Yellow",
    ResponseStatus_d >= 500, "Red",
    "Unknown"
)
| summarize Count = count() by StatusCategory
| order by StatusCategory
```

### Explanation:
1. **Filtering Logs:**  
   - The query filters `AzureDiagnostics` logs specific to Application Gateways (`ResourceType == "APPLICATIONGATEWAYS"`) and ensures the `ResponseStatus_d` field is not null or empty.
   
2. **Categorizing Status Codes:**  
   - `case()` assigns a category based on the numeric range of `ResponseStatus_d`:
     - `Green`: Status codes 399 or below (successful or informational responses).
     - `Yellow`: Status codes between 400 and 499 (client errors).
     - `Red`: Status codes 500 and above (server errors).
   
3. **Summarizing Results:**  
   - `summarize` counts how many responses fall into each category.
   
4. **Ordering:**  
   - The results are sorted by category name for a clear presentation.

This query gives you a concise view of how often your Azure Gateway logs show errors and allows you to quickly assess system health with color-coded status categories.


You can use the following Kusto query to analyze read latency metrics (ReadLatencyMs) from **InsightMetrics**:

```kql
InsightsMetrics
| where Name == "ReadLatencyMs"
| project TimeGenerated, Computer, Disk = Tags["Disk"], ReadLatencyMs = Val
| summarize AvgReadLatencyMs = avg(ReadLatencyMs) by Computer, Disk, bin(TimeGenerated, 1h)
| order by TimeGenerated desc
```

### Key Points:
- **Name == "ReadLatencyMs"**: Filters the data to show only the read latency metric.
- **project**: Selects specific columns for clarity (time, computer name, disk, and the read latency value).
- **summarize**: Aggregates the average read latency by computer, disk, and hourly intervals.
- **order by TimeGenerated desc**: Sorts the results to display the latest metrics first.

You can adjust the time bin (for example, `bin(TimeGenerated, 30m)` for 30-minute intervals) or filter further by specific computers or disks to narrow down your analysis.

If you’re looking to query **CPU utilization** using the `UtilizationPercentage` metric in **InsightsMetrics**, you can use the following KQL query:

```kql
InsightsMetrics
| where Name == "UtilizationPercentage"
| summarize AvgCpuUtilization = avg(Val) by Computer, bin(TimeGenerated, 5m)
| order by TimeGenerated desc
```

### Explanation:
- **InsightsMetrics**: The table containing performance metrics.
- **Name == "UtilizationPercentage"**: Filters for the CPU utilization percentage metric.
- **summarize AvgCpuUtilization**: Calculates the average CPU utilization for each computer over 5-minute intervals.
- **bin(TimeGenerated, 5m)**: Groups results into 5-minute time blocks.
- **order by TimeGenerated desc**: Sorts the results so the most recent data is displayed first.

#### Additional Filters:
- To focus on a specific computer:  
  ```kql
  | where Computer == "<Your_Computer_Name>"
  ```
- To find times when CPU utilization was above a certain threshold (e.g., 80%):  
  ```kql
  | where Val > 80
  ```

Let me know if you need further assistance!
If you have a known maximum memory capacity (for example, 32 GB) and want to see how the current memory usage compares to that limit, you can structure your Kusto query as follows:

```kql
InsightsMetrics
| where Name == "CommittedBytes"
| extend MemoryUsedGB = Val / (1024 * 1024 * 1024)
| extend MemoryMaxGB = 32
| extend MemoryUtilizationPercentage = (MemoryUsedGB / MemoryMaxGB) * 100
| summarize 
    AvgMemoryUsedGB = avg(MemoryUsedGB),
    AvgMemoryUtilization = avg(MemoryUtilizationPercentage)
    by Computer, bin(TimeGenerated, 5m)
| order by TimeGenerated desc
```

### Key Steps in the Query:
1. **Filter for Memory Usage Metric:**  
   The `Name == "CommittedBytes"` condition ensures that we only focus on committed memory, which typically reflects the memory in use.

2. **Convert CommittedBytes to GB:**  
   `Val / (1024 * 1024 * 1024)` converts the raw byte count into gigabytes.

3. **Set a Known Max Memory:**  
   Here, `MemoryMaxGB = 32` is explicitly defined as the system’s maximum memory capacity.

4. **Calculate Utilization Percentage:**  
   By dividing the used memory (in GB) by the known maximum memory (32 GB) and multiplying by 100, you get the percentage of memory utilization.

5. **Summarize Over Time Intervals:**  
   The query calculates the average memory used and the average utilization percentage over 5-minute bins. You can adjust the bin size as needed.

6. **Order Results:**  
   Sorting by `TimeGenerated desc` helps you quickly see the most recent memory usage compared to the 32 GB maximum.

This query provides a straightforward way to understand memory usage as a percentage of a fixed maximum value.

To query memory usage in **Azure Monitor Logs (Kusto Query Language - KQL)** from **InsightMetrics**, you can use the following query:

```kusto
InsightsMetrics
| where Name == "AvailableMB" or Name == "CommittedBytes"
| extend MemoryUsedMB = iff(Name == "CommittedBytes", Val / (1024 * 1024), null)
| summarize 
    AvgAvailableMB = avg(iff(Name == "AvailableMB", Val, null)), 
    AvgMemoryUsedMB = avg(MemoryUsedMB)
    by Computer, bin(TimeGenerated, 5m)
| order by TimeGenerated desc
```

### Explanation:
- **InsightsMetrics**: The table storing system performance metrics.
- **AvailableMB**: Represents available memory in megabytes.
- **CommittedBytes**: Represents the total memory committed (used) in bytes.
- **MemoryUsedMB**: Converts **CommittedBytes** to megabytes.
- **Summarize**: Computes the **average available memory** and **average used memory** grouped by the computer and 5-minute intervals.
- **Order by TimeGenerated desc**: Shows the latest results first.

#### Optional Enhancements:
- To filter for a specific VM, add:  
  ```kusto
  | where Computer == "<VM_NAME>"
  ```
- To monitor high memory usage (e.g., less than 500MB available):  
  ```kusto
  | where AvgAvailableMB < 500
  ```

Let me know if you need further refinements! 🚀
