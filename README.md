<div align="center">

# 🛡️ Wazuh on ZimaOS — Docker Setup & Port 1515 Fix

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
![Platform: ZimaOS](https://img.shields.io/badge/Platform-ZimaOS-informational)
![Wazuh Manager](https://img.shields.io/badge/Manager-4.12.0-blue)
![Wazuh Agent](https://img.shields.io/badge/Agent-4.7.2-blue)
![Docker](https://img.shields.io/badge/Docker-yes-2496ED)

</div>

**TL;DR:** Fix `wazuh-authd` binding to **1515**, set API creds (quote your `!`), run the Docker agent, attach it to the manager’s network, write a proper `<client><server>` block, and verify in the UI.

---

## Table of Contents
- [Screenshots](#screenshots)
- [Requirements](#requirements)
- [Pre-flight checks](#pre-flight-checks)
- [Setup — Manager](#setup--manager)
- [Setup — Agent (Docker)](#setup--agent-docker)
- [Verify](#verify)
- [Troubleshooting](#troubleshooting)
- [Notes & Tips](#notes--tips)
- [Credits](#credits)

---

## Screenshots

| Endpoints (Agent Active) |
| --- |
| ![Agent active](images/dashboard-agents.png) |

| Security Events (filtered to the agent) |
| --- |
| ![Security events](images/security-events.png) |

---

## Requirements

- ZimaOS with Docker
- Wazuh **manager** (container) — name: `single-node-wazuh.manager-1`
- LAN IP of manager (example): `192.168.0.7`
- Shell access to the host running Docker

> Versions used here: **Manager 4.12.0**, **Agent 4.7.2** (community agent image). Works fine for basic monitoring.

---

## Pre-flight checks

```bash
# Manager API answers (401 is OK here):
docker exec -it single-node-wazuh.manager-1 bash -lc 'curl -skI https://localhost:55000 | head -n1'

# authd (1515) is listening (inside manager container):
docker exec -it single-node-wazuh.manager-1 bash -lc '
grep -qi ":05eb" /proc/net/tcp /proc/net/tcp6 && echo "1515 LISTEN present" || echo "1515 not listening"
'
If 1515 is not listening, see Troubleshooting below.

Setup — Manager
Reset the API password for user wazuh (quotes matter if it contains !):

bash
Copy code
docker exec -it single-node-wazuh.manager-1 bash -lc '
/var/ossec/framework/python/bin/python3 - << "PY"
from wazuh.security import update_user
print(update_user(user_id="1", password="AgentBoot!234").render())
PY
'
Optional: allow re-enrollment without waiting:

bash
Copy code
docker exec -it single-node-wazuh.manager-1 bash -lc '
CONF=/var/ossec/etc/ossec.conf
sed -i "/<auth>/,/<\/auth>/c\<auth>\n  <disabled>no</disabled>\n  <port>1515</port>\n  <use_source_ip>no</use_source_ip>\n  <force>\n    <after_registration_time>0</after_registration_time>\n    <key_mismatch>yes</key_mismatch>\n  </force>\n</auth>" "$CONF"
/var/ossec/bin/wazuh-control restart
'
Setup — Agent (Docker)
A) One-liner (scripts included in this repo)
bash
Copy code
# 1) run the agent container
bash scripts/agent-run.sh 192.168.0.7 zimaos-docker-agent

# 2) attach to manager network + write server block + restart agent
bash scripts/link-agent-to-manager.sh
B) Manual (same steps, expanded)
bash
Copy code
# Create a data volume and run the agent container
docker volume create wazuh-agent-data
docker rm -f wazuh-agent 2>/dev/null || true

docker run -d --name wazuh-agent --restart unless-stopped \
  -e JOIN_MANAGER_MASTER_HOST='192.168.0.7' \
  -e JOIN_MANAGER_HOST='192.168.0.7' \
  -e JOIN_MANAGER_PROTOCOL='https' \
  -e JOIN_MANAGER_API_PORT='55000' \
  -e JOIN_MANAGER_USER='wazuh' \
  -e JOIN_MANAGER_PASSWORD='AgentBoot!234' \
  -e JOIN_MANAGER_PORT='1514' \
  -e NODE_NAME='zimaos-docker-agent' \
  -v wazuh-agent-data:/var/ossec \
  kennyopennix/wazuh-agent:latest

# Put the agent on the manager's Docker network
NET=$(docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{printf "%s " $k}}{{end}}' single-node-wazuh.manager-1 | awk '{print $1}')
docker network connect "$NET" wazuh-agent 2>/dev/null || true

# Write <client><server> block and restart the agent
docker exec -it wazuh-agent bash -lc '
CONF=/var/ossec/etc/ossec.conf
if grep -q "<client>" "$CONF"; then
  if grep -q "<server>" "$CONF"; then
    sed -i "/<server>/,/<\/server>/{s#<address>.*</address>#<address>single-node-wazuh.manager-1</address>#; s#<port>.*</port>#<port>1514</port>#; s#<protocol>.*</protocol>#<protocol>tcp</protocol>#}" "$CONF"
  else
    sed -i "s#</client>#  <server>\n    <address>single-node-wazuh.manager-1</address>\n    <port>1514</port>\n    <protocol>tcp</protocol>\n  </server>\n</client>#g" "$CONF"
  fi
else
  sed -i "s#</ossec_config>#  <client>\n    <server>\n      <address>single-node-wazuh.manager-1</address>\n      <port>1514</port>\n      <protocol>tcp</protocol>\n    </server>\n  </client>\n</ossec_config>#g" "$CONF"
fi
/var/ossec/bin/wazuh-control restart
'
Verify
bash
Copy code
# Manager side
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/agent_control -ls

# Agent logs
docker logs --tail=120 wazuh-agent
In the Wazuh UI

Endpoints → Agents: should show active.

Security events: set time range → “Last 15 minutes”, enable Auto-refresh.
Filter WQL:

ini
Copy code
agent.name="zimaos-docker-agent"
Generate a test FIM event

bash
Copy code
docker exec -it wazuh-agent bash -lc 'echo "hello $(date -u)" >> /home/wazuh_fim_test'
Troubleshooting
Port 1515 “Transport endpoint is not connected”
Quickest fix:

bash
Copy code
docker restart single-node-wazuh.manager-1
If still stuck, try:

bash
Copy code
bash scripts/troubleshoot-1515.sh
API login 401 loops
You used ! in the password without quotes. Re-run the container with 'AgentBoot!234' (single quotes).

Agent says “never_connected”
Not on same Docker network as manager → run link-agent-to-manager.sh.

Missing <client><server> block → ensure it points to single-node-wazuh.manager-1:1514.

Notes & Tips
The agent container shows “could not open /var/log/*” — normal. To collect host logs, mount them read-only and add matching <localfile> entries.

UI search power move:

pgsql
Copy code
agent.name="zimaos-docker-agent" AND rule.level>=5
Credits
Real-world debugging on ZimaOS (Docker) with a single-node Wazuh manager.

Community agent image for convenience.

bash
Copy code

---

## 2) Add the scripts

Create **`scripts/agent-run.sh`**:

```bash
#!/usr/bin/env bash
set -euo pipefail
MANAGER_IP=${1:-192.168.0.7}
AGENT_NAME=${2:-zimaos-docker-agent}
WZ_PASS=${WAZUH_API_PASS:-AgentBoot!234}

docker volume create wazuh-agent-data >/dev/null 2>&1 || true
docker rm -f wazuh-agent >/dev/null 2>&1 || true

docker run -d --name wazuh-agent --restart unless-stopped \
  -e JOIN_MANAGER_MASTER_HOST="$MANAGER_IP" \
  -e JOIN_MANAGER_HOST="$MANAGER_IP" \
  -e JOIN_MANAGER_PROTOCOL='https' \
  -e JOIN_MANAGER_API_PORT='55000' \
  -e JOIN_MANAGER_USER='wazuh' \
  -e JOIN_MANAGER_PASSWORD="$WZ_PASS" \
  -e JOIN_MANAGER_PORT='1514' \
  -e NODE_NAME="$AGENT_NAME" \
  -v wazuh-agent-data:/var/ossec \
  kennyopennix/wazuh-agent:latest
Create scripts/link-agent-to-manager.sh:

bash
Copy code
#!/usr/bin/env bash
set -euo pipefail

NET=$(docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{printf "%s " $k}}{{end}}' single-node-wazuh.manager-1 | awk '{print $1}')
docker network connect "$NET" wazuh-agent 2>/dev/null || true

docker exec -it wazuh-agent bash -lc '
set -e
CONF=/var/ossec/etc/ossec.conf
if grep -q "<client>" "$CONF"; then
  if grep -q "<server>" "$CONF"; then
    sed -i "/<server>/,/<\/server>/{s#<address>.*</address>#<address>single-node-wazuh.manager-1</address>#; s#<port>.*</port>#<port>1514</port>#; s#<protocol>.*</protocol>#<protocol>tcp</protocol>#}" "$CONF"
  else
    sed -i "s#</client>#  <server>\n    <address>single-node-wazuh.manager-1</address>\n    <port>1514</port>\n    <protocol>tcp</protocol>\n  </server>\n</client>#g" "$CONF"
  fi
else
  sed -i "s#</ossec_config>#  <client>\n    <server>\n      <address>single-node-wazuh.manager-1</address>\n      <port>1514</port>\n      <protocol>tcp</protocol>\n    </server>\n  </client>\n</ossec_config>#g" "$CONF"
fi
/var/ossec/bin/wazuh-control restart
sleep 4
tail -n 40 /var/ossec/logs/ossec.log
'
Create scripts/troubleshoot-1515.sh:

bash
Copy code
#!/usr/bin/env bash
set -euo pipefail

echo ">>> Restarting manager container..."
docker restart single-node-wazuh.manager-1
sleep 8

echo ">>> Checking 1515 LISTEN inside manager..."
docker exec -it single-node-wazuh.manager-1 bash -lc '
if grep -qi ":05eb" /proc/net/tcp /proc/net/tcp6; then
  echo "1515 LISTEN present."
else
  echo "1515 not listening; cleaning runtime and restarting..."
  /var/ossec/bin/wazuh-control stop || true
  rm -rf /var/ossec/var/run/* /var/ossec/queue/ossec/* /var/ossec/queue/sockets/* 2>/dev/null || true
  /var/ossec/bin/wazuh-control start
  sleep 6
  /var/ossec/bin/wazuh-control status || true
fi
tail -n 60 /var/ossec/logs/ossec.log | egrep -n "authd|Bind|ERROR|INFO" || true
'
Make scripts executable:

bash
Copy code
chmod +x scripts/*.sh
Commit & push:

bash
Copy code
git add README.md images/* scripts/*
git commit -m "Docs + scripts: Wazuh on ZimaOS, port 1515 fix, Docker agent"
git push
