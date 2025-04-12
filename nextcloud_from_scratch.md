## Nextcloud Server From Scratch...

This more of a checklist than a real guide or tutorial.  
This ***does NOT aim*** at being the perfect installation!

It ***does aim*** at being 

> [!WARNING]
> We assume that that you know when to change defaults user, passwords and domains as we go.

### Base Debian install

I will be using [Debian12 Netinstall](https://www.debian.org/CD/netinst/)  for the base server installation.

> [!WARNING]
> **If installing in a VM**  
> Make sure to have unrestricted access to your network  to circumvent DNS resolution problems.

- boot machine with img/iso
- chose the installer mode you like
- Partition with /var & /home  on separate partitions

> [!NOTE]
> Or use separate disk(s) for both 

- ⚠️ give a strong **root** passwd and keep it safe!

> [!NOTE]
>  We will be the only admin of the server.  
>  ***-Almost-*** no need for sudo

- ⚠️ user : nxtadmin, chose a passwd and write it down
- 🚫  NO GUI installation
- ssh install

### Post Debian Install

SSH into your machine or do the next steps on a local console if you can access one.

Create .ssh folder in /home/nxtadmin, then copy the wanted pubkey from your local machine to the server

```

ssh-copy-id -i /home/nxtadmin/.ssh/id_rsa.pub  nxtadmin@REMOTEIP
```

Or by copying the content of your pubkey to the authorized_keys file

```
#on local machine, select & copy the output
cat  /home/nxtadmin/.ssh/id_rsa.pub

#on remote machine
nano /home/nxtadmin/.ssh/authorized_keys
```

Logout and test  your config.  
You should be able to ssh into your server w/o being asked your passwd.

> [!CAUTION]
> For all the rest of this guide, we will be acting as **root**.

```
su -l root
```

We set some variables in sshd_config  to harden the install. No more passwd login allowed beyond this step.

```
nano /etc/ssh/sshd_config

# Add these values towards the end
PermitRootLogin no
PasswordAuthentication no

#restart ssh daemon
systemctl restart sshd
```

### Set up the Firewall

[Firewall](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu) will be installed and basic ports opened.

```
#install UFW
apt install ufw

#Config UFW
ufw allow SSH     
ufw allow WWW\ Full 

#enable the firewall
ufw enable
```

### Install Fail2Ban

We'll install [fail2ban](https://linuxhandbook.com/fail2ban-basic/). After installation, fail2ban will automagically be banning on our ssh port.

```
apt install fail2ban
```

#### Useful & required packages

```
apt install imagemagick
apt install curl unzip 
apt install certbot python3-certbot-apache
apt install sudo
```

> [!WARNING]
> **sudo** will only be installed to interact with the [Nextcloud  occ command ](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/occ_command.html)
> 
> __No other user will be allowed to use sudo, eg not in sudoer's__

### Apache2 Set Up

Install  apache2  and enable some needed mods

```
apt install apache2

#enable mods
a2enmod headers rewrite env dir mime
```

### Php Setup & Base Config

Install needed php modules

```

apt install php php-fpm php-cli php-mysql php.apcu php-common php-gd php-xml php-mbstring php-zip php-curl php-json php-bz2 php-intl php-bcmath php-gmp php-imagick

#after installation of modules to activate fpm
a2enmod proxy_fcgi setenvif
a2enconf php8.2-fpm
systemctl restart apache2
```

Tune some variables for fpm

```
nano /etc/php/8.2/fpm/php.ini 

#Search for and set these variables
max_execution_time = 300
memory_limit = 512M
post_max_size = 128M

systemctl restart php8.2-fpm
```

### MYSQL(MARIADB) SETUP

Install maria db and run the secure installation

```
apt install mariadb-server
mysql_secure_installation 
```

⚠️ Change the root password and answer Y to all the questions

```
		- Remove anonymous users? [Y/n] Y     
		- Disallow root login remotely? [Y/n] Y     
		- Remove test database and access to it? [Y/n] Y     
		- Reload privilege tables now? [Y/n] Y     
```

Restart the mysql server and login as root

```
systemctl restart mariadb
mysql -u root -p
```

Create the Nextcloud database,  Nextcloud user and grant privileges.  
⚠️ Keep your users & passwords safe.

```
CREATE DATABASE nextcloud_db;
GRANT ALL ON nextcloud_db.* TO 'nextclousuer'@'localhost' IDENTIFIED BY 'VERYSTRONGPASSWD';
FLUSH PRIVILEGES;
exit;
```

### Nextcloud Set Up

Arrived here, we can download and unpack the Nextcloud package.  
We'll create the /data directory inside our nextcloud directory and set the right permissions.

```
cd /var/www
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
mkdir /var/www/nextcloud/data
chown -R www-data.www-data nextcloud/
```

Create the virtual host configuration

```
nano /etc/apache2/sites-available/nextcloud.conf
```

This is  a basic config  to paste

```
<VirtualHost *:80>
	ServerName  nextcloud.example.com
  ServerAdmin admin@nextcloud.example.com
        
  DocumentRoot /var/www/nextcloud/
        
  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
        
  <Directory /var/www/nextcloud/>
  		Require all granted
     AllowOverride All
     Options FollowSymLinks MultiViews

     <IfModule mod_dav.c>
     		Dav off
     </IfModule>
 </Directory>

RewriteEngine on
RewriteCond %{SERVER_NAME} =nextcloud.example.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI[END,NE,R=permanent] 
</VirtualHost>
```

Activate and restart apache

```
a2ensite nextcloud.conf
systemctl restart apache2
```

### SSL Activation

Easy Peasy with [certbot](https://certbot.eff.org/) we installed earlier.  
Register your admin email with certbot (only needed once) and run the apache plugin.  
It will scan all you configs and ask you which domain to cert.

```
certbot register
certbot --apache
```

Activate SSL and restart apache

```
a2enmod ssl
systemctl restart apache2
```

#### SSL Self signed certs

If you need certs for development or local networks.  
  
[**SELF SIGNED SSL WITH NEXTCLOUD**](https://outside.rimshot.lu/index.php/f/3339)

### Fail2ban set up

Create a filter for nextcloud

```
nano /etc/fail2ban/filter.d/nextcloud.conf
```

Paste these Regex's

```
[Definition]
_groupsre = (?:(?:,?\s*"\w+":(?:"[^"]+"|\w+))*)
failregex = ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Login failed:^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Trusted domain error.
datepattern = ,?\s*"time"\s*:\s*"%%Y-%%m-%%d[T ]%%H:%%M:%%S(%%z)?"
```

Create a jail for nextcloud

```
nano /etc/fail2ban/jail.d/nextcloud.local
```

Paste these settings

```
 [nextcloud]
 backend = auto
 enabled = true
 port = 80,443
 protocol = tcp
 filter = nextcloud
 maxretry = 3
 bantime = 86400
 findtime = 43200
 logpath = /var/www/nextcloud/data/nextcloud.log
```

> [!NOTE]
> bantime, findtime and maxretry can be tuned to further reduce the background noise on your server.  
> With these settings, an IP will be banned for 24hrs (86400 s) if it hits 3 fails during the findtime of 12 hrs  
> There's also the options to use [incremental bans](https://stackoverflow.com/questions/66392687/fail2ban-how-to-ban-ip-permanently-after-it-was-baned-3-times-temporarily).

Restart fail2ban

```
systemctl restart fail2ban
```

### Set Up alias in bash

> [!NOTE]
> Create an alias for the Nextcloud occ command, preferably in your .bash_aliases

```
alias occ='sudo -u  www-data php /var/www/nextcloud/occ'
```

### Set Up crontab

Create a [crontab](https://tecadmin.net/crontab-in-linux-with-20-examples-of-cron-schedule/) for www-data. This is needed for switching background tasks to cron in Admin Settings.  
The line calls cron.php every 5 minutes

```
crontab -u www-data -e

#add this line at the end
 */5  *  *  *  * php -f /var/www/nextcloud/cron.php
```

### Nextcloud First login

Navigate with your browser to <https://nextcloud.example.com>

Fill out all the info needed and login.

Install the apps when prompted (chose what you need).

**Head to:**

> Personal Settings -> Fill out the Email adress.

(You need one to get the test mail)

**Head to:**

> Admin Settings -> Basic Settings 

❕ Ignore the Errors popping up at when the Overview is loaded

Switch Background Jobs from ***AJAX*** to ***CRON***

### Nextcloud config.php

Return to your server console.

```
nano /var/www/nextcloud/config/config.php 
```

Add the maintenance window we need for cron

```
'maintenance_window_start' => 1,
```

Add your phone country code

```
 'default_phone_region' => 'LU',
```

Setup your system email.  
No MTA on the server, sou we'll use an existing email.

```
'mail_smtpmode' => 'smtp',
'mail_sendmailmode' => 'smtp',
'mail_smtptimeout' => 30,
'mail_smtphost' => 'smtp.example.com',
'mail_from_address' => 'noreply',
'mail_domain' => 'example.com',
'mail_smtpport' => '25',
'mail_smtpauth' => true, 
'mail_smtpname' => 'hello@example.com',
'mail_smtppassword' => 'NastyPassword',
```

Add the cache (more about it here)

```
'memcache.local' => '\OC\Memcache\APCu',
```

### ⚠️ Nextcloud Office 

For this to work properly, you have to install the [Collabora CODE ](https://www.collaboraonline.com/code/#learnmorecode)server onto your machine.  
Since that is another beast, you' ll find the instructions for the installation, set up of server - including self signed SSL - and the Nextcloud configuration in a separate file.  
  
[**NEXTCLOUD-COLLABORA-GUIDE**](https://outside.rimshot.lu/index.php/f/3347)

###   
Nextcloud Post install

**System Emails & Cronjobs**

> Admin Settings  --->  Basic

Check if you can send Emails  
Check if the background jobs run properly

**System Check**

> Admin Settings  --->  Overview

Here you can check what is still missing.  
Most likely, we have not installed Redis and there's no High Performance Backend.  
You can do that if you need it.

**Testmail not sending**

If mail sending does not work because of CA issues, add this into your config.php

> [!WARNING]
> Be aware of the security risks

```
'mail_smtpstreamoptions' => array (
    'ssl' => array (
        'allow_self_signed' => true,
        'verify_peer' => false,
        'verify_peer_name' => false
        )
    ),
```

  
**Errors about the database**

You might to run some database repairs manually.   
Two of the most common tasks below.  
❗ Note the use of the [alias](https://github.com/p3titpoe/p3titpoe/edit/main/nextcloud_from_scratch.md#set-up-alias-in-bash) we set before.

```
#repair indices:
occ db:add-missing-indices

#mimetypes migrations
occ maintenance:repair --include-expensive
```

In general, the system will tell you what command to execute, that's where the alias comes in handy.

**Cache related issues**

There are some settings worth playing with if you have op-cache related errors.

Open the .ini file

```
nano /etc/php/8.2/fpm/php.ini
```

Search for these settings

```
opcache.enable=1
opcache.memory_consumption=512
opcache.interned_strings_buffer=16
opcache.revalidate_freq=2
opcache.save_comments=1
```

### 

### Conclusion

#### Useful Links

[https://docs.nextcloud.com/server/31/admin_manual/installation/source_installation.html ](https://docs.nextcloud.com/server/31/admin_manual/installation/source_installation.html%20)

<https://www.linuxtuto.com/how-to-install-nextcloud-on-debian-12/#google_vignette> 

<https://devrix.com/tutorial/ssl-certificate-authority-local-https/>

<https://docs.nextcloud.com/server/31/admin_manual/installation/server_tuning.html#enable-php-opcache>

<https://docs.nextcloud.com/server/31/admin_manual/installation/harden_server.html#setup-a-filter-and-a-jail-for-nextcloud>

<https://tweenpath.net/installation-collabora-debain-12/ >

<https://www.aptgetlife.co.uk/setup-fail2ban-for-nextcloud/>

<https://tweenpath.net/install-collabora-online-ubuntu-20-nextcloud/>
