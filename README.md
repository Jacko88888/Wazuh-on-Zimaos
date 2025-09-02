# Wazuh-on-Zimaos
Guide to run Wazuh on ZimaOS with Docker, fix authd port 1515, enroll a Docker agent, and verify in the UI.
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
