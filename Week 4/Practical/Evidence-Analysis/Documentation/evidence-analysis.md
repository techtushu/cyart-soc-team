# Velociraptor + FTK Imager — Evidence Analysis (Windows VM)

**Purpose:** step-by-step GitHub-ready documentation so a new analyst can reproduce the evidence-collection, hashing, and chain-of-custody workflow we did: enroll a Windows VM as a Velociraptor client, collect `netstat()` output, capture memory with FTK Imager, compute SHA256, and produce a chain-of-custody and short analysis.

---

## Table of contents

1. Overview
2. Scope & assumptions
3. Prerequisites
4. Architecture (quick)
5. Folder & file layout
6. Step-by-step instructions

   * Prepare the environment
   * Install / start Velociraptor server (host)
   * Generate client config / enroll client
   * Install Velociraptor client on Windows VM
   * Verify connectivity and quick troubleshooting
   * Collect `netstat()` using Velociraptor
   * Export and hash evidence
   * Capture memory with FTK Imager
   * Correlate netstat + memory (basic)
7. Chain-of-custody template (CSV + example)
8. Short investigation report template (Markdown)
9. Troubleshooting checklist (common issues)
10. Next steps & references

---

## 1) Overview

This doc shows a repeatable workflow to:

* Enroll a Windows 10 VM as a Velociraptor client.
* Run `SELECT * FROM netstat()` (Velociraptor artifact) and export the results.
* Use FTK Imager to capture a memory image of the VM.
* Calculate SHA256 for each evidence item and log them in a chain-of-custody file.
* Provide basic next-analysis steps (Volatility / netscan correlation).

Goal: produce defensible evidence that can be shipped in a report along with hashes and a chain-of-custody CSV file.

---

## 2) Scope & assumptions

* Host machine (analysis/server): Windows 10 or Linux/WSL where Velociraptor **server** runs (GUI + Frontend).
* Target machine (endpoint): Windows 10 VM (client) where we install Velociraptor **agent** and collect artifacts.
* FTK Imager is used inside the VM (or on the analysis workstation) to capture memory/disk.
* Analyst has Administrator privileges on host and VM.
* Network connectivity exists between host and VM (or both are on same physical host via host-only/bridged).

---

## 3) Prerequisites

* Velociraptor binary (Windows `amd64` and Linux/Windows server binary). Put `velociraptor.exe` on Windows and `velociraptor` on Linux/WSL if used.
* FTK Imager (or FTK Imager Lite) installed on the analyst machine or the VM.
* Windows 10 VM with admin access.
* Optional: Volatility 3 (for memory analysis), libewf / ewf-tools (for mounting E01), python3.

Useful utilities:

* PowerShell (Admin) on Windows
* `sha256sum` (Linux) or `Get-FileHash` (PowerShell) for hashing
* `scp` / file share to move client config to VM

---

## 4) Architecture (quick)

```
[Analyst Host: Velociraptor Server GUI] <----TLS----> [Velociraptor Client (Windows 10 VM)]
                                  |
                                  +-- (FTK Imager runs on VM and saves memory image to evidence share)
```

---

## 5) Folder & file layout (recommended)

```
/evidence/
  ├─ velociraptor/
  │   ├─ client.config.yaml
  │   ├─ server.config.yaml
  │   └─ netstat_results.csv
  ├─ images/
  │   ├─ WinVM_memory_2025-09-13.E01
  │   └─ WinVM_C_drive_2025-09-13.E01
  └─ hashes/
      ├─ evidence_hashes.txt
      └─ chain_of_custody.csv
```

---

## 6) Step-by-step instructions

### A — Prepare environment & snapshot

1. Create the Windows 10 VM (VirtualBox / Hyper-V). Give it enough RAM (4–8 GB) and disk (≥50 GB).
2. Before doing anything invasive, **take a VM snapshot** named `baseline` (this helps return to a pristine state).

### B — Install / start Velociraptor server (host)

You can run the server on the Windows host, or on WSL/Ubuntu. Two quick options are shown: instant/demo mode, or proper server+frontend.

**Instant/demo (fast, single-machine):**

* Place `velociraptor` binary on the host and run:

```powershell
# Windows: start instant (GUI + local client)
.\\velociraptor.exe gui
```

This starts a local frontend + GUI and a single client (useful for testing).

**Proper server (recommended):**

1. Generate server config (interactive):

```bash
# Linux/WSL or Windows PowerShell
./velociraptor config generate -i
# or on Windows: .\velociraptor.exe config generate -i
```

This produces `server.config.yaml` (and may produce `client.config.yaml` too depending on options).

2. Start the frontend (server + GUI):

```bash
# Linux/WSL
./velociraptor --config server.config.yaml frontend -v
# Windows PowerShell
.\velociraptor.exe --config server.config.yaml frontend -v
```

* Note: the `-v` flag enables verbose logging. The server will print the GUI URL (e.g., [https://0.0.0.0:8889](https://0.0.0.0:8889)).
* Open the GUI URL in your browser.

### C — Create / Export the client config

You can create a `client.config.yaml` from the server GUI (Admin → Enrollment) or CLI.

**CLI method (produce client.config.yaml):**

```bash
# From server host
./velociraptor --config server.config.yaml config client -v > client.config.yaml
# Windows: .\velociraptor.exe --config server.config.yaml config client -v > client.config.yaml
```

* Copy `client.config.yaml` to the Windows 10 VM (use SCP, SMB, or a shared folder).

### D — Install Velociraptor client on the Windows VM

On the VM (Administrator PowerShell):

1. Put `velociraptor.exe` and `client.config.yaml` in the same folder (e.g., `C:\Velociraptor`).
2. Install as a Windows service and start:

```powershell
cd C:\Velociraptor
.\velociraptor.exe service install --config client.config.yaml
Start-Service velociraptor
# verify
Get-Service velociraptor
```

3. In the server GUI, go to **Clients** → your VM should appear (might take one poll interval).

**Optional: speed up check-in in `client.config.yaml`**
Edit `client.config.yaml` and add under `Client:`

```yaml
Client:
  poll_delay: 60   # seconds between server checks (lab use only)
```

Restart the service after changing.

### E — Verify connectivity & quick troubleshooting

From the VM:

```powershell
Test-NetConnection <SERVER_IP_OR_HOST> -Port <PORT>
# check logs
Get-Content "C:\Path\To\velociraptor.log" -Tail 40
```

If client shows as offline in GUI but service is running, ensure correct `client.config.yaml` was used when installing the service, firewall rules allow traffic, and the poll delay hasn't just lapsed.

### F — Collect `netstat()` using Velociraptor

Two ways:

**1) Built-in artifact (GUI)**

* GUI → Clients → select VM → Collect Artifact → search `Windows.Network.Netstat` → Launch.
* Wait for results. Click the task and **Download Results → CSV**.

**2) Notebook / Live Query**

* Notebooks → New notebook → Run the VQL:

```sql
SELECT * FROM netstat();
```

* Export results from the notebook.

Save the CSV as `/evidence/velociraptor/netstat_results_YYYYMMDD.csv`.

### G — Export and compute SHA256

After exporting the results file, compute SHA256 and log it.

**PowerShell (Windows):**

```powershell
Get-FileHash -Path "C:\evidence\netstat_results_2025-09-13.csv" -Algorithm SHA256
```

**Linux:**

```bash
sha256sum netstat_results_2025-09-13.csv
```

Copy the hex into your `evidence_hashes.txt` and `chain_of_custody.csv`.

### H — Capture memory using FTK Imager

**Inside the VM** (recommended to capture volatile RAM):

1. Launch **FTK Imager** (Run as Administrator).
2. `File` → `Capture Memory`.

   * Destination: choose evidence folder or network share (preferably write-protected destination)
   * Filename: `WinVM_memory_YYYYMMDD.E01` (or `.aff` if using other tools)
3. Start capture. Wait until FTK finishes and **note the progress & completion screenshot**.
4. FTK Imager will show hashes (MD5 / SHA1). If it does not show SHA256, compute SHA256 with `Get-FileHash` or `sha256sum`.

**Capture disk (logical/physical) (optional):**

* `File` → `Create Disk Image` → choose Physical Drive or Logical Drive → Output: E01 (recommended) or Raw → enable hashing (if available) → Start.
* Save FTK imaging log. The FTK log and the hash values are evidence items.

### I — Correlate netstat + memory (basic)

1. Use the `netstat` CSV to identify suspicious Remote IPs, Ports, and PIDs.
2. From the memory image, use Volatility to confirm the PID and process details.

**Convert E01 to raw (on Linux) for Volatility if needed:**

```bash
# install libewf/ewf-tools (Debian/Ubuntu)
sudo apt install ewf-tools
# mount E01 in read-only mode
mkdir -p /mnt/ewf_mount
ewfmount WinVM_memory_YYYYMMDD.E01 /mnt/ewf_mount
# the raw stream will be at /mnt/ewf_mount/ewf1
# copy it out if you need a standalone raw file
dd if=/mnt/ewf_mount/ewf1 of=WinVM_memory_YYYYMMDD.raw bs=4096 status=progress
```

**Volatility 3 (basic commands):**

```bash
# example: list processes
volatility3 -f WinVM_memory_YYYYMMDD.raw windows.pslist
# scan net connections in memory
volatility3 -f WinVM_memory_YYYYMMDD.raw windows.netscan
# find cmdline for PID 1234
volatility3 -f WinVM_memory_YYYYMMDD.raw windows.cmdline --pid 1234
```

(If you use Volatility 2, use matching plugin names and the correct profile.)

---

## 7) Chain-of-Custody template (CSV)

**Filename:** `chain_of_custody.csv`

```
Item ID,Item,Description,Collected By,Collection Date (IST),File Name / Path,Hash (SHA256),Storage Location,Notes
1,Network Log,Velociraptor netstat export,Tushar Yadav,2025-09-13 22:10 IST,/evidence/velociraptor/netstat_results_2025-09-13.csv,<sha256>,/evidence/velociraptor/,Exported from Velociraptor GUI
2,Memory Dump,FTK Imager RAM capture,Tushar Yadav,2025-09-13 22:35 IST,/evidence/images/WinVM_memory_2025-09-13.E01,<sha256>,/evidence/images/,FTK Imager capture; saved FTK log
```

Replace `<sha256>` with the exact hash you computed.

---

## 8) Short investigation report template (Markdown)

**Filename:** `investigation_report.md`

```
# Incident Report — Velociraptor + FTK Evidence

**Incident ID:** IR-YYYYMMDD-001
**Analyst:** Tushar Yadav
**Date:** 2025-09-13

## Summary
(1–3 line summary of what you found)

## Evidence Collected
- Netstat export: `/evidence/velociraptor/netstat_results_2025-09-13.csv` (SHA256: ...)
- Memory image: `/evidence/images/WinVM_memory_2025-09-13.E01` (SHA256: ...)

## Steps Taken
1. Deployed Velociraptor server & enrolled Windows 10 VM client.
2. Collected `Windows.Network.Netstat` artifact via Velociraptor.
3. Exported CSV and computed SHA256.
4. Captured RAM using FTK Imager; computed SHA256.

## Findings
- (Table of suspicious connections and justification)

## Recommendations
- Isolate host
- Full malware triage / YARA scans / reimage

## Appendix
- Velociraptor query used: `SELECT * FROM netstat();`
- FTK Imager log: attached
```

---

## 9) Troubleshooting checklist (common issues)

* **Client shows offline but service running:** check `client.config.yaml` used to install service, check logs, ensure firewall allows server port, verify server IP and port in client config, reduce `poll_delay` in config for lab.
* **Velociraptor GUI not accessible:** check server process, confirm port, ensure TLS self-signed warnings accepted in browser.
* **FTK can't write image:** ensure destination has space and write permissions; prefer network share or external drive.
* **E01 not recognized by Volatility:** use `ewfmount` to create raw stream and dd to raw file before feeding to Volatility.

---

## 10) Next steps & recommended learning

* Deeper volatile analysis with Volatility (YARA, malfind, dlllist, handles).
* Automate Velociraptor hunts for periodic netstat collection.
* Integrate Velociraptor findings into your SIEM (Wazuh) for alerting and enrichment.

---

## Appendix: Quick copy-paste commands summary

**On server (Windows/PowerShell)**

```powershell
# generate configs interactively
.\velociraptor.exe config generate -i
# create client config with CLI
.\velociraptor.exe --config server.config.yaml config client -v > client.config.yaml
# start the frontend (server)
.\velociraptor.exe --config server.config.yaml frontend -v
```

**On client (Windows VM, PowerShell Admin)**

```powershell
# install agent as service
cd C:\Velociraptor
.\velociraptor.exe service install --config client.config.yaml
Start-Service velociraptor
# run quick netstat query locally (standalone)
.\velociraptor.exe gui    # opens local GUI with a local client
# or run a single VQL command locally
.\velociraptor.exe query "SELECT * FROM netstat()" --config client.config.yaml > netstat_results.json
```

**Hashing**

```powershell
Get-FileHash -Path "C:\evidence\file.ext" -Algorithm SHA256
```

**Convert E01 to raw (Linux)**

```bash
sudo apt install ewf-tools
mkdir -p /mnt/ewf_mount
ewfmount WinVM_memory.E01 /mnt/ewf_mount
# raw stream is /mnt/ewf_mount/ewf1
dd if=/mnt/ewf_mount/ewf1 of=WinVM_memory.raw bs=4096 status=progress
```

**Volatility 3 example**

```bash
volatility3 -f WinVM_memory.raw windows.pslist
volatility3 -f WinVM_memory.raw windows.netscan
volatility3 -f WinVM_memory.raw windows.cmdline --pid 1234
```

---
