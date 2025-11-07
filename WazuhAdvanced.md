# Wazuh Advanced Setup: Upgrade, Rules, Alerts & Retention

[YouTube video](https://youtu.be/KQohkp3pN2M)

---

## Requirements

- Wazuh **4.12** stack running (manager, indexer, dashboard)
- Access to Wazuh Dashboard (HTTPS)
- At least one Linux endpoint (agent)
- Working SMTP server for email alerts

---

## Upgrading Wazuh Docker Stack

When a new version of Wazuh is released, update your Docker stack safely with these steps.

### Backup before upgrading

```bash
cp -r wazuh-docker wazuh-docker.bak.$(date +%F)
```

### Upgrade

```bash
cd wazuh-docker/single-node
docker compose down
```

If you do not have custom changes, download the new version.

**If you have changes in your Docker file**  
See the official documentation: [Upgrade](https://documentation.wazuh.com/current/deployment-options/docker/upgrading-wazuh-docker.html)

Example for upgrading from `4.12` to `4.14`

In `generate-indexer-certs.yml` set:

```yml
services:
   generator:
      image: wazuh/wazuh-certs-generator:0.0.2
```

Recreate certificates:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
sudo chown -R 1000:1000 config/wazuh_indexer_ssl_certs/
```

Edit `config/wazuh_indexer/wazuh.indexer.yml`:

- Change `/usr/share/wazuh-indexer/certs/` to `/usr/share/wazuh-indexer/config/certs/`

Edit `docker-compose.yml`:

- Change `/usr/share/wazuh-indexer/certs/` to `/usr/share/wazuh-indexer/config/certs/`
- Change `/usr/share/wazuh-indexer/opensearch.yml` to `/usr/share/wazuh-indexer/config/opensearch.yml`
- Change `/usr/share/wazuh-indexer/opensearch-security/internal_users.yml` to `/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml`

Edit `docker-compose.yml` and change the following lines:

```yml
wazuh.manager:
   image: wazuh/wazuh-manager:4.14.0
wazuh.indexer:
   image: wazuh/wazuh-indexer:4.14.0
wazuh.dashboard:
   image: wazuh/wazuh-dashboard:4.14.0
```

Download the new `wazuh_manager.conf` and compare if you have custom changes:

```bash
wget https://raw.githubusercontent.com/wazuh/wazuh-docker/refs/tags/v4.14.0/single-node/config/wazuh_cluster/wazuh_manager.conf -O config/wazuh_cluster/wazuh_manager.conf
diff config/wazuh_cluster/wazuh_manager.conf ../../wazuh-docker.bak.$(date +%F)/single-node/config/wazuh_cluster/wazuh_manager.conf
```

If any customizations are present, reapply them.

### Pull and start containers

```bash
docker compose pull
docker compose up -d
```

### Verify the upgrade

- Dashboard loads and shows the new version.
- Agents show **Active**.

---

## Custom Rules

Edit **local rules** on the manager:  
Go to `Server management --> Rules` and edit `local_rules.xml`

Example:  
Overwrite existing rule *5710* with a higher level

```xml
<group name="syslog,sshd,">
 <rule id="5710" level="10" overwrite="yes">
   <if_sid>5700</if_sid>
    <match>illegal user|invalid user</match>
    <description>sshd: Attempt to login using a non-existent user</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>invalid_login,authentication_failed,pci_dss_10.2.4,pci_dss_10.2.5,pci_dss_10.6.1,gpg13_7.1,gdpr_IV_35.7.d,gdpr_IV_32.2,hipaa_164.312.b,nist_800_53_AU.14,nist_800_53_AC.7,nist_800_53_AU.6,tsc_CC6.1,tsc_CC6.8,tsc_CC7.2,tsc_CC7.3,</group>
  </rule>
</group>
```

Save the file, then **restart** the manager to load rules.

---

## Centralized Agent Configuration

Apply settings to a group:  
`Agents management --> Groups`, on your group click **edit**

Minimal example enabling FIM and logs:

> Examples:  
> FIM `/etc` - is monitored by default, change with your custom path  
> Change log path with your custom log

```xml
<agent_config>
  <syscheck>
    <frequency>43200</frequency>
    <directories realtime="yes" report_changes="yes">/etc</directories>
  </syscheck>

  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/auth.log</location>
  </localfile>
</agent_config>
```

---

## Check

Add a comment to a file under **/etc**, for example `/etc/hosts`, and login via SSH with a non-existing user.  
Check under your agent endpoint that the file change is registered and an event for the SSH login is created.

---

## Email Alerts

Edit the manager configuration:  
`Server management --> Settings`, *Edit configuration*  
Enable notifications and SMTP:

```xml
<global>
  <email_notification>yes</email_notification>
  <email_to>alerts@example.com</email_to>
  <smtp_server>mail.example.com</smtp_server>
  <email_from>wazuh@example.com</email_from>
</global>
```

Optional: alert level threshold (send only alerts ≥ 7):

```xml
<alerts>
  <email_alert_level>7</email_alert_level>
</alerts>
```

Click `Save` and `Restart Manager`.

Test an alert:

- Generate failed SSH logins or trigger your custom rule.
- Confirm the email arrives.

---

## Threat Detection Enhancements

### Vulnerability Detector

Edit the manager configuration:  
`Server management --> Settings`, *Edit configuration*

```xml
<vulnerability-detector>
  <enabled>yes</enabled>
  <interval>5m</interval>
  <run_on_start>yes</run_on_start>
  <provider name="debian">
    <enabled>yes</enabled>
  </provider>
  <provider name="nvd">
    <enabled>yes</enabled>
  </provider>
</vulnerability-detector>
```

Click `Save` and `Restart Manager`.  
Then open **Vulnerability Detection** in agent view to review results.

### MITRE ATT&CK view

- Open **MITRE ATT&CK** in the agent view.
- Filter by agent or rule to see mapped techniques.

---

## Verification

- Agents show **Active** and send events.
- FIM events present under **Integrity monitoring**.
- Email alerts received for level ≥ configured threshold.
- Vulnerabilities populated after first scan.

---

## Index Retention - Auto Delete After 30 Days

You can automatically roll over and delete old Wazuh indices using Index Management in the Wazuh Indexer.

### Create a 30-day retention policy

1. `Index Management --> Index Management` *State management policies*  **Create**
2. Name the policy (e.g., `logrotate`) and paste the following JSON:

```json
{
  "policy": {
    "policy_id": "wazuh-30d-retention",
    "description": "Rollover and delete Wazuh indices after 30 days",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [
          {
            "rollover": {
              "min_index_age": "1d",
              "min_primary_shard_size": "20gb"
            }
          }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": { "min_index_age": "30d" }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [ { "delete": {} } ]
      }
    ],
    "ism_template": [
      {
        "index_patterns": ["wazuh-alerts-4.*", "wazuh-archives-4.*"],
        "priority": 100
      }
    ]
  }
}
```

3. Save the policy.  
4. Go to *Indexes*, select all `wazuh-alerts-4.*` and `wazuh-archives-4.*` indexes, click on **Actions, Apply policy** and apply your newly created policy.

This ensures indices older than 30 days are automatically removed, keeping your stack lightweight.

---

## Next Steps

- Compare Wazuh vs Graylog (SIEM workflow)
- Add Windows/macOS agents
