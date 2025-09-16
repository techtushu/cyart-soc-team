# Caldera on Raspberry Pi (aarch64) + Wazuh Integration

**Purpose:** A step-by-step GitHub-ready documentation for new users showing how to run MITRE Caldera with a Sandcat agent on a Raspberry Pi 4 (aarch64/Kali), how to build an ARM agent, create safe adversary profiles & abilities, run operations, and integrate with Wazuh (running on Windows → WSL Ubuntu).

> Tested environment in this guide (adjust IPs/usernames to your setup):
>
> * Caldera Server: WSL Ubuntu on Windows, IP `192.168.1.41` (port `8888`)
> * Raspberry Pi: Kali Linux, `aarch64` (hostname: `cyberpi`)
> * Wazuh: Dashboard, manager, indexer on Windows (WSL Ubuntu)

---

## Table of Contents

1. Overview & goals
2. Prerequisites
3. Build and run Sandcat on Raspberry Pi (ARM64)
4. Integrate ARM binary into Caldera auto-deploy (optional)
5. Create an adversary profile (step-by-step)
6. Add safe abilities (copy/paste-ready)
7. Run an operation and interpret results
8. Integrate Caldera activity with Wazuh (detection)
9. Example Wazuh rule snippets
10. Troubleshooting
11. Security, ethics & cleanup
12. References & further reading

---

## 1. Overview & goals

This repo/document is meant for learning and lab testing only. It shows how to:

* Build the Sandcat agent for ARM64 on a Raspberry Pi
* Run the agent and verify check-in in Caldera UI
* Create a non-destructive adversary (discovery-focused) and abilities
* Run an operation and inspect results
* Create simple Wazuh detection rules for those actions

**Do not** use these techniques on networks or systems you do not own or have explicit authorization to test.

---

## 2. Prerequisites

* Basic Linux command-line familiarity
* Caldera installed on a server (here: WSL Ubuntu on Windows)
* Git installed on the Pi and server
* Go installed on the Pi (>= 1.19 recommended)
* Network connectivity between the Pi and Caldera server

Example package installs (on Pi):

```bash
sudo apt update
sudo apt install -y git golang build-essential
```

Verify architecture on the Pi:

```bash
uname -a  # should include "aarch64"
```

---

## 3. Build and run Sandcat on Raspberry Pi (ARM64)

1. Clone the Sandcat repo on the Pi:

```bash
cd ~
git clone https://github.com/mitre/sandcat.git
cd sandcat/gocat
```

2. Initialize and tidy Go modules (if needed):

```bash
# Only run these if the module is not initialized in that folder
go mod init gocat  # may be optional depending on repo state
go mod tidy
```

3. Build for ARM64:

```bash
GOOS=linux GOARCH=arm64 go build -o sandcat sandcat.go
```

4. Run the agent (replace server IP with your Caldera server):

```bash
./sandcat -server http://192.168.1.41:8888 -group red -v
```

5. Verify in Caldera UI → Agents: your `paw`/host should appear.

> If you prefer a long-running service, wrap `sandcat` with a systemd unit on the Pi (lab only).

---

## 4. Integrate ARM binary into Caldera auto-deploy (optional)

If you want Caldera's "Deploy Agent" to hand out the ARM binary rather than an x86 one:

1. Build the ARM binary as above on the Pi.
2. Copy it to the Caldera server (WSL Ubuntu) into the sandcat plugin directory.

Example SCP (from Pi to server):

```bash
scp ./sandcat user@192.168.1.41:/home/user/caldera/plugins/sandcat/sandcat_arm64
```

3. Edit the plugin or the endpoint the dashboard uses to serve binaries. Replace the default x86 binary or add a new download option for `aarch64`.
4. Restart Caldera service or container.

---

## 5. Create an adversary profile (step-by-step)

1. Login to Caldera UI → **Adversaries** → **Create**.
2. Fill details:

   * **Name:** `Pi_Discovery_Adversary`
   * **Description:** `Non-destructive discovery & host enumeration for Raspberry Pi labs`
3. Create the adversary (this creates an empty shell). You will add abilities next.

---

## 6. Add safe abilities (copy/paste-ready)

Create **Abilities → Create** for each ability. Important fields to set:

* **Platform:** `linux`
* **Executor:** `sh` (or `bash` if using bash-specific syntax)
* **Command:** copy from the blocks below
* **Technique** (optionally add MITRE ATT\&CK ID)

### Abilities (safe, read-only)

1. **Get System Identity** (T1033 / Discovery)

```sh
uname -a; hostname; whoami; id
```

2. **OS and Distro** (T1082)

```sh
cat /etc/os-release || lsb_release -a || echo "no /etc/os-release"
```

3. **List Users and Home Dirs** (T1087)

```sh
cut -d: -f1,3,6 /etc/passwd | awk -F: '{print $1,"UID="$2,"HOME="$3}'; ls -la /home || echo "no /home"
```

4. **Process Enumeration** (T1057)

```sh
ps aux --cols 200 | head -n 50
```

5. **Network Sockets & Listening** (T1049)

```sh
ss -tunlp || netstat -tunlp
```

6. **Network Configuration** (T1016)

```sh
ip addr show; ip route show; arp -an
```

7. **Check SSH Keys** (T1552.001)

```sh
[ -f /etc/ssh/ssh_host_rsa_key.pub ] && echo "host pub keys:" && ls -l /etc/ssh/*.pub; echo "user authorized_keys (if readable):"; for d in /home/*; do [ -f "$d/.ssh/authorized_keys" ] && echo "$d/.ssh/authorized_keys:" && sed -n '1,20p' "$d/.ssh/authorized_keys"; done
```

8. **SUID Files** (T1069)

```sh
find / -perm -4000 -type f -maxdepth 5 -exec ls -ld {} \; 2>/dev/null | head -n 100
```

9. **Scheduled Jobs** (T1053)

```sh
crontab -l 2>/dev/null || echo "no user crontab"; [ -d /etc/cron.* ] && ls -la /etc/cron.* || echo "no /etc/cron.*"; systemctl list-timers --all --no-pager
```

10. **Sudo Privileges** (T1069.002)

```sh
sudo -l 2>&1 || echo "sudo -l not permitted for this user"
```

---

## 7. Run an operation and interpret results

1. Caldera → **Operations → New Operation**.
2. Choose **Pi\_Discovery\_Adversary** and choose your Pi agent (or a group containing it).
3. Start operation.
4. Observe task logs per ability in the agent output pane.
5. Common notes:

   * If an ability times out, increase the timeout for that ability or make command less heavy.
   * Some commands may return little/no output depending on permissions; that is expected.

---

## 8. Integrate Caldera activity with Wazuh (detection)

Goal: create simple Wazuh rules that trigger when Caldera abilities run specific commands.

### Where to inspect logs

* On the Pi: Caldera agent runs commands; they may be visible in `auth.log`, `syslog`, or through process auditing (auditd). Commands executed by the sandcat agent appear as child processes of the sandcat process. Configure Wazuh/OSSEC to capture process execution logs.
* On the Wazuh manager: check `alerts` and `logs` in the Wazuh dashboard.

### Basic approach

1. Ensure command execution is logged on the Pi. Options:

   * Install/enable `auditd` and configure syscall audit rules for `execve` to capture command arguments.
   * Alternatively, log shell history for the sandcat user (less reliable).
2. Create Wazuh rules matching suspicious commands (or the exact commands you used in abilities).

---

## 9. Example Wazuh rule snippets

> Place custom rules in the manager rules directory (e.g., `/var/ossec/etc/rules/local_rules.xml` or `/var/ossec/etc/rules/` depending on version). Restart the manager after adding rules.

**Example:** Detect `ps aux` run by any user (match on command string in audit/syslog):

```xml
<group name="caldera-commands,">
  <rule id="100100" level="8">
    <if_sid>1001</if_sid>
    <decoded_as>json</decoded_as>
    <field name="command">ps aux</field>
    <description>Caldera test: 'ps aux' executed</description>
    <options>no_full_log</options>
  </rule>
</group>
```

**Note:** The exact XML will depend on how command data is forwarded to Wazuh (auditd, syslog, agent). If you forward `auditd` entries to Wazuh, match `exe`/`argv` fields.

**Auditd example rule** (auditd configuration on Pi) — append to `/etc/audit/rules.d/caldera.rules`:

```
-a always,exit -F arch=b64 -S execve -k exec_log
-a always,exit -F arch=b32 -S execve -k exec_log
```

After enabling, ensure Wazuh is collecting audit logs and create a rule to match `exec_log` messages.

---

## 10. Troubleshooting

* **`exec format error`** → Architecture mismatch. Rebuild for `GOARCH=arm64`.
* **`go: cannot find main module`** → Initialize module or `cd` into correct folder (e.g., `gocat`). Use `go mod init` and `go mod tidy` if required.
* **Agent not showing in Caldera UI** → Check network, firewall, server IP/port, and that the agent used `-server http://<ip>:<port>` with reachable IP.
* **Ability fails / times out** → Increase ability timeout or simplify command. Check agent logs for stderr.
* **Wazuh doesn't show alerts** → Ensure log forwarding (auditd/syslog) is configured and Wazuh manager rules match fields actually sent.

---

## 11. Security, ethics & cleanup

* Use only in authorized test environments.
* After testing, stop/remove agent binary and any temporary modifications.

Cleanup example (on Pi):

```bash
# stop agent if running in foreground: Ctrl+C or kill by PID
pkill sandcat
# remove binary
rm -f ~/sandcat/gocat/sandcat
```

---

## 12. References & further reading

* MITRE Caldera: [https://github.com/mitre/caldera](https://github.com/mitre/caldera)
* Sandcat repository: [https://github.com/mitre/sandcat](https://github.com/mitre/sandcat)
* Wazuh documentation: [https://documentation.wazuh.com/](https://documentation.wazuh.com/)
* auditd docs: man auditd

---

### Next steps I can help with (pick one)

* Produce a ready-to-import adversary manifest (YAML/JSON) you can upload to Caldera
* Produce a set of Caldera ability JSONs for bulk import
* Provide specific Wazuh rule XMLs tuned to your log forwarding format
* Create a repo structure and `git` commits and show commands to push to GitHub

*End of guide.*

