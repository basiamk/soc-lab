# Phase 1 — Lab Infrastructure Setup Notes

## Objective

Build a fully isolated, multi-VM virtual enterprise network that mirrors
a real small-to-medium business environment. Every subsequent phase depends
on this foundation being stable and correctly configured.

---

## Hypervisor Configuration

**VMware Workstation Pro** used as the hypervisor.

Network adapter types used:
- **NAT adapter** on pfSense Adapter 1 — provides internet access for the
  lab via VMware's NAT engine. pfSense treats this as its WAN interface.
- **LAN Segment (labnet)** on pfSense Adapter 2 and all other VMs —
  a completely isolated private network with no direct access to the host
  machine or external networks. All inter-VM traffic is routed through
  pfSense.

The LAN Segment is the VMware equivalent of a physical network switch.
When a VM's adapter is set to LAN Segment: labnet, it is virtually plugging
its ethernet cable into that switch. No traffic from labnet can reach the
internet or the host machine without going through pfSense.

---

## pfSense Installation

- Version: pfSense CE 2.8.1-RELEASE
- Filesystem: UFS (ZFS failed — see troubleshooting-log/001)
- Partition scheme: GPT

pfSense WAN interface (em0): DHCP from VMware NAT
pfSense LAN interface (em1): Static 192.168.1.1/24

pfSense firewall rule added to allow all LAN traffic:
- Action: Pass
- Interface: LAN
- Protocol: Any
- Source: LAN subnet
- Destination: Any

---

## Active Directory Configuration

Windows Server 2022 configured as Domain Controller:
- Domain: homelab.lab
- DC hostname: LAB-01
- IP: 192.168.1.10 (static)
- DNS: points to itself (192.168.1.10)
- Roles installed: AD DS, DNS Server

Windows 10 domain joined:
- Domain: homelab.lab
- Joined via: System Properties > Computer Name > Change > Domain
- Credential used: homelab\administrator

---

## Static IP Configuration

All VMs assigned static IPs. DNS points to DC (192.168.1.10) for domain
resolution, with 8.8.8.8 as secondary for internet resolution.

Ubuntu Server netplan configuration (/etc/netplan/00-installer-config.yaml):
```yaml
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.1.20/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.10, 8.8.8.8]
  version: 2
```

---

## Connectivity Verification

Full ping matrix verified — all VMs reached each other and 8.8.8.8
with 0% packet loss. pfSense accessible via HTTPS on 192.168.1.1.

---

## Phase 1 Milestones

- [x] pfSense installed and routing traffic
- [x] All VMs on labnet LAN segment
- [x] All VMs pinging each other
- [x] Active Directory domain (homelab.lab) operational
- [x] Windows 10 domain joined
- [x] pfSense firewall rules allowing internal traffic
- [x] Network topology documented
- [x] Snapshots taken: Phase1-complete-network-configured
