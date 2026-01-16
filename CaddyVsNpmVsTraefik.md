# Reverse Proxy 101: Caddy vs Nginx Proxy Manager vs Traefik

[YouTube Video](https://youtu.be/igInrPwZuzA)

---

## Overview

A **reverse proxy** sits in front of your services and handles:

- **HTTPS/TLS** (certificates, redirects, HSTS)
- **Routing** (which domain/path goes to which container/VM)
- **Auth** (basic auth, SSO/OAuth in some setups)
- **Rate limiting/IP allowlists** (depending on proxy)
- **Observability** (access logs, metrics)

This guide gives you a **homelab-first** comparison and a **working baseline setup** for:

- **Caddy** (simple + automatic HTTPS)
- **Nginx Proxy Manager (NPM)** (GUI-first)
- **Traefik** (dynamic Docker/Kubernetes-first)

### Versions (examples used)

- Caddy: 2.10.2  
- Nginx Proxy Manager: 2.13.5  
- Traefik: 3.6.6

---

## What you need

- A Linux host (I used Debian 13) running Docker and Docker Compose
- A domain name that resolves publicly to your WAN IP (DDNS is fine), e.g. `lab.example.com`
- Router/NAT port-forwarding to your proxy host:
  - TCP **80** --> proxy host (for Let’s Encrypt HTTP-01)
  - TCP **443** --> proxy host (for public HTTPS access) **optional**
- (Recommended) A dynamic DNS updater if your WAN IP changes

### HTTP-01 challenge (port 80)

For **standard Let’s Encrypt certificates** without DNS automation, we’ll use the **HTTP-01** challenge.

That means:

- Your domain’s **A/AAAA record must point to your public WAN IP**
- Port **80** must be reachable from the internet and forwarded to the proxy

> If you want **LAN-only** certificates without opening ports, you need DNS-01 instead.

---

## Quick decision guide

### Pick **Caddy** if…

- You want the **simplest** “it just works” HTTPS setup
- You prefer **config in a file** (Caddyfile)
- You want automatic TLS with minimal effort

### Pick **Nginx Proxy Manager** if…

- You want a **GUI** to manage hosts/certs
- You’re managing many routes and want click-ops
- You want easy Let’s Encrypt + access lists in UI

### Pick **Traefik** if…

- Your services change often (containers come/go)
- You want **labels-based** routing (GitOps-friendly)
- You may later move to **Kubernetes**

---

## Common architecture patterns

### Public-facing proxy (required for HTTP-01)

- DNS points `lab.example.com` to your **public WAN IP**
- Router forwards **80**_/443_ to your proxy host
- Proxy routes to internal services
- Use strict firewall rules, MFA, and limit exposed services

---

## HTTP-01: standard Let’s Encrypt certificates

We’ll request certificates using the **HTTP-01** challenge.

### Pick your hostnames

For this guide we’ll use:

- `whoami.<YOUR_DOMAIN>`

### Point DNS to your WAN IP

Create an A/AAAA record so:

- `whoami.<YOUR_DOMAIN>` or general `<YOUR_DOMAIN>` --> `<YOUR_WAN_IP>`

### Forward ports to the proxy

On your router/firewall:

- Forward **TCP 80** --> `<PROXY_LAN_IP>:80`
- Optionaly: Forward **TCP 443** --> `<PROXY_LAN_IP>:443`

> Let’s Encrypt must be able to reach `http://whoami.<YOUR_DOMAIN>/.well-known/acme-challenge/...` on port **80**.

---

## Baseline: test service (whoami)

Create a working directory:

```bash
mkdir -p ~/reverse-proxy-lab && cd ~/reverse-proxy-lab
```

Create a shared proxy network (run once):

```bash
docker network create proxy
```

Create `compose.whoami.yml`:

```yaml
services:
  whoami:
    image: traefik/whoami:latest
    container_name: whoami
    restart: unless-stopped
    networks: [proxy]

networks:
  proxy:
    external: true
```

We’ll reuse this service across proxies.

**Start the service:**

```bash
docker compose -f compose.whoami.yml up -d
```

---

> Note: only **one** reverse proxy can bind to ports **80/443** on the same host at a time. Stop the other proxy container(s) before starting a different one.

## Caddy

**Compose:**

Caddy can request Let’s Encrypt certs automatically using **HTTP-01** as long as port **80** is reachable.

Create `compose.caddy.yml`:

```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks: [proxy]

volumes:
  caddy_data:
  caddy_config:

networks:
  proxy:
    external: true
```

**Caddyfile routing:**

Create `Caddyfile`:

```caddyfile
{
  email <YOUR_EMAIL>
}

whoami.<YOUR_DOMAIN> {
  reverse_proxy whoami:80
}
```

**Start:**

```bash
docker compose -f compose.caddy.yml up -d
```

**Verify:**

Open: `https://whoami.<YOUR_DOMAIN>`

**Stop:**

```bash
docker compose -f compose.caddy.yml down
```

---

## Nginx Proxy Manager

**Compose:**

Create `compose.npm.yml`:

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"   # admin UI
      - "443:443"
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    networks: [proxy]

volumes:
  npm_data:
  npm_letsencrypt:

networks:
  proxy:
    external: true
```

**Start:**

```bash
docker compose -f compose.npm.yml up -d
```

**First login:**

Open: `http://<PROXY_IP>:81`

Create your admin user

**Add a Proxy Host (GUI):**

1. **Hosts --> Proxy Hosts --> Add Proxy Host**
1. Domain Names: `whoami.<YOUR_DOMAIN>`
1. Scheme: `http`
1. Forward Hostname/IP: `whoami`
1. Forward Port: `80`
1. **SSL tab --> Request a new SSL Certificate**
1. Enable **Force SSL** (optional: HTTP/2)
1. Save

**Verify:**

Open: `https://whoami.<YOUR_DOMAIN>`

**Stop:**

```bash
docker compose -f compose.npm.yml down
```

## Traefik (dynamic + labels, HTTP-01)

Traefik shines when you want “deploy container --> route appears automatically”.

**Compose:**

Create `compose.traefik.yml`:

```yaml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      # Let's Encrypt via HTTP-01 (port 80)
      - "--certificatesresolvers.le.acme.email=<LE_EMAIL>"
      - "--certificatesresolvers.le.acme.storage=/acme.json"
      - "--certificatesresolvers.le.acme.httpchallenge=true"
      - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"

    ports:
      - "80:80"
      - "443:443"
      # - "8080:8080" # dashboard (lock this down!)

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./acme.json:/acme.json

    networks: [proxy]

networks:
  proxy:
    external: true
```

**Prepare ACME storage:**

Traefik stores certs in `acme.json`. Create it and lock permissions:

```bash
touch acme.json
chmod 600 acme.json
```

**Update or create a new whoami to use labels:**

Update `compose.whoami.yml` or create `compose.whoami.traefik.yml`:

```yaml
services:
  whoami:
    image: traefik/whoami:latest
    container_name: whoami
    restart: unless-stopped
    networks: [proxy]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.<YOUR_DOMAIN>`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=le"
networks:
  proxy:
    external: true
```

**Start:**

```bash
docker compose -f compose.traefik.yml up -d
```

```bash
# If you created compose.whoami.traefik.yml
docker compose -f compose.whoami.yml down
docker compose -f compose.whoami.traefik.yml up -d
# Or if you updated compose.whoami.yml
docker compose -f compose.whoami.yml down
docker compose -f compose.whoami.yml up -d
```

**Verify:**

- `https://whoami.<YOUR_DOMAIN>`
- Dashboard: `http://<PROXY_IP>:8080` **if enabled** (protect it!)

---

## Security checklist (do this regardless of proxy)

- Do not expose admin dashboards (NPM: `:81`, Traefik: `:8080`) to the internet
- Add MFA/SSO if you expose anything publicly
- Use IP allowlists for admin endpoints
- HTTP-01 requires port **80** reachable from the internet; if you can’t open ports, switch to DNS-01
- Keep the proxy updated (image updates + restart policy)
- Log access requests somewhere (Loki/Graylog/Wazuh)
