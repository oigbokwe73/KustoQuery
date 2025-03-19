# KustoQuery

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
