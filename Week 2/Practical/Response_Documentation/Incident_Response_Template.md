# Incident Response Report – Phishing Simulation

## 1. Incident Summary
- **Incident ID:** IR-2025-08-21-PHISH-01
- **Date/Time Detected:** 2025-08-21 21:47 UTC
- **Detection Source:** Wazuh SIEM (custom phishing rule triggered)
- **Incident Type:** Phishing simulation (user clicked malicious link)
- **Severity Level:** High (rule level 10)

---

## 2. Timeline of Events
| Time (UTC) | Event |
|------------|-------|
| 21:44 | Custom Wazuh rule deployed to detect `PHISHING-CLICK` logs |
| 21:47 | User simulated click recorded in `/var/log/wazuh-custom.log` |
| 21:47 | Wazuh agent forwarded event to Manager |
| 21:47 | Wazuh rule `100100` triggered → *"Phishing simulation detected"* |
| 21:48 | SOC team notified via Wazuh dashboard |
| 21:55 | Analyst validated event and confirmed simulation |
| 22:05 | Response documentation created |

---

## 3. Detection
- **Rule ID:** 100100  
- **Rule Description:** *Phishing simulation detected*  
- **Log Source:** `/var/log/wazuh-custom.log`  
- **Log Example:**



---

## 4. Response Actions
- Alert triaged in SIEM dashboard.  
- Analyst confirmed that event matched controlled phishing simulation.  
- No production systems compromised.  
- Documentation & lessons learned logged.

---

## 5. Containment & Recovery
- No containment required (simulation only).  
- Verification steps executed:
- Ensured logs forwarded correctly.  
- Verified alert correlation worked as expected.  

---

## 6. Lessons Learned
- Wazuh custom rules successfully detect simulated phishing clicks.  
- Next steps: integrate with TheHive for case management.  
- Expand simulation to include real email headers for stronger validation.

---

## 7. Attachments
- `diagrams/sequence.png` (Phishing event lifecycle)  
- `diagrams/flowchart.png` (Response workflow)  
