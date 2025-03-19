# KustoQuery

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
