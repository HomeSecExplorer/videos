# Documentation for

#### Zabbix Installation & Pi-hole Monitoring: Step-by-Step Guide

[YouTube video](https://youtu.be/-340x3cjPvo)

##

#### Zabbix Doc's

[CURRENT](https://www.zabbix.com/documentation/current/en/manual)

##

## Zabbix Install

####  For current install instructions visit

[INSTALL](https://www.zabbix.com/download)

#### *Install MariaDB*

```
sudo apt install mariadb-server
```

```
sudo mysql_secure_installation
```

  - answer all prompt with **y** and enter a **secure password** when prompted

#### *Install Zabbix*

```
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_7.0-2+debian12_all.deb
sudo dpkg -i zabbix-release_7.0-2+debian12_all.deb
sudo apt update
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

#### *Configure DB*

```
mysql -uroot -p
```
  - enter your DB root password *(set earlier)*
```
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'SecurePassw0rd';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```
  - **remeber to change zabbix db user password**

```
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```
  - enter zabbix db user password

```
mysql -uroot -p
```
  - enter your DB root password *(set earlier)*

```
set global log_bin_trust_function_creators = 0;
quit;
```

#### *Configure Apache and Zabbix conf file*

```
sudo nano /etc/zabbix/zabbix_server.conf
```
  - ```DBPassword=SecurePassw0rd```


```
sudo nano /etc/apache2/sites-enabled/000-default.conf
```
  - change ```DocumentRoot /var/www/html``` to ```DocumentRoot /usr/share/zabbix```


#### *Restart und enable services*

```
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

## Configure PiHole

connect to PiHole via ssh

#### *Allow Zabbix IP to connet to PiHole API*

```
sudo nano /var/www/html/admin/scripts/pi-hole/php/auth.php
```
  - in the array of **$AUTHORIZED_HOSTNAMES** add ```'ZabbixIP',```


```
sudo systemctl restart pihole-FTL
```

#### *Get API Token*

```
sudo cat /etc/pihole/setupVars.conf | grep PASSWORD | cut -d "=" -f2
```


## Configure Zabbix

Go to *http://YouIP/zabbix* and go trough the process


#### *Login*

Login with **Admin** and password **zabbix**


#### *Change Admin password*

Under **Users/Users** click on *Admin* and set new password

### Add PiHole monitoring

#### *Download template*

Download fitting template to your local machine
<https://github.com/zabbix/community-templates/tree/main/Applications/template_pi-hole_api>

#### *Upload template*

Go to **Data collection/templates**

Top right corner click **Import** and import the file you just downloaded

#### Collect metric's

Go to **Data collection/hosts**

**You have Zabbix Agent installed on PiHole**
  - edit your pihole host and add *pihole template* under **template** section
  - under **macros** add *{$WEBPASSWORD}* with your *API Token*

**PiHole host not existing**
  - top right corner **Create host**
  - fill in **Name** and **Host Group**
  - under **template** select *pihole template*
  - click **add** under **Interfaces** and select *SNMP*, enter PiHole IP
  - under **macros** add *{$WEBPASSWORD}* with your *API Token*
