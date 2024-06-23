# Documentation for
#### Next Level DNS - Installing PiHole with DNS Over HTTPS
[YouTube video](https://youtu.be/TAvGvxFj9so)

##
#### PiHole Doc's
<https://docs.pi-hole.net/guides/dns/cloudflared/>

##

## Install cloudflared  
####  *Add cloudflare gpg key*
sudo mkdir -p --mode=0755 /usr/share/keyrings

curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

####  *Add repo to apt repositories*
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

####  *install cloudflared*
sudo apt-get update && sudo apt-get install cloudflared

sudo useradd -s /usr/sbin/nologin -r -M cloudflared

sudo nano /etc/default/cloudflared

    CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query

sudo chmod 744 /etc/default/cloudflared

sudo chown cloudflared:cloudflared /etc/default/cloudflared

sudo nano /etc/systemd/system/cloudflared.service

    [Unit]
    Description=cloudflared DNS over HTTPS proxy 5053
    After=syslog.target network-online.target
    [Service]
    Type=simple
    User=cloudflared
    EnvironmentFile=/etc/default/cloudflared
    ExecStart=/usr/local/bin/cloudflared proxy-dns $CLOUDFLARED_OPTS
    Restart=on-failure
    RestartSec=10
    KillMode=process
    [Install]
    WantedBy=multi-user.target

sudo nano /etc/default/cloudflaredquad9

    CLOUDFLARED_OPTS=--port 5054 --upstream https://9.9.9.9/dns-query --upstream https://149.112.112.112/dns-query

sudo chmod 744 /etc/default/cloudflaredquad9

sudo chown cloudflared:cloudflared /etc/default/cloudflaredquad9

sudo nano /etc/systemd/system/cloudflaredquad9.service

    [Unit]
    Description=cloudflared DNS over HTTPS proxy 5054
    After=syslog.target network-online.target
    [Service]
    Type=simple
    User=cloudflared
    EnvironmentFile=/etc/default/cloudflaredquad9
    ExecStart=/usr/local/bin/cloudflared proxy-dns $CLOUDFLARED_OPTS
    Restart=on-failure
    RestartSec=10
    KillMode=process
    [Install]
    WantedBy=multi-user.target

sudo systemctl daemon-reload

sudo systemctl start cloudflaredquad9.service

sudo systemctl start cloudflared.service

sudo systemctl enable cloudflared.service

sudo systemctl enable cloudflaredquad9.service

sudo systemctl status cloudflared.service

sudo systemctl status cloudflaredquad9.service

##
## Pihole Install

curl -sSL https://install.pi-hole.net | sudo PIHOLE_SKIP_OS_CHECK=true bash

#### *reset password*
pihole -a -p

#### *blocklist's*

https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts

https://www.technoy.de/lists/FireTVAds.txt

https://v.firebog.net/hosts/AdguardDNS.txt

https://v.firebog.net/hosts/Admiral.txt

https://adaway.org/hosts.txt

https://v.firebog.net/hosts/Easyprivacy.txt

https://v.firebog.net/hosts/Prigent-Ads.txt

https://v.firebog.net/hosts/static/w3kbl.txt

https://raw.githubusercontent.com/RPiList/specials/master/Blocklisten/notserious

https://www.technoy.de/lists/fake-streaming.txt

https://raw.githubusercontent.com/DandelionSprout/adfilt/master/Alternate versions Anti-Malware List/AntiMalwareHosts.txt

https://osint.digitalside.it/Threat-Intel/lists/latestdomains.txt

https://v.firebog.net/hosts/Prigent-Crypto.txt

https://raw.githubusercontent.com/blocklistproject/Lists/master/ransomware.txt

https://phishing.army/download/phishing_army_blocklist_extended.txt


#### *PiZero and other ARMv6*

wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm

sudo mv -f ./cloudflared-linux-arm /usr/local/bin/cloudflared

sudo chmod +x /usr/local/bin/cloudflared cloudflared -v

