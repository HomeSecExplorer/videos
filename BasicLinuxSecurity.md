# Documentation for

#### 10 Essential Steps to Harden Your Linux Server

[YouTube video](https://youtu.be/DmR8cWqlxPI)

##

## First Hardening Steps

> ### Example on Debian 12

### 1. Set Up SSH Key Authentication and Disable Root Login

On your PC:

```bash
ssh-keygen -t ed25519
ssh-copy-id user@your-server
```

On your Linux server:

```bash
sudo nano /etc/ssh/sshd_config
```

- Set:

    ```bash
    PermitRootLogin no
    PasswordAuthentication no
    ```

- Save & restart service

```bash
sudo systemctl restart ssh
```

### 2. Configure UFW (Firewall)

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw enable
```

### 3. Install Fail2Ban

```bash
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
```

### 4. Enable Unattended Security Updates

```bash
sudo apt install unattended-upgrades
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

- Add:

    ```bash
    APT::Periodic::Update-Package-Lists "1";
    APT::Periodic::Unattended-Upgrade "1";
    ```

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

- Add:

    ```bash
    Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}-security";
    };
    Unattended-Upgrade::Automatic-Reboot "false";
    ```

### 5. Harden Kernel Parameters (sysctl)

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

- Add:

    ```bash
    net.ipv4.icmp_echo_ignore_broadcasts = 1
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.tcp_syncookies = 1
    ```

- Save & load

```bash
sudo sysctl --system
```

### 6. Set Strong Password Policies

```bash
sudo apt install libpam-pwquality
sudo nano /etc/security/pwquality.conf
```

- Set:

    ```bash
    minlen = 14
    dcredit = -1
    ucredit = -1
    ocredit = -1
    lcredit = -1
    ```

### 7. Limit SSH Access and Timeout Settings

```bash
sudo nano /etc/ssh/sshd_config
```

- Add:

    ```bash
    MaxAuthTries 3
    ClientAliveInterval 15
    ClientAliveCountMax 3
    AllowUsers <yourusername>
    ```

- Save & restart service

```bash
sudo systemctl restart ssh
```

### 8. Set Login Banner Warning (Legal or Awareness)

```bash
echo "Unauthorized access is prohibited." | sudo tee /etc/issue.net
sudo nano /etc/ssh/sshd_config
```

- Add:

    ```bash
    Banner /etc/issue.net
    ```

- Save & restart service

```bash
sudo systemctl restart ssh
```

### 9. Enforce Secure File Permissions and umask Defaults

```bash
sudo nano /etc/profile
```

- Set:

    ```bash
    umask 027
    ```

### 10. Disable Unused Network Services and Ports

```bash
sudo ss -tulnp
sudo systemctl list-unit-files --type=service | grep enabled
# Disable unused services, e.g.:
sudo systemctl disable avahi-daemon
sudo systemctl disable cups
```

---

## Further steps/reading

- CIS Benchmark
- OpenSCAP
- SSH-Audit
- Lynis
- AppArmor/SELinux
- auditd
- rkhunter
- ClamAV
