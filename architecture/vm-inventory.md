# VM Inventory

## Host Machine

| Component | Specification |
|---|---|
| Model | HP Laptop 15-dw3000 |
| CPU | Intel Core i5-1135G7 (4 cores / 8 threads, 2.4 GHz base, 4.2 GHz boost) |
| RAM | 16 GB DDR4 3200 MT/s dual-channel |
| Storage C: | 475 GB SSD (VMs: Ubuntu Server only) |
| Storage E: | 797 GB HDD (VMs: all others) |
| OS | Windows 11 Home 25H2 |
| Hypervisor | VMware Workstation Pro |

---

## VM Resource Allocation

| VM | vCPU | RAM | Disk | Storage Drive | Network |
|---|---|---|---|---|---|
| pfSense | 1 core / 1 thread | 512 MB | 20 GB | E: HDD | LAN Segment: labnet |
| Windows Server 2022 | 1 core / 2 threads | 2048 MB | 50 GB | E: HDD | LAN Segment: labnet |
| Ubuntu Server | 1 core / 2 threads | 4096 MB | 60 GB | C: SSD | LAN Segment: labnet |
| Windows 10 | 1 core / 2 threads | 2048 MB | 40 GB | E: HDD | LAN Segment: labnet |
| Kali Linux | 1 core / 2 threads | 2048 MB | 40 GB | E: HDD | LAN Segment: labnet |
| Ubuntu Desktop | 1 core / 1 thread | 1536 MB | 30 GB | E: HDD | LAN Segment: labnet |
| **Total VMs** | | **12,288 MB** | **240 GB** | | |
| **Host reserve** | | **4,096 MB** | | | |

Note: Ubuntu Server kept on SSD for Wazuh indexer I/O performance.
All other VMs moved to HDD after host SSD reached critical capacity (16 GB free).
See troubleshooting-log/004.

---

## Software Inventory

### pfSense (192.168.1.1)
- pfSense CE 2.8.1-RELEASE
- Installed with UFS filesystem (ZFS failed due to VM RAM constraint)
- Remote syslog forwarding to 192.168.1.20:514 configured

### Windows Server 2022 (192.168.1.10)
- Windows Server 2022 Standard Evaluation
- Roles: Active Directory Domain Services, DNS Server
- Domain: homelab.lab
- Hostname: LAB-01
- Wazuh agent 4.11.2 installed (agent name: winserver2022-dc)
- Sysmon installed with SwiftOnSecurity config

### Ubuntu Server 24.04 (192.168.1.20)
- Ubuntu Server 24.04.4 LTS
- Hostname: lab-server-01
- Wazuh 4.11.2 all-in-one (manager + indexer + dashboard)
- JVM heap capped at 1 GB to prevent memory exhaustion
- UFW rules: 22, 80, 443, 514/udp, 1514, 1515, 55000

### Windows 10 Evaluation (192.168.1.30)
- Windows 10 Pro Evaluation (10.0.19045)
- Hostname: WINDOWS10DESKTO
- Domain joined: homelab.lab
- Wazuh agent 4.11.2 installed (agent name: Windows10Desktop)
- Sysmon installed with SwiftOnSecurity config
- RDP enabled on port 3389
- Test user: labuser (member of Remote Desktop Users)

### Kali Linux 2026.1 (192.168.1.40)
- Kali Linux 2026.1
- Wazuh agent 4.11.2 installed (agent name: kali-attacker)
- Tools used: Nmap, Hydra, Metasploit, Responder, Atomic Red Team

### Ubuntu Desktop 24.04 (192.168.1.50)
- Ubuntu Desktop 24.04
- Wazuh agent 4.11.2 installed (agent name: ubuntu-desktop)
- Used as analyst workstation for Wazuh dashboard access

---

## Snapshot Policy

Snapshots taken at each major phase milestone:
- Phase1-complete-network-configured
- Phase2-complete-wazuh-agents-deployed
- Phase2-complete-pfSense-syslog-working
- Wazuh-certs-fixed-Oct2026

Snapshots deleted periodically to manage disk space on E: drive.
