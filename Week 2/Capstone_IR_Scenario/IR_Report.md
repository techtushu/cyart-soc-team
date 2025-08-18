# Incident Response Report

## Incident Overview
- Date: 2025-08-18
- Reported By: SOC Analyst
- Summary: Multiple alerts indicating possible ransomware infection and data exfiltration.

## Timeline of Events
| Time | Event |
|------|-------|
| 09:05 | Wazuh alert: VPN brute force attempt |
| 09:15 | Sysmon alert: Malicious PowerShell execution |
| 09:25 | EDR alert: Process injection on Finance workstation |
| 09:45 | Firewall alert: Large outbound data transfer |

## Root Cause
- Attacker gained access through VPN brute force
- Executed malicious PowerShell script
- Injected into legitimate process
- Exfiltrated data to external server

## Containment Actions
- Isolated affected workstation
- Disabled compromised VPN account
- Blocked malicious IPs at firewall

## Remediation
- Force password reset for all VPN accounts
- Apply PowerShell execution restrictions
- Patch vulnerable software
- Conduct forensic review of exfiltrated files

## Lessons Learned
- Implement MFA on VPN
- Improve monitoring for outbound traffic anomalies
- Enhance user awareness training
