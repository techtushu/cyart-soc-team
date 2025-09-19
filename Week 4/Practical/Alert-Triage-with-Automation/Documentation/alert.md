# Wazuh + VirusTotal Automation — Step-by-step Guide

**Purpose:** Quick, reproducible guide for integrating Wazuh alerts with VirusTotal for automated file-hash validation. Designed for someone new to the stack (Wazuh manager on WSL/Ubuntu, Wazuh agents on devices like Raspberry Pi/Kali).

---

# Overview

This guide shows how to: 1) capture a Wazuh JSON alert, 2) extract a file hash (if present), 3) query VirusTotal's API v3 for analysis results, and 4) store a concise, machine-readable result for further triage or reporting. The flow avoids TheHive/Cortex to keep the setup minimal and reliable.

# Architecture (simplified)

Wazuh Agent → Wazuh Manager (alerts) → `/var/ossec/integrations/virustotal` script → VirusTotal API → `/var/ossec/logs/virustotal_<hash>.json`

# Prerequisites

* Wazuh Manager installed (this doc assumes manager on WSL/Ubuntu).
* Wazuh agent installed on endpoints (Raspberry Pi, Kali, etc.).
* Python 3 and `pip3` on Wazuh Manager.
* A valid VirusTotal API v3 key. (Get it from your VirusTotal profile.)
* `requests` Python package.

# Files & locations used in this guide

* Integration script (place in): `/var/ossec/integrations/virustotal` (executable)
* API key file (optional, recommended): `/var/ossec/integrations/.vt_api_key` (600 perms)
* Output log/results: `/var/ossec/logs/virustotal_<hash>.json`

# 1) Prepare VirusTotal API key

1. Login to [https://www.virustotal.com](https://www.virustotal.com) using your account.
2. Go to your **Profile → API key** and copy your v3 API key.
3. On the Wazuh Manager, save it to a file and protect it:

```bash
sudo tee /var/ossec/integrations/.vt_api_key >/dev/null <<'KEY'
YOUR_VT_API_KEY_GOES_HERE
KEY
sudo chmod 600 /var/ossec/integrations/.vt_api_key
sudo chown root:root /var/ossec/integrations/.vt_api_key
```

> **Why a file?** Avoids hardcoding keys in scripts and lets you rotate keys without editing code.

# 2) Install Python dependency

```bash
sudo apt update && sudo apt install -y python3-pip
sudo pip3 install requests
```

# 3) Create the `virustotal` integration script

Create `/var/ossec/integrations/virustotal` with the following content (make executable):

```python
#!/usr/bin/env python3
"""
Minimal Wazuh integration script that reads a Wazuh JSON alert file (passed as arg),
extracts a file hash (md5/sha1/sha256), queries VirusTotal v3, and writes a result file.
"""
import sys
import json
import os
import hashlib
import requests

API_KEY_FILE = '/var/ossec/integrations/.vt_api_key'
VT_URL_BASE = 'https://www.virustotal.com/api/v3/files/'


def read_api_key():
    try:
        with open(API_KEY_FILE) as kf:
            return kf.read().strip()
    except Exception:
        return None


def extract_hashes(obj):
    hashes = {}
    if isinstance(obj, dict):
        for k, v in obj.items():
            lk = k.lower()
            if lk in ('md5', 'sha1', 'sha256') and isinstance(v, str):
                hashes[lk] = v
            else:
                hashes.update(extract_hashes(v))
    elif isinstance(obj, list):
        for item in obj:
            hashes.update(extract_hashes(item))
    return hashes


def compute_sha256(path):
    h = hashlib.sha256()
    with open(path, 'rb') as f:
        for chunk in iter(lambda: f.read(8192), b''):
            h.update(chunk)
    return h.hexdigest()


def extract_paths(obj):
    paths = []
    if isinstance(obj, dict):
        for k, v in obj.items():
            if k.lower() in ('path', 'pathname', 'file_path', 'location') and isinstance(v, str):
                paths.append(v)
            else:
                paths.extend(extract_paths(v))
    elif isinstance(obj, list):
        for item in obj:
            paths.extend(extract_paths(item))
    return paths


def query_virustotal(hashval, api_key):
    headers = {'x-apikey': api_key}
    r = requests.get(VT_URL_BASE + hashval, headers=headers, timeout=30)
    return r


if __name__ == '__main__':
    if len(sys.argv) < 2:
        print('Usage: virustotal <wazuh_alert_json_file>')
        sys.exit(1)

    alert_file = sys.argv[1]

    try:
        with open(alert_file) as fh:
            alert = json.load(fh)
    except Exception as e:
        print('[-] Failed to read alert file:', e)
        sys.exit(1)

    # 1) find hashes in alert
    hashes = extract_hashes(alert)

    # 2) if no hash found, look for file paths and compute sha256 if file exists locally
    if not hashes:
        paths = extract_paths(alert)
        for p in paths:
            if os.path.isfile(p):
                try:
                    hashes['sha256'] = compute_sha256(p)
                    break
                except Exception:
                    continue

    if not hashes:
        print('[*] No file hashes or accessible file paths found in alert — skipping VirusTotal query')
        sys.exit(0)

    # prefer sha256, then sha1, then md5
    hashval = hashes.get('sha256') or hashes.get('sha1') or hashes.get('md5')
    api_key = read_api_key()
    if not api_key:
        print('[-] VirusTotal API key not found at', API_KEY_FILE)
        sys.exit(1)

    try:
        r = query_virustotal(hashval, api_key)
    except Exception as e:
        print('[-] VirusTotal query failed:', e)
        sys.exit(1)

    if r.status_code == 200:
        data = r.json()
        stats = data.get('data', {}).get('attributes', {}).get('last_analysis_stats', {})
        out = {
            'hash': hashval,
            'stats': stats,
            'source': 'wazuh',
            'alert_id': alert.get('id') or alert.get('rule', {}).get('id') or 'unknown'
        }
        outpath = f"/var/ossec/logs/virustotal_{hashval}.json"
        try:
            with open(outpath, 'w') as of:
                json.dump({'vt': out, 'raw': data}, of, indent=2)
            print(json.dumps(out))
        except Exception as e:
            print('[-] Failed to write output file:', e)
            sys.exit(1)
    else:
        print('[-] VirusTotal returned', r.status_code, r.text)
        sys.exit(1)
```

Save the file and make it executable:

```bash
sudo chmod +x /var/ossec/integrations/virustotal
```

# 4) Configure Wazuh to call the integration

Edit Wazuh manager config: `/var/ossec/etc/ossec.conf` and add (inside root `<ossec_config>`):

```xml
<integration>
  <name>virustotal</name>
  <alert_format>json</alert_format>
</integration>
```

Notes:

* The Wazuh integration system will invoke `/var/ossec/integrations/virustotal` and pass the alert JSON path as a single argument. This matches the script above.

# 5) Restart Wazuh Manager

```bash
sudo /var/ossec/bin/wazuh-control restart
# or
sudo systemctl restart wazuh-manager
```

Monitor logs for integration activity:

```bash
sudo tail -f /var/ossec/logs/ossec.log /var/ossec/logs/active-responses.log
```

# 6) Manual testing (without waiting for a real alert)

Create a sample alert file `/tmp/wazuh_test_alert.json` with this minimal structure (update `sha256` to desired hash):

```json
{
  "id": "test-001",
  "rule": {"description": "Suspicious File Download", "level": 10},
  "agent": {"name": "raspberrypi"},
  "data": {"sha256": "44d88612fea8a8f36de82e1278abb02f"}
}
```

Run the integration script as Wazuh would:

```bash
sudo /var/ossec/integrations/virustotal /tmp/wazuh_test_alert.json
```

Expected result:

* The script prints a short JSON summary to stdout.
* A file `/var/ossec/logs/virustotal_44d88612fea8a8f36de82e1278abb02f.json` is created with the VT stats and raw response.

# 7) Example `curl` test for VirusTotal (manual quick-check)

```bash
curl -i -H "x-apikey: YOUR_REAL_API_KEY" \
  "https://www.virustotal.com/api/v3/files/44d88612fea8a8f36de82e1278abb02f"
```

If successful, you’ll receive a JSON response with `last_analysis_stats` and other metadata.

# 8) Troubleshooting

* **Script not executed by Wazuh**: ensure `/var/ossec/integrations/virustotal` is executable and that `<name>virustotal</name>` exists in `ossec.conf`.
* **Permissions**: ensure the script can read the API key and write to `/var/ossec/logs/` (usually root-owned; Wazuh runs as root). If you run the script manually, use `sudo`.
* **VT API errors**: check your VT key, ensure it’s a v3 key and not revoked. Use `curl` to validate.
* **No hashes in alert**: Wazuh alerts may not contain file hashes. You can modify your detection rules to capture file hash attributes or have the agent compute and send them.


