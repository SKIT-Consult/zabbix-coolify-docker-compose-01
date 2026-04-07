# Zabbix on Coolify — Deployment Guide

## What's included
- MySQL 8.0 (database)
- Zabbix Server 7.4 (monitoring engine)
- Zabbix Web Nginx 7.4 (web interface, port 8080)

## What's NOT included (by design)
- Zabbix Agent — install on host separately (see below)
- Zabbix Proxy — not needed for single-server setup
- Java Gateway — not needed unless monitoring JMX
- SNMP Traps — not needed for this use case

## Deploy via Coolify

### Option A: Docker Compose Empty (fastest)
1. Coolify → Project → New Resource → Docker Compose Empty
2. Paste contents of `docker-compose.yml` into the editor
3. Add environment variables from `.env.example` (change passwords!)
4. Assign domain to `zabbix-web` service with port `8080`
5. Deploy

### Option B: Git Repository (versioned)
1. Push this folder to a public Git repo
2. Coolify → Project → New Resource → Public Repository
3. Paste repo URL
4. Build Pack → Docker Compose
5. Add environment variables from `.env.example` (change passwords!)
6. Assign domain to `zabbix-web` service with port `8080`
7. Deploy

## After deployment
1. Open assigned domain in browser
2. Login: `Admin` / `zabbix`
3. **Change default password immediately**
4. Check: Administration → System Information → Zabbix server is running = Yes

## Zabbix Agent 2 (host-level, outside Coolify)

Agent2 runs directly on the server, not inside Coolify.

```bash
# Add Zabbix repository (Ubuntu/Debian)
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu$(lsb_release -rs)_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu$(lsb_release -rs)_all.deb
sudo apt update

# Install agent2
sudo apt install zabbix-agent2 zabbix-agent2-plugin-*

# Configure
sudo nano /etc/zabbix/zabbix_agent2.conf
# Set:
#   Server=<zabbix-server-container-IP-or-name>
#   ServerActive=<zabbix-server-container-IP-or-name>
#   Hostname=<your-server-hostname>

# Enable Docker monitoring (agent2 needs Docker socket access)
sudo usermod -aG docker zabbix
sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2
```

### Connecting agent2 to Zabbix Server container

The agent2 on the host needs to reach the Zabbix Server container.
Two options:

**Option 1 (simple):** Find zabbix-server container IP:
```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <zabbix-server-container-name>
```
Use this IP in agent2 config. Note: IP may change on redeploy.

**Option 2 (stable):** Enable "Connect to Predefined Network" on the Zabbix stack in Coolify.
Then agent2 can reach zabbix-server via the Coolify predefined network.
The container name will include a UUID suffix — check with `docker ps`.

## Notes
- `log-bin-trust-function-creators=1` is required for MySQL with Zabbix 6.0+
- No custom Docker networks declared — Coolify handles networking
- Volumes persist MySQL data across redeployments
- Port 10051 is NOT exposed externally — only used internally between agent and server
