# Incident 005: Wazuh Dashboard SSL Certificate Path Mismatch

**Date:** 2026-10-13  
**Severity:** High (dashboard completely unavailable)  
**Time to resolve:** ~20 minutes  
**Phase:** Phase 3 — Threat Simulation  

---

## Symptom

Wazuh dashboard at https://192.168.1.20 returned "Unable to connect" in
Firefox despite Ubuntu Server being reachable via ping from Ubuntu Desktop.

Running systemctl status showed the service as "active (running)" but the
uptime reset to 3-4 seconds on every check:

```
Active: active (running) since Tue 2026-10-13 06:28:11 UTC; 3s ago
Memory: 148.8M (peak: 149.1M)
```

Restart counter had reached 245+ iterations. Port 443 showed no listener:

```bash
sudo ss -tlnp | grep 443
# No output
```

---

## Impact

Wazuh dashboard completely inaccessible. Could not monitor SIEM alerts or
verify attack detections during Phase 3 threat simulation session. Wazuh
manager and indexer remained operational — agents continued reporting in the
background. Alert data was accumulating but could not be reviewed.

---

## Diagnosis Process

1. Noted restart counter incrementing every 3-4 seconds — confirmed crash
   loop rather than slow startup.

2. Read crash logs using journalctl:

   ```bash
   sudo journalctl -u wazuh-dashboard --no-pager -n 50
   ```

   Key error identified:

   ```
   Error: ENOENT: no such file or directory,
   open '/etc/wazuh-dashboard/certs/dashboard-key.pem'
   ```

   ENOENT is a Linux/Unix error code meaning "Error: No Entry" — the file
   path does not exist on disk.

3. Listed actual certificate files in the certs directory:

   ```bash
   sudo ls -la /etc/wazuh-dashboard/certs/
   ```

   Files present:
   - root-ca.pem
   - wazuh-dashboard-key.pem
   - wazuh-dashboard.pem

4. Checked what the dashboard config expected:

   ```bash
   sudo grep -i "key\|cert\|pem" \
   /etc/wazuh-dashboard/opensearch_dashboards.yml
   ```

   Config showed:
   ```yaml
   server.ssl.key: "/etc/wazuh-dashboard/certs/dashboard-key.pem"
   server.ssl.certificate: "/etc/wazuh-dashboard/certs/dashboard.pem"
   opensearch.ssl.certificateAuthorities:
     ["/etc/wazuh-dashboard/certs/root-ca.pem"]
   ```

   Mismatch confirmed:
   - Config expected: dashboard-key.pem
   - Actual file: wazuh-dashboard-key.pem
   - Config expected: dashboard.pem
   - Actual file: wazuh-dashboard.pem

---

## Root Cause

The opensearch_dashboards.yml configuration file referenced SSL certificate
filenames using a short naming convention (dashboard-key.pem) while the
actual installed certificate files used a longer naming convention
(wazuh-dashboard-key.pem). This mismatch likely resulted from a difference
between Wazuh installer variants or a version where the naming convention
changed. The dashboard process crashed immediately on startup when it could
not open the SSL key file, triggering systemd's automatic restart — which
also crashed immediately, creating the observed loop.

---

## Fix Applied

Updated both mismatched paths in opensearch_dashboards.yml using sed:

```bash
sudo sed -i \
's|certs/dashboard-key.pem|certs/wazuh-dashboard-key.pem|' \
/etc/wazuh-dashboard/opensearch_dashboards.yml

sudo sed -i \
's|certs/dashboard.pem|certs/wazuh-dashboard.pem|' \
/etc/wazuh-dashboard/opensearch_dashboards.yml
```

Verified the config now matched actual filenames:

```bash
sudo grep -i "key\|cert\|pem" \
/etc/wazuh-dashboard/opensearch_dashboards.yml
```

Output confirmed correct paths:
```yaml
server.ssl.key: "/etc/wazuh-dashboard/certs/wazuh-dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/wazuh-dashboard.pem"
```

Started the dashboard:

```bash
sudo systemctl start wazuh-dashboard
sleep 90
```

---

## Verification

```bash
sudo systemctl status wazuh-dashboard | grep -E "Active|Memory"
# Active: active (running) since Tue 2026-10-13 06:44:47 UTC; 1min 30s ago
# Memory: 258.7M (stable)

sudo ss -tlnp | grep 443
# LISTEN 0  511  0.0.0.0:443  users:(("node",pid=11662,fd=19))
```

Restart counter no longer incrementing. Port 443 now serving.
Dashboard accessible at https://192.168.1.20. Login confirmed successful.

---

## Lesson Learned

SSL certificate path mismatches are a common cause of service failures after
system migrations, VM moves, version upgrades, or reinstallations. The error
message from journalctl provided the exact file path that was missing
(dashboard-key.pem), which made it straightforward to compare against what
actually existed on disk.

The correct diagnostic workflow for any service crash loop is:

1. Check systemctl status — note restart counter and memory
2. Read journalctl logs — find the exact error message
3. Parse the error: ENOENT means file not found, EACCES means permission
   denied, ECONNREFUSED means connection refused to a dependency
4. Match the error to a specific file, port, or service
5. Verify the actual state on disk or network matches what the config expects
6. Apply the targeted fix

This workflow applies to any TLS-dependent service: SIEM platforms, EDR
consoles, certificate authorities, API gateways, and ticketing systems.

---

## References

- OpenSearch Dashboards TLS configuration
- Wazuh installation guide: documentation.wazuh.com
- Linux error codes: ENOENT, EACCES, ECONNREFUSED
