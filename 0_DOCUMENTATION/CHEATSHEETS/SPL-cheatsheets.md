
# SIEM Comparison & Engineering Guide: Splunk vs. Azure Sentinel vs. IBM QRadar

A comprehensive technical comparison of market-leading SIEM platforms and a specialized query language reference for security engineers.

---

> [!CAUTION]
> ### ⚠️ Disclaimer
> This comparison is based on publicly available information, product documentation, and hands-on testing as of **January 2024**. Pricing, features, and capabilities change frequently. Always verify with official sources and conduct proof-of-concepts for your specific requirements.

---

## 🔍 1. Detailed Feature Breakdown: Data Collection & Parsing

### **Splunk**
* **Strengths:**
    * Universal Forwarder (lightweight, efficient)
    * 600+ Technology Add-ons (TA)
    * Heavy Forwarder for parsing/transformation
    * HTTP Event Collector (HEC)
    * Support for any log format
* **Weaknesses:**
    * Forwarder management overhead
    * Parsing happens at index time (rigid)
* **Metrics:**
    * **Data Sources:** 9.5/10
    * **Parsing Flexibility:** 9/10

---

### **Azure Sentinel**
* **Strengths:**
    * Native Azure integration (zero config for Azure resources)
    * 100+ Data Connectors (Microsoft + 3rd party)
    * CEF/Syslog support
    * Azure Monitor Agent (AMA)
    * REST API for custom sources
* **Weaknesses:**
    * Limited parsing customization
    * Some connectors are premium
* **Metrics:**
    * **Data Sources:** 8/10
    * **Parsing Flexibility:** 7/10

---

### **IBM QRadar**
* **Strengths:**
    * Protocol support (Syslog, SNMP, JDBC, etc.)
    * QRadar DSM Editor for custom parsing
    * 400+ Supported log sources
    * High-performance collectors
* **Weaknesses:**
    * Complex parsing configuration
    * Limited cloud-native support
* **Metrics:**
    * **Data Sources:** 8/10
    * **Parsing Flexibility:** 8/10

---

## 💻 2. Query & Search Capabilities

### **Comparative Overview**

| Query Type | Splunk (SPL) | Azure Sentinel (KQL) | IBM QRadar (AQL) |
| :--- | :--- | :--- | :--- |
| **Basic Search** | `index=security` | `SecurityEvent` | `SELECT * FROM events` |
| **Time Filter** | `earliest=-1h` | `| where TimeGenerated > ago(1h)` | `START '1 hours ago'` |
| **Aggregation** | `stats count by user` | `| summarize count() by user` | `GROUP BY user SELECT COUNT(*)` |
| **Joins** | `join type=left` | `| join kind=leftouter` | `JOIN ... ON ...` |
| **Performance** | Fast with indexing | Very fast (columnar) | Fast with tuning |

### **Example: Failed Login Detection**

**Splunk SPL**
```sql
index=wineventlog EventCode=4625 | stats count by user, src_ip | where count > 5 | table user, src_ip, count
```
**Azure Sentinel KQL**
```sql
SecurityEvent | where EventID == 4625 | summarize FailedCount = count() by Account, IpAddress | where FailedCount > 5
```
**IBM QRadar AQL**
```sql
SELECT username, sourceip, COUNT(*) FROM events WHERE eventname='Failed Login' GROUP BY username, sourceip HAVING COUNT(*) > 5
```
# 🔍 3. Splunk SPL Cheat Sheet
## Basic Search & Stats
```SQL

-- Filtering
index=security earliest=-1h NOT user=service_*
-- Aggregation
| stats count, dc(user) as unique_users by src_ip
-- Visualization
| timechart span=1h count by status
```
## Security Patterns
```SQL

-- Brute Force (Linux)
index=linux "Failed password" | bucket _time span=1m | stats count by src_ip, _time | where count > 10

-- Data Exfiltration
index=proxy url=*pastebin.com* | stats sum(bytes) as total_bytes by user | where total_byt
```
# Statistical Commands
```sql
| stats count by user
| stats count, avg(response_time) by src_ip
| stats earliest(_time) as first_seen, latest(_time) as last_seen by user
| stats values(event_id) as events, dc(user) as unique_users
| stats count(eval(status>=500)) as errors by hour

-- Top/Limit results
| top 10 user
| rare user  # Least common
```
# ⏰ Time Analysis
```sql
| timechart span=1h count by status
| timechart span=1d avg(response_time)
| bucket _time span=5m  # Group into 5min buckets
| timechart count(eval(action="block")) as blocks
```
# 🔗 Data Manipulation
```sql
-- Field extraction/creation
| eval risk_score = count * severity
| eval duration = end_time - start_time
| eval user_group = if(user="admin", "privileged", "standard")

-- Renaming fields
| rename src_ip as source_address

-- Field selection
| fields user, src_ip, timestamp, action
| fields - _raw, _time  # Remove fields

-- Sorting
| sort - count  # Descending
| sort user  # Ascending
```
# 🔄 Joins & Transactions
```sql
-- Transaction (related events)
| transaction user maxspan=5m
| transaction src_ip, dst_ip startswith="CONNECT" endswith="DISCONNECT"

-- Join (like SQL)
| join type=left user [search index=users | table user, department]
| join src_ip [search index=threat_intel | table ip, threat_type]

-- Append (union)
| append [search index=firewall]
```
# 🔎 Search Optimization
```sql
-- Fast filtering (index-time)
index=security sourcetype=access_* status=5*

-- Search-time filtering
| search user=admin

-- Efficient wildcards
src_ip=192.168.1.*  # Good
src_ip=*admin*      # Slow

-- Use summary indexing
| collect index=summary marker="report=hourly_stats"

-- Subsearch optimization
[search index=firewall action=deny | head 1000 | table src_ip]
```
# 🛡️ Security-Specific Patterns
```sql
-- Failed logins (Windows Event ID 4625)
index=windows EventCode=4625 
| stats count by user, src_ip 
| where count > 5

-- Brute force detection
index=linux sourcetype=linux_secure "Failed password"
| bucket _time span=1m
| stats count by src_ip, _time
| where count > 10

-- Port scanning
index=firewall action=deny 
| stats count by dst_port, src_ip
| where count > 20

-- Data exfiltration
index=proxy url=*pastebin.com* OR url=*github.com/raw*
| stats sum(bytes) as total_bytes by user
| where total_bytes > 1000000

-- Malware execution
index=endpoint process_name IN ("powershell.exe", "cmd.exe", "wscript.exe")
| stats count by parent_process, process_name
```
# 📈 Visualization & Reporting
```sql
-- Chart types
| chart count over user by status  # Column chart
| chart avg(response_time) over _time by region  # Line chart

-- Gauges & single value
| stats count as total_events
| gauge total_events 1000 5000

-- Geospatial
| iplocation src_ip
| geostats count latfield=lat longfield=lon

-- Convert to JSON/CSV
| outputlookup threat_ips.csv
| outputjson
```
# 🔗 Useful Functions
```sql
-- String functions
| eval lower_user = lower(user)
| eval upper_user = upper(user)
| eval substr_user = substr(user, 1, 3)
| eval contains_admin = if(match(user, "admin"), "yes", "no")

-- Math functions
| eval mb = round(bytes/1024/1024, 2)
| eval log_count = log10(count)

-- Time functions
| eval hour = strftime(_time, "%H")
| eval date = strftime(_time, "%Y-%m-%d")

-- Coalesce (first non-null)
| eval ip = coalesce(src_ip, dest_ip, "unknown")

-- Case/when
| eval priority = case(
    severity >= 8, "Critical",
    severity >= 5, "High",
    severity >= 3, "Medium",
    true(), "Low"
  )
```
# 🚨 Common Alert Templates
```sql
-- Threshold alert
index=security action=fail 
| stats count by user 
| where count > 10
| eval message = "User " + user + " has " + count + " failures"

-- Baseline deviation
index=network 
| timechart span=1h avg(bytes) as avg_bytes
| eval deviation = abs(bytes - avg_bytes)
| where deviation > (avg_bytes * 2)

-- Sequence detection
index=security 
| transaction user maxevents=3 
| search event_sequence="connect,authenticate,download"
```

## 📚 Resources & References

### **Official Documentation**
* [Splunk Enterprise Security Docs](https://docs.splunk.com/Documentation/ES)
* [Microsoft Sentinel Documentation](https://learn.microsoft.com/en-us/azure/sentinel/)
* [IBM QRadar Documentation](https://www.ibm.com/docs/en/qradar-common)

### **Community & Studies**
* [Gartner Magic Quadrant for SIEM](https://www.gartner.com/reviews/market/security-information-event-management)
* [Azure Sentinel GitHub Repository](https://github.com/Azure/Azure-Sentinel)
* [Splunk Answers (Community Forum)](https://community.splunk.com/)

