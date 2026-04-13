# Zabbix on Coolify — Docker Compose Stack

Docker Compose configuration for running Zabbix 7.4 monitoring stack on Coolify.

## Components

| Service | Image | Purpose | Port |
|---|---|---|---|
| `mysql-server` | `mysql:8.0-oracle` | Database backend | 3306 (localhost only) |
| `zabbix-server` | `zabbix/zabbix-server-mysql:alpine-7.4-latest` | Monitoring engine | 10051 (agent trapper) |
| `zabbix-web` | `zabbix/zabbix-web-nginx-mysql:alpine-7.4-latest` | Web interface | 8080 (HTTP) |

## Key configuration details

### Networking
- `extra_hosts: host.docker.internal:host-gateway` on zabbix-server — allows server to reach the host machine (for Zabbix Agent 2 and Coolify API)
- `ports: 10051:10051` on zabbix-server — allows host-installed Zabbix Agent 2 to send data to the server
- `ports: 127.0.0.1:3306:3306` on mysql-server — allows host-installed Agent 2 to monitor MySQL (localhost only, not exposed externally)
- No custom Docker networks — Coolify manages networking automatically
- Zabbix Web listens on port 8080 — assign domain in Coolify pointing to this port

### Volumes
- `mysql_data` — persistent MySQL storage (survives redeployments)
- `zabbix_server_alertscripts` — custom alert scripts
- `zabbix_server_externalscripts` — external check scripts

## Deploy via Coolify

1. Coolify > Project > New Resource > Docker Compose (or Git Repository)
2. Paste/link `docker-compose.yaml`
3. Set environment variables from `.env.example` (**change all passwords!**)
4. Assign domain to `zabbix-web` service with port `8080`
5. Deploy

## Environment variables

See `.env.example` for the full list. Required:

| Variable | Description |
|---|---|
| `MYSQL_DATABASE` | Database name (default: `zabbix`) |
| `MYSQL_USER` | Database user |
| `MYSQL_PASSWORD` | Database password (**change!**) |
| `MYSQL_ROOT_PASSWORD` | MySQL root password (**change!**) |
| `PHP_TZ` | Web UI timezone (default: `Europe/Berlin`) |
| `ZBX_CACHESIZE` | Zabbix server cache size (default: `32M`) |

## Zabbix Agent 2 (host-level)

Agent 2 runs directly on the server, **not** inside Docker. It provides CPU/RAM/disk metrics, Docker container monitoring, and MySQL monitoring.

### Installation (Ubuntu 24.04)

```bash
# Add Zabbix repo
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu$(lsb_release -rs)_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu$(lsb_release -rs)_all.deb
sudo apt update
sudo apt install -y zabbix-agent2 zabbix-agent2-plugin-*
```

### Configuration (`/etc/zabbix/zabbix_agent2.conf`)

```ini
ServerActive=127.0.0.1:10051
Server=127.0.0.1,10.0.0.0/8,172.0.0.0/8
Hostname=sjops.souljourneys.space
Plugins.Docker.Endpoint=unix:///var/run/docker.sock
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=5
```

**Key points:**
- `Server` includes `10.0.0.0/8` — Coolify uses this subnet for container networking
- `Hostname` must match the host technical name in Zabbix UI
- Docker socket access required for container monitoring

### Enable and start

```bash
sudo usermod -aG docker zabbix
sudo systemctl enable zabbix-agent2
sudo systemctl restart zabbix-agent2
```

## After deployment

1. Open assigned domain in browser
2. Login: `Admin` / `zabbix` → **change password immediately**
3. Verify: Administration > System Information > "Zabbix server is running" = Yes
4. Create host with interface IP `10.0.7.1` (Docker gateway to host) port `10050`
5. Link templates: Linux by Zabbix agent active, Docker by Zabbix agent 2

## Related repositories

| Repo | Description |
|---|---|
| [sj-monitoring](https://github.com/SKIT-Consult/sj-monitoring) | Zabbix configuration backups, dashboards, templates, research |
| [sj-n8n-backups](https://github.com/SKIT-Consult/sj-n8n-backups) | n8n workflow JSON exports |
| [sj-apple-apps](https://github.com/SKIT-Consult/sj-apple-apps) | macOS automation scripts for content production |
