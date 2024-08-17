# Documentation for

#### Basic Grafana Installation & Monitoring Your Server: Step-by-Step Guide

[YouTube video](https://youtu.be/DS0DlEam6JA)

##

#### Grafana Doc's

[Latest](https://grafana.com/docs/grafana/latest/)

##

## Pre-Installation

#### *Generate Certificate*

```
openssl req -x509 -nodes -newkey rsa:4096 -keyout grafana.key -out grafana.crt -sha256 -days 365
```

#### *Generate Prometheus Password Hash*

```
htpasswd -nBC 10 "" | tr -d ':\n'
```
  - Save your hash.

## Prometheus Installation

#### *Install Prometheus*

```
sudo apt install prometheus
```

```
sudo nano /etc/prometheus/prometheus.yml
```
  - Edit to your needs.

#### *Configure Prometheus*

Copy the certificate and set permissions:
```
sudo cp grafana.key /etc/prometheus/prom.key
sudo cp grafana.crt /etc/prometheus/prom.crt
sudo chown prometheus:prometheus /etc/prometheus/prom.*
sudo chmod 400 /etc/prometheus/prom.key /etc/prometheus/prom.crt
```

Edit the web config file:
```
sudo nano /etc/prometheus/web-config.yml
```

Add the following:
```
basic_auth_users:
  prom: YOURHASH
tls_server_config:
  cert_file: /etc/prometheus/prom.crt
  key_file: /etc/prometheus/prom.key
```

Set permissions on the file:
```
sudo chown prometheus:prometheus /etc/prometheus/web-config.yml
```

Edit service ARGS to use the web config file and allow remote-write so Alloy can push to Prometheus:
```
sudo nano /etc/default/prometheus
```

In **ARGS=""**, add:
```
--web.config.file /etc/prometheus/web-config.yml --web.enable-remote-write-receiver
```

Start and enable Prometheus:
```
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl restart prometheus
sudo systemctl disable --now prometheus-node-exporter
sudo systemctl status prometheus
```

## Grafana Alloy

#### *Add Grafana Repository*

Install the GPG package so you can add the Grafana GPG key:
```
sudo apt install gpg
```

Add the Grafana GPG key and repository:
```
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

```

Update the package list and install Grafana Alloy:
```
sudo apt update
sudo apt install alloy
```

#### *Configure Alloy*

Modify or replace the default configuration:
```
sudo nano /etc/alloy/config.alloy
```
```
logging {
  level = "warn"
}

prometheus.remote_write "default" {
  endpoint {
    url = "https://PrometheusServerIP:9090/api/v1/write"
    basic_auth {
      username = "prom"
      password = "PromPassword-NOThash"
    }
    tls_config {
      insecure_skip_verify = true
    }
  }
}

prometheus.exporter.unix "default" {
  include_exporter_metrics = true
  disable_collectors       = ["mdadm"]
}

prometheus.scrape "default" {
  scrape_interval = "10s"
  targets = concat(
    prometheus.exporter.unix.default.targets,
    [{
      // Self-collect metrics
      job         = "alloy",
      __address__ = "127.0.0.1:12345",
    }],
  )

  forward_to = [prometheus.remote_write.default.receiver]
}
```

Start and enable Alloy:
```
sudo systemctl enable alloy
sudo systemctl start alloy
sudo systemctl restart alloy
sudo systemctl status alloy
```

## Install Grafana

```
sudo apt-get install grafana-enterprise
```

#### *Configure Grafana*

Copy the certificate:
```
sudo mv grafana.key /etc/grafana/
sudo mv grafana.crt /etc/grafana/
```

Set permissions:
```
sudo chown grafana:grafana /etc/grafana/grafana.crt
sudo chown grafana:grafana /etc/grafana/grafana.key
sudo chmod 400 /etc/grafana/grafana.key /etc/grafana/grafana.crt
```

Modify the Grafana config:
```
sudo nano /etc/grafana/grafana.ini
```

Under **[server]**, add:
```
protocol = https
cert_file = /etc/grafana/grafana.crt
cert_key = /etc/grafana/grafana.key
```

Start and enable Grafana:
```
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl restart grafana-server
sudo systemctl status grafana-server
```

#### *Grafana GUI*

  - Default user **admin** with password **admin**
  - Set secure password
  - Add a Database *Connections-->Add new connection-->Prometheus-->Add new data source-->*
    - URL **https://127.0.0.1:9090**
    - Authentication **Basic authentication**
      - User **YourPrometheusUser** (prom)
      - Password **PromPassword-NOThash**
    - TLS settings--> set **Skip TLS certificate verification**
    - Save & test
  - Add Dashboard *Dashboards-->New-->New dashboard*
  - Import Dashboard *Dashboards-->New-->Import*

## Dashboard

[Overview](OverviewDashboard.json)

[Linux Host's](LinuxHostsDashboard.json)
