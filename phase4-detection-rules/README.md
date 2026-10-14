# Phase 4 — Custom Detection Rules

**Status:** Pending (starts after Phase 3 completes)

---

## Objective

Write custom Sigma and Wazuh XML detection rules for every gap identified
in Phase 3. Map all rules to MITRE ATT&CK technique IDs. Add POPIA-specific
rules for South African regulatory compliance.

---

## Planned Rules

| # | ATT&CK ID | Rule Name | Format | Status |
|---|---|---|---|---|
| 1 | T1046 | Internal port scan detection | Sigma + Wazuh XML | Pending |
| 2 | T1110 | RDP brute force threshold | Sigma + Wazuh XML | Pending |
| 3 | T1059.001 | Suspicious PowerShell execution | Sigma + Wazuh XML | Pending |
| 4 | T1003.001 | LSASS process access | Sigma + Wazuh XML | Pending |
| 5 | T1078 | After-hours privileged login | Wazuh XML | Pending |
| 6 | POPIA | Bulk file access (50+ files, 10 seconds) | Wazuh XML | Pending |
| 7 | POPIA | Mass data export detection | Wazuh XML | Pending |
| 8 | POPIA | After-hours admin login | Wazuh XML | Pending |
| 9 | T1486 | Ransomware file rename pattern | Sigma + Wazuh XML | Pending |
| 10 | T1557 | LLMNR poisoning indicator | Sigma + Wazuh XML | Pending |

---

## Tools

- Sigma: github.com/SigmaHQ/sigma
- pySigma: pip install pySigma pySigma-backend-elasticsearch
- Wazuh custom rules: /var/ossec/etc/rules/local_rules.xml
- YARA: for file-based malware pattern detection

---

## Reference

- SigmaHQ rule library: github.com/SigmaHQ/sigma/tree/master/rules
- Florian Roth (Neo23x0): github.com/Neo23x0
- Wazuh custom rules guide: documentation.wazuh.com/current/user-manual/ruleset/custom
