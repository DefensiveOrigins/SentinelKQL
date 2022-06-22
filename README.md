# SentinelKQL
Some supporting KQL queries for a blog

This very basic query produces a timechart representation of failed logins.

```kusto
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(24h)
| render timechart
```

Let's look at the IP addresses that are attempting and failing logins and sort them.

```kusto
SecurityEvent
| where EventID == 4625 or EventID == 4776
| where TimeGenerated > ago(12h)
| summarize count() by IpAddress
| order by count_
```

The next query provides a summary of failed logins by targeted account.

```kusto
// Counts failed logons by target account. 
SecurityEvent
| where EventID == 4625
| summarize count() by TargetAccount
| order by count_
```

The next query provides a summary of failed logins by targeted account and the IP from whence it came. 

```kusto
// Counts failed logons by target account and source IP address. 
SecurityEvent
| where EventID == 4625
| summarize count() by TargetAccount, IpAddress
| order by count_
```

The following KQL query gathers a geoIP list from another Github repo to support an early query.

```kusto
// Pierce’s RDP attribution query
let ipdata = externaldata(network:string,geoname_id:string,continent_code:string,continent_name:string,country_iso_code:string,country_name:string,is_anonymous_proxy:string,is_satellite_provider:string)
[@"https://raw.githubusercontent.com/datasets/geoip2-ipv4/master/data/geoip2-ipv4.csv"];
let IPs = union Event, SecurityEvent
| where EventID in ("4624","4625")
    and AccountType != "Machine"
    and IpAddress != "-" 
| project IpAddress, TimeGenerated, Activity, EventID;
IPs
| evaluate ipv4_lookup(ipdata, IpAddress, network)
| summarize Count = count() by IpAddress, network, country_iso_code, country_name
| order by Count 
```

The following KQL query gathers a geoIP list from another Github repo to support another query.

```kusto
// Pierce’s SSH attribution query
let ipdata = externaldata(network:string,geoname_id:string,continent_code:string,continent_name:string,country_iso_code:string,country_name:string,is_anonymous_proxy:string,is_satellite_provider:string)
[@"https://raw.githubusercontent.com/datasets/geoip2-ipv4/master/data/geoip2-ipv4.csv"];
let IPs = Syslog
| where SyslogMessage startswith "Failed Password"
| order by EventTime desc 
| extend User = extract("for(?s)(.*)from",1,SyslogMessage)
| extend IPaddr = extract("(([0-9]{1,3})\\.([0-9]{1,3})\\.([0-9]{1,3})\\.(([0-9]{1,3})))",1,SyslogMessage) 
| project HostName, SyslogMessage, EventTime, IPaddr, User;
IPs
| evaluate ipv4_lookup(ipdata, IPaddr, network)
| summarize Count = count() by IPaddr, network, country_iso_code, country_name
| order by Count
```

The following query provides a view of failed logins that includes another set of interesting results.

```kusto
// Interesting perspective on failed logins
SecurityEvent
| where TimeGenerated > ago(30m)
| where EventID == 4625
| project TimeGenerated, EventID, WorkstationName, Computer, Account, LogonTypeName, IpAddress
| extend AccountEntity = Account
| extend IPEntity = IpAddress
```

Another query, another perspective!
```kusto
union Event, SecurityEvent
| where TimeGenerated > ago(30m)
| where EventID == 4625 or EventID == 4776
| project Account , IpAddress
| summarize USERs = make_set(Account) by IpAddress
| where USERs[1] != ""
```
