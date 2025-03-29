# Documentation for
#### Install Gatus: Better Than Uptime Kuma? | Monitor All Your Services Easily
[YouTube video](https://youtu.be/CrDiMWzEoKw)

##
#### Gatus Doc's
<https://github.com/TwiN/gatus?tab=readme-ov-file#table-of-contents>

##

## Create config file
####  *Create folder*

```
mkdir -p ~/gatus
cd ~/gatus
```

####  *Edit config*

```
nano config.yaml
```

####  *Config*

```
# storage:
#   type: postgres
#   path: "postgres://<POSTGRES_USER>:<POSTGRES_PASSWORD>@<POSTGRES_SERVER>:5432/<GATUS_DB>?sslmode=disable"
metrics: true
alerting:
  discord:
    webhook-url: "https://discord.com/api/webhooks/your_webhook"
    default-alert:
      description: "{{ .Endpoint.Name }} is DOWN!"
endpoints:
  - name: Google
    group: Internet
    url: https://google.com
    interval: 2m
    client:
      insecure: true
    conditions:
      - "[STATUS] == 200"
  - name: DNS
    group: Internet
    url: <DNS_SERVER>
    interval: 2m
    dns:
      query-name: "one.one.one.one"
      query-type: "A"
    conditions:
      - "[BODY] == any(1.1.1.1, 1.0.0.1)"
      - "[DNS_RCODE] == NOERROR"
  - name: PVE Server 1
    group: Server
    url: icmp://<SERVER_IP_URL>
    interval: 2m
    conditions:
      - "[CONNECTED] == true"
  - name: Nextcloud
    group: Service
    url: https://<NEXTCLOUD.EXAMPLE.URL>
    interval: 2m
    client:
      insecure: true
    conditions:
      - "[STATUS] == 200"
```

## Docker
### *Run container*

```
docker run -d \
  --name gatus \
  -p 8080:8080 \
  -v ~/gatus/config.yaml:/config/config.yaml \
  twinproduction/gatus
```
