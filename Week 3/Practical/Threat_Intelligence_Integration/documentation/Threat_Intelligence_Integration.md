# Wazuh + AlienVault OTX — Threat Intelligence Integration (GitHub README)

**Author:** Tushar Yadav

**Purpose:** step-by-step guide to import AlienVault OTX IOCs into Wazuh (4.12), enrich alerts, and perform a T1078 (Valid Accounts) hunt. This README is written so a new person can follow and reproduce the lab.

---

## Overview

This repo documents how to:

1. Export IOCs from AlienVault OTX (manual export)
2. Convert the IOC list into a Wazuh-compatible CDB list
3. Configure Wazuh to use that CDB for alert enrichment
4. Create rules that trigger on IOC matches
5. Test end-to-end (wazuh-logtest / logger / agent → manager)
6. Perform threat hunting for MITRE T1078 (Valid Accounts)

This approach works with **Wazuh 4.12** (manager + indexer + dashboard on WSL Ubuntu) and a Wazuh Agent on a Raspberry Pi (Kali). It does not rely on the deprecated `alienvaultotx` integrator.

---

## Prerequisites

* Wazuh Manager, Indexer, Dashboard installed and running (tested on Wazuh 4.12).
* Wazuh Agent installed on the monitored host(s) (Raspberry Pi in this lab).
* SSH access and `sudo` on the Manager host.
* An AlienVault OTX account (optional if you will manually collect IOCs). *Do not store API keys in the repo.*
* `jq` recommended for parsing JSON on the manager: `sudo apt install jq`

---

## File layout (what we'll create)

* `/var/ossec/etc/lists/otx-iocs.txt` — raw IOC list (key\:value format)
* `/var/ossec/etc/lists/otx-iocs.txt.cdb` — compiled CDB list used by Wazuh
* `/var/ossec/etc/rules/local_rules.xml` — custom rules (IOC match + T1078 rule)
* small helper scripts (optional) to automate list updates (store outside repo or in a secure area)

---

## IMPORTANT: Security & Git hygiene

* **NEVER** commit API keys, private certificates, or any sensitive server config to a public repo.
* Add any files that may contain secrets to `.gitignore`.
* Use environment variables or a secrets store for automation scripts that use the OTX API.

---

## 1) Export IOCs from OTX (manual)

1. Login to [https://otx.alienvault.com/](https://otx.alienvault.com/)
2. Subscribe to a Pulse or open a Pulse and use **Export** → **CSV** (or copy IPs directly).
3. From the CSV or UI extract **IPv4 addresses**.

> Example IPs (for lab/testing only — replace with the OTX export you downloaded):

```
185.220.101.1
45.33.32.156
207.210.121.148
```

---

## 2) Prepare the IOC text list for Wazuh

On the Wazuh Manager (WSL Ubuntu):

```bash
sudo -s
cat > /var/ossec/etc/lists/otx-iocs.txt <<'EOF'
185.220.101.1:1
45.33.32.156:1
207.210.121.148:1
EOF
```

**Notes:**

* Each line must be `key:value`. The value can be `1` for all entries (the `:1` is required for the key\:value format).

---

## 3) Compile the list into a CDB

Wazuh requires a compiled CDB file for the `<list>` match. Different Wazuh packages may include different helper tools; check which you have.

### Method A — preferred (Wazuh helper script if present)

Check for helper scripts:

```bash
ls -l /var/ossec/framework/scripts | grep cdb
ls -l /var/ossec/framework/scripts | grep cdbsync
```

If a helper exists, run (example names — your install may differ):

```bash
# Example 1 (if present)
sudo /var/ossec/framework/scripts/wazuh-cdbsync /var/ossec/etc/lists/otx-iocs.txt

# Example 2 (older name)
sudo /var/ossec/framework/scripts/wazuh-cdbruleset -i /var/ossec/etc/lists/otx-iocs.txt
```

This should generate `/var/ossec/etc/lists/otx-iocs.txt.cdb` (or `otx-iocs.cdb`).

### Method B — find any `.cdb` already created by tooling

Some installations auto-generate a `.cdb` and append `.txt.cdb`. If you already have a file like `/var/ossec/etc/lists/otx-iocs.txt.cdb`, use that exact filename in the rules (see section 5).

### Troubleshooting

* If no helper scripts exist, search for `wazuh-cdbsync`, `wazuh-cdbruleset`, or check Wazuh docs for how your package builds CDB lists.
* Running the helper will create the `.cdb`; if it creates `otx-iocs.txt.cdb` use that exact name.

---

## 4) Register the list in Wazuh (`ossec.conf`)

Open `/var/ossec/etc/ossec.conf` and ensure the `<ruleset>` section contains a reference to your list. Example minimal snippet (inside `<ossec_config>`):

```xml
<ruleset>
  <!-- default dirs -->
  <decoder_dir>ruleset/decoders</decoder_dir>
  <rule_dir>ruleset/rules</rule_dir>

  <!-- your custom list -->
  <list>etc/lists/otx-iocs.txt.cdb</list>

  <!-- local rules dir -->
  <decoder_dir>etc/decoders</decoder_dir>
  <rule_dir>etc/rules</rule_dir>
</ruleset>
```

Save and test configuration later (see validation commands below).

---

## 5) Create custom rules (local\_rules.xml)

Create or edit `/var/ossec/etc/rules/local_rules.xml` and add the following two rules (IOC match + successful login mapped to T1078):

```xml
<!-- local_rules.xml -->
<group name="local,syslog,sshd,">
  <!-- Example: IOC match (IP list compiled to CDB) -->
  <rule id="100100" level="10">
    <if_sid>5760</if_sid> <!-- ties to sshd authentication failed, optional -->
    <list field="srcip">etc/lists/otx-iocs.txt.cdb</list>
    <description>Connection from IP found in OTX IOC list.</description>
    <mitre>
      <id>T1071</id>
    </mitre>
  </rule>

  <!-- Map successful ssh login to T1078 (Valid Accounts) -->
  <rule id="100110" level="8">
    <match>Accepted password for</match>
    <description>Successful SSH login — potential Valid Accounts abuse (T1078)</description>
    <mitre>
      <id>T1078</id>
    </mitre>
  </rule>

</group>
```

**Notes:**

* `100100` triggers when a log has `srcip` and that IP exists in the compiled CDB file.
* `100110` is a simple match rule for successful sshd log lines. You can refine by adding `<if_sid>` that corresponds to your sshd success rule if you prefer.

---

## 6) Validate config & restart Wazuh

Validate parsing of rules and restart services:

```bash
# Validate rules (analysisd)
sudo /var/ossec/bin/wazuh-analysisd -t

# Restart Wazuh (safe way)
sudo /var/ossec/bin/wazuh-control restart
# or
sudo systemctl restart wazuh-manager
```

Check important logs:

```bash
sudo tail -n 200 /var/ossec/logs/ossec.log
sudo tail -n 200 /var/ossec/logs/alerts/alerts.json
sudo tail -n 200 /var/ossec/logs/integrations.log
```

---

## 7) Test end-to-end (manager & agent)

### A) Test on Manager (quick) using `wazuh-logtest` (interactive)

Run:

```bash
sudo /var/ossec/bin/wazuh-logtest

# Paste a syslog-style sshd line (exact format is important):
Sep 05 00:10:15 myhost sshd[9999]: Failed password for root from 207.210.121.148 port 4444 ssh2

# or to test a successful login (T1078):
Sep 05 00:20:15 myhost sshd[9999]: Accepted password for user1 from 203.0.113.5 port 5555 ssh2
```

Expected output: you should see decoded fields (srcip, dstuser) and the filter phase should show the rules fired (e.g., `100100`, `5760`, `100110`).

### B) Test via `logger` (on manager or on agent)

To simulate logs being sent by an agent, run on the machine you want to emulate (agent or manager):

```bash
logger "sshd[1234]: Failed password for root from 207.210.121.148 port 5555 ssh2"
# or
logger "sshd[1234]: Accepted password for user1 from 203.0.113.5 port 5555 ssh2"
```

Then on the manager:

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json
```

You should observe alerts for your custom rule id(s).

---

## 8) Dashboard / Hunting queries (Wazuh Discover)

Use the Wazuh Dashboard (Discover) to run these KQL queries:

* IOC matches (enrichment):

```
rule.id:100100
```

* Failed SSH logins (brute force):

```
rule.id:5760
```

* Hunt for Valid Accounts (exclude system):

```
rule.id:100110 OR (rule.id:5760 AND user.name != "system")
```

* Find IOC matches that also involved successful logins (correlation):

```
rule.id:100100 AND (message: "Accepted password" OR rule.id:100110)
```

---

## 9) Write-up / Evidence table (example)

Include this in your repo or report (replace IDs/IPs as necessary):

| Alert ID | IP              | Reputation      | Notes                       |
| -------- | --------------- | --------------- | --------------------------- |
| 100100   | 207.210.121.148 | Malicious (OTX) | Known C2 server in IOC feed |

**50-word hunting summary (example):**

> Wazuh logs revealed multiple SSH authentication attempts. Several failed attempts (T1110) originated from IPs listed in the OTX feed. One successful login matched T1078 (Valid Accounts), indicating a potential credential misuse paired with known malicious infrastructure. Recommended: account password resets, MFA enforcement, and forensic review.

---

## 10) Automating IOC updates (optional)

Automate pulling/pushing OTX IOCs into Wazuh but **do not** commit your API key. Two approaches:

1. **Manual** – Download CSV from OTX UI periodically and run the compile step.
2. **Automated script** – store the API key in an environment variable or vault and run a script (cron) that:

   * Calls the OTX API (or uses `curl`) to fetch pulses or indicators
   * Extracts IPv4 addresses
   * Writes `ip:1` lines to `/var/ossec/etc/lists/otx-iocs.txt`
   * Runs the CDB compile utility on the text file
   * Restarts/Reloads Wazuh analysisd (if needed)

*Important:* keep any automation script outside the public repo or add it to `.gitignore` if it contains secrets.

---

## 11) Troubleshooting (common errors & fixes)

* **`Invalid element 'enabled'`**: remove `<enabled>` from integration blocks in `ossec.conf` (not supported by integratord).
* **`File not found inside 'integrations'`**: old `alienvaultotx` integration script not available in Wazuh 4.12 — we use the CDB approach instead.
* **`List match lookup="match" is not valid`**: new Wazuh versions require CDB lists; use `address_match` or compile to `.cdb` and reference the `.cdb` filename.
* **`No decoder matched`** in `wazuh-logtest`: ensure you feed a full syslog-style line with timestamp/hostname so decoders (syslog → sshd) chain correctly.
* **Rule not firing**: check `ossec.log` and `analysisd -t` for errors; confirm the `.cdb` filename used in the rule exactly matches the file in `/var/ossec/etc/lists/` (e.g., `otx-iocs.txt.cdb`).

---

## 12) Deliverables (what to upload to GitHub)

* `README.md` (this file)
* `examples/otx-iocs.txt` (sample with dummy entries *not real keys*) — **do not** include real OTX data or secrets.
* `examples/local_rules.xml` (your rule snippets)
* `run_tests.md` (short checklist for tests to run after deploy)

---

## Appendix — Quick commands reference

```bash
# Validate rules
sudo /var/ossec/bin/wazuh-analysisd -t

# Restart Wazuh
sudo /var/ossec/bin/wazuh-control restart
# or
sudo systemctl restart wazuh-manager

# Watch alerts
sudo tail -f /var/ossec/logs/alerts/alerts.json

# Test parsing (interactive)
sudo /var/ossec/bin/wazuh-logtest
# then paste a full syslog-style line

# Look for IOC matches
grep '"rule":{"id":"100100"' /var/ossec/logs/alerts/alerts.json
```

---

If you want, I can:

* Produce the `examples/` files as separate documents in this repo.
* Prepare a small automation script (cron-friendly) to refresh IOCs from OTX (I will not include any API key in the script; you supply it at runtime).

