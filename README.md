
When analyzing **AGWAccessLogs** (Application Gateway Access Logs) for key performance indicators (KPIs), you typically focus on metrics that help you understand traffic patterns, response performance, error rates, and data throughput. Below are a series of KQL queries to derive useful KPIs:

---

### 1. **Total Request Count**
This metric provides the total number of requests received by the Application Gateway, grouped by time.

```kql
AGWAccessLogs
| summarize TotalRequests = count() by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

**Use Case:**  
Understand overall traffic volume and patterns over time.

---

### 2. **Successful and Failed Requests**
Break down requests into successful (status codes < 400) and failed (status codes >= 400) to monitor the health of your applications.

```kql
AGWAccessLogs
| extend Success = iff(httpStatus_d < 400, "Success", "Failed")
| summarize TotalRequests = count(), SuccessfulRequests = countif(httpStatus_d < 400), FailedRequests = countif(httpStatus_d >= 400) by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

**Use Case:**  
Track changes in the success-to-failure ratio over time and quickly spot when something starts failing more frequently.

---

### 3. **Response Time Statistics**
Calculate average, maximum, and percentile response times to understand performance trends and outliers.

```kql
AGWAccessLogs
| summarize 
    AvgResponseTimeMS = avg(timeTaken_d * 1000), 
    MaxResponseTimeMS = max(timeTaken_d * 1000), 
    P95ResponseTimeMS = percentile(timeTaken_d * 1000, 95) 
    by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

**Use Case:**  
Monitor how responsive your application is and ensure SLAs are being met.

---

### 4. **Data Throughput (Bytes Sent/Received)**
Track how much data is flowing through your Application Gateway.

```kql
AGWAccessLogs
| summarize 
    TotalBytesSent = sum(bytesSent_d), 
    TotalBytesReceived = sum(bytesReceived_d) 
    by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

**Use Case:**  
Identify periods of high data transfer, which can help with capacity planning and cost analysis.

---

### 5. **Top Backend Pools by Requests**
Determine which backend pools handle the most traffic.

```kql
AGWAccessLogs
| summarize RequestCount = count() by backendPoolName_s, bin(TimeGenerated, 1h)
| order by RequestCount desc
```

**Use Case:**  
Pinpoint backends that are under heavy load or may require scaling.

---

### 6. **Status Code Distribution**
Break down requests by status code range to understand the overall health of responses.

```kql
AGWAccessLogs
| extend ResponseCategory = case(
    httpStatus_d >= 500, "5xx Server Errors",
    httpStatus_d >= 400, "4xx Client Errors",
    httpStatus_d >= 300, "3xx Redirections",
    httpStatus_d >= 200, "2xx Success",
    "Other"
)
| summarize Count = count() by ResponseCategory, bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

**Use Case:**  
Quickly identify trends in errors, redirects, or successes.

---

### 7. **Requests by URL Path**
Understand which endpoints receive the most traffic.

```kql
AGWAccessLogs
| summarize RequestCount = count() by requestUri_s, bin(TimeGenerated, 1h)
| order by RequestCount desc
```

**Use Case:**  
Identify high-traffic endpoints to optimize caching, scaling, or prioritization efforts.

---

### 8. **Client IP Distribution**
Identify the most frequent client IPs or regions.

```kql
AGWAccessLogs
| summarize RequestCount = count() by clientIp_s, bin(TimeGenerated, 1h)
| order by RequestCount desc
```

**Use Case:**  
Understand where your traffic is coming from and ensure appropriate geographical scaling and security policies.

---

**Combining Queries for Dashboards:**  
You can run these queries individually to investigate specific KPIs, or integrate them into a dashboard (using Azure Monitor Workbooks or a similar tool) to monitor them side-by-side. This allows you to maintain a real-time view of your Application Gateway’s health and performance.

There are a variety of metrics available within the Application Gateway logs that can help you gain deeper insights into performance, reliability, and traffic patterns. Below are some additional metrics and corresponding KQL examples:

1. **Response Status Distribution**  
   This metric shows how requests are distributed across different status code ranges (e.g., successful, client errors, server errors).  
   ```kql
   AzureDiagnostics
   | where ResourceType == "APPLICATIONGATEWAYS"
   | summarize SuccessfulRequests = countif(ResponseStatus_d < 400),
               ClientErrorRequests = countif(ResponseStatus_d >= 400 and ResponseStatus_d < 500),
               ServerErrorRequests = countif(ResponseStatus_d >= 500)
               by bin(TimeGenerated, 1h)
   ```

2. **Bytes Sent and Received**  
   This metric shows the amount of data transmitted and received through the gateway over time, which can help identify bandwidth usage or large traffic spikes.  
   ```kql
   AzureDiagnostics
   | where ResourceType == "APPLICATIONGATEWAYS"
   | summarize BytesSent = sum(ToLong(BytesSent_d)), 
               BytesReceived = sum(ToLong(BytesReceived_d))
               by bin(TimeGenerated, 1h)
   ```

3. **Request Counts Per Host**  
   If your Application Gateway handles multiple backends, you can analyze how many requests are routed to each hostname or backend pool.  
   ```kql
   AzureDiagnostics
   | where ResourceType == "APPLICATIONGATEWAYS"
   | summarize RequestsPerHost = count() by Host_s, bin(TimeGenerated, 1h)
   ```

4. **Backend Response Times**  
   You can examine the average time taken by the backend to respond to requests, providing insight into which backends might be causing latency.  
   ```kql
   AzureDiagnostics
   | where ResourceType == "APPLICATIONGATEWAYS" and BackendResponseTime_d != ""
   | summarize AvgBackendResponseTimeMS = avg(ToLong(BackendResponseTime_d) * 1000)
               by BackendPoolName_s, bin(TimeGenerated, 1h)
   ```

5. **Connection Counts**  
   Monitoring the number of connections established can help identify trends in user traffic and potential scaling issues.  
   ```kql
   AzureDiagnostics
   | where ResourceType == "APPLICATIONGATEWAYS"
   | summarize TotalConnections = count() by bin(TimeGenerated, 1h)
   ```

6. **Failed Requests Over Time**  
   Highlighting failed requests helps identify problem periods or trends in error rates.  
   ```kql
   AzureDiagnostics
   | where ResourceType == "APPLICATIONGATEWAYS" and ResponseStatus_d >= 400
   | summarize FailedRequests = count() by bin(TimeGenerated, 1h)
   ```

**Customizing These Metrics:**  
- **Adjust the Time Bin Size:** You can change `bin(TimeGenerated, 1h)` to a smaller or larger interval depending on how granular you want your insights.  
- **Add Additional Filters:** For example, if you only care about certain hosts, backend pools, or specific response statuses, you can add additional `where` conditions.  
- **Use Percentiles and Averages:** For metrics like response times or data throughput, consider adding `percentile()` or `avg()` to understand the distribution more clearly.

By combining these queries and metrics, you can gain a comprehensive view of how your Application Gateway is performing, identify issues quickly, and plan capacity or optimization efforts effectively.
If you’d like to focus specifically on request/response performance, you can use metrics related to request processing time or response latency. Application Gateway logs often include a field like `TimeTaken` (usually in milliseconds) that indicates how long it took to handle a request. You can aggregate this data over time to observe average, minimum, maximum, and percentile values for response performance.

Here’s a KQL query that analyzes request and response performance metrics over time:

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| extend RequestTimeMS = toint(TimeTaken)
| summarize
    AverageTimeMS = avg(RequestTimeMS),
    P95TimeMS = percentile(RequestTimeMS, 95),
    MaxTimeMS = max(RequestTimeMS),
    MinTimeMS = min(RequestTimeMS)
    by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

**Key Steps:**
1. **Filter for Application Gateway Logs:**  
   The `where ResourceType == "APPLICATIONGATEWAYS"` clause ensures you’re only working with logs related to your Application Gateway.

2. **Extract and Convert Request Time:**  
   - `TimeTaken` is typically a string or numeric field representing the request handling time in milliseconds.  
   - `toint()` converts it to an integer so you can perform numerical calculations.  
   - If your environment uses a different field for latency, replace `TimeTaken` with that field’s name.

3. **Aggregate Performance Metrics:**  
   - `avg(RequestTimeMS)` calculates the average request time.  
   - `percentile(RequestTimeMS, 95)` calculates the 95th percentile, a common measure to highlight high latency scenarios without being skewed by extreme outliers.  
   - `max()` and `min()` give you the range of observed response times.

4. **Summarize Over Time Intervals:**  
   - `bin(TimeGenerated, 1h)` groups the metrics into hourly intervals. Adjust this to a finer granularity if needed (e.g., `10m` or `5m`).

5. **Sort Results by Time:**  
   - Sorting by `TimeGenerated` helps you visualize performance trends over time, making it easier to spot increases in latency or performance degradation.

**What This Query Tells You:**  
- The average response time over each time interval.  
- The maximum and minimum request times, helping identify spikes or bottlenecks.  
- The 95th percentile response time, highlighting the experience of the slowest 5% of requests.

You can use these insights to understand how your Application Gateway handles traffic, diagnose potential performance issues, and track improvements over time.

# KustoQuery

You can use the **Heartbeat** table to calculate the uptime percentage by determining how frequently heartbeat signals were sent during a given time window. Each entry in the **Heartbeat** table typically represents a "ping" that indicates the monitored resource was online and responsive at that moment. 

Below is a Kusto query that calculates the uptime percentage within each hourly time bin:

```kql
Heartbeat
| summarize HeartbeatCount = count(), 
            TimeSlots = dcount(bin(TimeGenerated, 1m)) 
            by Computer, bin(TimeGenerated, 1h)
| extend UptimePercentage = (HeartbeatCount * 100.0) / TimeSlots
| project TimeGenerated, Computer, UptimePercentage
| order by TimeGenerated asc
```

**How the Query Works:**

1. **Count Heartbeats (HeartbeatCount):**
   - Each record in the **Heartbeat** table represents a status check. By counting the total number of heartbeat entries for a given computer in an hourly bin, we know how many times that computer reported itself as online.

2. **Count Distinct Time Slots (TimeSlots):**
   - If heartbeats are expected every minute, you can count how many unique minute slots occurred in that hour. This represents the number of expected heartbeats during a fully "up" hour.
   - Adjust the `bin(TimeGenerated, 1m)` interval if heartbeats occur at a different frequency.

3. **Calculate Uptime Percentage:**
   - The percentage is `(HeartbeatCount / TimeSlots) * 100`.
   - If all expected heartbeats are present, the percentage will be close to 100%. Fewer heartbeats indicate downtime, reducing the percentage.

4. **Projection:**
   - You can project fields like the timestamp, the computer’s name, and the calculated uptime percentage for clarity.

5. **Sorting:**
   - The query sorts the results by time, making it easier to see uptime trends over time.

**Important Notes:**
- Make sure the time interval for binning matches the expected frequency of heartbeats. If you expect a heartbeat every minute, a `1m` interval works well. If heartbeats occur at different intervals, adjust the binning accordingly.
- If a resource is consistently emitting heartbeats without missing intervals, the uptime percentage will be 100%. Missing heartbeats (indicating downtime) will lower this value.

This approach gives you a detailed breakdown of uptime percentages over time, using the **Heartbeat** table as the source of truth for availability.
To query uptime and downtime from `InsightsMetrics`, you typically look at a metric that reflects the availability or status of a resource. For example, if the metric reports availability as a percentage (e.g., 100% for fully up, 0% for fully down), you can analyze how often the resource is fully available versus partially or fully unavailable.

**Example Query:**
```kql
InsightsMetrics
| where Name == "Availability"  // Adjust "Availability" to the actual metric name for uptime if different
| extend UptimeStatus = case(
    Val == 100, "Uptime",
    Val > 0 and Val < 100, "Partial Downtime",
    Val == 0, "Downtime",
    "Unknown"
)
| summarize 
    UptimeCount = countif(UptimeStatus == "Uptime"),
    PartialDowntimeCount = countif(UptimeStatus == "Partial Downtime"),
    DowntimeCount = countif(UptimeStatus == "Downtime"),
    TotalEntries = count(),
    UptimePercentage = 100 * countif(UptimeStatus == "Uptime") / count(),
    DowntimePercentage = 100 * countif(UptimeStatus == "Downtime") / count()
    by bin(TimeGenerated, 1h)
| order by TimeGenerated asc
```

**How This Query Works:**
1. **Filter by Metric Name:**  
   Replace `"Availability"` with the actual metric name that reflects uptime or availability in your `InsightsMetrics` dataset.

2. **Categorize Uptime and Downtime:**  
   The `case()` function categorizes data into three groups:  
   - `Uptime` when the value is `100` (fully available).  
   - `Partial Downtime` when the value is greater than `0` but less than `100` (partially available).  
   - `Downtime` when the value is `0` (completely unavailable).

3. **Summarize by Time Bins:**  
   - Count how many instances fall into each category (`UptimeCount`, `PartialDowntimeCount`, `DowntimeCount`).
   - Calculate uptime and downtime percentages.
   - Use `bin(TimeGenerated, 1h)` to break the data into hourly intervals. Adjust the time bin if needed.

4. **Order by Time:**  
   Sorting results by `TimeGenerated` helps you see how uptime/downtime trends over time.

**What You’ll Get:**
- **Hourly intervals** showing the count of uptime, partial downtime, and full downtime events.
- **Percentages** indicating the proportion of time the resource was up or down.
- A clear trend of availability over the queried time period.

This query can help you understand the stability of your resource and identify periods of reduced availability or outright downtime.





Perfect! Let’s walk through a **detailed KQL** solution for tracking **Azure Application Gateway uptime** using **AzureDiagnostics** and **AzureMetrics** tables in **Log Analytics**.

Since Application Gateway logs are stored in the `AzureDiagnostics` table (for access logs, firewall logs, etc.), and availability can be tracked via **successful requests** or **metrics** like `UnhealthyHostCount`, you have multiple ways to approach it depending on the signals available.

---

## ✅ **Approach 1: Uptime Based on Successful Request Logs (`AzureDiagnostics`)**

This approach assumes:
- You define "uptime" as the presence of **successful HTTP requests** through the App Gateway.
- You want to detect periods with **no traffic or only failed traffic**.

```kusto
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and TimeGenerated > ago(7d)
| where Category == "ApplicationGatewayAccessLog"
| extend StatusCode = tostring(propertiesHttpStatus)
| extend RequestSuccess = iff(toint(StatusCode) between (200 .. 399), 1, 0)
| summarize
    TotalRequests = count(),
    SuccessfulRequests = countif(RequestSuccess == 1),
    FailedRequests = countif(RequestSuccess == 0),
    UptimePct = round(100.0 * SuccessfulRequests / TotalRequests, 2)
by bin(TimeGenerated, 1h), Resource
| order by TimeGenerated asc
```

### 📊 Output:
| TimeGenerated     | Resource             | TotalRequests | SuccessfulRequests | FailedRequests | UptimePct |
|-------------------|----------------------|----------------|---------------------|----------------|-----------|
| 2025-03-26 14:00  | appgw-prod-westus     | 120            | 118                 | 2              | 98.33     |

> This gives you hourly uptime % based on successful HTTP status codes.

---

## ✅ **Approach 2: Uptime Based on Backend Health (`AzureMetrics`)**

This is a more infrastructure-level view — using `UnhealthyHostCount` from metrics.

```kusto
AzureMetrics
| where Resource == "your-app-gateway-name"
| where MetricName == "UnhealthyHostCount"
| where TimeGenerated > ago(7d)
| summarize
    AvgUnhealthy = avg(Total),
    MaxUnhealthy = max(Total)
by bin(TimeGenerated, 5m), Resource
| extend Healthy = iff(MaxUnhealthy == 0, 1, 0)
| summarize
    TotalIntervals = count(),
    HealthyIntervals = countif(Healthy == 1),
    DowntimeIntervals = countif(Healthy == 0),
    UptimePct = round(100.0 * HealthyIntervals / TotalIntervals, 2)
by Resource
```

### 📊 Output:
| Resource            | TotalIntervals | HealthyIntervals | DowntimeIntervals | UptimePct |
|---------------------|----------------|------------------|-------------------|-----------|
| appgw-prod-westus   | 2016           | 2000             | 16                | 99.21     |

> This is great for **SLA monitoring**: uptime % by backend health.

---

## ✅ **Approach 3: Detect Missing Logs (Potential Downtime Periods)**

```kusto
range TimeSlot from ago(7d) to now() step 5m
| join kind=leftanti (
    AzureDiagnostics
    | where ResourceType == "APPLICATIONGATEWAYS"
    | where Category == "ApplicationGatewayAccessLog"
    | summarize by bin(TimeGenerated, 5m)
) on $left.TimeSlot == $right.TimeGenerated
```

### 📊 Output:
This query gives you **time slots where no App Gateway logs were received**, possibly indicating a downtime or traffic interruption.

---

## ✅ Bonus: Real-Time Uptime Status for Today

```kusto
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where TimeGenerated >= startofday(now())
| extend IsSuccess = iff(toint(propertiesHttpStatus) between (200 .. 399), 1, 0)
| summarize
    Total = count(),
    Success = countif(IsSuccess == 1),
    Failures = countif(IsSuccess == 0),
    UptimePct = round(100.0 * Success / Total, 2)
```

---

## 📈 Suggested Visuals for Azure Workbook or Power BI

| Visual Type    | Metric                         | Source            |
|----------------|--------------------------------|--------------------|
| Line Chart     | Uptime % per hour              | AzureDiagnostics   |
| Bar Chart      | Backend Unhealthy Host Count   | AzureMetrics       |
| Table          | Failed Requests Summary        | AzureDiagnostics   |
| KPI            | Uptime % today                 | Both               |

---

### Want to monitor for specific status codes like 502, 503?
```kusto
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where propertiesHttpStatus in ("502", "503")
| summarize count() by propertiesHttpStatus, bin(TimeGenerated, 1h)
```

---

Let me know:
- Your App Gateway's **name or region**
- If you’re using **WAF**, we can also pull WAF log-based downtime.
- Or if you'd like a **ready-to-import Workbook JSON**.
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

Below is a detailed **Kusto Query Language (KQL)** script for each of the **key SignInLogs indicators** you can use in **Azure Monitor Workbooks**, **Sentinel**, or **Power BI**.

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
