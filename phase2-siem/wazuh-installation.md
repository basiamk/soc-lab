# Wazuh Installation Notes

## Overview

Wazuh 4.11.2 deployed on Ubuntu Server 24.04 using the official all-in-one
installer. The all-in-one installation bundles Wazuh Manager, Wazuh Indexer
(OpenSearch), and Wazuh Dashboard into a single deployment.

---

## Installation

### Prerequisites

System updated before installation:
```bash
sudo apt update && sudo apt upgrade -y
```

### Installer command

```bash
curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh && \
sudo bash wazuh-install.sh -a
```

Installation time: approximately 15-20 minutes depending on internet speed.

### Post-installation credentials

Admin password generated and saved at installation completion.
Stored securely offline. Dashboard accessible at https://192.168.1.20.

---

## Service Management

Three services must be running in this order:

```bash
sudo systemctl start wazuh-indexer    # Start first — it is the database
sudo systemctl start wazuh-manager    # Start second
sudo systemctl start wazuh-dashboard  # Start last
```

Health check script saved at ~/check-wazuh.sh:
```bash
#!/bin/bash
FREE=$(free -m | awk 'NR==2{print $7}')
echo "Available RAM: ${FREE}MB"
if [ "$FREE" -lt 400 ]; then
    echo "WARNING: Low memory — Wazuh may become unstable"
fi
sudo systemctl status wazuh-manager | grep Active
sudo systemctl status wazuh-indexer | grep Active
sudo systemctl status wazuh-dashboard | grep Active
```

---

## UFW Firewall Rules

```bash
sudo ufw allow 443/tcp    # Dashboard HTTPS
sudo ufw allow 80/tcp     # HTTP redirect
sudo ufw allow 1514/tcp   # Agent communication
sudo ufw allow 1515/tcp   # Agent enrollment
sudo ufw allow 55000/tcp  # Wazuh API
sudo ufw allow 514/udp    # pfSense syslog
```

---

## Memory Management

Wazuh indexer JVM heap capped to prevent memory exhaustion on 4 GB VM:

File: /etc/wazuh-indexer/jvm.options
```
-Xms1g
-Xmx1g
```

logall disabled to prevent disk and RAM exhaustion:

File: /var/ossec/etc/ossec.conf
```xml
<logall>no</logall>
<logall_json>no</logall_json>
```

Weekly log rotation cron job:
```
0 2 * * 0 find /var/ossec/logs -name "*.log" -mtime +7 -delete
0 2 * * 0 find /var/ossec/logs -name "*.json" -mtime +7 -delete
```

---

## Active Agents

| Agent Name | IP | OS | Status |
|---|---|---|---|
| winserver2022-dc | 192.168.1.10 | Windows Server 2022 | Active |
| Windows10Desktop | 192.168.1.30 | Windows 10 | Active |
| kali-attacker | 192.168.1.40 | Kali Linux | Active |
| ubuntu-desktop | 192.168.1.50 | Ubuntu Desktop | Active |

Note: Ubuntu Server (192.168.1.20) is the Wazuh manager itself and does not
require a separate agent.

---

## ossec.conf Additions

Sysmon event collection added for Windows agents:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

pfSense syslog remote listener added:
```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.1.0/24</allowed-ips>
</remote>
```

---

## Known Issues

See troubleshooting-log/ for full incident reports:
- Memory exhaustion from logall=yes: troubleshooting-log/002
- pfSense syslog not forwarding: troubleshooting-log/003
- Dashboard SSL cert mismatch: troubleshooting-log/005
