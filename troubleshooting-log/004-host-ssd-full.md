# Incident 004: Host SSD Full — VM Storage Crisis

**Date:** 2026-05-20  
**Severity:** Critical (all VMs at risk of corruption)  
**Time to resolve:** ~60 minutes  
**Phase:** Phase 3 — Threat Simulation  

---

## Symptom

VMware Workstation displayed a critical error dialog:

```
The operation on the file "C:\Users\Basia\Documents\Virtual Machines\
Ubuntu Server 64-bit\Ubuntu Server 64-bit-000002-s002.vmdk" failed
(There is not enough space on the disk).

The file system where disk resides is full.
```

Ubuntu Server VM paused mid-session. Wazuh dashboard became inaccessible.

Host machine File Explorer showed:

```
Local Disk C:   16.2 GB free of 475 GB   [RED bar]
Local Disk E:   287 GB free of 797 GB     [BLUE bar]
```

---

## Impact

All VMs stored on C: were at risk of disk write failures and potential
corruption. Ubuntu Server paused. Wazuh unavailable. Phase 3 attack
simulations blocked. Host machine general performance degraded due to
near-zero free space on the system drive.

---

## Diagnosis Process

1. Identified disk usage on host C: drive — 475 GB drive with only 16 GB free.

2. Checked total VM storage consumption:
   - All 6 VMs stored on C: drive under Documents\Virtual Machines
   - VMs collectively consuming approximately 200+ GB
   - Ubuntu Server VM had grown due to Wazuh log accumulation
   - Multiple VM snapshots consuming additional space

3. Identified contributing factors:
   - logall=yes had been enabled (see incident 002) causing Wazuh to write
     every event to disk inside the Ubuntu Server VM
   - All VMs on SSD — no separation between OS drive and VM storage
   - Multiple snapshots per VM accumulated over Phase 1 and 2

4. Identified solution: E: drive had 287 GB free — sufficient for all VMs
   except Ubuntu Server which benefits from SSD performance for Wazuh indexer
   I/O.

---

## Root Cause

All VM virtual disk files were stored on the host SSD (C:). Wazuh verbose
logging (logall=yes from incident 002 mitigation period) caused Ubuntu Server
VM disk to grow rapidly. Combined with snapshots from Phases 1 and 2, the
475 GB SSD reached critical capacity.

The underlying architectural mistake was storing all VM files on the system
drive without a storage strategy, and not monitoring host disk usage.

---

## Fix Applied

1. Cancelled the VMware error dialog without retrying — prevented further
   writes to the full disk.

2. Deleted all non-essential snapshots via VMware Snapshot Manager for each VM.

3. Moved 5 of 6 VMs to E: drive using VMware's built-in Move VM function:
   - VM > Manage > Move VM > browse to E:\Virtual Machines\
   - Moved in order: pfSense, Windows Server 2022, Windows 10,
     Kali Linux, Ubuntu Desktop
   - Ubuntu Server kept on C: SSD for Wazuh indexer performance

4. Ran Windows Disk Cleanup on C: drive — recovered additional space from
   temporary files and Windows Update cache.

5. Post-move C: drive status:
   ```
   Local Disk C:   210 GB free of 475 GB   [BLUE bar — healthy]
   ```

6. Cleaned Wazuh logs inside Ubuntu Server VM:
   ```bash
   sudo find /var/ossec/logs -type f -delete
   sudo systemctl restart wazuh-indexer
   sudo systemctl restart wazuh-manager
   sudo systemctl restart wazuh-dashboard
   ```

---

## Verification

Host C: drive recovered from 16 GB to 210 GB free.
Host E: drive: 97 GB free after VM migration.
All VMs booted successfully after migration.
Network connectivity verified — all VMs pinging each other and pfSense.
Wazuh agents reconnected automatically after restart.

---

## Lesson Learned

Storage management is one of the most common operational challenges in running
a SIEM. Production SOC environments address this with:

- Retention policies: keep logs for X days, then delete or archive
- Storage alerts: automated alerts when disk reaches 70%, 80%, 85% full
- Tiered storage: recent logs on fast SSD (hot), older logs on cheap HDD
  (warm), archived logs on object storage (cold)
- Log rotation: compress and rotate logs automatically on a schedule
- Separation of concerns: OS, applications, and log storage on separate volumes

The correct architecture for this lab would have been: host OS on C: SSD,
Ubuntu Server VM on C: SSD for SIEM performance, all other VMs on E: HDD
from day one. Never store all VMs on the system drive without monitoring
available space.

---

## Preventive Measures Implemented

- Weekly log rotation cron job added to Ubuntu Server (see incident 002)
- Ubuntu Server remains on SSD, all other VMs on HDD
- Snapshot policy updated: maximum 2 snapshots per VM at any time
- Host disk space checked at the start of each lab session

---

## References

- VMware VM storage migration: docs.vmware.com
- Wazuh log management: documentation.wazuh.com
