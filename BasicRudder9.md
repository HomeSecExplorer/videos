# Rudder 9 - Automated Patch Management & Drift Detection

[YouTube Video](https://youtu.be/wHe1C18nmsA)

---

## Overview

This guide walks you through installing **Rudder 9** on Debian 13, enrolling one Linux node, enabling **automated patch management**, and using **drift detection** to ensure system compliance.

Rudder is a configuration management and compliance platform ideal for homelabs that want production‑grade workflows.

---

## Requirements

- Debian 13 VM
- Internet access
- One managed Linux node (Debian, Ubuntu, Rocky, etc.)
- SSH access between server and nodes

---

## 1. Install Rudder Server

### Add Rudder repo

```bash
sudo apt update
sudo apt install -y gnupg
sudo wget --quiet -O /etc/apt/keyrings/rudder_apt_key.gpg "https://repository.rudder.io/apt/rudder_apt_key.gpg"
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/rudder_apt_key.gpg] http://repository.rudder.io/apt/9.0/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/rudder.list
sudo apt update
```

### Install Rudder Server components

```bash
sudo apt install -y rudder-server
```

Create a User:

```bash
sudo rudder server create-user -u <USERNAME>
```

> When prompted enter a password

Access the web UI: `https://<RUDDER_IP>/rudder`  
Login with:

- **Username:** `<USERNAME>`
- **Password:** `<PASSWORD>`

---

## 2. Install Rudder Agent on a Node

Run this on the node you want to manage:

```bash
sudo apt update
sudo apt install -y gnupg
sudo wget --quiet -O /etc/apt/keyrings/rudder_apt_key.gpg "https://repository.rudder.io/apt/rudder_apt_key.gpg"
sudo echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/rudder_apt_key.gpg] http://repository.rudder.io/apt/9.0/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/rudder.list
sudo apt update
sudo apt install -y rudder-agent
sudo rudder agent policy-server <rudder server ip or hostname>
# force agent run and send inventory
sudo rudder agent inventory
```

On the Rudder server, approve the node: **Nodes --> Pending --> Accept Node**

Open the Node View.  
Here you can view: Node Score, Node inventory (OS, packages, services), Compliance, applied directives/rules.

---

## 3. Compliance

Rudder can audit and enforce compliance based on policies you define.

### Create a Directive

1. Navigate to **Configuration --> Directives**
2. Packages --> Click `Create`
3. Name: **Ensure htop and curl installed**
4. Parameters:
   a. Package name: `htop`
   b. **Add another 'Package'**
   c. Package name: `curl`
   d. (Optional) Set any additional options
5. Click **Save**

This directive tells Rudder: **“On agent run, ensure these packages are installed.”**

### Add Directive to Rule

To execute a directive, you must link it to a rule that is assigned to your agent.

1. **Configuration --> Rules**
2. Create or Edit a rule
3. In Rule/Directives
   a. Compliance by directives `Select`
   b. Select your Directives and `Confirm`
4. **Save**

**What happens on each agent run:**

- Rudder’s agent on the node checks the directive under it's rules
- If a listed package is missing (e.g. curl or htop), Rudder will install it automatically
- After execution, compliance status is updated in the node view: installed packages, compliance score, logs

> You can also create your own Techniques to check custom configurations.
>
> - File content checks (e.g. SSH config)
> - Service status (enabled / running)
> - Configuration file permissions
> - Cron jobs, environment settings, etc.

---

## 4. Removing a Node

To remove a system cleanly: **Node management --> Nodes --> Select Node --> Delete**

On the node:

```bash
sudo apt remove rudder-agent
```

---

## Next Steps

- Add more nodes (Debian, Ubuntu, Rocky)
- Enforce SSH hardening policies
- Configure default settings enforcement
