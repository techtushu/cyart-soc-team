# Threat Hunt Queries

## ELK Example
```
event.dataset: "powershell" AND powershell.command_line: "*Invoke-Expression*"
```

## Splunk Example
```
index=winlogs sourcetype="WinEventLog:Security" EventCode=4624 
| stats count by Account_Name, src_ip
```

