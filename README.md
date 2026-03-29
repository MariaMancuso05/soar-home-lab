# 🛡️ SOAR Home Lab — Wazuh + Shuffle + TheHive + Cortex

> **Automated threat detection, alert management, and incident response pipeline — built on open-source tools in a self-hosted lab environment.**

---

## 📌 Table of Contents

- [About the Project](#about-the-project)
- [Architecture](#architecture)
- [Shuffle Workflow](#shuffle-workflow)
- [Upcoming Features](#upcoming-features)
- [Tutorial — Part 1: Virtual Machine Setup](#tutorial--part-1-virtual-machine-setup)
- [Tutorial — Part 2: Tool Installation](#tutorial--part-2-tool-installation)
- [Coming Soon — Part 3](#coming-soon--part-3-node-configuration--cortex-integration)

---

## About the Project

This project implements a fully self-hosted **SOAR (Security Orchestration, Automation and Response)** pipeline designed for educational and lab purposes. The goal is to simulate a realistic SOC environment where security events are automatically detected, triaged, escalated, and notified — with zero manual intervention in the alert-to-case flow.

### What it does

When an attack or suspicious event is detected on a monitored machine, the pipeline automatically:

1. **Detects** the event via the Wazuh SIEM/XDR platform
2. **Receives** the alert in Shuffle via webhook
3. **Creates** an alert in TheHive (case management)
4. **Enriches** it with observable data (e.g. source IP)
5. **Opens** a case in TheHive for analyst investigation
6. **Notifies** the analyst via Telegram in real time

### Stack

| Tool | Role |
|------|------|
| **Wazuh** | SIEM / XDR — log collection, threat detection, rule engine |
| **Shuffle** | SOAR engine — workflow automation and orchestration |
| **TheHive** | Case management — alert/case lifecycle, observables |
| **Cortex** | *(integration in progress)* — automated IOC analysis and enrichment |

### Use Cases Covered

- SSH brute force detection → automatic case creation
- File Integrity Monitoring (FIM) alerts → TheHive escalation
- Suspicious network activity → real-time Telegram notification
- Source IP enrichment as observable on alert

---

## Architecture

The lab uses **3 Ubuntu VMs + 1 Kali Linux VM** on a private host-only network (`192.168.100.0/24`):

| VM | OS | RAM | Disk | Services |
|----|----|-----|------|----------|
| `wazuh-server` | Ubuntu 22.04 | 4 GB | 25 GB | Wazuh Manager + Indexer + Dashboard |
| `soar-server` | Ubuntu 22.04 | 4 GB | 20 GB | Shuffle + TheHive + Cortex (Docker) |
| `victim-server` | Ubuntu 22.04 | 1 GB | 12 GB | Wazuh Agent |
| `kali-attacker` | Kali Linux | 2 GB | existing | Red team tools |

**Network layout:**

| VM | Static IP | Key Ports |
|----|-----------|-----------|
| `wazuh-server` | `192.168.100.10` | 443 (dashboard), 1514/1515 (agent), 55000 (API) |
| `soar-server` | `192.168.100.20` | 3001 (Shuffle), 9000 (TheHive), 9001 (Cortex) |
| `victim-server` | `192.168.100.50` | 22 (SSH) |
| `kali-attacker` | `192.168.100.40` | — |

---

## Shuffle Workflow

The current workflow (`Wazuh SOAR Pipeline`) handles the full alert-to-notification chain automatically, from the moment Wazuh fires an alert to the creation of a case in TheHive and a real-time Telegram notification.

![Shuffle Workflow](workflow_screenshot.png)

### Node breakdown

**1. 🔵 Receive Wazuh alerts — Webhook trigger**
Entry point of the pipeline. Exposes an HTTP webhook URL that Wazuh calls every time an alert is fired. The incoming payload is a JSON object containing all alert fields: rule ID, description, severity level, source IP, affected host, and raw log.

**2. ⚙️ Repeat — Shuffle built-in**
Wazuh can batch multiple alerts into a single webhook call. This node iterates over each alert in the array individually, so every downstream node processes one alert at a time in a clean loop.

**3. 🐝 Create alert — TheHive**
Creates a new alert in TheHive, mapping the Wazuh JSON fields to TheHive's alert schema: title, severity, description, and tags. This gives the analyst an immediate, structured view of the event inside the case management platform.

**4. 🐝 Add IP to alert — TheHive**
Extracts the source IP from the Wazuh alert and attaches it as an **observable** to the alert just created. This enriches the alert with the attacker's IP as a trackable indicator of compromise (IOC), ready for future analysis by Cortex.

**5. 🐝 Create case — TheHive**
Automatically escalates the alert to a full **investigation case** in TheHive. From this point, the analyst has a complete case with timeline, observables, and task management — no manual triage needed.

**6. ✈️ Send Message — Telegram**
Sends a real-time notification to a configured Telegram chat, including the alert title, severity, and source IP. This ensures the analyst is immediately informed even when not actively monitoring the TheHive dashboard.

---

## Upcoming Features

The following capabilities are currently in development and will be added in the next release:

### Cortex Integration — Automated IOC Analysis

Cortex will be integrated directly into the Shuffle workflow to automatically run analyzers on observables extracted from Wazuh alerts. Planned analyzers include:

- **VirusTotal** — IP and hash reputation lookup
- **AbuseIPDB** — IP abuse score and geolocation
- **MaxMind GeoIP** — Geographic enrichment of source IPs

Results will be automatically attached to the corresponding TheHive case as analysis reports.

### Automated Incident Response

A response playbook will be added to the pipeline. When an alert exceeds a configured severity threshold, Shuffle will:

- Automatically block the attacker IP via `iptables` on the victim server
- Log the automated action as a task update inside the TheHive case
- Notify the analyst via Telegram with the action taken and timestamp

---

## Tutorial — Part 1: Virtual Machine Setup

> This section covers the initial configuration of all VMs before installing any tools.

### 1.1 — Create the VMs

Create the following VMs in VirtualBox or VMware with **two network adapters** each:
- **Adapter 1**: NAT (for internet access during installation)
- **Adapter 2**: Host-only on the `192.168.100.0/24` subnet (for inter-VM communication)

Recommended specs per VM are listed in the [Architecture](#architecture) table above.

### 1.2 — Configure Static IPs (all Ubuntu VMs)

After installing each Ubuntu Server, identify the second network interface name and configure a static IP:

```bash
ip a   # identify the second interface (e.g. enp0s8 on VirtualBox, ens37 on VMware)
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace the file content with the following, adjusting the IP for each VM:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:       # first adapter: NAT
      dhcp4: true
    enp0s8:       # second adapter: host-only
      dhcp4: false
      addresses: [192.168.100.10/24]   # change per VM: .10 / .20 / .50 / .40
```

Apply the configuration:

```bash
sudo netplan apply
```

> **IPs to assign:** `.10` → wazuh-server, `.20` → soar-server, `.50` → victim-server, `.40` → kali-attacker

### 1.3 — System Update and Common Tools (all Ubuntu VMs)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano net-tools
```

### 1.4 — Install Docker (soar-server only)

> ⚠️ Do **NOT** install Docker on `wazuh-server` — Wazuh uses its own native installer which is incompatible with Docker on the same machine.

```bash
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker && sudo systemctl start docker
sudo usermod -aG docker $USER && newgrp docker
```

---

## Tutorial — Part 2: Tool Installation

### 2.1 — Wazuh (on `wazuh-server`)

For Wazuh installation, follow the **official step-by-step guides** in this exact order:

| Component | Official Guide |
|-----------|---------------|
| **Wazuh Indexer** | [Step-by-step installation](https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/step-by-step.html) |
| **Wazuh Manager** | [Step-by-step installation](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html) |
| **Wazuh Dashboard** | [Step-by-step installation](https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/step-by-step.html) |

> 💡 Once the installation is complete, save the credentials and verify the dashboard is accessible at `https://192.168.100.10` before proceeding to the next step.

#### Configure the Wazuh → Shuffle Webhook

After setting up Shuffle (step 2.2) and obtaining your webhook URL, add the integration to Wazuh:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following block before the closing `</ossec_config>` tag:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://192.168.100.20:3001/api/v1/hooks/YOUR_HOOK_ID</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

```bash
sudo systemctl restart wazuh-manager
```

> Level `7` captures significant alerts such as brute force, file modifications, and privilege escalation. Raise to `10` to reduce noise during testing.

---

### 2.2 — Shuffle (on `soar-server`)

Clone the official Shuffle repository and start it with Docker Compose:

```bash
git clone https://github.com/Shuffle/Shuffle
cd Shuffle
docker-compose up -d
```

Wait 2–3 minutes, then access the UI at:

```
http://192.168.100.20:3001
```

Register an admin account on first access.

> 💡 To test the webhook immediately after setup:
> ```bash
> curl -X POST http://192.168.100.20:3001/api/v1/hooks/YOUR_HOOK_ID \
>   -H 'Content-Type: application/json' \
>   -d '{"test":"ok"}'
> ```

---

### 2.3 — Wazuh Agent (on `victim-server`)

From the Wazuh Dashboard (`https://192.168.100.10`), go to **Agents → Deploy new agent**, select **DEB (Ubuntu)**, and enter the manager IP. Copy and run the generated command on `victim-server`:

```bash
# Command generated by the dashboard — example:
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.0_amd64.deb
sudo WAZUH_MANAGER='192.168.100.10' dpkg -i ./wazuh-agent_*.deb
sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
```

Verify in the Wazuh dashboard: the agent should appear as **Active** within 30 seconds.

---

### 2.4 — TheHive + Cortex (on `soar-server`, via Docker)

Clone this repository to get the `TheHive-stack` directory and the `docker-compose.yml` already configured:

```bash
cd ~
git clone https://github.com/MariaMancuso05/soar-home-lab.git
```

Move into the stack directory and start the services:

```bash
cd ~/soar-home-lab/TheHive-stack
docker-compose up -d
```

> 💡 First startup may take a few minutes as Docker pulls the images (~2 GB total). You can monitor progress with:
> ```bash
> docker-compose logs -f
> ```

Access the services at:

| Service | URL | Default credentials |
|---------|-----|---------------------|
| TheHive | `http://192.168.100.20:9000` | `admin@thehive.local` / `secret` |
| Cortex | `http://192.168.100.20:9001` | Create account on first access |

---

## Coming Soon — Part 3: Node Configuration & Cortex Integration

The next section of this tutorial will cover:

- Step-by-step configuration of each Shuffle workflow node (field mapping, authentication, error handling)
- TheHive API key setup and Shuffle app authentication
- Telegram bot setup and message formatting
- Full Cortex integration: connecting Cortex to TheHive, enabling analyzers (VirusTotal, AbuseIPDB, MaxMind), and adding the analysis block to the Shuffle workflow
- End-to-end test: simulated brute force attack from Kali → full pipeline execution walkthrough

---

## Resources

- [Wazuh Documentation](https://documentation.wazuh.com)
- [Shuffle GitHub](https://github.com/Shuffle/Shuffle)
- [TheHive Documentation](https://docs.strangebee.com)
- [Cortex Analyzers](https://github.com/TheHive-Project/Cortex-Analyzers)
- [MITRE ATT&CK](https://attack.mitre.org)
