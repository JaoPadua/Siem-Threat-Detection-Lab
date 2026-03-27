# Wazuh SIEM + Telegram Alerts

A real-world security monitoring lab built on **Wazuh v4.14.1**, deployed on VirtualBox VMs, with real-time **Telegram bot notifications** for categorized threats. Integrates **ClamAV** for malware detection, **Suricata** for network IDS, and **FIM** for file integrity monitoring across Linux and Windows agents.

---

## Table of Contents

- [Lab Environment](#lab-environment)
- [Installation](#installation)
- [Agent Deployment](#agent-deployment)
- [ClamAV Setup](#clamav-setup)
- [Agent Group Configuration](#agent-group-configuration)
- [Custom Rules](#custom-rules)
- [Telegram Bot Integration](#telegram-bot-integration)
- [Alert Types](#alert-types)
- [Testing Procedures](#testing-procedures)
- [Agent Dashboard Features](#agent-dashboard-features)
- [Troubleshooting](#troubleshooting)

---

## Lab Environment

### Network Topology

```
OPNsense Firewall VM (Host-Only)
        |
        |--- Wazuh Manager VM (Host-Only + External for SSH)
        |--- Ubuntu Agent VM (Host-Only)
        |--- Windows Agent VM (Host-Only)
```

### Component Table

| Component | Role | Location |
|---|---|---|
| OPNsense | Firewall | VM |
| Wazuh Manager v4.14.1 | SIEM / Alert Manager | VM |
| Suricata | Network IDS | Same VM as Wazuh |
| ClamAV 1.4.3 | Antivirus / Malware Detection | Agent VMs |
| Telegram Bot | Alert Notifications | External |

### Key File Locations

| File | Path |
|---|---|
| Wazuh Rules | `/var/ossec/etc/rules/local_rules.xml` |
| Telegram Script | `/var/ossec/integrations/custom-telegram.py` |
| Telegram Shell Wrapper | `/var/ossec/integrations/custom-telegram` |
| Wazuh Manager Config | `/var/ossec/etc/ossec.conf` |
| ClamAV Scan Script | `/usr/local/bin/clamav-scan.sh` |
| ClamAV Detection Log | `/var/log/clamav/detections.log` |

---

## Installation

### Requirements

- 64-bit Intel, AMD, or ARM Linux processor (x86_64 or AARCH64)
- Recommended OS: Ubuntu 20.04 / 22.04 / 24.04, RHEL 7–10, Amazon Linux 2/2023

### Install Wazuh (All-in-One)

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

Once complete, the terminal will output:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP>
      User: admin
      Password: <ADMIN_PASSWORD>
INFO: Installation finished.
```

Save these credentials. Access the dashboard at `https://<WAZUH_DASHBOARD_IP>`.

---

## Agent Deployment

From the Wazuh Dashboard:

1. Click **Active** on the dashboard
2. Click **Deploy new agent**
3. Select the agent OS
4. For Linux — choose **Deb amd64** (Intel/AMD CPU)
5. Enter the Wazuh Server IP, Agent Name, and Group (`linux-agents` or `windows-agents`)
6. Paste the provided command into the terminal (Linux) or PowerShell as Administrator (Windows)

### Verify Agent is Active

```bash
sudo /var/ossec/bin/agent_control -l
```

---

## ClamAV Setup

### Ubuntu Agent

```bash
sudo apt update && sudo apt install clamav clamav-daemon -y
sudo systemctl stop clamav-freshclam
sudo freshclam
sudo systemctl start clamav-freshclam
sudo systemctl enable clamav-daemon
```

#### Create Detection Log Directory

```bash
sudo mkdir -p /var/log/clamav
sudo touch /var/log/clamav/detections.log
sudo chown -R clamav:clamav /var/log/clamav
sudo chmod 755 /var/log/clamav
sudo chmod 644 /var/log/clamav/detections.log
```

#### Create VirusEvent Script

This logs detections in a format Wazuh can parse and auto-removes the infected file.

```bash
sudo nano /usr/local/bin/clamav-event.sh
```

```bash
#!/bin/bash
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
HOSTNAME=$(hostname)
LOGFILE=/var/log/clamav/detections.log
VIRUS="$CLAM_VIRUSEVENT_VIRUSNAME"
FILE="$CLAM_VIRUSEVENT_FILENAME"

echo "$TIMESTAMP - MALWARE DETECTED: $VIRUS | FILE: $FILE | HOST: $HOSTNAME" >> $LOGFILE
rm -f "$FILE"
echo "$TIMESTAMP - FILE REMOVED: $FILE | HOST: $HOSTNAME" >> $LOGFILE
```

```bash
sudo chmod +x /usr/local/bin/clamav-event.sh
```

#### Auto-Scan Script (Every 5 Minutes, Low Priority)

```bash
sudo nano /usr/local/bin/clamav-scan.sh
```

```bash
#!/bin/bash
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
HOSTNAME=$(hostname)
LOGFILE=/var/log/clamav/detections.log

nice -n 19 ionice -c 3 clamscan \
  /home/*/Downloads /home/*/Desktop /home/*/Documents /tmp \
  --remove --quiet --no-summary 2>/dev/null | while IFS= read -r line; do
    if echo "$line" | grep -q "FOUND"; then
        VIRUS=$(echo "$line" | awk '{print $2}')
        FILE=$(echo "$line" | awk '{print $1}' | tr -d ':')
        echo "$TIMESTAMP - MALWARE DETECTED: $VIRUS | FILE: $FILE | HOST: $HOSTNAME" >> $LOGFILE
        echo "$TIMESTAMP - FILE REMOVED: $FILE | HOST: $HOSTNAME" >> $LOGFILE
    fi
done
```

```bash
sudo chmod +x /usr/local/bin/clamav-scan.sh
sudo crontab -e
# Add:
*/5 * * * * /usr/local/bin/clamav-scan.sh
```

### Windows Agent

1. Download from [clamav.net/downloads](https://www.clamav.net/downloads) and install to `C:\ClamAV\`
2. Configure `clamd.conf`:

```
TCPSocket 3310
TCPAddr 127.0.0.1
LogFile C:\ClamAV\logs\clamd.log
VirusEvent C:\ClamAV\scripts\virus_event.bat
```

3. Create `C:\ClamAV\scripts\virus_event.bat`:

```bat
@echo off
set INFECTED=%CLAM_VIRUSEVENT_FILENAME%
set VIRUS=%CLAM_VIRUSEVENT_VIRUSNAME%
set TIMESTAMP=%DATE% %TIME%
echo %TIMESTAMP% - MALWARE DETECTED: %VIRUS% | FILE: %INFECTED% | HOST: %COMPUTERNAME% >> C:\ClamAV\logs\detections.log
del /f "%INFECTED%"
```

4. Schedule scans via PowerShell (Admin):

```powershell
$Action = New-ScheduledTaskAction -Execute "C:\ClamAV\clamscan.exe" `
  -Argument "-r C:\Users --remove --log=C:\ClamAV\logs\detections.log"
$Trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes 5) `
  -Once -At (Get-Date)
Register-ScheduledTask -TaskName "ClamAV Scan" -Action $Action -Trigger $Trigger -RunLevel Highest
```

---

## Agent Group Configuration

### Create Groups (Wazuh Dashboard)

Navigate to: **Management → Groups → Add new group**

Create: `linux-agents` and `windows-agents`

### Linux Group Config

```xml
<agent_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/clamav/detections.log</location>
  </localfile>
  <syscheck>
    <frequency>300</frequency>
    <directories check_all="yes" realtime="yes" report_changes="yes">/home</directories>
    <directories check_all="yes" realtime="yes" report_changes="yes">/tmp</directories>
    <directories check_all="yes" realtime="yes" report_changes="yes">/var/www</directories>
  </syscheck>
</agent_config>
```

### Windows Group Config

```xml
<agent_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>C:\ClamAV\logs\detections.log</location>
  </localfile>
  <syscheck>
    <frequency>300</frequency>
    <directories check_all="yes" realtime="yes" report_changes="yes">C:\Users</directories>
    <directories check_all="yes" realtime="yes" report_changes="yes">C:\Temp</directories>
  </syscheck>
</agent_config>
```

### Assign Agents to Groups

```bash
sudo /var/ossec/bin/agent_groups -a -i 002 -g linux-agents
sudo /var/ossec/bin/agent_groups -a -i 003 -g windows-agents
```

---

## Custom Rules

Edit `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="clamav,malware,">
  <rule id="100200" level="12">
    <match>MALWARE DETECTED</match>
    <description>ClamAV: Malware detected on agent</description>
    <group>malware_detected,</group>
  </rule>

  <rule id="100201" level="10">
    <match>FILE REMOVED</match>
    <description>ClamAV: Infected file automatically removed</description>
    <group>malware_removed,</group>
  </rule>

  <rule id="100202" level="12">
    <if_sid>550,553,554</if_sid>
    <match>MALWARE</match>
    <description>FIM: Suspicious file change correlated with malware detection</description>
    <group>fim_malware,</group>
  </rule>
</group>
```

Validate rules before restarting:

```bash
sudo /var/ossec/bin/ossec-analysisd -t
```

---

## Telegram Bot Integration

### Step 1: Create a Bot

1. Open Telegram → search `@BotFather`
2. Send `/newbot`, follow the prompts
3. Save the bot token: `7123456789:AAFxxxxxxxxxxxxxxxx`

### Step 2: Get Your Chat ID

Send a message to your bot, then visit:
```
https://api.telegram.org/bot<TOKEN>/getUpdates
```
Copy the `"id"` value from the `"chat"` object. Group chat IDs are negative numbers.

### Step 3: Install requests Library

```bash
sudo /var/ossec/framework/python/bin/pip3 install requests
```

### Step 4: Create the Shell Wrapper

```bash
sudo nano /var/ossec/integrations/custom-telegram
```

```bash
#!/bin/sh
WPYTHON_BIN="framework/python/bin/python3"
SCRIPT_PATH_NAME="$0"
DIR_NAME="$(cd $(dirname ${SCRIPT_PATH_NAME}); pwd -P)"
SCRIPT_NAME="$(basename ${SCRIPT_PATH_NAME})"

case ${DIR_NAME} in
    */integrations)
        if [ -z "${WAZUH_PATH}" ]; then
            WAZUH_PATH="$(cd ${DIR_NAME}/..; pwd)"
        fi
        PYTHON_SCRIPT="${DIR_NAME}/${SCRIPT_NAME}.py"
    ;;
esac

${WAZUH_PATH}/${WPYTHON_BIN} ${PYTHON_SCRIPT} "$@"
```

### Step 5: Create the Python Alert Script

```bash
sudo nano /var/ossec/integrations/custom-telegram.py
```

```python
#!/usr/bin/env python
import sys, json, requests, re
from datetime import datetime, timezone, timedelta

CHAT_ID = "YOUR_CHAT_ID"

MALWARE_RULES = [100200, 100201, 100202]
RDP_RULES     = [60122, 60137, 60138]
SMTP_RULES    = [67551, 67552, 100032]
SSH_RULES     = [5712, 5763, 40101]
SQLI_RULES    = [31103, 31106]
DDOS_RULES    = []  # find with: grep -r -i "ddos\|flood" /var/ossec/ruleset/rules/ | grep "id="

alert_file = open(sys.argv[1])
hook_url   = sys.argv[3]
alert_json = json.loads(alert_file.read())
alert_file.close()

alert_level = alert_json['rule'].get('level', 'N/A')
description = alert_json['rule'].get('description', 'N/A')
agent       = alert_json['agent'].get('name', 'N/A')
agent_ip    = alert_json['agent'].get('ip', 'N/A')
rule_id     = int(alert_json['rule'].get('id', 0))
full_log    = alert_json.get('full_log', 'N/A')

try:
    gmt8 = timezone(timedelta(hours=8))
    ts = datetime.strptime(alert_json.get('timestamp', '')[:19], '%Y-%m-%dT%H:%M:%S')
    timestamp = ts.replace(tzinfo=gmt8).strftime('%Y-%m-%d %H:%M:%S GMT+8')
except:
    timestamp = alert_json.get('timestamp', 'N/A')

src_ip = "N/A"
if 'data' in alert_json and isinstance(alert_json['data'], dict):
    src_ip = alert_json['data'].get('srcip') or alert_json['data'].get('src_ip') or "N/A"
if src_ip == "N/A":
    m = re.search(r"\b(\d{1,3}(?:\.\d{1,3}){3})\b", full_log)
    if m: src_ip = m.group(1)

malware_name = infected_file = "N/A"
if rule_id in MALWARE_RULES:
    m1 = re.search(r"MALWARE DETECTED:\s*([^\|]+)", full_log)
    m2 = re.search(r"FILE[:\s]+([^\|]+)", full_log)
    if m1: malware_name  = m1.group(1).strip()
    if m2: infected_file = m2.group(1).strip()

if rule_id in MALWARE_RULES:
    emojis = {100200: ("🦠","MALWARE DETECTED","⚠️ File flagged — awaiting removal"),
              100201: ("🗑️","MALWARE REMOVED","✅ File automatically removed by ClamAV"),
              100202: ("📁","FIM MALWARE CORRELATION","⚠️ Suspicious file change detected")}
    emoji, category, status = emojis[rule_id]
    text = (f"{emoji} *{category}*\n─────────────────────\n"
            f"🖥️ *Agent:* `{agent}` (`{agent_ip}`)\n"
            f"📋 *Rule ID:* `{rule_id}`\n⚡ *Level:* `{alert_level}`\n"
            f"🦠 *Malware:* `{malware_name}`\n📄 *File:* `{infected_file}`\n"
            f"🕐 *Time:* `{timestamp}`\n─────────────────────\n🔰 *Status:* {status}")
elif rule_id in SSH_RULES:
    text = (f"🔐 *SSH BRUTE FORCE*\n─────────────────────\n"
            f"🖥️ *Agent:* `{agent}` (`{agent_ip}`)\n"
            f"📋 *Rule ID:* `{rule_id}`\n⚡ *Level:* `{alert_level}`\n"
            f"📝 *Description:* {description}\n🌐 *Source IP:* `{src_ip}`\n"
            f"🕐 *Time:* `{timestamp}`\n─────────────────────\n"
            f"🔰 *Status:* ⚠️ Multiple failed SSH attempts")
elif rule_id in RDP_RULES:
    text = (f"🖥️ *RDP ATTACK DETECTED*\n─────────────────────\n"
            f"🖥️ *Agent:* `{agent}` (`{agent_ip}`)\n"
            f"📋 *Rule ID:* `{rule_id}`\n⚡ *Level:* `{alert_level}`\n"
            f"📝 *Description:* {description}\n🌐 *Source IP:* `{src_ip}`\n"
            f"🕐 *Time:* `{timestamp}`\n─────────────────────\n"
            f"🔰 *Status:* ⚠️ RDP brute force/scan in progress")
elif rule_id in SMTP_RULES:
    text = (f"📧 *SMTP ATTACK DETECTED*\n─────────────────────\n"
            f"🖥️ *Agent:* `{agent}` (`{agent_ip}`)\n"
            f"📋 *Rule ID:* `{rule_id}`\n⚡ *Level:* `{alert_level}`\n"
            f"📝 *Description:* {description}\n🌐 *Source IP:* `{src_ip}`\n"
            f"🕐 *Time:* `{timestamp}`\n─────────────────────\n"
            f"🔰 *Status:* ⚠️ SMTP bot/brute force attempt")
elif rule_id in SQLI_RULES:
    text = (f"💉 *SQL INJECTION ATTEMPT*\n─────────────────────\n"
            f"🖥️ *Agent:* `{agent}` (`{agent_ip}`)\n"
            f"📋 *Rule ID:* `{rule_id}`\n⚡ *Level:* `{alert_level}`\n"
            f"📝 *Description:* {description}\n🌐 *Source IP:* `{src_ip}`\n"
            f"🕐 *Time:* `{timestamp}`\n─────────────────────\n"
            f"🔰 *Status:* ⚠️ SQL Injection pattern detected")
else:
    text = (f"⚠️ *WAZUH ALERT*\n─────────────────────\n"
            f"🖥️ *Agent:* `{agent}` (`{agent_ip}`)\n"
            f"📋 *Rule ID:* `{rule_id}`\n⚡ *Level:* `{alert_level}`\n"
            f"📝 *Description:* {description}\n🌐 *Source IP:* `{src_ip}`\n"
            f"🕐 *Time:* `{timestamp}`\n─────────────────────")

requests.post(hook_url,
    headers={'content-type': 'application/json', 'Accept-Charset': 'UTF-8'},
    json={'chat_id': CHAT_ID, 'text': text, 'parse_mode': 'Markdown'})
sys.exit(0)
```

### Step 6: Set Permissions

```bash
sudo chmod 750 /var/ossec/integrations/custom-telegram
sudo chmod 750 /var/ossec/integrations/custom-telegram.py
sudo chown root:ossec /var/ossec/integrations/custom-telegram
sudo chown root:ossec /var/ossec/integrations/custom-telegram.py
```

### Step 7: Configure ossec.conf

Add inside `<ossec_config>` in `/var/ossec/etc/ossec.conf`:

```xml
<integration>
  <name>custom-telegram</name>
  <hook_url>https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage</hook_url>
  <rule_id>100200,100201,100202,60122,60137,60138,67551,67552,40101,5712,5763,31103,31106</rule_id>
  <alert_format>json</alert_format>
</integration>
```

### Step 8: Restart Wazuh Manager

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

---

## Alert Types

| Alert | Emoji | Rule IDs |
|---|---|---|
| Malware Detected | 🦠 | 100200 |
| Malware Removed | 🗑️ | 100201 |
| FIM Correlation | 📁 | 100202 |
| SSH Brute Force | 🔐 | 5712, 5763, 40101 |
| RDP Attack | 🖥️ | 60122, 60137, 60138 |
| SMTP Attack | 📧 | 67551, 67552, 100032 |
| SQL Injection | 💉 | 31103, 31106 |
| Generic Fallback | ⚠️ | All others |

### Alert Flow

```
Attack / Malware event occurs
        ↓
ClamAV / Suricata detects it
        ↓
VirusEvent script logs to detections.log + removes file
        ↓
Wazuh Agent ships log to Wazuh Manager
        ↓
Custom rule fires (100200 / 100201 / etc.)
        ↓
Wazuh Integration triggers custom-telegram.py
        ↓
Telegram Bot sends categorized alert
```

---

## Testing Procedures

### Test ClamAV — EICAR File (Safe Test Signature)

```bash
# Create test file
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > ~/Downloads/eicar_test.com

# Run scan
sudo /usr/local/bin/clamav-scan.sh

# Verify removal
ls ~/Downloads/

# Check detection log
cat /var/log/clamav/detections.log
```

### Test SSH Brute Force

```bash
sudo apt install hydra -y
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://AGENT_IP -t 4
```

### Test SQL Injection

```bash
sudo apt install sqlmap -y
sqlmap -u "http://AGENT_IP/index.php?id=1" --batch --level=3
# or:
curl "http://AGENT_IP/index.php?id=1' OR 1=1--"
```

### Test RDP Brute Force

```bash
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://AGENT_IP -t 4
```

### Test DDoS Simulation

```bash
sudo apt install hping3 -y
sudo timeout 30 hping3 -S --flood -V -p 80 AGENT_IP
```

### Monitor Alerts in Real Time

```bash
# Terminal 1 — live alerts
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i "malware\|100200\|100201"

# Terminal 2 — integration logs
sudo tail -f /var/ossec/logs/integrations.log

# Terminal 3 — ClamAV detections (on agent)
tail -f /var/log/clamav/detections.log
```

### Manual Telegram Script Test

```bash
cat > /tmp/test_alert.json << 'EOF'
{
  "timestamp": "2026-03-11T10:00:00.000+08:00",
  "rule": { "id": "5712", "description": "SSHD brute force trying to get access to the system.", "level": 10 },
  "agent": { "name": "ubuntu-agent", "ip": "192.168.1.100" },
  "data": { "srcip": "203.0.113.5" },
  "full_log": "Failed password for root from 203.0.113.5 port 4422 ssh2"
}
EOF

sudo /var/ossec/framework/python/bin/python3 \
  /var/ossec/integrations/custom-telegram.py \
  /tmp/test_alert.json "" \
  "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage"
```

---

## Agent Dashboard Features

Each agent dashboard includes: Threat Hunting, File Integrity Monitoring (FIM), Configuration Assessment (SCA), MITRE ATT&CK mapping, and Vulnerability Detection.

| Feature | Purpose |
|---|---|
| Threat Hunting | Search events for brute force, privilege escalation, unusual processes |
| FIM | Detect file create/modify/delete in monitored paths (`/home`, `/tmp`, `C:\Users`) |
| SCA | CIS benchmark compliance scoring with remediation guidance |
| MITRE ATT&CK | Map detections to attacker tactics and techniques |
| Vulnerability Detection | CVE tracking by severity across installed packages |

---
## Screenshots

### Wazuh Dashboard
![Wazuh Dashboard](screenshots/wazuh_login.png)
![Dashboard](screenshots/dashboard2.png)
![Dashboard](screenshots/dashboard3.png)

### Agent Setup
![Creating Agents](screenshots/creating_agents.png)
![Creating Agents 2](screenshots/creating_agents2.png)
![Agent Dashboard](screenshots/Agent_Dashboard.png)

### File Integrity Monitoring (FIM)
![FIM](screenshots/FIM.png)
![FIM 2](screenshots/FIM2.png)
![FIM 3](screenshots/FIM3.png)
![FIM 4](screenshots/FIM4.png)
![FIM 5](screenshots/FIM5.png)

## Troubleshooting

| Problem | Solution |
|---|---|
| Wazuh Manager won't start | Run `sudo /var/ossec/bin/ossec-analysisd -t` to check rule syntax |
| No Telegram alerts | Check `sudo tail -f /var/ossec/logs/integrations.log` |
| Permission denied on scripts | Re-run `chmod 750` and `chown root:ossec` on both integration files |
| `ModuleNotFoundError: requests` | Run `sudo /var/ossec/framework/python/bin/pip3 install requests` |
| Agent not sending logs | Run `sudo systemctl restart wazuh-agent` and verify group config |
| ClamAV log not found | Recreate with `sudo mkdir -p /var/log/clamav && sudo touch /var/log/clamav/detections.log` |
| freshclam locked error | Run `sudo systemctl stop clamav-freshclam && sudo pkill freshclam && sudo freshclam` |
| Telegram API 400 error | Check for unescaped special characters in Markdown message formatting |
| Wrong alert category | Verify the Rule ID is in the correct `_RULES` list in `custom-telegram.py` |

---

## Deployment Checklist

- [ ] Wazuh Manager installed and running
- [ ] Ubuntu and Windows agents connected
- [ ] ClamAV installed and scanning on all agents
- [ ] Agent groups created and configs pushed
- [ ] Custom rules added to `local_rules.xml`
- [ ] Telegram Bot created via @BotFather
- [ ] `requests` library installed in Wazuh's Python
- [ ] Integration scripts created with correct permissions
- [ ] `ossec.conf` updated with `<integration>` block
- [ ] Wazuh Manager restarted
- [ ] EICAR test passed — Telegram alert received
- [ ] SSH brute force test passed
- [ ] SQL injection test passed

---

## Disclaimer

This project was built and tested in an **isolated VirtualBox lab environment** for educational purposes and cybersecurity skill development. Do not use any of the attack simulation tools against systems you do not own or have explicit permission to test.