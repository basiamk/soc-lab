# Phase 5 — SOAR Automation & POPIA Incident Response

**Status:** Pending (starts after Phase 4 completes)

---

## Objective

Connect Wazuh to automated response actions via TheHive and Python playbooks.
Build a complete POPIA-aligned incident response workflow including a Python
script that generates a draft breach notification document.

---

## Planned Deliverables

### TheHive Integration
- Deploy TheHive 5 on Ubuntu Server via Docker
- Configure Wazuh to auto-create TheHive cases on high-severity alerts
- Test: trigger alert > verify case created automatically

### Python Playbooks
- Playbook 1: on brute force alert, query AbuseIPDB for IP reputation,
  append result to TheHive case
- Playbook 2: on malware alert, submit hash to VirusTotal API, attach
  report to case
- Playbook 3: pfSense API integration to auto-block malicious IPs

### POPIA Incident Response Playbook
Five-step playbook aligned to South African legal requirements:
1. Detection: how breach was discovered
2. Containment: immediate actions taken
3. Assessment: what personal information was affected, how many data subjects
4. Notification: draft letter to Information Regulator within 72 hours
5. Remediation: what was fixed

### POPIA Breach Notification Script
Python script that takes incident details as input and generates a
professionally formatted PDF breach notification document, referencing
POPIA Section 22 and the 72-hour reporting requirement under the
Cybercrimes Act 2021.

---

## SA Regulatory References

- POPIA Section 22: Notification of security compromises
- Information Regulator: justice.gov.za/inforeg
- Cybercrimes Act 2021: 72-hour breach notification obligation
- Joint Standard 2 (FSCA/PA June 2025): mandatory cybersecurity governance
  for banks, insurers, and pension funds

---

## Tools

- TheHive 5: docs.strangebee.com
- Cortex: github.com/TheHive-Project/Cortex
- Python requests library: docs.python-requests.org
- AbuseIPDB API: abuseipdb.com
- VirusTotal API: virustotal.com
- Shuffle SOAR (alternative): shuffler.io
