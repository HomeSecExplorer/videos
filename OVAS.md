# Documentation for

#### Basic OpenVAS Installation & Scan Your Server: Step-by-Step Guide

[YouTube video](https://youtu.be/KUFCrVwUctU)

##

#### OpenVAS Doc's

[Latest](https://greenbone.github.io/docs/latest/index.html)

##

## *Pre Install*

```
sudo apt install ca-certificates curl gnupg
```

## *Install Docker*

Add GPG key and repository:
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package list and install Docker:
```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Add current User to docker group and relogin:
```
sudo usermod -aG docker $USER && su $USER
```

## *Install OpenVAS*

Create directory:
```
mkdir -p $HOME/greenbone-community-container
```

Change into the directory and download the Docker Compose file:
```
cd $HOME/greenbone-community-container && curl -f -L https://greenbone.github.io/docs/latest/_static/docker-compose-22.4.yml -o docker-compose.yml
```

Create a certificate and copy into the directory as **serverkey.pem** and **servercert.pem**

Edit the Docker Compose file:
```
nano docker-compose.yml
```
  - Under **gsa** 
    - Add **environment:** and with
      ```- GSAD_ARGS=--no-redirect --http-sts --gnutls-priorities=SECURE256:-VERS-TLS-ALL:+VERS-TLS1.2:+VERS-TLS1.3```
    - Change **- 127.0.0.1:9293:80** to **- 443:443**
    - Under **volumes** add
      - ```- /home/<username>/greenbone-community-container/serverkey.pem:/var/lib/gvm/private/CA/serverkey.pem```
      - ```- /home/<username>/greenbone-community-container/servercert.pem:/var/lib/gvm/CA/servercert.pem```
  - Under **gvmd:** in **enviroment>**, set mail settings according your needs. For available options, check documentation.

Download and start the containers:
```
docker compose -f $HOME/greenbone-community-container/docker-compose.yml -p greenbone-community-edition pull
docker compose -f $HOME/greenbone-community-container/docker-compose.yml -p greenbone-community-edition up -d
```

Set a secure admin password:
```
docker compose -f $HOME/greenbone-community-container/docker-compose.yml -p greenbone-community-edition exec -u gvmd gvmd gvmd --user=admin --new-password='SecurePassw0rd'
```

Sync feed:
```
docker compose -f $HOME/greenbone-community-container/docker-compose.yml -p greenbone-community-edition pull notus-data vulnerability-tests scap-data dfn-cert-data cert-bund-data report-formats data-objects
```
Copy feed:
```
docker compose -f $HOME/greenbone-community-container/docker-compose.yml -p greenbone-community-edition up -d notus-data vulnerability-tests scap-data dfn-cert-data cert-bund-data report-formats data-objects
```

## *Web GUI*

Open URL in Browser: *https://YourIP*

Check feed status under *Administration > Feed Status*. It shouldn't be older than 2 days

Create 2 scan's
  - Go to Scans
  - Create  new task
    - Enter a Name *Default Scan 70 **or** 30*
    - Select or create your targets
    - Select or create your ports
    - Select or create a schedule
    - Create one task with **70%** *Min QoD* and the second one with **30%**
    - Scan Config: **Full and fast**
    - Order for target hosts: **Random**
    - set *Maximum concurrently scanned hosts* according your network. For example, set it to **2**
