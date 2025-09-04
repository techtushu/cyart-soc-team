# 🔎 Advanced Log Analysis

## 🎯 Key Objectives
- 🔗 Correlate logs across multiple sources  
- 📊 Detect anomalies in log activity  
- 🌍 Apply enrichment for deeper context  

---

## 🔗 Log Correlation
| Source             | Example Indicator 🚨                     | Possible Link 🔗         |
|--------------------|-------------------------------------------|--------------------------|
| Windows Security   | Failed Logins (Event ID 4625)             | Account brute-force      |
| Firewall Logs      | Repeated outbound to rare IPs             | Possible C2 traffic      |
| Endpoint Logs      | New process execution (PowerShell, etc.)  | Malware activity         |

---

## 📊 Anomaly Detection
| Behavior ⏰            | Example 📌                      | Detection Method ⚙️        |
|------------------------|----------------------------------|-----------------------------|
| Unusual Login Time     | User logs in at 3:00 AM          | Statistical baseline        |
| High Data Transfer     | 10GB upload to unknown IP        | Threshold-based rules       |
| Repeated Failures      | 50 failed logins in 5 mins       | SIEM correlation / rules    |

---

## 🌍 Log Enrichment
| Enrichment Type 🌐 | Tool / Feed 🔧           | Benefit ✅                  |
|--------------------|--------------------------|-----------------------------|
| GeoIP              | MaxMind, SIEM plugin     | Detect unusual locations    |
| WHOIS/Domain Info  | VirusTotal, AbuseIPDB    | Identify malicious domains  |
| Threat Intel Feeds | OTX, MISP, commercial    | Context for prioritization  |

