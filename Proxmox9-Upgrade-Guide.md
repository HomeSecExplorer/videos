# Documentation for

#### Upgrade Proxmox 8 to 9 - Fast Guide

[YouTube video](https://youtu.be/XLZyC_1IX_Q)

---

#### Resources  
- [Official guide](https://pve.proxmox.com/wiki/Upgrade_from_8_to_9)

---

## 1  Prep & Sanity Checks

```bash
# Make sure you’re fully up‑to‑date on 8.x
apt update
apt upgrade -y
pve8to9 --full
```

## 2  Switch to Squid 19.2 (for Ceph Users Only)

If you're running Ceph, verify in the official Proxmox upgrade guide whether you need to upgrade Ceph **before** upgrading PVE.

In `/etc/apt/sources.list.d/ceph.list` change the repo to **squid**
`deb http://<REPO>download.proxmox.com/debian/ceph squid main`

Alternatively, you can remove the old `.list` repository files and switch to the newer deb822-style repository format:

- Enterprise

```bash
cat > /etc/apt/sources.list.d/ceph.sources << EOF
Types: deb
URIs: https://enterprise.proxmox.com/debian/ceph-squid
Suites: trixie
Components: enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

- No-subscription

```bash
cat > /etc/apt/sources.list.d/ceph.sources << EOF
Types: deb
URIs: http://download.proxmox.com/debian/ceph-squid
Suites: trixie
Components: no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

## 3  Switch Debian Base Repos to 13 “Trixie”

```bash
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list
```

## 4  Switch the Proxmox Repository to 9

- Enterprise

```bash
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list.d/pve-enterprise.list
```

Alternatively, you can remove the old `.list` repository files and switch to the newer deb822-style repository format:

```bash
cat > /etc/apt/sources.list.d/pve-enterprise.sources << EOF
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: pve-enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

- No-subscription

Allready changed in step **2**.
Alternatively, you can remove the old repository in `/etc/apt/sources.list` and switch to the newer deb822-style repository format:

```bash
cat > /etc/apt/sources.list.d/proxmox.sources << EOF
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

## 5  Refresh APT & Verify

```bash
apt update
pve8to9
```

## 6  Do the Actual Dist-upgrade

```bash
apt dist-upgrade
```

## 7  Reboot PVE

```bash
reboot
```

## 8  Post-upgrade Validation

```bash
# Confirm versions
pveversion  # should show pve-manager/9.x
uname -r    # >6.14.x-pve

# Quick smoke test
qm list              # VMs present?
pct list             # CTs present?
ping -c3 8.8.8.8     # network okay?
```

## 9  (Optional) Update Old Repos to New deb822 Style

```bash
apt modernize-sources
```
