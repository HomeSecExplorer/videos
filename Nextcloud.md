# Documentation for

#### Basic Nextcloud Installation : Step-by-Step Guide

[YouTube video](https://youtu.be/f3wAeAeF3Ec)

##

####  Doc's

[Latest](https://docs.nextcloud.com/server/latest/admin_manual/contents.html)

##

## *Install Apache and PHP*

Install apache:
```
sudo apt install apache2
```

Enable and check apache:
```
sudo systemctl enable apache2
sudo systemctl status apache2
```

Install PHP:
```
sudo apt install php php-curl php-cli php-mysql php-gd php-common php-xml php-json php-intl php-pear php-imagick php-dev php-common php-mbstring php-zip php-soap php-bz2 php-bcmath php-gmp php-apcu libmagickcore-dev
```

Configure PHP:
```
sudo nano /etc/php/8.2/apache2/php.ini
```
Change:
  - **date.timezone** to your local Timezone
  - **memory_limit** to a appropriate size, e.g. 512M
  - **upload_max_filesize** to the maximum file size you can upload, e.g. 4096M
  - **post_max_size** to a bigger size then *upload_max_filesize*
  -  **max_execution_time** to a appropriate size, e.g. 300
  - **file_uploads = On**
  - **allow_url_fopen = On**
  - **display_errors = Off**
  - **output_buffering = Off**
  - **zend_extension=opcache**
  - under **[opcache]**
    - ```
      opcache.enable = 1
      opcache.interned_strings_buffer = 8
      opcache.max_accelerated_files = 10000
      opcache.memory_consumption = 128
      opcache.save_comments = 1
      opcache.revalidate_freq = 1
      ```

In:
```
sudo nano /etc/php/8.2/mods-available/apcu.ini
```
Add:
```
apc.enable_cli=1
```

Restart apache:
```
sudo systemctl restart apache2
```

## *Install MariaDB*

Install MariaDB:
```
sudo apt install mariadb-server
```

Enable and check MariaDB:
```
sudo systemctl enable mariadb
sudo systemctl status mariadb
```

Apply basic security settings:

```
sudo mysql_secure_installation
```
  - answer all prompt's with **y** and enter a **secure password** when prompted

Login to MariaDB:
```
sudo mariadb -u root -p
```

Create User and DB for Nextcloud:

  - **remember to set a secure password**

```
CREATE DATABASE nextcloud_db;
CREATE USER nextcloud@localhost IDENTIFIED BY 'SecurePassw0rd';
GRANT ALL PRIVILEGES ON nextcloud_db.* TO nextcloud@localhost;
FLUSH PRIVILEGES;
quit;
```

## *Download Nextcloud and configure Apache*

Change to the directory:
```
cd /var/www
```

Download latest Nextcloud archive:
```
sudo wget https://download.nextcloud.com/server/releases/latest.tar.bz2
```

Unpack and delete archive:
```
sudo tar -xjf latest.tar.bz2
sudo rm latest.tar.bz2
```

Set owner and group:
```
sudo chown -R www-data:www-data nextcloud
```

Add cron job:
```
sudo crontab -u www-data -e
```
  - Add
```
*/5  *  *  *  * php -f /var/www/nextcloud/cron.php
```

Create config for Apache:
```
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

  - **remember to set your own certificate**
```
<VirtualHost *:80>
  ServerName nextcloud.example.com
  Redirect permanent / https://nextcloud.example.com/
</VirtualHost>
<VirtualHost *:443>
  ServerName nextcloud.example.com
  DocumentRoot /var/www/nextcloud/
  ErrorLog ${APACHE_LOG_DIR}/nc-error.log
  CustomLog ${APACHE_LOG_DIR}/nc-access.log combined
  SSLEngine on
  SSLCertificateFile      /etc/ssl/certs/ssl-cert-snakeoil.pem
  SSLCertificateKeyFile   /etc/ssl/private/ssl-cert-snakeoil.key

  <Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>
  </Directory>
</VirtualHost>
```

Enable module:
```
sudo a2enmod rewrite ssl headers env dir mime
```

Disable default site:
```
sudo a2dissite 000-default.conf
```

Enable nextcloud site:
```
sudo a2ensite nextcloud.conf
sudo systemctl restart apache2
```

## *Install Nextcloud*

Open **https://YourServer** and follow the setup

Edit Nextcloud config:
```
sudo nano /var/www/nextcloud/config/config.php
```
  - under **'trusted_domains' =>** edit the array and add your domain name
  - change **'overwrite.cli.url' =>** to your domain name
  - add ```'memcache.local' => '\OC\Memcache\APCu',``` before **);**

Open **https://YourServer**

Go to **Administration settings/Basic settings**
  - Make sure *Background jobs* is **Cron**
  - Configure Email server to receive mails

Go to **Administration settings/Security**
  - Edit your password settings

Install the apps you like

Nextcloud is running, setup users, add apps and change other settings to your needs
