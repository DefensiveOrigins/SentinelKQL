# SentinelKQL
Some supporting KQL queries for a blog

The following KQL query gathers a geoIP list from another Github repo to support an early query.

```kusto
// blog query 1 - IP, Netblock, Country of origin, Country name, Count
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
