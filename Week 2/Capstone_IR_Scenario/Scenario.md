# Capstone Incident Response Scenario

## Scenario
On 2025-08-18, the SOC receives multiple alerts:
- **Wazuh:** Brute force attempt on VPN gateway from IP 203.0.113.45
- **EDR:** Suspicious process injection detected on Finance workstation
- **Sysmon:** PowerShell downloading file from hxxp://malicious[.]example/malware.ps1
- **Firewall:** Large outbound data transfer to unknown external IP

## Your Tasks
1. **Incident Triage**
   - Record all alerts in the Incident_Triage_Log.csv
   - Assign severity (Low, Medium, High, Critical)
   - Document initial triage notes

2. **Chain of Custody**
   - Create at least one custody form for collected evidence (e.g., packet capture, memory dump)

3. **Analysis**
   - Identify the attack chain (Initial Access → Execution → Exfiltration)
   - Suggest containment & remediation steps

4. **Reporting**
   - Write a short IR summary in `IR_Report.md`

## Deliverables
- Updated Incident_Triage_Log.csv
- Completed Chain_of_Custody_Form.md
- IR_Report.md
