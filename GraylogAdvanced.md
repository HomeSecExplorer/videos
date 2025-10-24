# Documentation for

#### Graylog Advanced | Log Collection and Alerts

[YouTube video](https://youtu.be/iI44ANoylsU)

This guide builds on the base Graylog installation and adds:

- Linux system log collection with **rsyslog**
- Docker container logs via **GELF UDP**
- Dashboard
- Alerting for SSH brute‑force and Docker errors

---

#### Graylog Doc's

[Latest](https://go2docs.graylog.org/current/downloading_and_installing_graylog/debian_installation.htm)

---

## Requirements

- Graylog 6.3 running and reachable via Web UI
- Syslog and GELF UDP ports open (514 and 12201)
- Linux host with **rsyslog**
- Docker host using the **GELF logging driver**

---

## Prerequisites

### Create Inputs in Graylog

**Syslog UDP input**  
Go to:  
`System --> Inputs --> Syslog UDP --> Launch new input`  
Set:

- Title: Syslog UDP 514
- Bind address: 0.0.0.0
- Port: 514
- Store full message: true

Click `Set-up Input`:

- `Create Stream`
- Title: Syslog

If port 514 fails, use 1514 instead.

**GELF UDP input**  
Go to:  
`System --> Inputs --> GELF UDP --> Launch new input`  
Set:

- Title: GELF UDP 12201
- Bind address: 0.0.0.0
- Port: 12201

Click `Set-up Input`:

- `Create Stream`
- Title: Docker Logs

Start both inputs after saving.

---

## Linux Log Forwarding (rsyslog)

On your Linux host, create or edit:  
`/etc/rsyslog.d/90-remote-syslog.conf`  
Add this line:

```conf
*.* @<GRAYLOG_IP>:514;RSYSLOG_SyslogProtocol23Format
```

Restart rsyslog:

```bash
sudo systemctl restart rsyslog
```

Verify that logs appear in Graylog under **Search**.

> If you used port 1514 for the input, update the config line accordingly.

---

## Extend Syslog

**Extract SSH data**  
Add a regex extractor on the Syslog UDP input with this pattern:

`System --> Inputs --> Syslog UDP --> Manage extractors -- > Create extractor`  
On `message` click `Select extractor type --> Regular expression`

SSH User:

- Regular expression: `Failed password for[ invalid user]? (\S+) from \d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}`
- Store as field: `ssh_user`
- Extractor titel: SSH User

SSH source IP:

- Regular expression: `Failed password for[ invalid user]? \S+ from (\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})`
- Store as field: `ssh_src_ip`
- Extractor titel: SSH source IP

---

## Docker Log Forwarding

In your Docker Compose file, add GELF logging:

Example:

```yaml
services:
  app:
    image: nginx:latest
    logging:
      driver: gelf
      options:
        gelf-address: "udp://<GRAYLOG_IP>:12201"
        tag: "nginx"
```

Apply changes:

```bash
docker compose up -d
```

Logs will appear in Graylog under the `Docker Logs` stream.

---

## Dashboards

Create a dashboard named `Overview`.

Create a `Aggregation` widget for each:

**Failed SSH logins**

- Stream: Syslog
- Query: `message:"Failed password" OR message:"Invalid user"`
- Metric: Count
- Group by: timestamp
- Visualization: Line chart

**Top attacking IPs**

- Stream: Syslog
- Query: `message:"Failed password" OR message:"Invalid user"`
- Group by: `ssh_src_ip`
- Metric: Count
- Visualization: Bar chart

**Containers with errors**

- Stream: Docker Logs
- Query: `level:>=3 OR message:ERROR`
- Group by: `container_name`
- Metric: Count
- Visualization: Bar chart

**Service restarts**

- Stream: Syslog
- Query: `application_name:systemd AND (message:"Started " OR message:"Stopped ")`
- Metric: Count
- Group by: timestamp
- Visualization: Area chart

Save the dashboard and set the time range to **Last 24 hours**.

---

## Alerts

Go to `Alerts & Events --> Event Definitions --> Create new`

**SSH brute‑force detection**

- Title: SSH brute‑force spike
- Priority: High
- Condition Type: Filter & Aggregation
- Stream: Syslog
- Search Query: `message:"Failed password" OR message:"Invalid user"`
- Search within the last: 5 minutes
- Execute search every: 5 minutes
- Create Events for Definition if: Aggregation of results reaches a threshold
  - Condition: If `count`, is `>=`, Threshold `10`
- Notification: e.g. Email

**Container error surge**

- Title: Container error surge
- Priority: Medium
- Condition Type: Filter & Aggregation
- Stream: Docker Logs
- Search Query: `level:>=3 OR message:ERROR`
- Search within the last: 10 minutes
- Execute search every: 10 minutes
- Create Events for Definition if: Aggregation of results reaches a threshold
  - Condition: If `count`, is `>=`, Threshold `50`
- Notification: e.g. Email
