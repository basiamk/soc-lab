# Incident 001: pfSense ZFS Installation Failure

**Date:** 2026-04-27  
**Severity:** High (blocked lab setup)  
**Time to resolve:** ~45 minutes  
**Phase:** Phase 1 — Lab Infrastructure  

---

## Symptom

pfSense CE 2.8.1 installer failed with the error:

```
Installation has failed! The system is NOT installed.
Please check the installation log in /tmp/install-log.txt.
```

This occurred after selecting ZFS as the filesystem and 2.8.1 as the version.
The installer downloaded all 182 packages successfully but crashed during the
final integrity verification step.

On first boot after the failure attempt, the system dropped to a Mountroot>
prompt:

```
Mountroot> zfs:zroot/ROOT/default
Trying to mount root from zfs:zroot/ROOT/default []...
Mounting from zfs:zroot/ROOT/default failed with error 2: unknown file system.
```

---

## Impact

pfSense could not be installed. Lab network could not be configured.
All subsequent phases blocked.

---

## Diagnosis Process

1. Booted into installer shell and read the installation log:

   ```bash
   cat /tmp/install-log.txt
   ```

   Key line:
   ```
   Checking integrity...Child process pid=3509 terminated abnormally: Killed
   ```

2. The word "Killed" indicates the OS killed the process due to memory
   exhaustion — the OOM (Out of Memory) killer terminated the integrity check
   process because RAM ran out.

3. Checked VM RAM allocation in VMware: the pfSense VM had only 512 MB RAM.
   pfSense 2.8.1's integrity check during ZFS installation requires more RAM
   than this to complete.

4. Attempted ZFS Mountroot fix — confirmed ZFS pool was never written because
   the installation failed before completing.

---

## Root Cause

Insufficient VM RAM. The ZFS installation integrity check process was killed
by the Linux OOM killer when RAM was exhausted. pfSense 2.8.1 requires at
minimum 1 GB RAM for a clean ZFS installation. The pfSense VM had only 512 MB.

---

## Fix Applied

Two changes made:

1. Increased pfSense VM RAM to 1024 MB in VMware settings:
   VM > Settings > Memory > 1024 MB

2. Changed filesystem from ZFS to UFS during reinstallation. UFS is simpler,
   has no RAM overhead during installation integrity checks, and is fully
   sufficient for a lab firewall.

   At the Installation Options screen: pressed F to change File System from
   ZFS to UFS, then continued.

---

## Verification

pfSense installed successfully with UFS. On reboot (with ISO disconnected),
the system booted directly to the pfSense console menu showing:

```
WAN (wan) -> em0 -> v4/DHCP4: 192.168.177.133/24
LAN (lan) -> em1 -> v4: 192.168.1.1/24
```

Both interfaces assigned correctly.

---

## Lesson Learned

ZFS has higher RAM requirements than other filesystems, particularly during
installation integrity verification. In constrained VM environments, UFS is
the practical choice for a lab firewall — it provides all required
functionality with minimal overhead. Always check minimum system requirements
before selecting filesystem options during OS installation.

---

## References

- pfSense CE hardware requirements: docs.netgate.com/pfsense/en/latest
- ZFS vs UFS for pfSense: Netgate documentation
