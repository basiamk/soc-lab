# Incident 002: Wazuh Manager Memory Exhaustion

**Date:** 2026-05-20  
**Severity:** High (SIEM completely unavailable)  
**Time to resolve:** ~30 minutes  
**Phase:** Phase 2 / Phase 3 boundary  

---

## Symptom

Wazuh manager failed to restart with timeout error:

```
Job for wazuh-manager.service failed because a timeout was exceeded.
See "systemctl status wazuh-manager.service" for details.
```

Status output showed:
```
Active: failed (Result: timeout)
Memory: 2.8G (peak: 3.1G)
Tasks: 225
```

The manager had consumed 2.8 GB of the 4 GB available on Ubuntu Server,
leaving insufficient memory to restart cleanly.

---

## Impact

Wazuh manager failed to start. No alerts were being processed. The dashboard
was accessible but showed no new events. All Phase 3 attack simulations were
blocked — there was no functioning SIEM to detect against.

---

## Diagnosis Process

1. Checked service status — confirmed memory consumption at 2.8 GB peak.

2. Checked what was consuming disk space (root cause of high memory was
   the archives subsystem):

   ```bash
   sudo du -sh /var/ossec/logs/*
   ```

   Archives directory had grown significantly due to logall=yes.

3. Connected the timeline: logall=yes had been enabled earlier to verify
   pfSense syslog was arriving. This caused Wazuh to store every single
   event rather than only alerts. Combined with pfSense forwarding all
   syslog traffic, the volume overwhelmed the indexer.

---

## Root Cause

logall=yes in ossec.conf caused Wazuh to archive every single event to disk
and index everything in the OpenSearch indexer. pfSense configured to forward
"Everything" via syslog generated a high volume of events. The combination
caused the Wazuh indexer JVM heap to grow beyond available RAM, eventually
preventing the manager from restarting.

---

## Fix Applied

1. Stopped all Wazuh services:
   ```bash
   sudo systemctl stop wazuh-manager wazuh-indexer wazuh-dashboard
   sleep 5
   ```

2. Disabled logall:
   ```bash
   sudo sed -i 's/<logall>yes<\/logall>/<logall>no<\/logall>/' \
   /var/ossec/etc/ossec.conf
   sudo sed -i 's/<logall_json>yes<\/logall_json>/<logall_json>no<\/logall_json>/' \
   /var/ossec/etc/ossec.conf
   ```

3. Cleared accumulated archive files:
   ```bash
   sudo find /var/ossec/logs/archives -name "*.log" -delete
   sudo find /var/ossec/logs/archives -name "*.json" -delete
   ```

4. Capped JVM heap in /etc/wazuh-indexer/jvm.options:
   ```
   -Xms1g
   -Xmx1g
   ```

5. Added weekly log rotation cron job:
   ```
   0 2 * * 0 find /var/ossec/logs -name "*.log" -mtime +7 -delete
   0 2 * * 0 find /var/ossec/logs -name "*.json" -mtime +7 -delete
   ```

6. Started services in correct order:
   ```bash
   sudo systemctl start wazuh-indexer
   sleep 20
   sudo systemctl start wazuh-manager
   sleep 15
   sudo systemctl start wazuh-dashboard
   ```

---

## Verification

```
free -h:
  Mem: 4.1Gi total, 2.6Gi used, 1.5Gi available
```

All three services showing active (running). Available RAM healthy at 1.5 GB.

---

## Lesson Learned

SIEM storage management is a significant operational challenge in production
environments. Enterprise SIEMs implement retention policies, storage alerts,
and tiered storage (hot/warm/cold) specifically to prevent this. logall is
useful for short debugging sessions but must never be left enabled in a
resource-constrained environment. The correct approach is alert-based logging:
store only events that match detection rules plus high-severity events.

---

## References

- Wazuh ossec.conf reference: documentation.wazuh.com
- OpenSearch JVM configuration: opensearch.org/docs
