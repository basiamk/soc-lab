# SOC Automation & Purple Team Lab

## Network Topology
| VM | Role | IP |
|---|---|---|
| pfSense | Firewall/Gateway | 192.168.1.1 |
| Windows Server 2022 | Domain Controller | 192.168.1.10 |
| Ubuntu Server | SIEM (Wazuh) | 192.168.1.20 |
| Windows 10 Eva | Target | 192.168.1.30 |
| Kali Linux | Attacker | 192.168.1.40 |
| Ubuntu Desktop | Analyst | 192.168.1.50 |

## Phase 1 - Complete ✅
- Deployed pfSense firewall with NAT/LAN separation
- Configured isolated labnet LAN segment
- All VMs communicating through pfSense
- Active Directory domain configured on Windows Server 2022
