# Incident 006: Network Segmentation — Kali Moved to Dedicated ATTACKNET Segment

**Date:** 2026-10-14  
**Severity:** Improvement (planned architectural change)  
**Time to implement:** ~60 minutes  
**Phase:** Phase 3 — Threat Simulation  

---

## Background

This is not a break-fix incident. It is a deliberate architectural improvement
identified during Phase 3 setup. It is documented here because the
implementation involved real troubleshooting steps and because the decision
and its outcomes are worth preserving for reference.

---

## Problem Statement

In the original lab topology, Kali Linux (the attacker VM) sat on the same
VMware LAN Segment (labnet) as all target VMs. This meant attack traffic
between Kali and Windows 10 never crossed pfSense. Consequences:

- pfSense firewall logs contained no record of attack simulations
- Network-layer and endpoint-layer events could not be correlated in Wazuh
- pfSense IDS/IPS rules could not be tested against attack traffic
- Firewall rules could not be used as a security control between attacker
  and targets
- The lab did not reflect how real enterprise networks segment untrusted
  or potentially compromised hosts

The network diagram flagged this as a known limitation. Rather than
accepting it, the decision was made to implement proper segmentation before
continuing Phase 3 attack simulations. A lab that cuts corners teaches
corner-cutting habits.

---

## Impact Before Fix

Every attack simulation run from Kali generated endpoint telemetry only.
pfSense had no visibility on attack traffic. Wazuh could not correlate a
network-layer event with an endpoint event for the same attack. The lab
was less realistic than it could be and several planned Phase 3 techniques
(T1557 LLMNR Poisoning, Suricata IDS testing) were not possible in the
original topology.

---

## Changes Made

### 1. New VMware LAN Segment created

A second LAN Segment named `attacknet` created in VMware Workstation:
- Edit > Virtual Network Editor > Add Network
- Type: Host-only (internal, no DHCP, no host access)
- Name: attacknet

### 2. Third network adapter added to pfSense

pfSense VM powered off. Third adapter added:
- VM > Settings > Add > Network Adapter
- LAN Segment: attacknet
- pfSense powered back on — detected as em2

### 3. OPT1 interface configured on pfSense

Interfaces > Assignments > em2 assigned as OPT1.

OPT1 configuration:
- Description: ATTACKNET
- IPv4 type: Static
- IPv4 address: 192.168.2.1/24

### 4. Five firewall rules added on ATTACKNET interface

Rules added in this exact order (pfSense evaluates top-down, first match wins):

| Order | Action | Protocol | Source | Destination | Port | Description |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP | 192.168.2.40 | 192.168.1.20 | 1514 | Kali Wazuh agent to manager |
| 2 | Pass | Any | 192.168.2.40 | 192.168.1.30 | Any | Kali attacks to Windows 10 target |
| 3 | Pass | TCP | 192.168.2.40 | 192.168.1.20 | 443 | Kali to Wazuh dashboard HTTPS |
| 4 | Block | Any | 192.168.2.40 | 192.168.1.10 | Any | Block Kali direct DC access |
| 5 | Pass | Any | 192.168.2.0/24 | Any | Any | ATTACKNET outbound internet |

Rule 4 must sit above Rule 5. If Rule 5 were first, pfSense would match
it before reaching the Block rule and allow Kali to reach the DC.

### 5. Kali moved to attacknet segment

Kali VM powered off. Network adapter changed from LAN Segment: labnet to
LAN Segment: attacknet. Kali powered back on.

### 6. Kali static IP reconfigured

```bash
nmcli connection modify "Wired connection 1" \
  ipv4.addresses 192.168.2.40/24 \
  ipv4.gateway 192.168.2.1 \
  ipv4.dns "192.168.1.10,8.8.8.8" \
  ipv4.method manual

nmcli connection up "Wired connection 1"
```

---

## Troubleshooting Encountered

### Windows 10 not responding to ping from new segment

After reconfiguration, ping from Kali (192.168.2.40) to Windows 10
(192.168.1.30) returned 100% packet loss despite pfSense Rule 2 allowing
all traffic between them.

Root cause: Windows Defender Firewall treats traffic from 192.168.2.x as
coming from a different network zone than 192.168.1.x. The ICMP inbound
rule was scoped to the domain/private profile only, which did not cover
the new source subnet.

Fix: Enabled the built-in ICMP inbound rule via Windows Defender Firewall
with Advanced Security:
- Inbound Rules > File and Printer Sharing (Echo Request - ICMPv4-In)
- Enabled both domain and private profile entries

This is a realistic scenario — a SOC analyst reconfiguring network
segments will encounter Windows Firewall zone behaviour as a consequence.

---

## Verification

Full connectivity test run from Kali after all changes:

```
ping 192.168.2.1   — 0% packet loss  (pfSense OPT1 gateway reachable)
ping 192.168.1.30  — 0% packet loss  (Windows 10 reachable via pfSense)
ping 192.168.1.20  — 0% packet loss  (Wazuh reachable via pfSense)
ping 192.168.1.10  — 100% packet loss (DC blocked by Rule 4 as intended)
ping 8.8.8.8       — 0% packet loss  (internet reachable via pfSense NAT)
```

Port scan confirmed Windows 10 attack surface accessible:

```
nmap -p 3389,445,135 192.168.1.30
PORT     STATE    SERVICE
135/tcp  open     msrpc
445/tcp  filtered microsoft-ds
3389/tcp open     ms-wbt-server
```

Traceroute confirmed attack traffic now crosses pfSense:

```
traceroute 192.168.1.30
1  192.168.2.1 (192.168.2.1)  — pfSense ✅
2  192.168.1.30               — Windows 10 (does not respond to TTL-expired)
```

Hop 1 being pfSense confirms all attack traffic crosses the firewall before
reaching targets. The remaining hops showing * * * is expected — Windows 10
does not respond to TTL-exceeded ICMP messages by default.

Wazuh agent on Kali confirmed active after IP change:

```bash
sudo systemctl status wazuh-agent | grep Active
# Active: active (running)
```

Agent verified connected on Wazuh manager:

```bash
sudo /var/ossec/bin/agent_control -l
# kali-attacker: Active
```

---

## Updated Network Topology

| VM | Role | Segment | IP |
|---|---|---|---|
| pfSense LAN | Gateway to labnet | labnet | 192.168.1.1 |
| pfSense OPT1 | Gateway to attacknet | attacknet | 192.168.2.1 |
| Windows Server 2022 | Domain Controller | labnet | 192.168.1.10 |
| Ubuntu Server | Wazuh SIEM | labnet | 192.168.1.20 |
| Windows 10 | Target | labnet | 192.168.1.30 |
| Ubuntu Desktop | Analyst workstation | labnet | 192.168.1.50 |
| Kali Linux | Attacker | attacknet | 192.168.2.40 |

---

## What This Enables

**Cross-source alert correlation:** A single attack now generates a pfSense
firewall log entry AND a Windows endpoint event in Wazuh simultaneously.
Correlating these two data sources for the same attack is a core threat
hunting skill.

**Firewall as a security control:** pfSense Rule 4 demonstrates network
segmentation in practice. Kali cannot reach the Domain Controller directly.
Any technique that requires DC access (BloodHound AD enumeration, Kerberoasting
from Kali) must either go through an allowed path or be blocked — exactly
as a real enterprise firewall would behave.

**Suricata IDS testing:** Attack traffic now crosses pfSense, meaning
Suricata can be installed on the OPT1 interface to provide network-layer
intrusion detection on top of Wazuh endpoint detection. This is the
architecture used in real enterprise SOCs — NIDS at the network layer,
SIEM at the endpoint layer.

**Realistic lateral movement simulation:** An attacker who has compromised
Kali still has to navigate firewall rules to reach other targets. Techniques
that assume flat network access will behave differently in this environment,
which is closer to what a real post-breach scenario looks like.

---

## Lesson Learned

Network segmentation is not just a security recommendation — it is a
detection multiplier. Moving the attacker to a separate segment did not
just improve security posture, it fundamentally changed what the SIEM can
see. In production environments, network architecture decisions made by
infrastructure teams directly determine what a SOC can detect. A SOC
analyst who understands network segmentation can have a meaningful
conversation with network engineers about detection gaps caused by flat
network design.

The Windows Firewall zone behaviour encountered during this change is also
a realistic lesson. Reconfiguring network segments without updating host
firewall rules is a common source of connectivity failures in enterprise
environments after network changes.

---

## References

- pfSense firewall rules documentation: docs.netgate.com/pfsense/en/latest
- VMware LAN Segment configuration: docs.vmware.com
- Windows Defender Firewall ICMP rules: Microsoft documentation
