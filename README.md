# Home SOC Lab — Wazuh SIEM on VirtualBox

**Author:** Dillon Wilson  
**Date:** May 2026  
**Stack:** Wazuh v4.14.5 · Ubuntu 26.04 · Oracle VirtualBox · Nmap  
**Purpose:** Simulate a Tier 1 SOC analyst environment to practice log analysis, intrusion detection, and incident documentation

---

## Project Overview

Built a fully functional home Security Operations Center (SOC) lab using free, open-source tools on a personal Windows machine. The lab consists of a Wazuh SIEM server monitoring a live Ubuntu agent VM, with simulated attacks triggering real alerts mapped to the MITRE ATT&CK framework.

This project demonstrates hands-on skills relevant to:
- SOC Tier 1 analyst roles (alert triage, log analysis, incident documentation)
- Help desk / IT support roles (Active Directory, ticketing, network troubleshooting)
- Entry-level cybersecurity positions requiring practical SIEM experience

---

## Lab Architecture

```
┌─────────────────────────────────────────────┐
│              Windows Host Machine            │
│                                             │
│  ┌──────────────────┐  ┌─────────────────┐  │
│  │  Wazuh Server VM │  │ Agent-Ubuntu VM │  │
│  │  (192.168.56.101)│◄─│ (dill-VirtualBox│  │
│  │                  │  │  192.168.56.x)  │  │
│  │  - Wazuh Manager │  │                 │  │
│  │  - Wazuh Indexer │  │ - Wazuh Agent   │  │
│  │  - Wazuh Dashboard│  │ - Nmap          │  │
│  └──────────────────┘  └─────────────────┘  │
│           │                                  │
│    Browser (https://192.168.56.101)          │
└─────────────────────────────────────────────┘
```

**Network:** VirtualBox Host-Only Adapter (`192.168.56.x` subnet) + NAT adapter for internet access on agent

---

## Tools & Technologies

| Tool | Version | Purpose |
|------|---------|---------|
| Oracle VirtualBox | 7.x | Hypervisor — runs both VMs |
| Wazuh OVA | 4.14.5 | SIEM server, indexer, and dashboard |
| Ubuntu | 26.04 LTS | Agent VM (monitored endpoint) |
| Nmap | 7.98 | Network scanner for attack simulation |
| Python | 3.x | Vulnerability report automation |

---

## Setup Steps

### Phase 1 — Downloaded Required Software
- Oracle VirtualBox from `virtualbox.org`
- Wazuh 4.14.5 OVA from `documentation.wazuh.com` (~13 MB agent package)
- Ubuntu 26.04 Desktop ISO from `ubuntu.com`

### Phase 2 — Deployed Wazuh Server VM
- Imported `wazuh-4.14.5.ova` into VirtualBox
- Allocated **8192 MB RAM** and **4 CPUs**
- Configured networking: Bridged Adapter on host network
- Booted VM and confirmed server IP: `192.168.56.101`
- Accessed Wazuh dashboard at `https://192.168.56.101` from Windows browser

### Phase 3 — Deployed Ubuntu Agent VM
- Created new VM: Ubuntu 26.04 (64-bit), 2 GB RAM, 25 GB disk
- Installed Ubuntu with minimal configuration (no internet during setup)
- Added second network adapter (NAT) for internet access post-install
- Fixed DNS resolution: locked `/etc/resolv.conf` to `8.8.8.8`
- Installed Wazuh agent pointing to server IP:
```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb && \
sudo WAZUH_MANAGER='192.168.56.101' dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent --now
```
- Confirmed agent status: **active (running)**
- Agent appeared in Wazuh dashboard as **Active**

### Phase 4 — Simulated Attacks
Ran three attack simulations and monitored dashboard alerts in real time:

```bash
# 1. Brute-force SSH login simulation
for i in {1..10}; do ssh wronguser@localhost; done

# 2. File integrity monitoring trigger
sudo touch /etc/malicious_test.sh
sudo rm /etc/malicious_test.sh

# 3. Network port scan against Wazuh server
sudo apt install nmap -y
nmap -sV 192.168.56.101
```

---

## Detections & Alerts

Wazuh detected and categorized all simulated attacks automatically:

| Simulation | Alerts Generated | MITRE ATT&CK Tactic | Severity |
|-----------|-----------------|---------------------|----------|
| SSH brute-force (10 failed logins) | 5 Authentication Failures | Credential Access — Password Guessing | Medium |
| File creation/deletion in `/etc/` | Logged by File Integrity Monitor | Defense Evasion | Low–Medium |
| Nmap port scan (`-sV`) | Network scan alerts | Discovery — Network Service Scanning | Medium |
| sudo command usage | 9 Authentication Successes | Privilege Escalation — Sudo/Sudo Caching | Low |

**Total alerts generated: 91**  
**MITRE ATT&CK categories detected: Valid Accounts, Sudo and Sudo Caching, Password Guessing**

**Nmap scan results on Wazuh server:**
```
PORT    STATE  SERVICE  VERSION
22/tcp  open   ssh      OpenSSH 8.7 (protocol 2.0)
443/tcp open   ssl/https
```
Clean result — only expected ports open, no unnecessary attack surface.

---

## Incident Log

### Incident 001 — Brute-Force SSH Attempt
```
Date:         2026-05-21
Time:         ~21:00 EDT
Agent:        dill-VirtualBox (192.168.56.x)
Attack Type:  Brute-force SSH login (10 attempts)
Rule Fired:   Authentication failure alerts
MITRE Tactic: Credential Access — T1110 (Brute Force)
Alert Count:  5 authentication failures
Severity:     Medium
Root Cause:   Simulated — repeated failed SSH login loop
Action Taken: Confirmed detection. In production: block source IP via firewall,
              enforce key-based auth, disable password authentication in sshd_config
Resolved By:  N/A (controlled test)
```

### Incident 002 — Suspicious File Creation in /etc/
```
Date:         2026-05-21
Time:         ~21:00 EDT
Agent:        dill-VirtualBox (192.168.56.x)
Attack Type:  File created and deleted in sensitive directory
Rule Fired:   File Integrity Monitoring (FIM)
MITRE Tactic: Defense Evasion — T1070 (Indicator Removal)
Alert Count:  2 FIM events (create + delete)
Severity:     Medium
Root Cause:   Simulated — touch/rm commands targeting /etc/
Action Taken: Confirmed detection. In production: investigate process that created
              file, check for persistence mechanisms, audit /etc/ permissions
Resolved By:  N/A (controlled test)
```

### Incident 003 — Network Port Scan Detected
```
Date:         2026-05-21
Time:         ~21:53 EDT
Agent:        dill-VirtualBox scanning 192.168.56.101
Attack Type:  Nmap -sV service version scan
Ports Found:  22/tcp (SSH - OpenSSH 8.7), 443/tcp (HTTPS)
MITRE Tactic: Discovery — T1046 (Network Service Scanning)
Severity:     Medium
Root Cause:   Simulated — nmap run from agent VM against Wazuh server
Action Taken: Confirmed detection. In production: identify scan source, check for
              lateral movement, verify no unauthorized ports were exposed
Resolved By:  N/A (controlled test)
```

---

## Lab Screenshots

> Screenshots taken throughout the build and attack simulation phases.

### VM Setup
| | |
|---|---|
| ![Wazuh OVA imported and running in VirtualBox](<screenshots/01_wazuh_vm_running.png>) | 
| *Wazuh v4.14.5 OVA imported and running — 8192MB RAM, 4 CPUs* | 

### Wazuh Dashboard
| | |
|---|---|
| ![Wazuh login screen](<screenshots/03_wazuh_login.png>) | ![Wazuh API connection confirmed](<screenshots/04_wazuh_api.png>) |
| *Wazuh dashboard login at https://192.168.56.101* | *Server API online — Wazuh v4.14.5 confirmed* |

### Agent Connected
| | |
|---|---|
| ![Wazuh agent active running status](<screenshots/05_agent_active.png>) | ![Agent install terminal output](<screenshots/06_agent_install.png>) |
| *Wazuh agent active (running) on Ubuntu — all sub-services confirmed* | *Agent installation: 28.2 MB/s download, clean setup* |

### Attack Simulations & Alerts
| | |
|---|---|
| ![Nmap scan results showing open ports](<screenshots/07_nmap_scan.png>) | ![Wazuh threat hunting dashboard with 91 alerts](<screenshots/08_wazuh_alerts.png>) |
| *Nmap -sV scan against Wazuh server — ports 22 and 443 open* | *Threat Hunting dashboard: 91 total alerts, MITRE ATT&CK mapped* |

### Nmap Install & DNS Fix
| | |
|---|---|
| ![Nmap installing on Ubuntu](<screenshots/09_nmap_install.png>) | |
| *Nmap 7.98 installing on Ubuntu agent after apt update* | |

> **To add screenshots:** Place your PNG files in a `/screenshots/` folder in this repo and rename them to match the filenames above.

---

## 🔧 Troubleshooting Notes

Real issues encountered and resolved during this build — useful for anyone replicating this lab:

**Issue 1: Host-Only adapter not available in VirtualBox**  
*Cause:* Host-Only network not created yet  
*Fix:* File → Tools → Network Manager → Create host-only adapter

**Issue 2: Ubuntu VM had no internet (`Temporary failure in name resolution`)**  
*Cause:* Host-only subnet has no DNS or default route  
*Fix:* Added NAT as second adapter, locked `/etc/resolv.conf` to `8.8.8.8` using `chattr +i`

**Issue 3: `sudo poweroff` blocked by GUI session**  
*Fix:* `sudo poweroff -i` overrides inhibitor

**Issue 4: Snapshot failed (`VERR_DISK_FULL`)**  
*Cause:* Host Windows drive low on space  
*Fix:* Free disk space or move VM files to a larger drive

**Issue 5: `dhclient` not found on Ubuntu 26.04**  
*Fix:* Use `sudo dhcpcd enp0s8` instead (Ubuntu 26.04 uses dhcpcd)

---

## Skills Demonstrated

- **SIEM deployment** — Stood up Wazuh server with indexer and dashboard from OVA
- **Agent configuration** — Installed and configured Wazuh agent on Linux endpoint
- **Attack simulation** — Executed brute-force, FIM, and network scan scenarios
- **Alert triage** — Read and interpreted Wazuh alerts with Rule IDs and MITRE mappings
- **Incident documentation** — Wrote structured incident logs mirroring SOC ticket format
- **Linux administration** — Systemctl, networking, DNS, package management
- **Network analysis** — Nmap service scanning and port analysis
- **Troubleshooting** — Resolved DNS, networking, and VM configuration issues independently

---

## Related Projects

- [Network Vulnerability Scanner + PDF Report](../nmap-report/) — Python script that automates Nmap scanning and generates professional vulnerability reports
- Active Directory Home Lab 

