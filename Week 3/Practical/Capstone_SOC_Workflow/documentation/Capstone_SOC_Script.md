# SOC Capstone — Suricata + Wazuh (GitHub)

**Author:** Tushar Yadav
**Use Case:** Raspberry Pi (Kali Linux) with Suricata sensor + Wazuh Manager/Indexer/Dashboard on Windows (WSL Ubuntu)

---

## 1. Objective

Build a mini SOC pipeline:

* Detect suspicious traffic with Suricata.
* Forward alerts into Wazuh.
* Correlate & visualize events in the Wazuh dashboard.
* Document, escalate, and respond.

---

## 2. Architecture

```
[Raspberry Pi: Kali + Suricata]  --->  [WSL Ubuntu: Wazuh Manager/Indexer/Dashboard]
```

---

## 3. Prerequisites

* Raspberry Pi running Kali Linux (ARM64)
* Windows machine with WSL Ubuntu running Wazuh stack
* Root/sudo access
* Internet connectivity

---

## 4. Install & Configure Suricata on Kali Pi

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Suricata
sudo apt install -y suricata

# Enable promiscuous mode
sudo ip link set eth0 promisc on

# Test run
sudo suricata -i eth0 -l /var/log/suricata/
```

**Troubleshooting:**

* If no rules are loaded → install ruleset:

```bash
sudo suricata-update
sudo systemctl restart suricata
```

* Logs: `/var/log/suricata/eve.json`

---

## 5. Configure Wazuh Agent to Collect Suricata Logs

Edit agent config (`/var/ossec/etc/ossec.conf`):

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Restart Wazuh agent:

```bash
sudo systemctl restart wazuh-agent
```

---

## 6. Test Detection

* Run ET test:

```bash
curl http://testmyids.com
```

* Or simulate attack with Metasploit (e.g., Samba exploit). Suricata should generate alerts → Wazuh dashboard shows event.

---

## 7. Create Custom Rule in Wazuh

Edit `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100100" level="10">
  <decoded_as>json</decoded_as>
  <field name="alert.signature">ET MALWARE</field>
  <description>Suricata detected ET MALWARE event</description>
  <mitre>
    <id>T1071</id>
  </mitre>
</rule>
```

Restart manager:

```bash
sudo systemctl restart wazuh-manager
```

---

## 8. Escalation: TheHive Case

Create new case in TheHive (via UI or API).

**100-word summary template:**

```
Suricata sensor detected malicious traffic from external IP attempting to exploit Samba service. The alert was ingested into Wazuh and correlated against MITRE ATT&CK technique T1071. The source IP and payload match a known exploit attempt. Impact: Potential lateral movement and compromise if successful. Immediate actions taken: alert triaged, rules validated, and CrowdSec configured to block offending IP. Recommended follow-up: full packet capture analysis, IOC extraction, and notify incident response team for containment.
```

**TheHive JSON template (case import):**

```json
{
  "title": "Suricata Alert - ET MALWARE",
  "description": "Suricata detected malware traffic (ET MALWARE). Event ingested in Wazuh, MITRE T1071. Action taken: IP blocked.",
  "severity": 2,
  "tlp": 2,
  "pap": 2,
  "status": "New"
}
```

---

## 9. Reporting: Google Docs (SANS-style)

**200-word template:**

```
Executive Summary:
On [date], Suricata detected suspicious traffic on the Raspberry Pi sensor. The Wazuh stack ingested and correlated the event, mapping it to MITRE ATT&CK T1071. The malicious traffic originated from [IP], targeting Samba service. This was confirmed via test exploit simulation. The event was escalated to TheHive, assigned medium severity. Immediate containment was executed with CrowdSec. Recommended actions: further IOC analysis, endpoint checks, and threat intelligence enrichment.

Technical Details:
- Source IP: [IP]
- Destination IP: [IP]
- Rule Triggered: ET MALWARE
- Log Source: Suricata eve.json → Wazuh agent → Wazuh manager
- MITRE Technique: T1071
- Response: IP blocked, evidence stored

Outcome:
The SOC successfully detected, triaged, and contained the event.
```

---

## 10. Evidence Handling Checklist

* Save raw `eve.json` logs
* Export Suricata pcap (if enabled)
* Document in TheHive with TLP/PAP
* Store in GitHub repo (`/evidence` folder)
* Maintain chain of custody in `CHAIN_OF_CUSTODY.md`

---

## 11. Scripts

`scripts/start-suricata.sh`:

```bash
#!/bin/bash
sudo ip link set eth0 promisc on
sudo suricata -c /etc/suricata/suricata.yaml -i eth0 -D
```

`scripts/collect-logs.sh`:

```bash
#!/bin/bash
tar -czvf evidence_$(date +%F).tar.gz /var/log/suricata/eve.json
```

`scripts/push-evidence.sh`:

```bash
#!/bin/bash
git add evidence_*.tar.gz
git commit -m "Add evidence logs $(date)"
git push origin main
```

---

## 12. Troubleshooting

* **No alerts:** Ensure rules installed (`suricata-update`).
* **eve.json empty:** Check Suricata interface and config.
* **Agent not sending logs:** Verify `/var/ossec/logs/ossec.log`.
* **Wazuh dashboard empty:** Check Filebeat and indexer status.

---

## 13. Repo Structure

```
/README.md
/scripts/
  start-suricata.sh
  collect-logs.sh
  push-evidence.sh
/evidence/
CHAIN_OF_CUSTODY.md
```

---

