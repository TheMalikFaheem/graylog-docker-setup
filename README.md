# Graylog Docker Compose Stack

A production-ready Graylog 5 setup with MongoDB 6 and OpenSearch 2.

## Stack Overview

| Service | Image | Port |
|---|---|---|
| Graylog | `graylog/graylog:5.2` | `9000` (Web UI) |
| OpenSearch | `opensearchproject/opensearch:2.11.0` | `9200` (internal) |
| MongoDB | `mongo:6.0` | `27017` (internal) |

## Quick Start

### 1. Prerequisites

- Docker Engine 20.10+
- Docker Compose v2+
- At least **4 GB RAM** available

### 2. Set kernel parameter (Linux host — required for OpenSearch)

```bash
sudo sysctl -w vm.max_map_count=262144

# Make persistent across reboots:
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 3. Configure environment

```bash
cp .env.example .env
```

Edit `.env` and set:

**Generate `GRAYLOG_PASSWORD_SECRET`:**
```bash
# Install pwgen if needed: sudo apt install pwgen
pwgen -N 1 -s 96
```

**Generate `GRAYLOG_ROOT_PASSWORD_SHA2`:**
```bash
echo -n "YourAdminPassword" | sha256sum | awk '{print $1}'
```

### 4. Start the stack

```bash
docker compose up -d
```

### 5. Access Graylog

Open your browser at: **http://localhost:9000**

- **Username:** `admin`
- **Password:** whatever you hashed in step 3

---

## Input Ports

| Port | Protocol | Purpose |
|---|---|---|
| `9000` | TCP | Graylog Web UI & REST API |
| `514` | UDP/TCP | Syslog |
| `12201` | UDP/TCP | GELF (Docker log driver, apps) |
| `5555` | TCP | Raw/Plaintext |
| `5044` | TCP | Beats (Filebeat/Metricbeat) |

## Useful Commands

```bash
# View logs
docker compose logs -f graylog

# Stop stack
docker compose down

# Stop and remove all data volumes (DESTRUCTIVE)
docker compose down -v

# Restart a single service
docker compose restart graylog
```

## Sending Logs via GELF (Docker containers)

Add this to any container you want to ship logs from:

```yaml
logging:
  driver: gelf
  options:
    gelf-address: "udp://YOUR_GRAYLOG_HOST:12201"
    tag: "my-service"
```

## Notes

- OpenSearch security is **disabled** for internal cluster communication. Enable it for production with TLS certificates.
- MongoDB and OpenSearch ports are **not exposed** to the host by default for security.
- Data is persisted in named Docker volumes (`mongodb_data`, `opensearch_data`, `graylog_data`, `graylog_journal`).
