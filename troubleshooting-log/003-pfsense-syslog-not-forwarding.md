# Incident 003: pfSense Syslog Not Forwarding to Wazuh

**Date:** 2026-04-29  
**Severity:** Medium (visibility gap — firewall events missing from SIEM)  
**Time to resolve:** ~40 minutes  
**Phase:** Phase 2 — SIEM and Log Pipeline  

---

## Symptom

pfSense configured to forward syslog to 192.168.1.20:514 but no pfSense
events appeared in Wazuh Threat Hunting. Grep for pfSense events in archives
returned nothing:

```bash
sudo grep "192.168.1.1" /var/ossec/logs/archives/archives.log
# No output
```

tcpdump on Ubuntu Server showed 0 packets on UDP 514:

```bash
sudo tcpdump -i ens33 udp port 514 -n
# 0 packets captured
```

---

## Impact

pfSense firewall events — blocked connections, allowed traffic, DHCP leases,
authentication events — were not reaching Wazuh. Network perimeter visibility
was absent from the SIEM.

---

## Diagnosis Process

1. Verified pfSense remote logging configuration was correct in the GUI:
   - Enable Remote Logging: checked
   - Source Address: LAN
   - Remote log server: 192.168.1.20:514
   - Remote Syslog Contents: Everything

   Configuration looked correct.

2. Ran tcpdump on Ubuntu Server to verify packets were arriving:
   ```bash
   sudo tcpdump -i ens33 udp port 514 -n -v
   ```
   0 packets captured — pfSense was not sending.

3. Checked UFW status on Ubuntu Server:
   ```bash
   sudo ufw status
   ```
   Port 514/udp was ALLOW. Firewall was not blocking.

4. Checked whether Wazuh was listening on UDP 514:
   ```bash
   sudo ss -ulnp | grep 514
   # No output
   ```
   Wazuh was not listening on port 514 — the remote syslog listener was
   missing from ossec.conf.

5. Checked ossec.conf remote block — only the agent communication listener
   (port 1514, secure connection) was present. No syslog listener existed.

6. After adding the syslog listener and restarting Wazuh, ran tcpdump again
   — still 0 packets. pfSense syslogd had cached the old configuration.

7. Rebooted pfSense — immediately after reboot tcpdump showed packets:
   ```
   192.168.1.1:514 > 192.168.1.20:514: SYSLOG local0.info
   ```

---

## Root Cause

Two separate issues combined:

1. Wazuh ossec.conf was missing a syslog remote listener block. The existing
   remote block handled agent communication (port 1514, secure protocol) but
   did not listen for UDP syslog on port 514. Without this block Wazuh ignores
   incoming syslog packets even if UFW allows them.

2. pfSense's syslogd daemon had the old configuration cached in memory. Even
   after saving the remote logging settings in the GUI, syslogd needed a
   restart to pick up the new destination. A full pfSense reboot was the
   cleanest way to restart syslogd.

---

## Fix Applied

1. Added syslog remote listener to ossec.conf inside the first ossec_config
   block, after the existing secure remote block:

   ```xml
   <remote>
     <connection>syslog</connection>
     <port>514</port>
     <protocol>udp</protocol>
     <allowed-ips>192.168.1.0/24</allowed-ips>
   </remote>
   ```

2. Restarted Wazuh manager:
   ```bash
   sudo systemctl restart wazuh-manager
   ```

3. Verified Wazuh now listening on UDP 514:
   ```bash
   sudo ss -ulnp | grep 514
   # UNCONN 0  0  0.0.0.0:514  0.0.0.0:*  users:(("wazuh-remote"...))
   ```

4. Rebooted pfSense via dashboard: Diagnostics > Reboot.

5. Verified packets arriving with tcpdump:
   ```bash
   sudo tcpdump -i ens33 udp port 514 -n
   # 192.168.1.1:514 > 192.168.1.20:514: SYSLOG local0.info
   ```

---

## Verification

pfSense syslog packets confirmed arriving at Ubuntu Server via tcpdump.
Wazuh processing confirmed by checking alerts log.

---

## Lesson Learned

Configuring syslog forwarding requires changes on both sides — the sender
(pfSense) and the receiver (Wazuh). It is easy to assume the receiver will
automatically accept any incoming traffic on an open port, but application-
level listeners must be explicitly configured. In this case UFW allowed port
514 but Wazuh was not bound to that port at all.

Additionally, daemon processes cache their configuration in memory. After
changing logging destinations in a GUI, the underlying daemon (syslogd, rsyslog,
etc.) often requires a restart before the change takes effect. A service reload
or full reboot is frequently the final step in syslog configuration changes.

---

## References

- Wazuh remote syslog configuration: documentation.wazuh.com/current/user-manual
- pfSense syslog forwarding: docs.netgate.com/pfsense/en/latest/monitoring/logs
