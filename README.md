# SOC Detection Lab — Wazuh SIEM with MITRE ATT&CK Mapping
### Author: Abhai Panichiyil
### Based on: MSc Research — _Implementing Zero Trust Security in Multi-Cloud and Hybrid Cloud Environments_
**Stack**: Wazuh 4.14.4 · Ubuntu 24.04 · Docker · Minikube · Calico

### Table of Contents

1. [Project Overview](#)
2. [Zero Trust Defence Architecture](#)
3. [Lab Environment](#)
4. [Wazuh Installation](#)
5. [Attack Simulations](#)
6. [Detections & Alerts](#)
7. [MITRE ATT&CK Mapping](#)
8. [Results Summary](#)
9. [Repository Structure](#)
10. [Key Concepts Reference](#)

---

###Project Overview

This lab demonstrates a Security Operations Centre (SOC) detection capability using Wazuh 4.14.4 deployed on an Ubuntu 24.04 VM. Real attack simulations are executed locally — SSH brute force, privilege escalation, and suspicious file creation — and detected by Wazuh in real time, with alerts automatically mapped to MITRE ATT&CK techniques.

This lab is the detection and visibility layer that complements the K8s Security Lab, which enforces network-level blocking via Calico NetworkPolicies. Together, the two labs implement a dual-layer Zero Trust defence strategy: \
LayerToolFunctionNetworkCalico CNIBlocks lateral movement between Kubernetes namespacesHost/SIEMWazuhDetects and alerts on attack attempts at the OS level
The combination demonstrates that Zero Trust is not a single product — it is a layered strategy where blocking and detection work together.

### Zero Trust Defence Architecture
<pre>
┌──────────────────────────────────────────────────────────┐
│                    Ubuntu 24.04 VM                       │
│                  6GB RAM · 4 CPUs · VirtualBox           │
│                                                          │
│   ┌─────────────────────┐   ┌────────────────────────┐   │
│   │    Wazuh SIEM       │   │   Minikube Cluster     │   │
│   │                     │   │                        │   │
│   │  ┌───────────────┐  │   │  [Nginx]  frontend ns  │   │
│   │  │   Manager     │  │   │     │                  │   │
│   │  │   Indexer     │  │   │  [Flask]  backend ns   │   │
│   │  │   Dashboard   │  │   │     │                  │   │
│   │  └───────────────┘  │   │  [Postgres] database ns│   │
│   │                     │   │                        │   │
│   │  Detects:           │   │  Calico blocks:        │   │
│   │  · Brute force      │   │  · Lateral movement    │   │
│   │  · Priv escalation  │   │  · Unauthorised access │   │
│   │  · File changes     │   │  · Policy violations   │   │
│   └─────────────────────┘   └────────────────────────┘   │
│                                                          │
└──────────────────────────────────────────────────────────┘
</pre>

Attack simulation → Wazuh detects → Calico blocks → Zero Trust enforced

---

### Lab Environment

ComponentVersionPurposeUbuntu24.04 LTSHost OS for the entire labVirtualBox7.1.0Hypervisor running on Windows hostWazuh4.14.4SIEM — all-in-one deploymentMinikubeLatestSingle-node Kubernetes clusterDocker29.3.0Container runtime / Minikube driverCalicoLatestKubernetes CNI — enforces NetworkPolicies
VM Specs: 6GB RAM · 4 CPUs · SSD storage · VMSVGA graphics
Wazuh Components Installed:
Wazuh was deployed using the all-in-one installer, which provisions three components on a single node:

Wazuh Manager — the core engine that receives logs, applies detection rules, and generates alerts
Wazuh Indexer — stores all alert and log data (built on OpenSearch)
Wazuh Dashboard — web UI for visualising alerts, agent status, and MITRE ATT&CK coverage

---

### Wazuh Installation

Install Command
<pre>
bashcurl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
</pre>
The -a flag runs the all-in-one installer, provisioning the manager, indexer, and dashboard together. Installation takes approximately 10-15 minutes. The indexer stage can appear to stall for several minutes — this is normal.
Access the Dashboard
URL:  https://127.0.0.1
User: admin
Pass: (generated at end of install — save immediately)
For access from the Windows host browser, configure port forwarding in VirtualBox:
Host port 8443 → Guest port 443
Verify Services Running
bashsudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
All three should show Active (running).

---

### Attack Simulations

All attacks were simulated locally on the Ubuntu VM. The goal is to generate realistic attack telemetry that Wazuh can detect, rather than attacking external systems. \
- Simulation 1 — SSH Brute Force
Technique: T1110 — Brute Force \
bashfor i in {1..20}; do ssh fakeuser@localhost; done \
This loop attempts 20 SSH logins using a non-existent username. Each failed attempt generates an authentication failure log entry in /var/log/auth.log, which Wazuh monitors continuously.
What Wazuh sees: Repeated authentication failures from the same source in a short window — a classic brute force pattern.
- Simulation 2 — Privilege Escalation
Technique: T1068 — Exploitation for Privilege Escalation \
bashsudo su \
Switching to root via sudo generates PAM (Pluggable Authentication Module) session events, which Wazuh captures and flags as privilege escalation activity.
- Simulation 3 — Suspicious File Creation
Technique: T1105 — Ingress Tool Transfer \
bashsudo touch /etc/backdoor-test.sh \
Creating a file in /etc/ — a sensitive system directory — triggers Wazuh's file integrity monitoring (syscheck). Wazuh monitors changes to critical directories and alerts when new files appear. 

Cleanup: Remove the test file after the simulation: sudo rm /etc/backdoor-test.sh

---

### Detections & Alerts

- SSH Brute Force — Rule 5710 
Wazuh Rule 5710 fires on each attempt to authenticate with a non-existent user: \
Rule 5710: sshd: Attempt to login using non-existent user \

Trigger: Each failed SSH login to a non-existent account \
Log source: /var/log/auth.log \
MITRE mapping: T1110 — Brute Force \
Evidence: Alert spike visible in Wazuh dashboard timeline

- Privilege Escalation — Rule 5402 \
Rule 5402: Successful sudo to ROOT executed

Trigger: sudo su executed successfully \
Supporting rules: 5501 (PAM session open), 5502 (PAM session close) \
MITRE mapping: T1068 — Exploitation for Privilege Escalation \
Evidence: Alert visible in Security Events panel with MITRE tag \

- File Integrity — Syscheck
Wazuh syscheck: File added to the system

Trigger: New file created in monitored directory (/etc/) \
Default scan interval: 6 hours (can be forced manually) \
MITRE mapping: T1105 — Ingress Tool Transfer \
Evidence: Syscheck alert in dashboard with file path and hash \

---

### MITRE ATT&CK Mapping

Technique IDTechnique NameSimulationWazuh RuleResultT1110Brute ForceSSH loop (20 attempts)Rule 5710✅ DetectedT1068Privilege Escalationsudo suRule 5402✅ DetectedT1105Ingress Tool Transfer/etc/backdoor-test.shSyscheck✅ DetectedT1021Lateral Movement: Remote ServicesK8s namespace crossingCalico block✅ Blocked (see K8s lab)T1046Network Service ScanningBlocked connection attemptsCalico timeout✅ Blocked (see K8s lab)T1041Exfiltration Over C2 ChannelDatabase egress deny-allCalico block✅ Blocked (see K8s lab)

The first three techniques are detected by Wazuh at the host level. The last three are blocked by Calico at the network level. Together they demonstrate complementary Zero Trust controls across two layers.

---

###Results Summary
SimulationToolRule FiredMITREOutcomeSSH brute force (20 attempts)Wazuh5710T1110✅ DetectedPrivilege escalation via sudoWazuh5402, 5501, 5502T1068✅ DetectedSuspicious file in /etc/Wazuh syscheckFile integrity alertT1105✅ DetectedFrontend → Database (K8s)CalicoNetworkPolicy blockT1021✅ BlockedBackend → Frontend (K8s)CalicoNetworkPolicy blockT1046✅ BlockedDatabase egressCalicoEgress deny-allT1041✅ Blocked
All simulated attack techniques were either detected by Wazuh or blocked by Calico — no technique went unaddressed across the two-layer defence.

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

_ What is Wazuh?
Wazuh is an open-source SIEM (Security Information and Event Management) platform. It collects logs from monitored systems, applies detection rules, generates alerts, and maps findings to the MITRE ATT&CK framework. It provides the visibility layer that complements network-level controls like Calico.

- What is SIEM?
A Security Information and Event Management system aggregates log data from across an environment, correlates events, and surfaces suspicious activity as alerts. Without a SIEM, an attacker could operate inside your environment for days without being noticed. Wazuh provides this visibility.

- What is MITRE ATT&CK?
MITRE ATT&CK is a globally recognised framework that categorises adversary tactics and techniques based on real-world observations. Mapping detections to ATT&CK techniques allows security teams to understand not just that something happened, but what stage of an attack it represents.

- Why Wazuh + Calico Together?
Calico prevents unauthorised traffic from reaching its destination — it blocks the attack at the network layer. Wazuh detects that an attack was attempted — it provides visibility at the host layer. A Zero Trust architecture needs both: blocking alone leaves you blind, and detection alone doesn't stop anything.

- What is File Integrity Monitoring (syscheck)?
Wazuh's syscheck module monitors critical directories (like /etc/) for changes — new files, modified files, deleted files, and permission changes. If an attacker drops a malicious script or modifies a config file, syscheck catches it and alerts.

---

_Last updated: April 2026
Environment: Wazuh 4.14.4 · Ubuntu 24.04 · Minikube v1.33.x · Calico · VirtualBox 7.1.0_
