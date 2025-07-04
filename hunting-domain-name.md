## Work in progress

### Windows

```kql
let dt_lookBack = 1h;
let ioc_lookBack = 14d;
let EventQueryName = SecurityEvent
| where TimeGenerated >= ago(dt_lookBack) and Channel contains "Microsoft-Windows-Sysmon" and EventData contains "QueryName"
| extend parsed = parse_xml(EventData)
| mv-apply Data = parsed.EventData.Data on (summarize QueryName = tolower(take_anyif(Data["#text"], Data["@Name"] == "QueryName")));
let TIDomainName = ThreatIntelligenceIndicator
| where isnotempty(DomainName) and TimeGenerated >= ago(ioc_lookBack)
| extend DomainName = tolower(DomainName);
EventQueryName
| join kind=innerunique TIDomainName on $left.QueryName == $right.DomainName
| summarize arg_max(TimeGenerated, *) by Computer, QueryName
```

### Linux

From `TI map Domain entity to Syslog` rule:

```kql
let dt_lookBack = 1h;  // Define the time range to look back for syslog data (1 hour)
let ioc_lookBack = 14d;  // Define the time range to look back for threat intelligence indicators (14 days)
// Create a list of top-level domains (TLDs) from the threat feed for later validation
let list_tlds = ThreatIntelligenceIndicator
  | where isnotempty(DomainName)
  | where TimeGenerated > ago(ioc_lookBack)
  | summarize LatestIndicatorTime = arg_max(TimeGenerated, *) by IndicatorId
  | where Active == true and ExpirationDateTime > now()
  | extend parts = split(DomainName, '.')
  | extend tld = parts[(array_length(parts)-1)]
  | summarize count() by tostring(tld)
  | summarize make_list(tld);
// Fetch the latest active domain indicators from the threat intelligence data within the specified time range
let Domain_Indicators = ThreatIntelligenceIndicator
  | where isnotempty(DomainName)
  | where TimeGenerated >= ago(ioc_lookBack)
  | summarize LatestIndicatorTime = arg_max(TimeGenerated, *) by IndicatorId
  | where Active == true and ExpirationDateTime > now()
  | extend TI_DomainEntity = DomainName;
// Join the threat intelligence indicators with syslog data on matching domain entities
Domain_Indicators
  | join kind=innerunique (
    Syslog
    | where TimeGenerated > ago(dt_lookBack)
    // Extract domain patterns from syslog messages
    | extend domain = extract("(([a-z0-9]+(-[a-z0-9]+)*\\.)+[a-z]{2,})",1, tolower(SyslogMessage))
    | where isnotempty(domain)
    | extend parts = split(domain, '.')
    // Split out the top-level domain (TLD)
    | extend tld = parts[(array_length(parts)-1)]
    // Validate parsed domain by checking if the TLD is in the list of TLDs in our threat feed
    | where tld in~ (list_tlds)
    | extend Syslog_TimeGenerated = TimeGenerated
  ) on $left.TI_DomainEntity==$right.domain
  | where Syslog_TimeGenerated < ExpirationDateTime
  // Retrieve the latest syslog timestamp for each indicator and domain combination
  | summarize Syslog_TimeGenerated = arg_max(Syslog_TimeGenerated, *) by IndicatorId, domain
  // Select the desired columns for the final result set
  | project Syslog_TimeGenerated, Description, ActivityGroupNames, IndicatorId, ThreatType, ExpirationDateTime, ConfidenceScore, SyslogMessage, Computer, ProcessName, domain, HostIP, Url, Type, TI_DomainEntity
  // Extract the hostname from the Computer field
  | extend HostName = tostring(split(Computer, '.', 0)[0])
  // Extract the DNS domain from the Computer field
  | extend DnsDomain = tostring(strcat_array(array_slice(split(Computer, '.'), 1, -1), '.'))
  // Assign the Syslog_TimeGenerated value to the timestamp field
  | extend timestamp = Syslog_TimeGenerated
```
