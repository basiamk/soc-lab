# Wazuh Agent Deployment

## Overview

Wazuh agents installed on 4 VMs. Each agent connects back to the Wazuh
manager at 192.168.1.20 on port 1514 (TCP, encrypted).

Ubuntu Server (192.168.1.20) is the Wazuh manager itself and does not
need a separate agent — it is monitored by the manager directly.

---

## Agent Installation Method

All agents deployed using the auto-generated command from the Wazuh dashboard:
- Dashboard > Agents > Deploy new agent
- Select OS and architecture
- Enter server address: 192.168.1.20
- Enter agent name
- Copy generated PowerShell or bash command
- Run on target VM

---

## Deployed Agents

### Windows10Desktop (192.168.1.30)

OS: Windows 10, x86_64  
Agent name: Windows10Desktop  
Install method: PowerShell as Administrator  

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.11.2-1.msi `
  -OutFile $env:tmp\wazuh-agent; `
msiexec.exe /i $env:tmp\wazuh-agent /q `
  WAZUH_MANAGER='192.168.1.20' `
  WAZUH_AGENT_NAME='Windows10Desktop'
NET START WazuhSvc
```

### winserver2022-dc (192.168.1.10)

OS: Windows Server 2022, x86_64  
Agent name: winserver2022-dc  
Install method: PowerShell as Administrator (same command structure)  

### kali-attacker (192.168.1.40)

OS: Debian amd64 (Kali is Debian-based)  
Agent name: kali-attacker  
Install method: bash  

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### ubuntu-desktop (192.168.1.50)

OS: Ubuntu amd64  
Agent name: ubuntu-desktop  
Install method: bash (same as Kali)  

---

## Agent Status Verification

All 4 agents verified Active in Wazuh dashboard > Agents view.

---

## Copy-Paste Issue Resolution

VMware clipboard sharing was not working between VMs by default.
Resolved by:
- VM > Settings > Options > Guest Isolation
- Enable copy and paste: checked
- Enable drag and drop: checked
- Restarted the VM

As a workaround when clipboard sharing unavailable, commands were typed
manually from screenshots of the generated command.
