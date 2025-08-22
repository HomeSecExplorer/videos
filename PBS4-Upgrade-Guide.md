# Documentation for

#### How to Upgrade PBS 3.4 to 4.0 (Under 5 Minutes)

[YouTube video](https://youtu.be/KTtlgEJBPVM)

---

#### References
- [Official guide](https://pbs.proxmox.com/wiki/Upgrade_from_3_to_4)

---

## Safety First
- Verify existing backups.
- Optional: enable read‑only maintenance mode on datastores during the upgrade.

---

## 1  Prep & Sanity Checks

```bash
apt update
apt upgrade -y
pbs3to4 --full
```

## 2 (Optional) Maintenance Mode
```bash
proxmox-backup-manager datastore update <DATASTORE> --maintenance-mode read-only
```

## 3  Switch Debian Base Repos to 13 “Trixie”

```bash
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list
```

## 4  Switch the PBS Repository to 4

- Enterprise

```bash
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list.d/pbs-enterprise.list
```

Alternatively, you can remove the old `.list` repository files and switch to the newer deb822-style repository format:

```bash
cat >/etc/apt/sources.list.d/pbs-enterprise.sources <<'EOF'
Types: deb
URIs: https://enterprise.proxmox.com/debian/pbs
Suites: trixie
Components: pbs-enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```
- No-subscription

Allready changed in step **3**.
Alternatively, you can remove the old repository in `/etc/apt/sources.list` and switch to the newer deb822-style repository format:

```bash
cat >/etc/apt/sources.list.d/proxmox.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pbs
Suites: trixie
Components: pbs-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

## 5  Refresh APT & Verify

```bash
apt update
pbs3to4
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
systemctl status proxmox-backup-proxy.service proxmox-backup.service
proxmox-backup-manager versions
uname -r

# Quick smoke test
proxmox-backup-manager datastore list  # Datastores present?
proxmox-backup-manager datastore update <DATASTORE> --delete maintenance-mode
ping -c3 8.8.8.8     # network okay?
```

## 9  (Optional) Update Old Repos to New deb822 Style

```bash
apt modernize-sources
```
