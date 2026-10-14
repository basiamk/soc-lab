# Network Topology

## Overview

Fully isolated virtual enterprise network running on VMware Workstation Pro.
pfSense CE 2.8.1 acts as the perimeter firewall and gateway. All VM-to-VM
traffic is routed through pfSense on a VMware LAN Segment (labnet), isolating
the lab completely from the host machine and the broader network.

---

## IP Address Table

| VM | Role | IP Address | OS | Notes |
|---|---|---|---|---|
| pfSense CE 2.8.1 | Firewall / Gateway | 192.168.1.1 | FreeBSD | WAN: VMware NAT, LAN: labnet |
| Windows Server 2022 | Domain Controller | 192.168.1.10 | Windows Server 2022 | DNS server for domain |
| Ubuntu Server 24.04 | Wazuh SIEM | 192.168.1.20 | Ubuntu 24.04 LTS | Wazuh 4.11.2 all-in-one |
| Windows 10 Evaluation | Target endpoint | 192.168.1.30 | Windows 10 | Domain joined: homelab.lab |
| Kali Linux 2026.1 | Attacker | 192.168.1.40 | Kali Linux | Attack simulations |
| Ubuntu Desktop 24.04 | Analyst workstation | 192.168.1.50 | Ubuntu Desktop | Wazuh dashboard access |

**Subnet:** 192.168.1.0/24  
**Default gateway:** 192.168.1.1  
**Primary DNS:** 192.168.1.10 (Domain Controller)  
**Secondary DNS:** 8.8.8.8  
**Active Directory domain:** homelab.lab  
**Domain Controller hostname:** LAB-01.homelab.lab  

---

## Network Segmentation

```
INTERNET
    |
[VMware NAT Adapter]
    |
[pfSense WAN - em0 - DHCP from VMware NAT]
    |
[pfSense LAN - em1 - 192.168.1.1/24]
    |
[VMware LAN Segment: labnet]
    |
    +-- Windows Server 2022   192.168.1.10
    +-- Ubuntu Server         192.168.1.20
    +-- Windows 10            192.168.1.30
    +-- Kali Linux            192.168.1.40
    +-- Ubuntu Desktop        192.168.1.50
```

---

## pfSense Interface Configuration

| Interface | Adapter | Mode | IP |
|---|---|---|---|
| WAN (em0) | VMware NAT | DHCP client | Dynamic (VMware NAT range) |
| LAN (em1) | VMware LAN Segment: labnet | Static | 192.168.1.1/24 |

---

## pfSense Firewall Rules

| Rule | Source | Destination | Action | Purpose |
|---|---|---|---|---|
| LAN allow all | LAN subnet | Any | Pass | Allow all internal traffic |
| Default block | Any | Any | Block | Default deny inbound |

---

## Remote Logging

pfSense forwards all syslog events to Wazuh on UDP 514.

Configuration: Status > System Logs > Settings > Remote Logging  
Remote log server: 192.168.1.20:514  
Contents: Everything  
Verified: tcpdump packet capture confirmed packets arriving at Ubuntu Server

---

## Active Directory Configuration

| Setting | Value |
|---|---|
| Domain name | homelab.lab |
| Domain Controller | LAB-01.homelab.lab |
| DC IP | 192.168.1.10 |
| Domain joined clients | Windows 10 (192.168.1.30) |
| DNS zone | homelab.lab |
| DHCP | Disabled (all static IPs) |

---

## Connectivity Verification

All VMs verified to reach each other via ping and pfSense via HTTPS.
Kali confirmed internet access through pfSense NAT (ping 8.8.8.8).

Verified commands:
```bash
ping -c 2 192.168.1.1    # pfSense
ping -c 2 192.168.1.10   # DC
ping -c 2 192.168.1.20   # Wazuh
ping -c 2 192.168.1.30   # Windows 10
ping -c 2 8.8.8.8        # Internet via pfSense NAT
```

All returned 0% packet loss.
