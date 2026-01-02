# 🔍 Azure Sentinel KQL Cheat Sheet

![KQL](https://img.shields.io/badge/KQL-Kusto_Query_Language-blue)
![Sentinel](https://img.shields.io/badge/Platform-Azure_Sentinel-purple)

## 📋 Quick Reference

### 🔍 Basic Queries
```kql
// Basic table query
SecurityEvent
SecurityEvent | take 100
SecurityAlert | count

// Time ranges
SecurityEvent | where TimeGenerated > ago(1h)
SecurityEvent | where TimeGenerated between (datetime(2024-01-01) .. datetime(2024-01-02))
SigninLogs | where TimeGenerated > startofday(ago(7d))

// Multiple conditions
SecurityEvent | where EventID == 4625 and Account != "SYSTEM"
```
## 📊 Aggregation & Statistics

### **Azure Sentinel KQL**

```kql
// Basic stats
SecurityEvent | summarize count() by Account
SecurityEvent | summarize Count=count(), UniqueUsers=dcount(Account) by Computer

// Multiple aggregations
SecurityEvent 
| summarize 
    Total=count(),
    Failed=countif(EventID == 4625),
    SuccessRate=todouble(countif(EventID == 4624)) / count() * 100
    by Computer

// Time series
SecurityEvent 
| make-series count() default=0 on TimeGenerated from ago(7d) to now() step 1h
| render timechart

// Percentiles
Perf | summarize percentiles(CounterValue, 50, 95, 99) by Computer
```
## 🔗 Data Manipulation

### **Azure Sentinel KQL**

```kql
// Extend (add columns)
SecurityEvent | extend RiskScore = iif(EventID == 4625, 10, 1)
SigninLogs | extend Location = strcat(City, ", ", Country)

// Project (select columns)
SecurityEvent | project TimeGenerated, Computer, Account, EventID
SecurityEvent | project-away SourceIP, DestinationIP  // Remove columns

// Sort & limit
SecurityEvent | top 10 by TimeGenerated desc
SecurityEvent | order by TimeGenerated asc

// Distinct values
SecurityEvent | distinct Account
```
## 🔄 Joins & Unions
```kql
// Inner join
SecurityEvent 
| join kind=inner (SecurityAlert) on $left.Account == $right.AccountName

// Left join
SecurityEvent
| join kind=leftouter (ThreatIntelligenceIndicator) on $left.IpAddress == $right.NetworkIP

// Union (combine tables)
union SecurityEvent, WindowsFirewall
| where TimeGenerated > ago(1h)

// Append (add rows)
SecurityEvent
| union SigninLogs
```
## 🛡️ Security Detection Patterns
```kql
// Failed logins
SecurityEvent
| where EventID == 4625
| summarize FailedCount = count() by Account, IpAddress
| where FailedCount > 5

// Brute force detection
SigninLogs
| where ResultType == "50057"  // User locked out
| summarize count() by UserPrincipalName, bin(TimeGenerated, 5m)
| where count_ > 3

// Port scanning
AzureDiagnostics
| where ResourceType == "NETWORKSECURITYGROUPS"
| where direction_s == "Inbound"
| summarize ScanAttempts = count() by srcIpAddress_s, destPort_d
| where ScanAttempts > 100

// Data exfiltration
CommonSecurityLog
| where DeviceVendor == "Palo Alto Networks"
| where DestinationPort == 443
| where BytesSent > 100000000
| summarize TotalSent = sum(BytesSent) by SourceIP

// PowerShell execution
SecurityEvent
| where EventID == 4104  // PowerShell module logging
| where CommandLine has_cs "Invoke-"
```
## 📈 Time Series Analysis
```kql
// Anomaly detection
let timeframe = 1h;
let threshold = 2.0;
SecurityEvent
| make-series count() on TimeGenerated from ago(7d) to now() step timeframe
| extend anomalies = series_decompose_anomalies(count_, threshold)

// Moving average
SecurityEvent
| make-series count() on TimeGenerated from ago(30d) to now() step 1d
| extend ma = series_fir(count_, repeat(1, 7), true, true)  // 7-day moving avg

// Time comparison
SecurityEvent
| summarize Today=count() by bin(TimeGenerated, 1h)
| join kind=inner (
    SecurityEvent
    | where TimeGenerated between (ago(2d) .. ago(1d))
    | summarize Yesterday=count() by bin(TimeGenerated, 1h)
) on TimeGenerated
```
## 🔍 Advanced Operators
```kql
// Parse patterns
Syslog
| parse SourceIP with "src=" ip ":"
| parse EventText with "user=" user " " action " "

// Let statements (variables)
let startTime = ago(1h);
let endTime = now();
SecurityEvent
| where TimeGenerated between (startTime .. endTime)

// Materialize (cache results)
let suspiciousIPs = materialize(
    ThreatIntelligenceIndicator
    | where Active == true
    | distinct NetworkIP
);
SecurityEvent
| where IpAddress in (suspiciousIPs)

// Functions
let detectBruteForce = (threshold: int) {
    SecurityEvent
    | where EventID == 4625
    | summarize count() by Account, bin(TimeGenerated, 5m)
    | where count_ > threshold
};
detectBruteForce(10)
```
##🚨 Alert & Hunting Queries
```kql
// Account creation & immediate use
let newAccounts = SecurityEvent
| where EventID == 4720
| project NewAccount = TargetAccount, CreationTime = TimeGenerated;
SecurityEvent
| where EventID == 4624
| join kind=inner newAccounts on $left.Account == $right.NewAccount
| where TimeGenerated - CreationTime < timespan(5m)

// Lateral movement detection
SecurityEvent
| where EventID in (4624, 4625)
| summarize 
    SuccessLogons = countif(EventID == 4624),
    FailedLogons = countif(EventID == 4625)
    by Account, Computer, bin(TimeGenerated, 1h)
| where FailedLogons > 3 and SuccessLogons > 0

// Data staging detection
SecurityEvent
| where EventID == 4663  // File accessed
| where ObjectName endswith ".zip" or ObjectName endswith ".rar"
| summarize FilesCompressed = count() by Account, Computer
```
## ⚡ Performance Optimization
```kql
// Use time filters first
SecurityEvent
| where TimeGenerated > ago(1h)  // ← Always do this first
| where EventID == 4625

// Reduce data early
SecurityEvent
| where TimeGenerated > ago(1h)
| summarize by Account, Computer  // Reduce before joins
| join kind=inner (ThreatIntel) on $left.Account == $right.AccountName

// Use materialize() for repeated subqueries
let criticalHosts = materialize(
    Heartbeat
    | where Computer in~ ("server1", "server2", "dc1")
);
SecurityEvent | where Computer in (criticalHosts)

// Partition large tables
SecurityEvent
| where Computer startswith "web"  // Partition by naming convention
```
## 🔗 Useful Functions
```kql
// String functions
| extend lowerUser = tolower(UserPrincipalName)
| extend containsAdmin = UserPrincipalName contains "admin"
| extend domain = split(UserPrincipalName, "@")[1]

// Time functions
| extend hour = datetime_part("hour", TimeGenerated)
| extend dayOfWeek = dayofweek(TimeGenerated)
| extend duration = endTime - startTime

// IP functions
| extend isPrivate = ipv4_is_private(SourceIP)
| extend ipRange = ipv4_netmask_suffix(SourceIP, "255.255.255.0")

// Array functions
| extend processes = dynamic(["cmd.exe", "powershell.exe", "wscript.exe"])
| where Process in~ (processes)

// Geospatial
| extend location = geo_info_from_ip_address(ClientIP)
| extend country = location.country
```
