If you need to get information about the size of billable data ingested per Log Analytics table with an indication of Azure subscription responsible for generation of that data, use the following KQL query:

```
//We employ 'find' operator to return all tables containing specific columns (with billable data)
//This is computationally costly - use for limited range of lookback periods (ideally up to 14 days, I do not recommend to go beyond 30 days)
let lookback = 30d;
find withsource=SourceTable where TimeGenerated between(startofday(ago(lookback))..startofday(now())) project _BilledSize, _IsBillable, _SubscriptionId
| where _IsBillable == true
| summarize BillableDataBytes = sum(_BilledSize) by _SubscriptionId, SourceTable
| project BillableDataMB = format_bytes(BillableDataBytes,0,"MB"), _SubscriptionId, SourceTable
//Let's perform cross-cluster join with Azure Resource Graph (ARG) instance to map Azure subcription IDs to human readable names
| join hint.remote=local kind=inner (
//We are querying resourcecontainers ARG table and listing all subscriptions registered in our tenant
arg("").resourcecontainers 
| where type == "microsoft.resources/subscriptions"
//Some transformations required to ensure that subscriptionID is correctly extracted and mapped to correct data type
| project SubID = split(id,"/",2), name
| mv-expand bagexpansion=bag SubID to typeof(string) limit 2000
) on $left._SubscriptionId == $right.SubID
|project BillableDataMB, SourceTable, SubID, SubName = name
| sort by SubName asc
```
It will return amount of data ingested into each Log Analytics table alongside the ID and the name of an Azure subscription responsible for generating such data:
![image](https://github.com/user-attachments/assets/a510d797-981f-41fa-84a9-3ffdecba1103)

## Helpful sources
- [Analyze usage in a Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/analyze-usage)
- [KQL find operator](https://learn.microsoft.com/en-us/kusto/query/find-operator)
