# FleetDM – Complete Homelab Guide

[YouTube Video](https://youtu.be/XPC33deBf8Q)

---

## Overview

FleetDM is a modern **endpoint visibility and management platform** built on top of **osquery**.

It allows you to:

- Inventory software and hardware
- Run live queries across endpoints
- Detect vulnerabilities (CVEs)
- Enforce security policies
- Monitor Linux, Windows, and macOS systems

This guide shows how to deploy **FleetDM using Docker**, enroll endpoints, and use it effectively in a homelab or lab environment that mirrors production workflows.

---

## Requirements

- Debian / Ubuntu host (VM or bare metal)
- 4 CPU cores
- 8 GB RAM (minimum)
- Docker + Docker Compose
- At least one endpoint:
  - Linux, Windows, or macOS

---

## 1. Setup Fleet with Docker Compose

### Download the official Compose files

```bash
mkdir -p ~/fleetdm && cd ~/fleetdm

curl -O https://raw.githubusercontent.com/fleetdm/fleet/refs/heads/main/docs/solutions/docker-compose/docker-compose.yml
curl -O https://raw.githubusercontent.com/fleetdm/fleet/refs/heads/main/docs/solutions/docker-compose/env.example
cp env.example .env
```

#### Set required values in `.env`

Generate a server key:

```bash
openssl rand -base64 32
```

Edit `.env` and set at least:

```bash
MYSQL_ROOT_PASSWORD=<YOUR_SECURE_ROOT_PASSWORD>
MYSQL_PASSWORD=<YOUR_SECURE_FLEET_DB_PASSWORD>
FLEET_SERVER_PRIVATE_KEY=<PASTE_THE_BASE64_KEY_HERE>
```

### TLS choice (pick one)

**Option A (recommended): reverse proxy terminates TLS**

If you put Fleet behind Caddy/Traefik/Nginx, set:

```bash
FLEET_SERVER_TLS=false
```

**Option B: Fleet serves HTTPS directly (simple lab)**

Create self-signed certs:

```bash
mkdir -p certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout certs/fleet.key   -out certs/fleet.crt   -subj "/CN=<FQDN_OR_IP>"
chmod 644 certs/fleet.key certs/fleet.crt
```

And set:

```bash
FLEET_SERVER_TLS=true
```

### Start Fleet

```bash
docker compose up -d
docker compose ps
```

> Notes (important)
> - The official stack includes a one-time container called **fleet-init** that fixes Docker volume permissions.
>   Expect it to show as `Exited (0)` after the first run, that’s normal.
> - Fleet runs DB migrations automatically on startup.

### Install fleetctl

1. Install [Node.js](https://nodejs.org/en/download) including npm
   Example:
   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
   \. "$HOME/.nvm/nvm.sh"
   nvm install 24
   node -v
   npm -v
   ```
1. Install fleetctl: `npm install -g fleetctl`
1. Configure fleetctl: `fleetctl config set --address https://127.0.0.1:1337 --tls-skip-verify`

---

## 2. Initial Fleet Setup

Open your browser:

- If `FLEET_SERVER_TLS=true`: `https://<FLEET_HOST>:1337`
- If `FLEET_SERVER_TLS=false`: `http://<FLEET_HOST>:1337`

Follow the guide for initial setup.

---

## 3. Enroll hosts (Linux example)

1. In the Fleet web UI, go to **Hosts**
1. Click **Add hosts**
1. Open the **Linux** tab
1. Choose **Workstation** or **Server**
1. Copy the command shown (it will be a `fleetctl package ...` command with the correct `--fleet-url` and `--enroll-secret`)
   - Make sure you set `--type=` matches your Linux distribution (e.g.,deb for Debian/Ubuntu,rpm for Rocky/RHEL).

Run the copied command on your admin machine (the Docker host where you installed fleetctl). This generates a Linux installer package.
> If you’re using a self-signed certificate, add `--insecure` or `--fleet-certificate=<certs/fleet.crt>`.

### Install `fleetd` on the Linux host (install the generated package)

1. Copy the generated package to the Linux host you want to enroll.
1. Install the package, e.g. `sudo dpkg -i fleet-osquery_1.50.2_amd64.deb`
1. Check that the agent is running: `sudo systemctl status orbit.service`

After a minute or two, the host should appear in **Fleet --> Hosts**.

---

## 4. Understanding Hosts & Inventory

In **Hosts**, Fleet shows:

- OS version
- Kernel
- Installed software
- Hardware details
- Last check-in time

This replaces manual asset tracking.

---

## 5. Running Live Queries

Go to **Queries --> Add query**.

Example: list logged-in users

```sql
SELECT * FROM logged_in_users;
```

Example: running SSH processes

```sql
SELECT pid, name, path FROM processes WHERE name = 'sshd';
```

Run queries on:

- A single host
- All hosts
- A label

Results return in real time.

---

## 6. Policies (Compliance Checks)

Policies are queries that return **pass / fail**.

Example: Check if `htop` is installed

```sql
SELECT 1 FROM deb_packages WHERE name = 'htop';
```

Fleet will show:

- Passed
- Failed
- Unknown

This is **audit**, not enforcement.

---

## 7. Controls (OS settings, Scripts, Variables)

In Fleet you’ll also see a **Controls** section. This is where Fleet groups features that go beyond “query + inventory” and moves into **device management**.

What you’ll typically find there:

- **OS settings**: manage configuration profiles.
- **Scripts**: run scripts on hosts.
- **Variables**: reusable variables for scripts/policies.

> **Important:** Some Controls features require enabling **MDM** (Mobile Device Management) and may be platform- or edition-dependent.
> In many homelabs you’ll use Fleet mostly for **inventory + queries + policies**, and only use Controls if you manage macOS/Windows endpoints.

### Optional: enable MDM (if you want OS settings)

1. Go to **Controls --> OS settings**
2. Click **Turn on** (this enables Fleet MDM features)
3. Follow the on-screen instructions to complete the setup for your platform

If you only manage Linux servers, you can skip this section for now and focus on **Hosts**, **Queries**, and **Policies**.

### Run scripts on hosts (optional)

Scripts let you run **one-off** or **on-demand** actions from the Fleet UI (think: “remote maintenance”, not full config management).

1. Go to **Controls --> Scripts**
2. Click **Add script**
3. Name: `Install htop`
4. Script (example):

   ```bash
   #!/bin/bash
   set -euo pipefail
   sudo apt-get update
   sudo apt install htop -y
   ```

5. Save

**Run it:**

- Go to **Hosts --> select a host --> Actions --> Run script**
- Pick `Install htop` and confirm

**Where to see results:**

- Host page --> **Activity**
- Or **Controls --> Scripts** (recent runs / output)

> Tip: Use **Controls --> Variables** to store values (API tokens, URLs, etc.) and reference them in scripts so you don’t hardcode secrets.

**Security note:** Scripts run with the agent’s privileges. Treat this like SSH access: restrict who can run scripts and audit changes.

---

## 8. Vulnerability Management

Fleet automatically:

- Detects installed software
- Matches CVEs

Go to **Software-->Vulnerabilities**:

- Review detected CVEs
- View affected hosts

This works without additional agents.

---

## Common Use Cases

- Asset inventory
- Software tracking
- Compliance check

Fleet excels at **visibility**, not configuration enforcement.

---

## Fleet vs Other Tools

| Tool | Strength |
|----|----|
| FleetDM | Endpoint visibility, queries, CVEs |
| Rudder | Configuration enforcement |

Fleet complements, it does not replace.
