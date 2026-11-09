# NTP Clock Skew Baseline

**Date recorded:** 2026-10-15  
**Phase:** Phase 1 gap closure (merged roadmap requirement)  

## Configuration

All VMs use pfSense (192.168.1.1) as their NTP source.  
pfSense upstream: 2.pfsense.pool.ntp.org (Stratum 2 pool)  
pfSense stratum: 3  
VM stratum: 4  

## Measured offsets (timedatectl timesync-status output)

| VM | IP | Offset (ms) | Jitter (ms) | Delay (ms) | Recorded |
|---|---|---|---|---|---|
| Ubuntu Server | 192.168.1.20 | +8.003 | 7.482 | 1.314 | 2026-10-15 |
| Kali Linux | 192.168.2.40 | +143.933 (settling) | 0 | 0.743 | 2026-10-15 |
| Windows Server 2022 | 192.168.1.10 | TBD | TBD | TBD | pending |
| Windows 10 | 192.168.1.30 | TBD | TBD | TBD | pending |
| Ubuntu Desktop | 192.168.1.50 | TBD | TBD | TBD | pending |

## Significance

Maximum measured clock skew determines the precision of detection latency 
measurements in Phase 4. A skew of +8ms on Ubuntu Server means detection 
latency figures are accurate to within approximately 8ms. This is acceptable 
for the research questions in this project.

## Method

Each VM configured via /etc/systemd/timesyncd.conf (Linux) or w32tm (Windows).
Verification command: timedatectl timesync-status
