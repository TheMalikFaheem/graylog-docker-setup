# Graylog Log Ingestion & Analysis Guide

A complete reference for shipping **System Logs**, **Server Logs**, **Application Logs**, and **Remote Server Logs** into your Graylog instance.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Logs (Linux)](#1-system-logs-linux)
3. [Server Logs (Nginx & Apache)](#2-server-logs-nginx--apache)
4. [Application Logs (Docker Containers)](#3-application-logs-docker-containers)
5. [Remote Server Logs](#4-remote-server-logs-separate-server)
6. [Viewing & Analyzing Logs in Graylog UI](#5-viewing--analyzing-logs-in-graylog-ui)
7. [Creating Streams & Alerts](#6-creating-streams--alerts)

---

## Prerequisites

- Graylog is running and accessible (see `README.md` for setup)
- Graylog Web UI is open at `http://YOUR_GRAYLOG_IP:9000`

### First — Create Inputs in Graylog

Every log source needs a **Graylog Input** to receive data.
Go to: **System → Inputs → Select Input Type → Launch**

| Log Source       | Input Type to Create in Graylog |
|------------------|----------------------------------|
| System (Syslog)  | Syslog UDP or Syslog TCP         |
| Nginx / Apache   | Beats or Syslog                  |
| Docker Containers| GELF UDP                         |
| Remote Server    | Beats (port 5044) or Syslog      |

---

## 1. System Logs (Linux)

Linux system logs include authentication events, kernel messages, cron jobs, and general system activity.

### Key Log Files on Ubuntu

| Log File               | Purpose                              |
|------------------------|--------------------------------------|
| `/var/log/syslog`      | General system activity              |
| `/var/log/auth.log`    | SSH logins, sudo usage, auth events  |
| `/var/log/kern.log`    | Kernel messages                      |
| `/var/log/dmesg`       | Boot and hardware messages           |
| `/var/log/cron.log`    | Cron job execution logs              |
| `/var/log/dpkg.log`    | Package install/remove history       |
| `/var/log/boot.log`    | System boot log                      |

---

### Method A — Rsyslog (Recommended for System Logs)

**Rsyslog** is pre-installed on Ubuntu and can forward logs directly to Graylog via Syslog UDP.

#### Step 1 — In Graylog UI: Create a Syslog UDP Input

1. Go to **System → Inputs**
2. Select **Syslog UDP** → Click **Launch new input**
3. Set:
   - **Title:** `Ubuntu System Logs`
   - **Port:** `514`
   - **Bind address:** `0.0.0.0`
4. Click **Save**

#### Step 2 — On your Ubuntu server: Configure Rsyslog

```bash
sudo nano /etc/rsyslog.d/90-graylog.conf
```

Paste the following:

```bash
# Forward ALL system logs to Graylog via UDP Syslog
*.* @YOUR_GRAYLOG_IP:514;RSYSLOG_SyslogProtocol23Format

# Forward only Auth logs (SSH, sudo events)
auth,authpriv.* @YOUR_GRAYLOG_IP:514;RSYSLOG_SyslogProtocol23Format

# Forward Kernel logs
kern.* @YOUR_GRAYLOG_IP:514;RSYSLOG_SyslogProtocol23Format
```

> **Note:** Use `@` for UDP and `@@` for TCP.

#### Step 3 — Restart Rsyslog

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

#### Step 4 — Verify

In Graylog UI → **Search** → You should see system log messages flowing in within seconds.

---

### Method B — Journald (systemd logs)

For systemd-based logs, use `systemd-journal-gatewayd` or ship via Filebeat.

```bash
# View all systemd journal logs locally
journalctl -f

# View logs for a specific service
journalctl -u ssh -f
journalctl -u nginx -f
journalctl -u docker -f

# View logs since last boot
journalctl -b

# View kernel logs
journalctl -k

# View auth/sudo logs
journalctl _COMM=sudo -f
```

To forward journald to rsyslog (so it gets picked up automatically):

```bash
sudo nano /etc/systemd/journald.conf
```

Set:

```ini
[Journal]
ForwardToSyslog=yes
```

```bash
sudo systemctl restart systemd-journald
```

---

## 2. Server Logs (Nginx & Apache)

### 2A — Nginx Logs

#### Key Nginx Log Files

| File                          | Purpose           |
|-------------------------------|-------------------|
| `/var/log/nginx/access.log`   | All HTTP requests |
| `/var/log/nginx/error.log`    | Errors & warnings |

---

#### Method — Filebeat (Best for Nginx & Apache)

**Filebeat** is a lightweight log shipper by Elastic. It reads log files and sends them to Graylog.

##### Install Filebeat on Ubuntu

```bash
# Import Elastic GPG key
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

# Add Elastic repository
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | \
    sudo tee /etc/apt/sources.list.d/elastic-8.x.list

# Install Filebeat
sudo apt update && sudo apt install -y filebeat
```

##### Configure Filebeat for Nginx

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Replace the entire content with:

```yaml
filebeat.inputs:

  # ── Nginx Access Logs ──────────────────────────────────────────────────────
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      log_type: nginx_access
      server_name: web-server-01
    fields_under_root: true

  # ── Nginx Error Logs ───────────────────────────────────────────────────────
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    fields:
      log_type: nginx_error
      server_name: web-server-01
    fields_under_root: true

# ── Output: Send to Graylog Beats Input ────────────────────────────────────
output.logstash:
  hosts: ["YOUR_GRAYLOG_IP:5044"]

# ── Logging ────────────────────────────────────────────────────────────────
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

##### In Graylog UI — Create a Beats Input

1. Go to **System → Inputs**
2. Select **Beats** → Click **Launch new input**
3. Set:
   - **Title:** `Nginx Logs via Filebeat`
   - **Port:** `5044`
4. Click **Save**

##### Start Filebeat

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
sudo systemctl status filebeat

# Test Filebeat config
sudo filebeat test config
sudo filebeat test output
```

---

### 2B — Apache Logs

#### Key Apache Log Files

| File                              | Purpose           |
|-----------------------------------|-------------------|
| `/var/log/apache2/access.log`     | All HTTP requests |
| `/var/log/apache2/error.log`      | Errors & warnings |

Add these inputs to your `/etc/filebeat/filebeat.yml`:

```yaml
filebeat.inputs:

  - type: log
    enabled: true
    paths:
      - /var/log/apache2/access.log
    fields:
      log_type: apache_access
      server_name: web-server-01
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /var/log/apache2/error.log
    fields:
      log_type: apache_error
      server_name: web-server-01
    fields_under_root: true

output.logstash:
  hosts: ["YOUR_GRAYLOG_IP:5044"]
```

```bash
sudo systemctl restart filebeat
```

---

## 3. Application Logs (Docker Containers)

Docker containers can send logs directly to Graylog using the **GELF log driver** — no extra agent needed.

### Step 1 — In Graylog UI: Create a GELF UDP Input

1. Go to **System → Inputs**
2. Select **GELF UDP** → Click **Launch new input**
3. Set:
   - **Title:** `Docker Application Logs`
   - **Port:** `12201`
4. Click **Save**

---

### Step 2 — Configure Docker Containers to Use GELF

#### Option A — Per Container in docker-compose.yml

Add a `logging` block to any service:

```yaml
services:
  my-app:
    image: my-app:latest
    logging:
      driver: gelf
      options:
        gelf-address: "udp://YOUR_GRAYLOG_IP:12201"
        tag: "my-app"
        labels: "service,environment"
    environment:
      - service=my-app
      - environment=production
```

#### Option B — Set GELF as Default Docker Log Driver (all containers)

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "gelf",
  "log-opts": {
    "gelf-address": "udp://YOUR_GRAYLOG_IP:12201",
    "tag": "{{.Name}}"
  }
}
```

```bash
sudo systemctl restart docker
```

> ⚠️ Restart existing containers for the new log driver to take effect.

---

### Step 3 — View Docker Logs in Graylog

In Graylog UI → **Search** → Filter by field:
- `container_name:my-app`
- `image_name:nginx`
- `tag:my-app`

---

### Viewing Docker Logs Locally (Quick Commands)

```bash
# View logs of a running container
docker logs <container_name> -f

# Last 100 lines
docker logs <container_name> --tail 100

# Logs with timestamps
docker logs <container_name> -f --timestamps

# Logs for a specific time range
docker logs <container_name> --since 2024-01-01T00:00:00 --until 2024-01-02T00:00:00
```

---

## 4. Remote Server Logs (Separate Server)

To collect logs from a **completely separate Ubuntu server** and send them to your central Graylog instance.

```
[ Remote Server ]  ──── Filebeat / Rsyslog ────►  [ Graylog Server ]
  (web-02, db-01, etc.)                           (central log collector)
```

---

### Method A — Filebeat on Remote Server (Recommended)

#### On the Remote Server: Install Filebeat

```bash
# On the REMOTE server (not Graylog server)
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | \
    sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update && sudo apt install -y filebeat
```

#### Configure Filebeat on Remote Server

```bash
sudo nano /etc/filebeat/filebeat.yml
```

```yaml
filebeat.inputs:

  # ── System Logs ────────────────────────────────────────────────────────────
  - type: log
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log
      - /var/log/kern.log
    fields:
      log_type: system
      server_name: remote-server-01        # ← Identify which server
      environment: production
    fields_under_root: true

  # ── Nginx Logs (if running on remote server) ───────────────────────────────
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
      - /var/log/nginx/error.log
    fields:
      log_type: nginx
      server_name: remote-server-01
    fields_under_root: true

  # ── Custom App Logs ────────────────────────────────────────────────────────
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log
      - /opt/myapp/logs/*.log
    fields:
      log_type: application
      app_name: myapp
      server_name: remote-server-01
    fields_under_root: true
    multiline:                            # Handle multi-line stack traces
      pattern: '^\d{4}-\d{2}-\d{2}'
      negate: true
      match: after

# ── Output: Send to CENTRAL Graylog Server ────────────────────────────────
output.logstash:
  hosts: ["CENTRAL_GRAYLOG_IP:5044"]      # ← Point to your Graylog server

logging.level: info
```

#### Start Filebeat on Remote Server

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
sudo systemctl status filebeat

# Verify connectivity to Graylog
sudo filebeat test output
```

---

### Method B — Rsyslog Forwarding from Remote Server

For simple syslog forwarding without installing Filebeat:

#### On the Remote Server

```bash
sudo nano /etc/rsyslog.d/90-graylog-remote.conf
```

```bash
# Forward all logs to central Graylog (UDP)
*.* @CENTRAL_GRAYLOG_IP:514;RSYSLOG_SyslogProtocol23Format

# Or use TCP (more reliable)
*.* @@CENTRAL_GRAYLOG_IP:514;RSYSLOG_SyslogProtocol23Format
```

```bash
sudo systemctl restart rsyslog
```

#### In Graylog UI

Go to **System → Inputs** → Make sure a **Syslog UDP** input is running on port `514`.

---

### Method C — Shipping Docker Logs from Remote Server

On the **remote server** running Docker containers:

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "gelf",
  "log-opts": {
    "gelf-address": "udp://CENTRAL_GRAYLOG_IP:12201",
    "tag": "{{.Name}}",
    "env": "server_name=remote-server-01"
  }
}
```

```bash
sudo systemctl restart docker
```

---

### Firewall Rules on Graylog Server (Allow Remote Servers)

```bash
# Allow Beats from remote servers
sudo ufw allow 5044/tcp

# Allow Syslog from remote servers
sudo ufw allow 514/udp
sudo ufw allow 514/tcp

# Allow GELF from remote Docker hosts
sudo ufw allow 12201/udp
sudo ufw allow 12201/tcp

sudo ufw reload
```

---

## 5. Viewing & Analyzing Logs in Graylog UI

### Basic Search

Open `http://YOUR_GRAYLOG_IP:9000` → Click **Search**

```
# Search for a keyword
ssh failed

# Search by field
source:remote-server-01

# Search by log type
log_type:nginx_access

# Search by app name
app_name:myapp

# Combine filters
source:remote-server-01 AND log_type:nginx_error

# Search in time range (last 1 hour by default — use the time picker)
```

---

### Useful Built-in Fields

| Field          | Example Value       | Description               |
|----------------|---------------------|---------------------------|
| `source`       | `web-server-01`     | Hostname of the sender    |
| `message`      | `Failed password`   | The actual log message    |
| `timestamp`    | `2024-01-01 10:00`  | When the log was received |
| `level`        | `3` (error)         | Syslog severity level     |
| `facility`     | `auth`              | Log facility/category     |
| `container_name` | `nginx`           | Docker container name     |
| `log_type`     | `nginx_access`      | Custom field via Filebeat |
| `server_name`  | `remote-server-01`  | Custom field via Filebeat |

---

### Create a Dashboard

1. Go to **Dashboards → Create new dashboard**
2. Click **Add widget**
3. Choose:
   - **Message count** — how many logs per minute
   - **Field value chart** — log_type distribution
   - **Quick values** — top sources
4. Save and name your dashboard (e.g., `Production Overview`)

---

## 6. Creating Streams & Alerts

### Streams — Separate Logs by Source

Streams let you route and filter logs automatically.

1. Go to **Streams → Create Stream**
2. Example — Create a stream for `remote-server-01`:
   - **Title:** `Remote Server 01 Logs`
   - **Rule:** Field `server_name` must match `remote-server-01`
3. Click **Start Stream**

---

### Alerts — Get Notified on Critical Events

1. Go to **Alerts → Notifications → Create Notification**
2. Choose notification type: **Email**, **Slack**, **HTTP webhook**
3. Go to **Alerts → Event Definitions → Create Event Definition**
4. Example triggers:
   - SSH failed login: message contains `Failed password`
   - High error rate: `log_type:nginx_error` count > 50 in 5 minutes
   - Disk/memory warnings: message contains `out of memory`

---

## Summary: Which Method to Use?

| Scenario                          | Recommended Method       | Port |
|-----------------------------------|--------------------------|------|
| Linux system logs (same server)   | Rsyslog                  | 514  |
| Nginx / Apache log files          | Filebeat → Beats Input   | 5044 |
| Docker container logs             | GELF log driver          | 12201|
| Remote server — log files         | Filebeat on remote host  | 5044 |
| Remote server — syslog            | Rsyslog forwarding       | 514  |
| Remote Docker host                | GELF log driver → Graylog| 12201|
