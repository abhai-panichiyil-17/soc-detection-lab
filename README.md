# SOC Detection Lab — Wazuh SIEM with MITRE ATT&CK Mapping
### Author: Abhai Panichiyil
### Based on: MSc Research — _Implementing Zero Trust Security in Multi-Cloud and Hybrid Cloud Environments_
**Stack**: Wazuh 4.14.4 · Ubuntu 24.04 · Docker · Minikube · Calico

### Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Lab Environment](#lab-environment)
4. [Wazuh Installation](#wazuh-installation)
5. [Attack Simulations](#attack-simulations)
6. [Detections & Alerts](#detections--alerts)
7. [MITRE ATT&CK Mapping](#mitre-attck-mapping)
8. [Results Summary](#results-summary)
9. [Repository Structure](#repository-structure)
10. [Key Concepts Reference](#key-concepts-reference)

---

### Project Overview

This lab demonstrates a Security Operations Centre (SOC) detection capability using Wazuh 4.14.4 deployed on an Ubuntu 24.04 VM. Real attack simulations are executed locally — SSH brute force, privilege escalation, and suspicious file creation — and detected by Wazuh in real time, with alerts automatically mapped to MITRE ATT&CK techniques.

This lab is the detection and visibility layer that complements the K8s Security Lab, which enforces network-level blocking via Calico NetworkPolicies. Together, the two labs implement a dual-layer Zero Trust defence strategy: 

| Layer | Tool | Function |
|-------|------|----------|
| Network | Calico CNI | Blocks lateral movement between Kubernetes namespaces | 
|Host/SIEM | Wazuh | Detects and alerts on attack attempts at the OS level |

The combination demonstrates that Zero Trust is not a single product — it is a layered strategy where blocking and detection work together.

### Architecture
<pre>
┌──────────────────────────────┐
│         Ubuntu 24.04 VM      │
│                              │         
│                              │
│   ┌─────────────────────┐    │
│   │    Wazuh SIEM       │    │   
│   │                     │    │      
│   │  ┌───────────────┐  │    │  
│   │  │   Manager     │  │    │    
│   │  │   Indexer     │  │    │  
│   │  │   Dashboard   │  │    │     
│   │  └───────────────┘  │    │  
│   │                     │    │                        
│   │  Detects:           │    │  
│   │  · Brute force      │    │  
│   │  · Priv escalation  │    │  
│   │  · File changes     │    │
│   └─────────────────────┘    │
│                              │
└──────────────────────────────┘
</pre>

---

### Lab Environment

Tools Used:

| Component | Version | Purpose |
|-----------|---------|---------|
| Ubuntu | 24.04 LTS | Host OS for the entire lab |
| VirtualBox | 7.1.0 | Hypervisor running on Windows host |
| Wazuh | 4.14.4 | SIEM — all-in-one deployment |

Development Environment Specs:

| Component | Host Machine (Physical) | Virtual Machine (Guest) |
| :--- | :--- | :--- |
| OS | Windows 11 | Ubuntu 24.04 LTS |
| CPU | Intel i7-13620H (16 Threads) | 10 vCPUs |
| RAM | 16 GB | 9 GB (9080 MB) |
| Storage | 1 TB NVMe SSD | 500 GB VDI (Dynamic) | 
| GPU | NVIDIA RTX 4060 (8 GB) | VMSVGA (128 MB VRAM) |
| Hypervisor | N/A | VirtualBox 7.1.0 |

Wazuh Components Installed: \
Wazuh was deployed using the all-in-one installer, which provisions three components on a single node:

- Wazuh Manager — the core engine that receives logs, applies detection rules, and generates alerts
- Wazuh Indexer — stores all alert and log data (built on OpenSearch)
- Wazuh Dashboard — web UI for visualising alerts, agent status, and MITRE ATT&CK coverage

---

### Wazuh Installation

Install Command
<pre>
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
</pre>
The _-a_ flag runs the all-in-one installer, provisioning the manager, indexer, and dashboard together. Installation takes approximately 10-15 minutes. The indexer stage can appear to stall for several minutes — this is normal.
Access the Dashboard
<pre> URL:  https://127.0.0.1
</pre>
> Note:
> - Username: admin
> - Password generated at the end of installation

Verify Services Running
<pre>
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
</pre>
All three should show _Active (running)_.

---

### Attack Simulations

All attacks were simulated locally on the Ubuntu VM. The goal is to generate realistic attack telemetry that Wazuh can detect, rather than attacking external systems. \
- Simulation 1 — SSH Brute Force
Technique: T1110 — Brute Force
<pre>
for i in {1..20}; do ssh fakeuser@localhost; done 
</pre>
This loop attempts 20 SSH logins using a non-existent username. Each failed attempt generates an authentication failure log entry in _/var/log/auth.log_, which Wazuh monitors continuously.
What Wazuh sees: Repeated authentication failures from the same source in a short window which is a classic brute force pattern.

- Simulation 2 — Privilege Escalation
Technique: T1068 — Exploitation for Privilege Escalation 
<pre>sudo su 
</pre>    
Switching to root via _sudo_ generates PAM (Pluggable Authentication Module) session events, which Wazuh captures and flags as privilege escalation activity.

- Simulation 3 — Suspicious File Creation
Technique: T1105 — Ingress Tool Transfer 
<pre>sudo touch /etc/backdoor-test.sh 
</pre>
Creating a file in _/etc/_ which is a sensitive system directory triggers Wazuh's file integrity monitoring (syscheck). Wazuh monitors changes to critical directories and alerts when new files appear. 

Cleanup: Remove the test file after the simulation: 
<pre>sudo rm /etc/backdoor-test.sh
</pre>
---

### Detections & Alerts

* SSH Brute Force — Rule 5710 
Wazuh Rule 5710 fires on each attempt to authenticate with a non-existent user: \
Rule 5710: sshd: Attempt to login using non-existent user 

  - Trigger: Each failed SSH login to a non-existent account 
  - Log source: /var/log/auth.log 
  - MITRE mapping: T1110 — Brute Force 
  - Evidence: Alert spike visible in Wazuh dashboard timeline

* Privilege Escalation — Rule 5402 
Rule 5402: Successful sudo to ROOT executed

  - Trigger: sudo su executed successfully 
  - Supporting rules: 5501 (PAM session open), 5502 (PAM session close) 
  - MITRE mapping: T1068 — Exploitation for Privilege Escalation 
  - Evidence: Alert visible in Security Events panel with MITRE tag 

* File Integrity — Syscheck \
Wazuh syscheck: File added to the system

  - Trigger: New file created in monitored directory (/etc/) 
  - Default scan interval: 6 hours (can be forced manually) 
  - MITRE mapping: T1105 — Ingress Tool Transfer 
  - Evidence: Syscheck alert in dashboard with file path and hash 

---

### MITRE ATT&CK Mapping

| Technique ID | Technique Name | Simulation | Wazuh Rule | Result |
|--------------|----------------|------------|-------------|--------|
| T1110 | Brute Force | SSH loop (20 attempts) | Rule 5710 | ✅ Detected |
| T1068 | Privilege Escalation | sudo su | Rule 5402 | ✅ Detected |
| T1105 | Ingress Tool Transfer | /etc/backdoor-test.sh |Syscheck | ✅ Detected |

---

### Results Summary
 
| Simulation | Tool | Rule Fired | MITRE | Outcome |
|------------|-----|-------------|-------|---------|
| SSH brute force (20 attempts) | Wazuh | 5710 | T1110 | ✅ Detected |
| Privilege escalation via sudo | Wazuh | 5402, 5501, 5502 | T1068 | ✅ Detected |
| Suspicious file in /etc/ | Wazuh syscheck | File integrity alert | T1105 | ✅ Detected |

All simulated attack techniques were detected by Wazuh. No technique went unaddressed across the defence.

---

### Repository Structure
<pre>
soc-detection-lab/
├── README.md
├── config/
│   └── ossec.conf               # Wazuh agent/manager config
├── screenshots/
│   ├── dashboard/
│   │   └── overview.png         # Wazuh dashboard overview
│   └── alerts/
│       ├── ssh-brute-force.png  # Rule 5710 alert evidence
│       ├── privilege-escalation.png  # Rule 5402 alert evidence
│       └── mitre-mapping.png    # MITRE ATT&CK coverage view
└── docs/
    └── attack-runbook.md        # Step-by-step attack simulation guide
</pre>

---

### Key Concepts Reference

- What is Wazuh?
Wazuh is an open-source SIEM (Security Information and Event Management) platform. It collects logs from monitored systems, applies detection rules, generates alerts, and maps findings to the MITRE ATT&CK framework. It provides the visibility layer that complements network-level controls like Calico.

- What is SIEM?
A Security Information and Event Management system aggregates log data from across an environment, correlates events, and surfaces suspicious activity as alerts. Without a SIEM, an attacker could operate inside your environment for days without being noticed. Wazuh provides this visibility.

- What is MITRE ATT&CK?
MITRE ATT&CK is a globally recognised framework that categorises adversary tactics and techniques based on real-world observations. Mapping detections to ATT&CK techniques allows security teams to understand not just that something happened, but what stage of an attack it represents.

- Why Wazuh + Calico Together?
Calico prevents unauthorised traffic from reaching its destination as it blocks the attack at the network layer. Wazuh detects that an attack was attempted and it provides visibility at the host layer. A Zero Trust architecture needs both as blocking alone leaves you blind, and detection alone doesn't stop anything.

- What is File Integrity Monitoring (syscheck)?
Wazuh's syscheck module monitors critical directories (like _/etc/_) for changes such as adding new files, modifying and deleting files, and permission changes. If an attacker drops a malicious script or modifies a config file, syscheck catches it and alerts.

---

_Last updated: April 2026
Environment: Wazuh 4.14.4 · Ubuntu 24.04 · Minikube v1.33.x · Calico · VirtualBox 7.1.0_
