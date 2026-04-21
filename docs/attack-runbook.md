# Attack Simulation Runbook

## Purpose
Step-by-step commands used to generate attack telemetry for Wazuh detection testing.
All simulations run locally on the Ubuntu 24.04 VM.

---

## Simulation 1 — SSH Brute Force
**MITRE:** T1110 — Brute Force
**Expected Rule:** 5710

```bash
for i in {1..20}; do ssh fakeuser@localhost; done
```

**Expected result:** Rule 5710 fires for each attempt.
Check dashboard: Security Events → filter rule.id:5710

---

## Simulation 2 — Privilege Escalation
**MITRE:** T1068 — Privilege Escalation
**Expected Rule:** 5402

```bash
sudo su
```

**Expected result:** Rules 5402, 5501, 5502 fire.
Check dashboard: Security Events → filter rule.id:5402

---

## Simulation 3 — Suspicious File Creation
**MITRE:** T1105 — Ingress Tool Transfer
**Expected:** Syscheck alert

```bash
sudo touch /etc/backdoor-test.sh
```

**Cleanup after test:**
```bash
sudo rm /etc/backdoor-test.sh
```

**Expected result:** Syscheck file integrity alert fires.
Check dashboard: Integrity Monitoring panel

---

## Force Syscheck Scan (default interval is 6 hours)

```bash
sudo /var/ossec/bin/agent_control -r -u 000
```
