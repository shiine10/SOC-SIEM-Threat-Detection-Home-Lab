# 🛡️ SOC & SIEM Threat Detection Home Lab (Wazuh + Kali Linux + Windows 10)

## 📌 Executive Summary

This project demonstrates the design, deployment, and operation of a virtualized **Security Operations Center (SOC)** and **SIEM (Security Information and Event Management)** environment using **Wazuh**, **Kali Linux**, and **Windows 10**.

The goal of this lab is to simulate real-world cyberattacks, capture endpoint telemetry, monitor security alerts in real time, and conduct threat hunting activities using MITRE ATT&CK framework mapping.

---

## 🏗️ Network & Lab Architecture

```
                       +-------------------------+
                       |   VMware Workstation    |
                       +------------+------------+
                                    |
        +---------------------------+---------------------------+
        |                           |                           |
        v                           v                           v
+------------------+       +------------------+       +-------------------+
|  windows-agent   |       |   wazuh-server   |       |    kali-linux     |
| (Victim Machine) |       |  (SIEM / Manager)|       | (Attacker Machine)|
|  192.168.10.131  |       |  192.168.10.128  |       |   192.168.10.129  |
+------------------+       +------------------+       +-------------------+
```

### 🖥️ Machine Inventory & Network Addressing

| Machine Hostname | Operating System | Role in Lab | IP Address | Subnet Mask | Default Gateway | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `wazuh-server` | Ubuntu Linux / Wazuh | SIEM Manager, Indexer & Dashboard | `192.168.10.128` | `255.255.255.0` | `192.168.10.2` | 
| `windows-wazuh-agent` | Windows 10 Pro | Endpoint Victim / Telemetry Source | `192.168.10.131` | `255.255.255.0` | `192.168.10.2` | 
| `kali-linux` | Kali Linux 2025.3 | Penetration Testing / Attacker | `192.168.10.129` | `255.255.255.0` | `192.168.10.2` | 

---

## ⚙️ Component Setup & Deployment

### 1. Wazuh SIEM Server (`192.168.10.128`)
* Deployed as the central log aggregation manager, indexer, and web interface.
* **Service Status:** Verified running via `systemctl start wazuh-manager` and `systemctl restart wazuh-manager`.
* **Network Listener:** Port `55000` (API) and `1514` (Agent Log Collector).

### 2. Endpoint Agent Configuration (`192.168.10.131`)
* Deployed **Wazuh Windows Agent v4.14.5** on Windows 10 Pro build `10.0.19045`.
* Established continuous telemetry forwarding to manager `192.168.10.128`.
* Verified **Active** status under the Wazuh Agent Management console.

### 3. Attack Workstation (`192.168.10.129`)
* Kali Linux platform configured on the same host-only/NAT subnet (`192.168.10.0/24`) for targeted adversary simulations.

---

## 🔍 Threat Detection Scenarios & Threat Hunting Analysis

### 🚨 Scenario 1: Pass-the-Hash / RDP Remote Authentication Detection
* **Objective:** Detect lateral movement and unauthorized NTLM/RDP logon attempts.
* **Attack Execution (Red Team):** Simulated remote authentication from Kali Linux (`192.168.10.129`) targeting the Windows 10 Administrator account (`192.168.10.131`).
* **SIEM Telemetry & Rule Matching (Blue Team):**
  * **Wazuh Rule ID:** `92657`
  * **Alert Severity Level:** Level 6
  * **Description:** *Successful Remote Logon Detected - User: \Admin - NTLM authentication, possible pass-the-hash attack - Possible RDP connection.*
* **MITRE ATT&CK Mapping:**
  * **Tactic:** Defense Evasion, Lateral Movement, Persistence, Privilege Escalation, Initial Access
  * **Techniques:** `T1550.002` (Pass the Hash), `T1078.002` (Domain Accounts), `T1021.001` (Remote Desktop Protocol)

---

### 🚨 Scenario 2: Automated RDP Brute-Force & Credential Access Simulation
* **Objective:** Detect dictionary attacks targeting endpoint user accounts and trigger high-severity SIEM alerts upon multiple authentication failures.
* **Attack Execution (Red Team):** Executed a targeted password brute-force attack from Kali Linux using `Hydra` against the Windows 10 host:
  ```bash
  hydra -l Admin -P /usr/share/wordlists/rockyou.txt rdp://192.168.10.131 -t 4
  ```
* **SIEM Telemetry & Rule Matching (Blue Team):**
  * **Event Log Captured:** Windows Event ID `4625` (An account failed to log on) & Event ID `4740` (User account locked out).
  * **Wazuh Rule IDs:** `60115` (Multiple failed logons) and `60204` (Account Lockout detected).
  * **Alert Severity Level:** Level 10 (High Priority Alert).
* **MITRE ATT&CK Mapping:**
  * **Tactic:** Credential Access (`TA0006`)
  * **Technique:** `T1110.001` (Brute Force: Password Guessing)
* **SOC Analyst Triage & Response:**
  1. Identify source IP (`192.168.10.129`) and lock out target host interface if external.
  2. Verify if any subsequent successful logon event (Windows Event ID `4624`) occurred from the same IP.
  3. Initiate host isolation protocol if unauthorized access is confirmed.

---

### 🚨 Scenario 3: Suspicious Obfuscated PowerShell Execution (LOLBins / Living off the Land)
* **Objective:** Detect malicious process creation and obfuscated command-line arguments utilizing native Windows tools (PowerShell).
* **Attack Execution (Red Team):** Executed a Base64 encoded PowerShell string on the Windows 10 host to simulate stager download / evasion:
  ```powershell
  powershell -ExecutionPolicy Bypass -EncodedCommand QnkgTWFsaWNpb3VzIENvbW1hbmQ=
  ```
* **SIEM Telemetry & Rule Matching (Blue Team):**
  * **Event Log Captured:** Sysmon Event ID `1` (Process Creation) and PowerShell Event ID `4104` (Script Block Logging).
  * **Wazuh Rule ID:** `91802` (PowerShell encoded command line detected).
  * **Alert Severity Level:** Level 8 (Suspicious Activity).
* **MITRE ATT&CK Mapping:**
  * **Tactic:** Execution (`TA0002`), Defense Evasion (`TA0005`)
  * **Techniques:** `T1059.001` (Command and Scripting Interpreter: PowerShell), `T1027` (Obfuscated Files or Information)
* **SOC Analyst Triage & Response:**
  1. Extract Base64 encoded payload and decode using CyberChef / terminal to inspect the underlying code.
  2. Inspect parent process (`parent_command_line`) to trace how PowerShell was invoked (e.g., CMD, MS Office macro, or browser download).
  3. Perform endpoint memory scan using EDR/AV tool.

---

## 🧪 Future Testing Roadmap & Expansion (In Progress)

This lab is built to be modular. Additional scenarios being added to this environment include:

* [ ] **Scenario 4: Mimikatz LSASS Memory Dumping**
  * Tracking Sysmon Event ID 10 (ProcessAccess) alerts for unauthorized LSASS memory reading.
* [ ] **Scenario 5: Active Directory & Kerberoasting Detection**
  * Integrating Windows Server 2019 Active Directory to monitor TGS-REQ ticket requests for service accounts.
* [ ] **Scenario 6: Automated Active Response Rules**
  * Configuring Wazuh Active Response scripts to automatically block attacking IP addresses (`192.168.10.129`) at the Windows Firewall level after 5 failed login attempts.

---

## 🛠️ How to Replicate This Lab

1. **Hypervisor Setup:** Import 3 Virtual Machines into VMware Workstation.
2. **Network Configuration:** Set all VMs to the same virtual network adapter (`VMnet8` or host-only). Verify connectivity using `ip a` and `ipconfig`.
3. **SIEM Installation:** Deploy the Wazuh All-in-One image on Ubuntu Linux.
4. **Agent Deployment:** Install the Wazuh MSI package on Windows 10, target manager IP `192.168.10.128`, and start the service.
5. **Simulate Attack:** Execute tests from Kali Linux against Windows 10.
6. **Analyze Logs:** Open `http://192.168.10.128` in your browser and monitor the **Threat Hunting** and **Events** dashboards.

---

## 📄 Core Competencies Demonstrated

* **SIEM Administration:** Wazuh server deployment, custom rule tuning, Sysmon integration, agent management.
* **Threat Hunting & Incident Response:** Event log correlation, rule matching, MITRE ATT&CK taxonomy, triage workflows.
* **Red/Blue Teaming:** Attack simulation (Hydra, Metasploit, PowerShell obfuscation) and endpoint defense verification.
