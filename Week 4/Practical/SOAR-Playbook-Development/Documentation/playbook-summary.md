# Wazuh SOAR-style Playbook (Phishing) — GitHub README

**Target audience:** new SOC engineers or students. Setup tested with:

* **Agent:** Raspberry Pi 4 running Linux (Wazuh Agent installed). Agent hostname: `cyberpi` (IP `192.168.1.41`).
* **Manager / Indexer / Dashboard:** Windows host running WSL Ubuntu (Wazuh Manager, Indexer, Dashboard). Manager IP: `192.168.1.34`.

---

# 1. Overview

This repository documents a SOAR-style playbook implemented with **Wazuh** (detection) and optional Active Responses (blocking). The focus: detect a phishing event (JSON `event_type: phishing`) → generate an alert (rule `100200`) → (optional) automatically block the attacker IP and create a ticket.

This README is intended to be copy-pasted into a GitHub repo as `README.md` so a new person can reproduce the lab.

---

# 2. Architecture (simple)

```
[ Raspberry Pi 4 (Wazuh Agent) ] ---> (TCP 1514) ---> [ Wazuh Manager (WSL Ubuntu) + Indexer + Dashboard ]
                                                                |
                                                                +--> Elasticsearch indices (wazuh-alerts-4.x)
                                                                |
                                                                +--> (optional) Active Response scripts on manager/agent
```

---

# 3. Files in this repo

* `docs/local_rules.xml` — phishing rule (ID `100200`) and brief comments
* `docs/agent-ossec.conf-snippet.md` — agent config snippets to add
* `docs/manager-ossec.conf-snippet.md` — manager config snippets to add
* `scripts/block-phishing.sh` — optional active-response script (iptables)
* `docs/test-and-verify.md` — step-by-step test instructions and expected output
* `README.md` — this file

---

# 4. Prerequisites

* Raspberry Pi 4 with Linux and **Wazuh agent** installed and registered with manager.
* Windows machine with **WSL Ubuntu** hosting Wazuh Manager, Indexer (Elasticsearch/OpenSearch) and Wazuh Dashboard.
* Basic `sudo` access on both systems.
* Internet access to install packages (if not already installed).

Optional:

* `CrowdSec` for centralized blocking or `TheHive` for ticketing.

---

# 5. Detection: local\_rules.xml (copy to manager)

Create or edit `/var/ossec/etc/rules/local_rules.xml` on the **manager** and add the phishing rule. Ensure rule IDs are unique across your rules.

```xml
<group name="phishing,">
  <rule id="100200" level="10">
    <decoded_as>json</decoded_as>
    <field name="event_type">phishing</field>
    <description>Phishing alert detected</description>
    <options>no_full_log</options>
  </rule>
</group>
```

**Notes:**

* Use a unique `id` (we use `100200`). Duplicate IDs will cause unexpected behavior.
* After editing, restart manager: `sudo systemctl restart wazuh-manager`.

---

# 6. Agent (Raspberry Pi) setup: ossec.conf snippets

Edit `/var/ossec/etc/ossec.conf` on the Pi and ensure the `<client>` section points to the manager IP and add a JSON test localfile.

**Client/server config (required):**

```xml
<client>
  <server>
    <address>192.168.1.34</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

**Add a JSON localfile for clean testing (recommended):**

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/phishing-test.log</location>
</localfile>
```

Save, then restart the agent:

```bash
sudo systemctl restart wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
```

You should see a line: `Monitoring file: '/var/log/phishing-test.log'`.

---

# 7. Manager steps (WSL Ubuntu)

1. Place `local_rules.xml` in `/var/ossec/etc/rules/local_rules.xml`.
2. Restart manager:

```bash
sudo systemctl restart wazuh-manager
```

3. Verify the rule matches using `wazuh-logtest`:

```bash
sudo /var/ossec/bin/wazuh-logtest
# then paste one JSON line:
{"event_type":"phishing","srcip":"192.168.1.102"}
```

Expected output: rule `100200` should match and show "Alert to be generated."

---

# 8. Testing detection (end-to-end)

On the **Raspberry Pi agent** run:

```bash
sudo echo '{"event_type":"phishing","srcip":"192.168.1.102"}' | sudo tee -a /var/log/phishing-test.log
```

On the **manager** watch alerts:

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json
```

You should see an alert entry similar to:

```
{"rule":{"id":"100200","description":"Phishing alert detected"},"agent":{"name":"cyberpi","ip":"192.168.1.41"},"data":{"event_type":"phishing","srcip":"192.168.1.102"},"location":"/var/log/phishing-test.log"}
```

Also confirm the alert is visible on the Wazuh Dashboard and indexed in `wazuh-alerts-4.x-*`.

---

# 9. Optional - Active Response (blocking)

> Note: In this lab we validated detection. You may choose to **not** enable automatic blocking. The commands below show how to enable it later.

**Script (place at `/var/ossec/active-response/bin/block-phishing.sh`)**

```bash
#!/bin/bash
# Simple block script for demonstration. Use with caution.
ACTION=$1
USER=$2
IP=$3
LOGFILE="/var/ossec/logs/active-responses.log"

if [ "$ACTION" = "start" ]; then
  echo "$(date) Blocking IP: $IP" >> $LOGFILE
  # Example iptables block (requires root)
  /sbin/iptables -A INPUT -s $IP -j DROP || echo "iptables add failed" >> $LOGFILE
fi

if [ "$ACTION" = "stop" ]; then
  echo "$(date) Unblocking IP: $IP" >> $LOGFILE
  /sbin/iptables -D INPUT -s $IP -j DROP || echo "iptables delete failed" >> $LOGFILE
fi
exit 0
```

Make it executable and set ownership:

```bash
sudo chown root:wazuh /var/ossec/active-response/bin/block-phishing.sh
sudo chmod 750 /var/ossec/active-response/bin/block-phishing.sh
```

**Manager `ossec.conf` snippets (order matters — define `<command>` before `<active-response>`):**

```xml
<command>
  <name>block-phishing</name>
  <executable>block-phishing.sh</executable>
  <expect>srcip</expect>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <command>block-phishing</command>
  <location>any</location>
  <rules_id>100200</rules_id>
  <timeout>600</timeout>
</active-response>
```

Restart manager after changes and test by writing the phishing event again. Check `active-responses.log` for the block message.

**Important:** If your Wazuh manager refuses to restart, check `sudo /var/ossec/bin/wazuh-control config-check` and inspect `/var/ossec/logs/ossec.log` for the precise syntax error.

---

# 10. Evidence / Documentation template for GitHub

Use this example `TEST_RESULTS.md` content to show proof for a reviewer:

```
## Test run (phishing detection)
- Agent: cyberpi (192.168.1.41)
- Manager: 192.168.1.34
- Test event: {"event_type":"phishing","srcip":"192.168.1.102"}

Alerts.json excerpt:
<paste the alert json from /var/ossec/logs/alerts/alerts.json>

Active-responses log: (if enabled)
<paste /var/ossec/logs/active-responses.log or explain why blocking was skipped>
```

---

# 11. Troubleshooting checklist

* Agent not listed on manager: `sudo /var/ossec/bin/agent_control -ls`.
* Rule doesn't match in `wazuh-logtest`? Double-check `local_rules.xml` and restart manager.
* No localfile monitoring on agent: check `/var/ossec/logs/ossec.log` for "Monitoring file" lines.
* Manager restart error: run `sudo /var/ossec/bin/wazuh-control config-check` and fix syntax.
* Duplicate rule IDs: ensure unique IDs in `local_rules.xml`.

---

# 12. Security & hardening notes

* Do **not** store API keys or secrets in repo. Use environment files (`/var/ossec/active-response/bin/.ar_env`) with 640 perms.
* Test blocking scripts in a lab/VLAN — iptables rules can lock you out.
* Prefer using `cscli` (CrowdSec) or a firewall manager for production blocking.

---

# 13. 50-word summary (Google Docs ready)

This Wazuh playbook detects phishing via a JSON `event_type` rule (ID 100200). Alerts are generated and indexed in the Wazuh Dashboard. Automated blocking is optional; detection was validated using a Raspberry Pi agent and a WSL-hosted Wazuh manager. Documentation includes optional active-response scripts and testing steps.

---

# 14. How to publish to GitHub

1. `git init`
2. Add files (`local_rules.xml`, snippets, scripts, README.md)
3. `git add . && git commit -m "Wazuh SOAR playbook: phishing detection + optional response"`
4. Create remote repo on GitHub and push:

   ```bash
   git remote add origin git@github.com:youruser/wazuh-soar-playbook.git
   git push -u origin main
   ```

---

# 15. Next steps / Enhancements

* Add AbuseIPDB/VT enrichment to active-response script.
* Integrate with TheHive for automatic case creation using `thehive4py`.
* Replace simple iptables blocking with CrowdSec decisions or firewall manager.
* Create a small demo video showing alert -> dashboard -> optional block.

---

If you want, I can:

* generate separate files for each item (rules, script, test doc) in this repo canvas,
* or produce a single compressed ZIP you can download.

Tell me which format you prefer.
