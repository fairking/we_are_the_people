# Private Smartphone Cloud

I was thinking about to install the nextcloud on RockyLinux, but found some issue there [Rocky Linux - Rocky is not bootable](https://forums.rockylinux.org/t/rocky-is-not-bootable/4474). So I decided that Fedora is the best candidate. I bought a Home Server for a decent price [B6 Mini PC](https://www.aliexpress.com/item/4001178155704.html).
Please find my steps bellow how I did setup my server to run [Nextcould](https://nextcloud.com/):

### After Fedora installation please log into the Home Server via ssh. I recommend [WinSCP](https://winscp.net/) for windows:
![image](https://user-images.githubusercontent.com/13495631/140672951-f4a8fc6b-77dc-4eef-a932-181765ca22ad.png)
### Or you can do it via your smartphone using [ConnectBot](https://f-droid.org/en/packages/org.connectbot/).

>> Please execute linux commands line by line

# Set the static IP (make sure the router is out of DHCP range)
```
nmcli con show -a
ifconfig <UUID>
nmcli connection modify <UUID> IPv4.address 192.168.0.5/24
nmcli connection modify <UUID> IPv4.gateway 192.168.0.1
nmcli connection modify <UUID> IPv4.dns "192.168.0.1,176.103.130.130,176.103.130.131"
nmcli connection modify <UUID> IPv4.method manual
nmcli connection down <UUID> && nmcli connection up <UUID>
```
### You will luse a connection to the server, so you have to login into ssh again with different host.

# Setup Apache
```
dnf update
dnf install wget curl bzip2 nano unzip policycoreutils-python-utils -y
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent –-add-service=https
firewall-cmd --permanent --add-port=8781/tcp
semanage port -a -t http_port_t -p tcp 8781
firewall-cmd --reload
dnf install httpd
dnf install php php-gd php-mbstring php-intl php-pecl-apcu php-mysqlnd php-pecl-redis php-opcache php-imagick php-zip php-process php-bcmath php-gmp
systemctl enable --now httpd
nano /etc/httpd/conf.d/nextcloud.conf
```
### Add the following to the nextcloud.conf file
```
Listen 8781

<VirtualHost *:8781>
  DocumentRoot /var/www/nextcloud/
  ServerName  192.168.0.5:8781

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

# Install Database
```
dnf install mariadb mariadb-server
systemctl enable --now mariadb
mysql_secure_installation
mysql -p
create database nextcloud;
create user nextcloud@localhost identified by 'Admin12345!';
grant all privileges on nextcloud.* to nextcloud@localhost;
flush privileges;
exit;
```

# Install Nextcloud:
```
cd /var/www/
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip -d /var/www/
mkdir /var/data
mkdir /var/data/nextcloud
chown -R apache:apache /var/www/nextcloud
chown -R apache:apache /var/data/nextcloud
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/config(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/apps(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/data/nextcloud(/.*)?'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/.httpaccess'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/.user.ini'
semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/nextcloud/3rdparty/aws/aws-sdk-php/src/data/logs(/.*)?'
restorecon -Rv '/var/www/nextcloud/'
restorecon -Rv '/var/data/nextcloud/'
systemctl restart httpd
```
### The recommend PHP memory limit for Nextcloud is 512M. You can edit the `memory_limit` variable in the `nano /etc/php.ini` configuration file and restart your httpd service `systemctl restart httpd`.

### Go to website http://192.168.0.5:8781/ to setup the instance.

# Increase Root disk space quota
```
vgs
df -hT
```
### Please add as many space as you have free (in this example is 200Gb)
```
lvresize -L +200G --resizefs /dev/mapper/fedora_fedora-root
```

# DDNS
### Find your ip address
```
curl –s https://icanhazip.com
```
### Use freedns.afraid.org to register your dns
### And then
```
dnf install ddclient
nano /etc/ddclient.conf
```
### Add the following config lines (replace `mycloud.mooo.com` with your domain):
```
## FreeDNS.afraid.org
use=if, if=eth0
server=freedns.afraid.org
protocol=freedns
login=my_login
password=****
mycloud.mooo.com
```
### Save the file and then enable the service:
```
systemctl enable ddclient.service
systemctl start ddclient.service
```

# Router needs to be configured (Go to the router -> Port forwarding)
```
# Local IP address	| Local Port range	| External Port range	| Protocol  |	Enabled	Delete
# 192.168.0.5	      |	8781-8781         |	443-443	            |	TCP       |	
```

# Add the trusted_domains to the nextcloud config (`nano /var/www/nextcloud/config/config.php`):
```
  array (
    0 => 'mycloud.mooo.com',
    1 => '192.168.0.5:8781',
  ),
```

# LetsEncrypt
```
dnf install certbot python3-certbot-apache mod_ssl
systemctl restart httpd
certbot --apache
```
### Needs to rename (disable) `/etc/httpd/conf.d/nextcloud-le-ssl.conf.bak` and move the ssl certificates to `/etc/httpd/conf.d/nextcloud.conf` due to ERR_SSL_PROTOCOL_ERROR

# Cron
```
nano /etc/systemd/system/nextcloudcron.service
```
### Add the following lines to the file:
```
[Unit]
Description=Nextcloud cron.php job

[Service]
User=apache
ExecStart=/usr/bin/php -f /var/www/nextcloud/cron.php
KillMode=process
```
### Save the file and then create another one:
```
nano /etc/systemd/system/nextcloudcron.timer
```
### Add the following lines to the file:
```
[Unit]
Description=Run Nextcloud cron.php every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Unit=nextcloudcron.service

[Install]
WantedBy=timers.target
```
### Enable the cron
```
systemctl enable --now nextcloudcron.timer
```

# Optional: Caching (Redis) (currently NOT WORKING):
```
dnf install redis
systemctl enable redis --now
nano /etc/redis/redis.conf
```

### Add the following lines to the end of the file:
```
maxmemory 1gb 
maxmemory-policy allkeys-lru
requiredpass {YOUR_PASSWORD}
unixsocket /run/redis/redis.sock
unixsocketperm 775
bind 127.0.0.1
daemonize yes
supervised auto
stop-writes-on-bgsave-error no
rdbcompression yes
```

### You might also need a change in the file `/usr/lib/systemd/system/redis.service`:
Rename `--daemonize no` to `--daemonize yes`.

### Restart the service after config changes:
```
systemctl restart redis
```

### Add the following lines to the nextcloud config `nano /var/www/nextcloud/config/config.php` with your redis password:
```
  'memcache.local' => '\OC\Memcache\APCu',
  'memcache.locking' => '\OC\Memcache\Redis',
  'memcache.distributed' => '\OC\Memcache\Redis',
  'redis' => [
     'host'     => '/run/redis/redis.sock',
     'port'     => 6379,
     'dbindex'  => 0,
     'password' => '{YOUR_REDIS_PASSWORD}',
     'timeout'  => 1.5,
  ],
```

### Add session lock to the end of `nano /etc/php.ini` file:
```
[redis]
redis.session.locking_enabled=1
redis.session.lock_retries=-1
redis.session.lock_wait_time=10000
```

### And then setting up the redis connection:
```
usermod -g apache redis
usermod -a -G redis apache
chown -R redis:apache /run/redis
systemctl restart httpd
```


# Optional: Adding extra hard drive
### Option 1 - Extending ROOT (+200G change to your size)
```
lsblk
pvcreate /dev/sdb
pvs
vgs
vgextend fedora_fedora /dev/sdb
lvresize -L +200G --resizefs /dev/mapper/fedora_fedora-root
```
### Option 2 - Extending as a separate mounted folder
```
mkdir /data
fdisk -l
fdisk /dev/sdb
n # Create partition
p # Partition type
1 # Partition number
<Enter> # First sector (default)
<Enter> # Last sector (default)
w # Save changes and exit
mkfs.ext4 /dev/sdb
mount /dev/sdb /data
```

# Optional: Samba (currently NOT WORKING)
```
dnf install samba
systemctl enable smb --now
firewall-cmd --get-active-zones
firewall-cmd --permanent --zone=FedoraServer --add-service=samba
firewall-cmd --reload
useradd --system samba_home
passwd samba_home
mkdir /data/samba_home
chown -R samba_home /data/samba_home
smbpasswd -a samba_home
semanage fcontext --add --type "samba_share_t" /data/samba_home
restorecon -R /data/samba_home
nano /etc/samba/smb.conf
```
### Add the following lines to the file:
```
[global]
    min protocol = SMB2
[samba_home]
    comment = My Home Server
    path = /data/samba_home
    writeable = yes
    browseable = yes
    public = yes
    create mask = 0644
    directory mask = 0755
    write list = user
    force user = samba_home
```
### Save the file and then
```
systemctl restart smb
```


# After all of that steps we can install Nextcloud mobile app and sync all our photos and videos with the Home Cloud.
https://user-images.githubusercontent.com/13495631/140672253-d78be6bc-ed2a-4906-908f-aeb6ce57deac.mp4

![image](https://user-images.githubusercontent.com/13495631/140672399-f75d4bcc-5832-42d4-877a-67d41cc4b0f2.png)


# Please ask any questions here but BEFORE PLEASE search the command you are executing: https://github.com/fairking/we_are_the_people/issues

![image](https://user-images.githubusercontent.com/13495631/140669294-6ba9bb14-869b-4f61-9d5b-94b012262541.png)

Please ask questions into the [section](https://github.com/fairking/we_are_the_people/issues/1). Thank you for your help.
