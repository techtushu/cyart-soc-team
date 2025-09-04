# Advanced Log Analysis – SSH Brute Force & Sudo Caching (Wazuh + Elastic Security)

**Author:** Tushar Yadav
**Scope:** What I set up, exact commands I ran, why I ran them, and how I correlated/enriched/detected events.
**Stack used:** Raspberry Pi (Kali Linux with Wazuh Agent) + Windows 10 (WSL Ubuntu with Wazuh Manager/Indexer/Dashboard), Filebeat, Elastic Security (optional add‑on tasks), Google Sheets for documentation.

---

## 0) High‑Level Outcomes

* **Captured** SSH brute force attempts and **observed** sudo timestamp caching activity from Raspberry Pi (Kali Linux).
* **Verified ingestion** path end‑to‑end (Raspberry Pi Agent → Wazuh Manager on WSL Ubuntu → Filebeat/Indexer → Dashboard).
* **Built** correlation table (failed logons ↔ outbound DNS/HTTP).
* **Created** an Elastic Security rule for high data egress and a **GeoIP** enrichment pipeline.
* **50‑word summary:** *Repeated SSH failures followed by successful login and burst of sudo commands indicate potential brute force and privilege use. Outbound DNS lookups and HTTP requests from the same host/time window strengthen suspicion. GeoIP enrichment confirmed external destinations. A 1‑minute egress spike (>1MB) triggered an alert for possible exfiltration.*

---

## 1) Architecture & Data Flow

```
[Raspberry Pi (Kali) Wazuh Agent] → [WSL Ubuntu Wazuh Manager] → Filebeat → Wazuh Indexer (OpenSearch) → Wazuh Dashboard (on Windows)
```

**Why:** Understand where to start/stop services and where to look when data is missing.

---

## 2) Prerequisites

* **Raspberry Pi** running **Kali Linux** with Wazuh Agent installed.
* **Windows 10/11** with **WSL (Ubuntu)** hosting:

  * `wazuh-manager`
  * `wazuh-indexer`
  * `wazuh-dashboard`
  * `filebeat`
* (Optional) **Elastic Stack** / **Elastic Security** for the enhanced tasks.
* Attack tools (local test on Raspberry Pi): `hydra`, `nmap`, and SSH client.

> If re‑installing from scratch on WSL, first remove the distro (Windows Settings → Apps → Ubuntu → Uninstall), then reinstall from Microsoft Store. After installing Wazuh again, follow the service commands below.

---

## 3) Start/Verify Services (WSL Ubuntu)

Open **PowerShell** on Windows:

```powershell
wsl -d Ubuntu
```

**Why:** Enters the Linux environment where Wazuh services run.

Inside **WSL (Ubuntu)**, start and enable services:

```bash
sudo systemctl daemon-reload                       # Why: refresh unit files

sudo systemctl enable --now wazuh-manager          # Why: core event processing
sudo systemctl enable --now wazuh-indexer          # Why: stores/searches events
sudo systemctl enable --now wazuh-dashboard        # Why: web UI
sudo systemctl enable --now filebeat               # Why: ships manager alerts to indexer
```

Check status if something looks off:

```bash
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard filebeat
```

Logs to tail for troubleshooting:

```bash
sudo journalctl -u wazuh-manager -f
sudo journalctl -u filebeat -f
sudo journalctl -u wazuh-indexer -f
sudo journalctl -u wazuh-dashboard -f
```

**Why:** Confirms ingestion path and surfaces errors (e.g., Filebeat lock issues).

If Filebeat ever complains about a stale lock:

```bash
sudo rm -f /var/lib/filebeat/filebeat.lock  # Why: clear abandoned lock
sudo systemctl restart filebeat
```

---

## 4) Access the Dashboard

* Open browser on Windows: `https://`172.21.138.169 (or the port your Wazuh Dashboard is bound to).
* Log in with the Wazuh dashboard user you configured.

**Why:** Validate data is arriving; use Discover/Security events to filter.

---

## 5) Generate SSH Brute Force Signals (on Raspberry Pi)

**Option A – hydra (recommended):**

```bash
sudo apt update && sudo apt install -y hydra
# Replace user/host accordingly. This fires incorrect passwords quickly to trigger auth failures.
hydra -l pi -P /usr/share/wordlists/rockyou.txt -t 4 ssh://<wsl-manager-ip> -s 22
```

**Why:** Creates many `sshd` authentication failures that Wazuh should alert on.

**Option B – quick loop (slower, fewer events):**

```bash
for i in {1..15}; do ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no pi@<wsl-manager-ip> || true; done
```

**Why:** Also generates repeated failures without extra tools.

**Evidence to capture:**

* `/var/log/auth.log` on Raspberry Pi with `Failed password` or `Invalid user`.
* Wazuh alerts in Dashboard (search for `sshd` or `authentication failure`).

---

## 6) Demonstrate Sudo Timestamp Caching (on Raspberry Pi)

1. **Invalidate** the sudo cache and authenticate once:

```bash
sudo -k                 # Why: clear cached credentials
sudo ls                 # Why: force a password prompt (enter it once)
```

2. **Run multiple sudo commands without password:**

```bash
for i in {1..10}; do sudo -n date; done
```

**Why:** The `-n` (non‑interactive) succeeds while the timestamp is valid; shows a burst of privileged actions after one auth.

**Evidence to capture:**

* `/var/log/auth.log` entries for `sudo` with no password prompt after initial auth.
* Corresponding Wazuh alerts on sudo usage.

---

## 7) What I Looked For in Wazuh

* **SSHD failures (from Raspberry Pi):** search Discover for `sshd` and filter by agent name.
* **Sudo bursts (from Raspberry Pi):** filter by `program: sudo` (or `rule.description` containing `sudo`).
* **Timeline:** use time picker around your test window to see clustering.

**Why:** Confirms detection of brute force attempts and sudo caching behavior.

---

## 8) Enhanced Tasks in Elastic Security (Optional but included for the task)

> If you have Elastic Security available, perform these steps to align with the task brief.

### 8.1 Ingest Sample (Boss of the SOC) & Correlate 4625 → Outbound

**Goal:** Correlate Windows **Event ID 4625** (failed logon) with outbound traffic in the **same time window**.

**KQL examples (Elastic Discover/Security):**

```kql
# Failed logon events
index:* AND event.code:4625

# Outbound network activity (Packetbeat/Firewall/ECS logs)
index:* AND network.direction:outbound AND destination.ip:* AND (event.category:network or network.transport:*)
```

**Correlation approach:**

1. Filter `event.code:4625`, note **host.name**, **user.name**, **source.ip**, and **@timestamp**.
2. In a ±5–10 min window, filter outbound events where `host.name` or `source.ip` matches the same host.
3. Join in a spreadsheet (see template below) to document pairs.

**Google Sheets/CSV template (copy‑paste):**

```csv
Timestamp,Event ID,Source IP,Destination IP,Notes
2025-08-18 12:00:00,4625,192.168.1.100,8.8.8.8,Suspicious DNS request
```

### 8.2 Anomaly Detection Rule – High Volume Egress (1 minute)

**Elastic Security Rule (KQL) – bytes\_out > 1MB in 1m:**

```text
Index patterns: packetbeat-*, firewall-*, logs-*  
Custom query (KQL): network.bytes_out >= 1048576 and event.duration <= 60000000

# Scheduling
Runs every: 1 minute
Look-back: 1 minute
```

**Why:** Flags bursts of outbound data (exfiltration hint). Adjust field names to your ECS source (e.g., `destination.bytes`, `network.bytes` if that’s what you have).

### 8.3 GeoIP Enrichment Pipeline (Elastic Ingest)

Create a pipeline to add GeoIP on `destination.ip`:

```json
PUT _ingest/pipeline/geoip-dest
{
  "processors": [
    { "geoip": { "field": "destination.ip", "target_field": "destination.geo" } }
  ]
}
```

Attach pipeline to an index (example for Packetbeat):

```json
PUT packetbeat-*/_settings
{
  "index": { "default_pipeline": "geoip-dest" }
}
```

**Why:** Adds country/city/ASN to destination IPs for better triage.

---

## 9) Correlation Table (Filled Example)

Use this in your README or a Google Sheet. Expand with your own rows.

| Timestamp           | Event ID | Source IP    | Destination IP | Notes                 |
| ------------------- | -------- | ------------ | -------------- | --------------------- |
| 2025-09-04 21:56:00 | 4625     | 192.168.1.34 | 192.168.1.41   | sudo and sudo caching |
| 2025-09-04 21:45:16 | 4625     | 192.168.1.34 | 192.168.1.41   | Brute Force           |

---

## 10) Evidence Collection Checklist

* **Screenshots:**

  * Wazuh Dashboard showing SSH failures timeline.
  * Wazuh Dashboard showing sudo burst events.
  * Elastic Discover results for `event.code:4625` and outbound traffic.
  * Elastic Security alert for 1‑minute egress spike.
* **Raw logs:** `/var/log/auth.log` snippets from Raspberry Pi (sanitized).
* **Rule & pipeline JSON/YAML:** the KQL rule config and the GeoIP pipeline.

---

## 11) Troubleshooting Notes I Hit

* **“Not catching nmap alerts”:** baseline Wazuh rules may not alert on simple scans without IDS telemetry. Pair with Suricata (Security Onion) or tune Wazuh decoders/rules.
* **Filebeat lock error:** remove lock and restart (see §3).
* **Where are Wazuh alerts?** `wazuh-alerts-*` in Dashboard/Discover. Adjust time picker.
* **No data in dashboard?** Verify Raspberry Pi agent is connected and all WSL services running.

---

## 12) Optional: Security Onion Parity (Quick Notes)

If using Security Onion instead, capture SSH brute force on Zeek logs and sudo activity from Syslog/OSQuery; correlate in Kibana/OpenSearch using host and time window. Use Suricata for scan detection and Zeek `conn.log` for egress volume. GeoIP comes built‑in via Logstash/ingest pipelines.

---

## 13) Repro Steps (Copy/Paste Quickstart)

```bash
# On Windows PowerShell
wsl -d Ubuntu

# On WSL Ubuntu – start services
sudo systemctl enable --now wazuh-manager wazuh-indexer wazuh-dashboard filebeat

# On Raspberry Pi (Kali) – generate SSH failures
sudo apt update && sudo apt install -y hydra
hydra -l pi -P /usr/share/wordlists/rockyou.txt -t 4 ssh://<wsl-manager-ip> -s 22

# On Raspberry Pi (Kali) – demonstrate sudo caching
sudo -k; sudo ls   # enter password once
for i in {1..10}; do sudo -n date; done

# On Windows browser – open dashboard
https://localhost:5601
```

---

## 14) How to Include in This Repo

* `README.md` → this file.
* documents/ → your Elastic rule query & schedule notes.
* `/documents` → the ingest pipeline.
* `screenshoots/` → screenshots (PNG/JPG) and sanitized log samples.
* `documents` → your event join table.

---

## 15) Next Improvements

* Add Suricata/Zeek (Security Onion) to enrich network visibility.
* Tune Wazuh rules for `sshd` and `sudo` with custom thresholds.
* Automate evidence export with a small Python script (Kibana/OS APIs).

---

**End of document.**
