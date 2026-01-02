
## 📄 AQL-CheatSheet.md

```markdown
# 🔍 IBM QRadar AQL Cheat Sheet

![AQL](https://img.shields.io/badge/AQL-Ariel_Query_Language-blue)
![QRadar](https://img.shields.io/badge/Platform-IBM_QRadar-red)

## 📋 Quick Reference

### 🔍 Basic Queries
```sql
-- Basic select
SELECT * FROM events
SELECT * FROM events LAST 1 HOURS
SELECT * FROM flows LAST 24 HOURS

-- Specific fields
SELECT username, sourceip, eventname FROM events
SELECT starttime, sourceip, destinationip, destinationport FROM flows

-- Time ranges
SELECT * FROM events WHERE starttime > '2024-01-01 00:00:00'
SELECT * FROM events WHERE starttime < '2024-01-02 23:59:59'
SELECT * FROM events WHERE starttime LAST 30 MINUTES

-- Filtering
SELECT * FROM events WHERE severity > 5
SELECT * FROM events WHERE eventname ILIKE '%failed%'
SELECT * FROM events WHERE sourceip = '192.168.1.100'
```
## 📊 Aggregation & Grouping
```sql
-- Basic aggregation
SELECT sourceip, COUNT(*) as event_count 
FROM events 
GROUP BY sourceip

SELECT username, COUNT(DISTINCT sourceip) as unique_ips
FROM events 
GROUP BY username

-- Multiple aggregations
SELECT 
    sourceip,
    COUNT(*) as total_events,
    SUM(severity) as total_severity,
    AVG(severity) as avg_severity,
    MAX(severity) as max_severity
FROM events 
GROUP BY sourceip

-- Having clause
SELECT sourceip, COUNT(*) as attempts
FROM events 
WHERE eventname ILIKE '%login%'
GROUP BY sourceip
HAVING COUNT(*) > 10
```
## ⏰ Time Analysis
```sql
-- Time grouping
SELECT 
    HOUR(starttime) as hour_of_day,
    DAYNAME(starttime) as day_name,
    COUNT(*) as event_count
FROM events 
GROUP BY HOUR(starttime), DAYNAME(starttime)

-- Time series
SELECT 
    QUARTER(starttime) as quarter_hour,
    sourceip,
    COUNT(*) as event_count
FROM events 
GROUP BY QUARTER(starttime), sourceip

-- Time windows
SELECT 
    sourceip,
    COUNT(*) as events_last_hour
FROM events 
WHERE starttime > NOW() - 1 HOUR
GROUP BY sourceip
```
## 🔗 Joins & Subqueries
```sql
-- Basic join
SELECT 
    e.*,
    a.categoryname 
FROM events e
JOIN eventcategories a ON e.category = a.categoryid

-- Subquery
SELECT sourceip, event_count
FROM (
    SELECT sourceip, COUNT(*) as event_count
    FROM events
    GROUP BY sourceip
) AS ip_counts
WHERE event_count > 100

-- EXISTS clause
SELECT *
FROM events e
WHERE EXISTS (
    SELECT 1 
    FROM reference_set_blocklist b
    WHERE b.value = e.sourceip
)

-- IN clause with subquery
SELECT *
FROM events
WHERE sourceip IN (
    SELECT value 
    FROM reference_set_malicious_ips
)
```
## 🛡️ Security Detection Patterns
```sql
-- Failed authentication
SELECT 
    username,
    sourceip,
    COUNT(*) as failed_attempts
FROM events 
WHERE 
    eventname ILIKE '%failed%' 
    OR eventname ILIKE '%denied%'
    AND starttime > NOW() - 15 MINUTES
GROUP BY username, sourceip
HAVING COUNT(*) > 5

-- Port scanning detection
SELECT 
    sourceip,
    destinationip,
    COUNT(DISTINCT destinationport) as unique_ports,
    COUNT(*) as total_attempts
FROM flows 
WHERE 
    starttime > NOW() - 10 MINUTES
    AND direction = 'INBOUND'
GROUP BY sourceip, destinationip
HAVING COUNT(DISTINCT destinationport) > 20

-- Data exfiltration
SELECT 
    sourceip,
    destinationip,
    destinationport,
    SUM(sourcebytes) as total_bytes_sent,
    MAX(sourcebytes) as max_chunk
FROM flows
WHERE 
    starttime > NOW() - 1 HOUR
    AND sourcebytes > 1000000  -- 1MB chunks
GROUP BY sourceip, destinationip, destinationport
HAVING SUM(sourcebytes) > 100000000  -- 100MB total

-- Malware execution patterns
SELECT 
    sourceip,
    destinationip,
    COUNT(*) as connections,
    GROUP_CONCAT(DISTINCT destinationport) as ports_used
FROM events
WHERE 
    (eventname ILIKE '%powershell%' OR eventname ILIKE '%cmd%')
    AND starttime > NOW() - 5 MINUTES
GROUP BY sourceip, destinationip
HAVING COUNT(*) > 10
```
## 🔍 Reference Sets
```sql
-- Check against reference sets
SELECT *
FROM events
WHERE sourceip IN REFERENCE_SET('malicious_ips')

SELECT *
FROM flows 
WHERE 
    sourceip IN REFERENCE_SET('internal_ips')
    AND destinationip NOT IN REFERENCE_SET('internal_ips')

-- Create dynamic reference sets
SELECT DISTINCT sourceip
INTO REFERENCE SET 'suspicious_ips_temp'
FROM events
WHERE severity > 7
  AND starttime > NOW() - 1 HOUR

-- Use reference sets in conditions
SELECT 
    CASE 
        WHEN sourceip IN REFERENCE_SET('high_risk_ips') THEN 'HIGH'
        WHEN sourceip IN REFERENCE_SET('medium_risk_ips') THEN 'MEDIUM'
        ELSE 'LOW'
    END as risk_level,
    COUNT(*) as event_count
FROM events
GROUP BY risk_level
```
## 📈 Advanced Patterns
```sql
-- Session analysis
SELECT 
    sourceip,
    username,
    MIN(starttime) as session_start,
    MAX(starttime) as session_end,
    COUNT(*) as events_in_session,
    LIST(eventname, ' | ') as activities
FROM events
WHERE 
    starttime > NOW() - 1 HOUR
    AND username IS NOT NULL
GROUP BY sourceip, username
HAVING COUNT(*) > 5

-- Baseline deviation
WITH hourly_baseline AS (
    SELECT 
        HOUR(starttime) as hour,
        sourceip,
        AVG(COUNT(*)) OVER (
            PARTITION BY sourceip 
            ORDER BY HOUR(starttime) 
            ROWS BETWEEN 24 PRECEDING AND 1 PRECEDING
        ) as avg_events
    FROM events
    WHERE starttime > NOW() - 7 DAYS
    GROUP BY HOUR(starttime), sourceip
)
SELECT 
    hour,
    sourceip,
    COUNT(*) as current_events,
    avg_events,
    CASE 
        WHEN COUNT(*) > avg_events * 2 THEN 'ABNORMAL'
        ELSE 'NORMAL'
    END as status
FROM hourly_baseline
JOIN events ON HOUR(events.starttime) = hourly_baseline.hour
WHERE events.starttime > NOW() - 1 HOUR
GROUP BY hour, sourceip, avg_events

-- Correlation search
SELECT 
    e1.username,
    e1.sourceip,
    e1.eventname as first_event,
    e2.eventname as second_event,
    TIMESTAMPDIFF('MINUTE', e1.starttime, e2.starttime) as minutes_between
FROM events e1
JOIN events e2 ON e1.username = e2.username 
    AND e1.sourceip = e2.sourceip
    AND e2.starttime > e1.starttime
    AND e2.starttime <= e1.starttime + 5 MINUTES
WHERE 
    e1.eventname = 'User Logon'
    AND e2.eventname ILIKE '%file_access%'
```
## ⚡ Performance Tips
```sql
-- Use indexes (QRadar automatically indexes time and common fields)
SELECT * FROM events WHERE starttime > '2024-01-01'  -- Uses time index

-- Limit results early
SELECT * FROM (
    SELECT * FROM events 
    WHERE starttime > NOW() - 1 HOUR
    ORDER BY starttime DESC
    LIMIT 1000
) ORDER BY starttime ASC

-- Avoid SELECT * when possible
SELECT sourceip, eventname, severity FROM events  -- Better than SELECT *

-- Use WHERE before GROUP BY
SELECT sourceip, COUNT(*)
FROM events
WHERE severity > 3  -- Filter first
GROUP BY sourceip

-- Use appropriate time ranges
SELECT * FROM events LAST 1 HOURS  -- Better than fixed time if recent

-- Use reference sets for IP lists
SELECT * FROM events 
WHERE sourceip IN REFERENCE_SET('watchlist')  -- Faster than IN with values
```
## 🔗 Useful Functions
```sql
-- String functions
SELECT 
    UPPER(username) as upper_user,
    LOWER(eventname) as lower_event,
    SUBSTRING(sourceip, 1, 3) as ip_prefix,
    CONCAT(sourceip, ' -> ', destinationip) as connection,
    REPLACE(eventname, 'FAILED', 'DENIED') as normalized_name

-- Time functions
SELECT 
    NOW() as current_time,
    DAYNAME(starttime) as day_name,
    HOUR(starttime) as hour_of_day,
    WEEK(starttime) as week_number,
    TIMESTAMPDIFF('SECOND', starttime, endtime) as duration_seconds

-- IP functions
SELECT 
    INET_NTOA(sourceip) as readable_ip,  -- Convert integer IP to string
    IS_IPV4(sourceip) as is_ipv4,
    IP_CATEGORY(sourceip) as ip_category

-- Math functions
SELECT 
    ROUND(AVG(severity), 2) as avg_severity,
    CEIL(SUM(sourcebytes)/1024/1024) as total_mb,
    FLOOR(AVG(destinationport)) as avg_port

-- Conditional
SELECT 
    CASE 
        WHEN severity >= 8 THEN 'CRITICAL'
        WHEN severity >= 5 THEN 'HIGH'
        WHEN severity >= 3 THEN 'MEDIUM'
        ELSE 'LOW'
    END as priority_level,
    COUNT(*) as event_count
FROM events
GROUP BY priority_level
```
## 🚨 Rule Building Examples
```sql
-- Brute force rule
SELECT 
    sourceip,
    username,
    COUNT(*) as failed_attempts,
    MIN(starttime) as first_attempt,
    MAX(starttime) as last_attempt
FROM events
WHERE 
    devicetype = 'Authentication Server'
    AND (eventname ILIKE '%fail%' OR severity >= 5)
    AND starttime > NOW() - 15 MINUTES
GROUP BY sourceip, username
HAVING COUNT(*) >= 5

-- Data theft detection
SELECT 
    sourceip,
    destinationip,
    SUM(sourcebytes) as bytes_sent,
    COUNT(DISTINCT destinationport) as ports_used,
    LIST(DISTINCT username) as users_involved
FROM flows
WHERE 
    starttime > NOW() - 1 HOUR
    AND sourcebytes > 1000000  -- Large transfers
    AND destinationip NOT IN REFERENCE_SET('internal_ips')
GROUP BY sourceip, destinationip
HAVING SUM(sourcebytes) > 50000000  -- 50MB total

-- Lateral movement
SELECT 
    src.username as source_user,
    src.sourceip as source_host,
    dst.username as destination_user,
    dst.sourceip as destination_host,
    COUNT(*) as connection_attempts
FROM events src
JOIN events dst ON src.username = dst.username
    AND src.sourceip != dst.sourceip
    AND dst.starttime BETWEEN src.starttime AND src.starttime + 10 MINUTES
WHERE 
    src.eventname = 'Successful Logon'
    AND dst.eventname = 'Successful Logon'
    AND src.starttime > NOW() - 1 HOUR
GROUP BY src.username, src.sourceip, dst.username, dst.sourceip
```
## 📚 Resources & References

### **Official Documentation**
* [Splunk Enterprise Security Docs](https://docs.splunk.com/Documentation/ES)
* [Microsoft Sentinel Documentation](https://learn.microsoft.com/en-us/azure/sentinel/)
* [IBM QRadar Documentation](https://www.ibm.com/docs/en/qradar-common)

### **Community & Studies**
* [Gartner Magic Quadrant for SIEM (2025)](https://www.microsoft.com/en-us/security/blog/2025/10/16/microsoft-named-a-leader-in-the-2025-gartner-magic-quadrant-for-siem/)
* [Azure Sentinel GitHub Repository](https://github.com/Azure/Azure-Sentinel)
* [Splunk Answers (Community Forum)](https://community.splunk.com/t5/Find-Answers/ct-p/en-us-splunk-answers)
