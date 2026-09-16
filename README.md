# Install LibreNMS on Ubuntu 24.04 with MySQL

This guide installs [LibreNMS](https://www.librenms.org/), a web application that monitors servers, switches, routers, and other network devices.

## The goal

When you finish, you will have:

- a LibreNMS web page served by nginx;
- PHP-FPM running the LibreNMS application;
- MySQL storing monitoring data;
- SNMP collecting information from your devices; and
- automatic polling, scheduled tasks, and log rotation.

You will then be able to open LibreNMS in a browser, create an administrator account, add a device, and see its health and performance over time.

This guide is for a **fresh Ubuntu 24.04 server**. It uses **MySQL** instead of the MariaDB database used in the official example. The rest of the installation follows the [official LibreNMS installation guide](https://docs.librenms.org/Installation/Install-LibreNMS/).

## Before you start

You need:

- an Ubuntu 24.04 server with a static IP address;
- a user that can run `sudo`, or a root shell;
- the server's hostname or a DNS name, such as `librenms.example.com`;
- access to ports 80 and 443 from your browser; and
- SNMP access to every device you want to monitor.

Run the commands below as `root`. If you are not root, add `sudo` to commands that need it. Replace these example values as you work:

| Example | Replace it with |
|---|---|
| `librenms.example.com` | Your DNS name or server IP address |
| `Africa/Nairobi` | Your PHP timezone |
| `Etc/UTC` | Your server timezone |
| `password` | A strong MySQL password |
| `RANDOMSTRINGGOESHERE` | A private SNMP community string |

## 1. Update Ubuntu and add PHP

Ubuntu 24.04 includes PHP 8.3, while this installation uses PHP 8.5. Add the Sury PHP repository first, then refresh the package list.

```bash
apt update && apt upgrade -y
apt install -y lsb-release ca-certificates curl
curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
dpkg -i /tmp/debsuryorg-archive-keyring.deb
echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
apt update
```

## 2. Install the required software

These packages provide the web server, PHP, database, SNMP tools, graphing tools, and utilities LibreNMS needs.

```bash
apt install -y acl curl fping git mysql-server mysql-client mtr-tiny nginx-full nmap \
php8.5-cli php8.5-curl php8.5-fpm php8.5-gd php8.5-gmp php8.5-mbstring php8.5-mysql php8.5-snmp php8.5-xml php8.5-zip \
python3-command-runner python3-dotenv python3-pip python3-psutil python3-pymysql python3-redis python3-setuptools python3-systemd \
rrdtool snmp snmpd traceroute unzip whois
```

## 3. Create the LibreNMS user and download LibreNMS

LibreNMS should run as its own unprivileged user. This keeps the application from running as `root`.

```bash
useradd librenms -d /opt/librenms -M -r -s "$(which bash)"
cd /opt
git clone https://github.com/librenms/librenms.git
chown -R librenms:librenms /opt/librenms
chmod 771 /opt/librenms
```

Allow the web server and LibreNMS to write logs, graphs, cache files, and application data:

```bash
setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
```

## 4. Install PHP dependencies

Run Composer as the `librenms` user so the downloaded files have the correct owner.

```bash
su - librenms
./scripts/composer_wrapper.php install --no-dev
exit
```

## 5. Set the timezones

Use the same timezone in PHP and on the server. Edit both PHP configuration files and set `date.timezone` to your timezone.

```bash
vi /etc/php/8.5/fpm/php.ini
vi /etc/php/8.5/cli/php.ini
```

For example:

```ini
date.timezone = Africa/Nairobi
```

Then set the operating system timezone:

```bash
timedatectl set-timezone Etc/UTC
```

## 6. Create the MySQL database

MySQL stores LibreNMS users, device details, alerts, and other application data.

First, configure MySQL:

```bash
vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add these lines under `[mysqld]`:

```ini
innodb_file_per_table=1
lower_case_table_names=0
```

Restart MySQL:

```bash
systemctl enable mysql
systemctl restart mysql
```

Create the database and its local user:

```bash
mysql -u root
```

At the `mysql>` prompt, run this SQL. Replace `password` with a strong password and keep it for the web installer.

```sql
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
exit
```

The database user can connect only from this server and can access only the LibreNMS database. For additional MySQL hardening, run `mysql_secure_installation`.

## 7. Configure PHP-FPM

PHP-FPM runs the PHP code used by the LibreNMS web page.

```bash
cp /etc/php/8.5/fpm/pool.d/www.conf /etc/php/8.5/fpm/pool.d/librenms.conf
vi /etc/php/8.5/fpm/pool.d/librenms.conf
```

In `librenms.conf`, make these changes:

- change `[www]` to `[librenms]`;
- set `user = librenms`;
- set `group = librenms`; and
- set `listen = /run/php-fpm-librenms.sock`.

The socket path must exactly match the nginx configuration in the next step.

## 8. Configure nginx

nginx receives browser requests and passes PHP requests to PHP-FPM.

Create the site configuration:

```bash
vi /etc/nginx/sites-enabled/librenms.vhost
```

Paste this configuration and replace `librenms.example.com` with your DNS name or server IP:

```nginx
server {
 listen 80;
 server_name librenms.example.com;
 root /opt/librenms/html;
 index index.php;

 charset utf-8;
 gzip on;
 gzip_types text/css application/javascript text/javascript application/x-javascript image/svg+xml text/plain text/xsd text/xsl text/xml image/x-icon;

 location / {
  try_files $uri $uri/ /index.php?$query_string;
 }

 location ~ [^/]\.php(/|$) {
  fastcgi_pass unix:/run/php-fpm-librenms.sock;
  fastcgi_split_path_info ^(.+\.php)(/.+)$;
  include fastcgi.conf;
 }

 location ~ /\.(?!well-known).* {
  deny all;
 }
}
```

Apply the web server configuration:

```bash
rm /etc/nginx/sites-enabled/default
systemctl restart php8.5-fpm
systemctl reload nginx
```

## 9. Open the web ports

Only do this if UFW is enabled:

```bash
ufw allow 80/tcp
ufw allow 443/tcp
```

This guide starts with HTTP so that the installer works. Add HTTPS with Let's Encrypt before exposing LibreNMS to the public internet.

## 10. Configure LibreNMS helpers

Add the `lnms` command and its shell completion:

```bash
ln -s /opt/librenms/lnms /usr/bin/lnms
cp /opt/librenms/misc/lnms-completion.bash /etc/bash_completion.d/
```

Configure the SNMP service. SNMP is how LibreNMS reads metrics from monitored devices.

```bash
cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
vi /etc/snmp/snmpd.conf
```

Replace `RANDOMSTRINGGOESHERE` with a private community string. Use the same community string when configuring devices for monitoring.

```bash
curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
chmod +x /usr/bin/distro
systemctl enable snmpd
systemctl restart snmpd
```

## 11. Enable automatic tasks

These files make polling, housekeeping, and log cleanup run automatically:

```bash
cp /opt/librenms/dist/librenms.cron /etc/cron.d/librenms
cp /opt/librenms/dist/librenms-scheduler.service /opt/librenms/dist/librenms-scheduler.timer /etc/systemd/system/
systemctl enable librenms-scheduler.timer
systemctl start librenms-scheduler.timer
cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```

## 12. Finish in the browser

Open this address:

```text
http://librenms.example.com/install
```

The installer checks the server and asks for the MySQL details:

| Field | Value |
|---|---|
| Database host | `localhost` |
| Database name | `librenms` |
| Database user | `librenms` |
| Database password | The password created in Step 6 |

Create your LibreNMS administrator account when prompted. When the installer creates `config.php`, set its owner:

```bash
chown librenms:librenms /opt/librenms/config.php
```

## 13. Add your first device

Log in at:

```text
http://librenms.example.com/
```

Add a device from:

```text
http://librenms.example.com/addhost
```

Start with the LibreNMS server itself or another device that has SNMP enabled. After polling begins, LibreNMS will show availability, graphs, and alerts for that device.

## Troubleshooting

The validator checks configuration, permissions, and database connectivity:

```bash
su - librenms
./validate.php
```

Read the validator output from top to bottom and fix the first error before checking again.

Useful next steps:

- [Performance tuning](https://docs.librenms.org/Support/Performance/)
- [Alerting](https://docs.librenms.org/Alerting/)
- [Device groups](https://docs.librenms.org/Extensions/Device-Groups/)
- [Auto discovery](https://docs.librenms.org/Extensions/Auto-Discovery/)
- [HTTPS and security hardening](https://docs.librenms.org/Installation/Install-LibreNMS/)
