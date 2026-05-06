# Graylog Docker Compose Stack

A production-ready **Graylog 5** log management setup with **MongoDB 6** and **OpenSearch 2**, orchestrated via Docker Compose.

---

## Stack Overview

| Service    | Image                                      | Port             |
|------------|--------------------------------------------|------------------|
| Graylog    | `graylog/graylog:5.2`                      | `9000` (Web UI)  |
| OpenSearch | `opensearchproject/opensearch:2.11.0`      | `9200` (internal)|
| MongoDB    | `mongo:6.0`                                | `27017` (internal)|

---

## Full Setup Guide for Ubuntu

### Step 1 — Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

---

### Step 2 — Install Required Dependencies

```bash
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https \
    software-properties-common \
    pwgen \
    git
```

---

### Step 3 — Install Docker Engine

#### 3.1 Add Docker's Official GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

#### 3.2 Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### 3.3 Install Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 3.4 Start & Enable Docker Service

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

#### 3.5 Run Docker Without sudo (Recommended)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

#### 3.6 Verify Docker Installation

```bash
docker --version
docker compose version
docker run hello-world
```

---

### Step 4 — Set Kernel Parameter for OpenSearch

OpenSearch requires a higher virtual memory limit. This is **mandatory**.

```bash
# Apply immediately (current session)
sudo sysctl -w vm.max_map_count=262144

# Make it persist across reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### Step 5 — Clone This Repository

```bash
git clone https://github.com/TheMalikFaheem/graylog-docker-setup.git
cd graylog-docker-setup
```

---

### Step 6 — Configure Environment Variables

```bash
cp .env.example .env
nano .env
```

#### 6.1 Generate `GRAYLOG_PASSWORD_SECRET` (min 64 chars)

```bash
pwgen -N 1 -s 96
```

Copy the output and paste it as the value of `GRAYLOG_PASSWORD_SECRET` in `.env`.

#### 6.2 Generate `GRAYLOG_ROOT_PASSWORD_SHA2`

```bash
echo -n "YourAdminPassword" | sha256sum | awk '{print $1}'
```

Copy the hash and paste it as the value of `GRAYLOG_ROOT_PASSWORD_SHA2` in `.env`.

#### 6.3 Set External URI

If accessing from a remote server or IP, update this in `.env`:

```env
GRAYLOG_HTTP_EXTERNAL_URI=http://YOUR_SERVER_IP:9000/
```

---

### Step 7 — Start the Stack

```bash
docker compose up -d
```

Check all containers are running:

```bash
docker compose ps
```

Watch live logs:

```bash
docker compose logs -f graylog
```

---

### Step 8 — Access Graylog Web UI

Open your browser and go to:

```
http://localhost:9000
```

Or if on a remote server:

```
http://YOUR_SERVER_IP:9000
```

- **Username:** `admin`
- **Password:** the plain-text password you hashed in Step 6.2

---

## Input Ports Reference

| Port    | Protocol | Purpose                        |
|---------|----------|--------------------------------|
| `9000`  | TCP      | Graylog Web UI & REST API      |
| `514`   | UDP/TCP  | Syslog                         |
| `12201` | UDP/TCP  | GELF (apps, Docker log driver) |
| `5555`  | TCP      | Raw / Plaintext                |
| `5044`  | TCP      | Beats (Filebeat / Metricbeat)  |

---

## Useful Commands

```bash
# View running containers
docker compose ps

# Follow Graylog logs
docker compose logs -f graylog

# Follow all service logs
docker compose logs -f

# Stop the stack (data preserved)
docker compose down

# Stop and delete all volumes (DESTRUCTIVE — removes all data)
docker compose down -v

# Restart a single service
docker compose restart graylog

# Pull latest images
docker compose pull

# Rebuild and restart
docker compose up -d --force-recreate
```

---

## Sending Logs via GELF from Docker Containers

Add this logging block to any Docker container you want to ship logs from:

```yaml
logging:
  driver: gelf
  options:
    gelf-address: "udp://YOUR_GRAYLOG_HOST:12201"
    tag: "my-service-name"
```

---

## Firewall Setup (UFW)

If UFW is enabled on your Ubuntu server, open the necessary ports:

```bash
# Allow Graylog Web UI
sudo ufw allow 9000/tcp

# Allow Syslog
sudo ufw allow 514/tcp
sudo ufw allow 514/udp

# Allow GELF
sudo ufw allow 12201/tcp
sudo ufw allow 12201/udp

# Allow Beats
sudo ufw allow 5044/tcp

# Reload UFW
sudo ufw reload
sudo ufw status
```

---

## Troubleshooting

### OpenSearch fails to start

```bash
# Check current value
sysctl vm.max_map_count

# If less than 262144, set it:
sudo sysctl -w vm.max_map_count=262144
```

### Graylog Web UI not loading

```bash
# Check if all containers are healthy
docker compose ps

# Check Graylog logs for errors
docker compose logs graylog --tail=50
```

### Permission denied running Docker

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## Security Notes

- MongoDB and OpenSearch ports (`27017`, `9200`) are **NOT exposed** to the host — only accessible internally between containers.
- Always replace the default `GRAYLOG_PASSWORD_SECRET` and `GRAYLOG_ROOT_PASSWORD_SHA2` before deploying.
- For production, consider putting Graylog behind **Nginx** with **SSL/TLS**.

---

## Minimum System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU      | 2 cores | 4+ cores    |
| RAM      | 4 GB    | 8+ GB       |
| Disk     | 20 GB   | 50+ GB      |
| OS       | Ubuntu 20.04+ | Ubuntu 22.04 LTS |
