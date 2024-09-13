# Documentation for

#### Basic Graylog Installation & Log Your Server: Step-by-Step Guide

[YouTube video](https://youtu.be/5szt-IHc9XI)

##

#### Graylog Doc's

[Latest](https://go2docs.graylog.org/current/downloading_and_installing_graylog/debian_installation.htm)

##

## *Pre Install*

**When you run on a Proxmox VM change cpu to host**

```
sudo timedatectl set-timezone UTC
sudo apt-get -y install lsb-release ca-certificates curl gnupg gnupg2
```

## *Install MongoDB*

Add GPG key and repository:
```
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /etc/apt/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ signed-by=/etc/apt/keyrings/mongodb-server-7.0.gpg ] http://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

Update package list and install MongoDB:
```
sudo apt update
sudo apt install -y mongodb-org
```

Start and enable MongoDB service:
```
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
sudo systemctl status mongod.service
```

## *Install OpenSearch*

Add GPG key and repository:
```
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /etc/apt/keyrings/opensearch-keyring
echo "deb [signed-by=/etc/apt/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
```

Update the package list and install OpenSearch (currently maximum supported version is 2.15):
```
sudo apt update
sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD=$(openssl rand -base64 32) apt -y install opensearch=2.15.0
```

To prevent OpenSearch from being upgraded, mark it for hold:
```
sudo apt-mark hold opensearch
```

Edit the OpenSearch configuration file:
```
sudo nano /etc/opensearch/opensearch.yml
```
Change the **plugins.security.ssl.http.enabled** from *true* to **false**
```
plugins.security.ssl.http.enabled: false
```

Add the following to the configuration:
```
cluster.name: graylog
node.name: ${HOSTNAME}
network.host: 127.0.0.1
discovery.type: single-node
action.auto_create_index: false
plugins.security.disabled: true
```

Edit Java options:
```
sudo nano /etc/opensearch/jvm.options
```
Set **Xms** and **Xmx** value to **half** of your systems memory

Set kernel parameter:
```
sudo sysctl -w vm.max_map_count=262144
```
```
sudo nano /etc/sysctl.conf
```
Add:
```
vm.max_map_count=262144
```

Start and enable the OpenSearch service:
```
sudo systemctl enable opensearch.service
sudo systemctl start opensearch.service
sudo systemctl status opensearch.service
```

## *Install Graylog*

Add the repository:
```
wget https://packages.graylog2.org/repo/packages/graylog-6.0-repository_latest.deb
sudo dpkg -i graylog-6.0-repository_latest.deb
```

Update the package list and install Graylog:
```
sudo apt-get update
sudo apt-get install graylog-server 
```

Generate a secret for Graylog:
```
openssl rand -base64 96
```
Example output:
  - *7xUdq9ztGVbG/nF/G70X232k36FPqaQCEmr/chS9cOzMj6xx716pnbj3z7R0nw+vZ7907lqXT7M7LBwAvY5HMPdCluq9dEIBo4cqtxyxVWYP0pq7bqcxUygG5bgx+vwO*

Generate the admin password:
```
echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
```
This will prompt you to enter a password, and produce an output like this:
  - *Enter Password: SecretPassw0rd*
  - *e6a97b6c2bbca8a2141de63e1bce00ba93fad72573adce185100b8c1928b8a2e*

Edit the Graylog configuration file:
```
sudo nano /etc/graylog/server/server.conf
```

And set the following options:
  - password_secret = *7xUdq9ztGVbG/nF/G70X232k36FPqaQCEmr/chS9cOzMj6xx716pnbj3z7R0nw+vZ7907lqXT7M7LBwAvY5HMPdCluq9dEIBo4cqtxyxVWYP0pq7bqcxUygG5bgx+vwO* **replace with your secret**
  - root_password_sha2 = *e6a97b6c2bbca8a2141de63e1bce00ba93fad72573adce185100b8c1928b8a2e* **replace with your secret**
  - root_email = *YourMail*
  - http_bind_address = 0.0.0.0:9000
  - elasticsearch_hosts = http://127.0.0.1:9200
  - transport_email_* set to your needs

Start and enable the Graylog service:
```
sudo systemctl enable graylog-server.service
sudo systemctl start graylog-server.service
sudo systemctl status graylog-server.service
```

Optional: Mark these packages on hold:
```
sudo apt-mark hold mongodb-org
sudo apt-mark hold graylog-server
```

## *Web GUI*

Open **https://YouIP:9000** in your browser
Login:
  - User: **admin**
  - Password: **Your secret password**

Add a Syslog input:
  - System-->Inputs
  - Select **Syslog UDP**
    - Title: **Syslog**
    - Tick **Store full message**

Under System --> Content Packs, install packages according to your needs

Add an alert:
  - Alerts click Event Definitions
  - Create
    - Title: **SSH**
    - Next
    - Condition Type: **Filter & Aggregation**
    - Search Query: **Failed user ssh**
    - Search within the last: **1** minutes
    - Execute search every: **1** miuntes
    - Next
    - Next
    - Create Notification according to your needs
    - Next
    - Create

## *Send Syslog to Graylog*

On your systems, add a rsyslog configuration file:
```
sudo nano /etc/rsyslog.d/remote.conf
```
Add:
```
*.*@YourGraylogIP:514;RSYSLOG_SyslogProtocol23Format
```

Restart rsyslog:
```
sudo systemctl restart rsyslog.service
sudo systemctl status rsyslog.service
```
