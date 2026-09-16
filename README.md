<div align="center">

# 📡 LibreNMS on Ubuntu 24.04 (MySQL Edition)

### A fully annotated, copy‑paste‑ready install guide — every command explained

![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![MySQL](https://img.shields.io/badge/Database-MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.5-777BB4?style=for-the-badge&logo=php&logoColor=white)
![nginx](https://img.shields.io/badge/Webserver-nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![LibreNMS](https://img.shields.io/badge/Monitoring-LibreNMS-1A1A1A?style=for-the-badge&logo=librenms&logoColor=white)

</div>

---

## 🧭 Linux command primer — quick guide to commands used in this document

This guide uses a number of common Linux commands and patterns. If you're new to Linux, here's a short reference for the exact commands used below and what they do.

- apt update && apt upgrade -y
  - Refresh package lists and upgrade installed packages. `-y` answers "yes" to prompts.

- apt install <package(s)>
  - Installs one or more packages from the configured repositories (e.g., `apt install nginx`).

- curl, wget
  - Download files from the network. `curl -O <url>` or the `-sSLo` form used in this doc saves to a path.

- dpkg -i <file.deb>
  - Installs a local .deb package file (used here to install a keyring package).

- echo "..." > /path/to/file
  - Writes text into a file, replacing its contents. Use `>>` to append instead.

- git clone <repo-url>
  - Downloads a Git repository to the current directory.

- cd /path
  - Change the current working directory.

- useradd <username> [flags]
  - Create a system user. Flags like `-d` (home dir), `-r` (system account), and `-s` (shell) are used here.

- chown -R user:group /path
  - Change owner and group of files; `-R` applies recursively.

- chmod 771 /path
  - Change file/directory permissions. Numeric modes like `771` are explained in-line in the doc.

- setfacl -m / -d -m
  - Modify POSIX ACLs (access control lists) to grant more granular permissions than chmod can.

- su - <user> / exit
  - Switch to another user account (`su - librenms`) and `exit` returns to the previous user. Use `sudo -i -u <user>` as an alternative.

- systemctl enable|start|restart <service>
  - Manage systemd services. `enable` makes a service start at boot, `start` runs it now, `restart` reloads config and restarts it.

- mysql -u root
  - Opens an interactive MySQL shell as the `root` database user. SQL commands are run at the `mysql>` prompt.

- vi /etc/whatever or editor of your choice
  - Edit configuration files. You can use `nano`, `vi`, or any editor you prefer.

- ln -s /source /target
  - Create a symbolic link. Used to make commands available system-wide.

- cp /source /destination
  - Copy files. Used to install default configs into /etc or /opt paths.

- rm /etc/nginx/sites-enabled/default
  - Remove files. Use carefully; `rm -rf` removes recursively and forcibly.

- chmod +x /path/to/script
  - Make a file executable so you can run `./script`.

- ./script or /usr/bin/command
  - Execute a script in the current directory (`./script`) or a command installed in a system path.

- ufw allow <port>/tcp
  - Open a firewall port using Ubuntu's uncomplicated firewall. Only needed if `ufw` is enabled.

- cp /opt/librenms/dist/librenms.cron /etc/cron.d/librenms
  - Installing a file into `/etc/cron.d/` registers scheduled jobs with cron.

Notes & safety tips

- Many commands in this doc require root privileges. The guide suggests running as root or prefixing commands with `sudo`.
- When copying commands from the web: inspect them before running — especially commands that write to system paths (`/etc`, `/usr/bin`) or that remove files.
- When editing files like MySQL's config or PHP's php.ini, keep a backup (e.g., `cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.bak`) before making changes.

---

> 🟢 **Status:** Ubuntu 24.04 is an **officially supported** LibreNMS platform — no workaround OS needed. This guide follows the [official LibreNMS docs](https://docs.librenms.org/Installation/Install-LibreNMS/)

## 📑 Table of Contents

| | | | |
|---|---|---|---|
| [🧰 What you'll end up with](#-what-youll-end-up-with) | [1️⃣ Update & add PHP repo](#1️⃣-update-the-server-and-add-the-php-repository) | [2️⃣ Install packages](#2️⃣-install-al[...]
| [3️⃣ Create system user](#3️⃣-create-the-librenms-system-user) | [4️⃣ Download LibreNMS](#4️⃣-download-librenms) | [5️⃣ File permissions](#5️⃣-set-file-permissions) |
| [6️⃣ Composer](#6️⃣-install-php-dependencies-via-composer) | [7️⃣ Timezone](#7️⃣-set-the-timezone) | [8️⃣ 🐬 MySQL config](#8️⃣--configure-mysql) |
| [9️⃣ PHP-FPM](#9️⃣-configure-php-fpm) | [🔟 nginx](#-configure-nginx) | [1️⃣1️⃣ Firewall](#1️⃣1️⃣-allow-access-through-the-firewall) |
| [1️⃣2️⃣ CLI tool](#1️⃣2️⃣-enable-lnms-command-line-completion) | [1️⃣3️⃣ SNMP](#1️⃣3️⃣-configure-snmpd) | [1️⃣4️⃣ Cron](#1️⃣4️⃣-set-up-the-cron-jo[...]
| [1️⃣5️⃣ Scheduler](#1️⃣5️⃣-enable-the-polling-scheduler) | [1️⃣6️⃣ Log rotation](#1️⃣6️⃣-enable-log-rotation) | [1️⃣7️⃣ Web installer](#1️⃣7️⃣-run[...]
| [1️⃣8️⃣ ✅ Final steps](#1️⃣8️⃣-final-steps) | [🆘 Troubleshooting](#-troubleshooting) | [📚 Next reads](#-good-next-reads) |

---

## 🧰 What You'll End Up With

A working LibreNMS install on Ubuntu 24.04 (`noble`), serving over **nginx + PHP-FPM**, storing data in **MySQL**, polling devices over **SNMP**, on a schedule managed by **systemd timers**.

> 👤 Run every command below as **root** (or prefix with `sudo`). Only commands at a `mysql>` prompt skip `sudo`.

---

## 1️⃣ Update the Server and Add the PHP Repository

Ubuntu 24.04's own repos ship **PHP 8.3**, but LibreNMS needs **PHP 8.4+** (8.5 recommended) — so even on the *officially supported* OS version, the [official docs](https://docs.librenms.org/Ins[...]

```bash
apt update && apt upgrade -y
```
🔹 `apt update` refreshes the list of packages available from Ubuntu's repos.
🔹 `apt upgrade -y` installs pending updates; `-y` auto-confirms so it won't pause for input.

```bash
apt install -y lsb-release ca-certificates curl
```
🔹 `lsb-release` — lets scripts detect this is `noble` (24.04).
🔹 `ca-certificates` — lets `curl`/`apt` verify HTTPS certificates.
🔹 `curl` — downloads files.

```bash
curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
dpkg -i /tmp/debsuryorg-archive-keyring.deb
```
🔹 Downloads and installs **Sury's GPG signing key**, so `apt` trusts packages coming from their PHP repo instead of rejecting them as unsigned.

```bash
echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
apt update
```
🔹 Registers the Sury PHP repo (`$(lsb_release -sc)` auto-fills `noble`), then refreshes `apt`'s package list to include it.

---

## 2️⃣ Install All Required Packages

```bash
apt install acl curl fping git mysql-server mysql-client mtr-tiny nginx-full nmap \
php8.5-cli php8.5-curl php8.5-fpm php8.5-gd php8.5-gmp php8.5-mbstring php8.5-mysql php8.5-snmp php8.5-xml php8.5-zip \
python3-command-runner python3-dotenv python3-pip python3-psutil python3-pymysql python3-redis python3-setuptools python3-systemd \
rrdtool snmp snmpd traceroute unzip whois
```

<details>
<summary>📦 <strong>Click to expand — what each package is for</strong></summary>

| Package | Purpose |
|---|---|
| `acl` | Fine-grained file permissions (`setfacl`), used in Step 5 |
| `curl` | Downloading files from within scripts |
| `fping` | Fast ping utility LibreNMS uses to check if devices are alive |
| `git` | Clones the LibreNMS source code from GitHub |
| 🐬 `mysql-server` / `mysql-client` | The database engine + CLI tool. *(Official docs use `mariadb-server`/`mariadb-client` here — this is the MySQL swap, straight from Ubuntu's default repos[...]
| `mtr-tiny` | Traceroute/ping combo tool for network path diagnostics |
| `nginx-full` | The web server serving the LibreNMS UI |
| `nmap` | Network scanning, used by some discovery features |
| `php8.5-cli` / `php8.5-fpm` | PHP itself — CLI scripts + web server integration |
| `php8.5-curl` | PHP HTTP requests |
| `php8.5-gd` | Image/graph generation |
| `php8.5-gmp` | Big-number math |
| `php8.5-mbstring` | Multi-byte string handling |
| `php8.5-mysql` | PHP ↔ database driver (works with MySQL *and* MariaDB) |
| `php8.5-snmp` | PHP ↔ SNMP device communication |
| `php8.5-xml` / `php8.5-zip` | Parsing config/data formats |
| `python3-pymysql` | Python ↔ database access (poller scripts) — works against MySQL too |
| `python3-redis` | Caching |
| `python3-dotenv` | Reads the `.env` config file |
| `python3-psutil` / `python3-systemd` | System info for Python scripts |
| `python3-pip` / `python3-setuptools` | Python package management |
| `python3-command-runner` | Runs shell commands from Python safely |
| `rrdtool` | Stores/graphs the time-series performance data (bandwidth, CPU, etc.) |
| `snmp` / `snmpd` | SNMP client tools + daemon |
| `traceroute` | Network path tracing |
| `unzip` | Extracts zip archives |
| `whois` | Domain/IP lookup features |

</details>

---

## 3️⃣ Create the `librenms` System User

```bash
useradd librenms -d /opt/librenms -M -r -s "$(which bash)"
```

| Flag | Meaning |
|---|---|
| `-d /opt/librenms` | Sets this as the user's home directory |
| `-M` | **Don't** create that home dir yet — `git clone` will do it in Step 4 |
| `-r` | Makes it a system account (no password login — it's a service account) |
| `-s "$(which bash)"` | Sets the login shell to bash, found dynamically |

> ⚠️ Everything LibreNMS does on disk runs as **this** unprivileged user — never as root. That's intentional and important for security.

---

## 4️⃣ Download LibreNMS

```bash
cd /opt
git clone https://github.com/librenms/librenms.git
```
🔹 `cd /opt` — the conventional home for third-party software on Linux.
🔹 `git clone` — downloads the full LibreNMS source into `/opt/librenms`.

---

## 5️⃣ Set File Permissions

```bash
chown -R librenms:librenms /opt/librenms
chmod 771 /opt/librenms
```
🔹 `chown -R` — makes the `librenms` user/group own every file recursively.
🔹 `chmod 771` — owner + group get read/write/execute (`7`+`7`); everyone else gets execute-only (`1`), so outsiders can traverse the path but not browse it.

```bash
setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
```
🔹 Sets ACLs on the folders the web server needs to write into (graphs, logs, cache, storage). `-d` ("default") makes **new** files inherit group read/write/execute; `-R` applies it to everythi[...]

---

## 6️⃣ Install PHP Dependencies via Composer

```bash
su - librenms
./scripts/composer_wrapper.php install --no-dev
exit
```
🔹 `su - librenms` — switches to the `librenms` user so ownership stays correct.
🔹 `composer_wrapper.php install --no-dev` — **Composer** is PHP's package manager; this pulls in every third-party PHP library LibreNMS needs. `--no-dev` skips testing-only packages.
🔹 `exit` — back to root.

> 🌐 **Behind a proxy?** Install Composer manually:
> ```bash
> wget https://getcomposer.org/composer-stable.phar
> mv composer-stable.phar /usr/bin/composer
> chmod +x /usr/bin/composer
> ```

---

## 7️⃣ Set the Timezone

```bash
vi /etc/php/8.5/fpm/php.ini
vi /etc/php/8.5/cli/php.ini
```
🔹 In **both** files, set `date.timezone = Africa/Nairobi` (swap for [your zone](https://www.php.net/manual/en/timezones.php)). PHP needs its own timezone, independent of the OS.

```bash
timedatectl set-timezone Etc/UTC
```
🔹 Sets the **OS-level** timezone (replace `Etc/UTC` with yours) — keep this in sync with the PHP setting above.

---

## 8️⃣ 🐬 Configure MySQL

> This is the **one step that meaningfully differs** from the official MariaDB-based docs. Everything else in this guide is unmodified.

| | MariaDB *(official docs)* | MySQL *(this guide)* |
|---|---|---|
| Config file | `/etc/mysql/mariadb.conf.d/50-server.cnf` | `/etc/mysql/mysql.conf.d/mysqld.cnf` |
| Config section | `[mariadbd]` | `[mysqld]` |
| systemd service | `mariadb` | `mysql` |
| Client command | `mysql -u root` | `mysql -u root` *(identical)* |

```bash
vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
Add under `[mysqld]`:
```ini
innodb_file_per_table=1
lower_case_table_names=0
```

> 💡 **Good to know:**
> - `innodb_file_per_table=1` gives each table its own file on disk (easier space management). **MySQL 5.6+ already defaults to this** — you're just making it explicit.
> - `lower_case_table_names=0` keeps table names case-sensitive, which LibreNMS's schema expects. **On Linux, MySQL already defaults to `0`** too — Windows/macOS installs are the ones that defa[...]
> - ⚠️ **MySQL-specific gotcha:** `lower_case_table_names` can only be set **before** the data directory is first initialized — MySQL refuses to change it afterward on a system with existin[...]

```bash
systemctl enable mysql
systemctl restart mysql
```
🔹 Note the service name: **`mysql`**, not `mariadb`. `enable` = start on boot; `restart` = apply the config.

```bash
mysql -u root
```
🔹 Opens an interactive MySQL session. Fresh Ubuntu `mysql-server` installs authenticate root via your Linux login automatically (`auth_socket` plugin) — no separate password needed yet.

At the `mysql>` prompt (swap `password` for something secure):
```sql
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
exit
```
✅ This SQL is **identical** on MySQL and MariaDB — it's standard SQL, not a vendor extension.
- `CREATE DATABASE` — new empty DB, `utf8mb4` for full Unicode support (emoji included).
- `CREATE USER` — a *database* login called `librenms`, restricted to `localhost`.
- `GRANT ALL PRIVILEGES` — full rights over the `librenms` database only.

> 🔒 **Recommended:** run `mysql_secure_installation` before this step to set a root password, drop anonymous users, and disable remote root login.

---

## 9️⃣ Configure PHP-FPM

```bash
cp /etc/php/8.5/fpm/pool.d/www.conf /etc/php/8.5/fpm/pool.d/librenms.conf
vi /etc/php/8.5/fpm/pool.d/librenms.conf
```
Edit:
- `[www]` → `[librenms]` *(just a label)*
- `user =` / `group =` → both `librenms`
- `listen =` → `listen = /run/php-fpm-librenms.sock` — **must match** what nginx points to in Step 10

> 💾 Optional: delete `www.conf` if this server won't run any other PHP apps.

---

## 🔟 Configure nginx

```bash
vi /etc/nginx/sites-enabled/librenms.vhost
```
```nginx
server {
 listen      80;
 server_name librenms.example.com;
 root        /opt/librenms/html;
 index       index.php;

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
📝 Replace `librenms.example.com` with your real domain/IP.

| Block | What it does |
|---|---|
| `root /opt/librenms/html` | Only the `html` subfolder is web-accessible — not the whole app |
| `location /` | Serves matching files directly, else routes to `index.php` |
| `location ~ .php` | Forwards `.php` requests to PHP-FPM via the socket from Step 9 |
| `location ~ /\.` | Blocks access to hidden files (e.g. `.env`), except `.well-known` |

```bash
rm /etc/nginx/sites-enabled/default
systemctl reload nginx
systemctl restart php8.5-fpm
```
🔹 Removes nginx's default site so it doesn't conflict, reloads nginx config, and restarts PHP-FPM to pick up the pool file.

---

## 1️⃣1️⃣ Allow Access Through the Firewall

`ufw` is off by default on Ubuntu — only run this if you've enabled it:
```bash
ufw allow 80/tcp
ufw allow 443/tcp
```

---

## 1️⃣2️⃣ Enable `lnms` Command-Line Completion

```bash
ln -s /opt/librenms/lnms /usr/bin/lnms
cp /opt/librenms/misc/lnms-completion.bash /etc/bash_completion.d/
```
🔹 Symlinks the `lnms` management CLI system-wide, then adds Tab-completion for it. Pure convenience.

---

## 1️⃣3️⃣ Configure `snmpd`

```bash
cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
vi /etc/snmp/snmpd.conf
```
🔹 Find `RANDOMSTRINGGOESHERE` → replace with your own **private** SNMP community string.

```bash
curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
chmod +x /usr/bin/distro
systemctl enable snmpd
systemctl restart snmpd
```
🔹 Installs a helper script SNMP uses to report the OS distro, then enables + restarts the SNMP daemon.

---

## 1️⃣4️⃣ Set Up the Cron Job

```bash
cp /opt/librenms/dist/librenms.cron /etc/cron.d/librenms
```
🔹 Installs LibreNMS's pre-written schedule for periodic housekeeping tasks.

---

## 1️⃣5️⃣ Enable the Polling Scheduler

```bash
cp /opt/librenms/dist/librenms-scheduler.service /opt/librenms/dist/librenms-scheduler.timer /etc/systemd/system/
systemctl enable librenms-scheduler.timer
systemctl start librenms-scheduler.timer
```
🔹 Installs a `systemd` service + timer pair — the modern replacement for cron-driven device polling — then enables and starts it.

---

## 1️⃣6️⃣ Enable Log Rotation

```bash
cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```
🔹 Keeps `/opt/librenms/logs` from growing forever by auto-compressing/cleaning old logs.

---

## 1️⃣7️⃣ Run the Web Installer

Visit:
```
http://librenms.example.com/install
```
Follow the wizard — it checks requirements, connects to your `librenms` MySQL database from Step 8, and creates your admin account. **The DB step is identical for MySQL or MariaDB** — same ho[...]

Save the generated config into `/opt/librenms/config.php`, then:
```bash
chown librenms:librenms /opt/librenms/config.php
```

---

## 1️⃣8️⃣ Final Steps

🎉 Log in at `http://librenms.example.com/`

> 🔴 **Security note:** Don't expose this to the public internet yet — it's HTTP-only. Add HTTPS (Let's Encrypt/Certbot) and harden the server first.

**Add your first device** (start with the server itself):
```
http://librenms.example.com/addhost
```

---

## 🆘 Troubleshooting

Run LibreNMS's built-in validator any time something looks off — it checks config, permissions, and DB connectivity:
```bash
su - librenms
./validate.php
```

---

## 📚 Good Next Reads

- 🚀 [Performance tuning](https://docs.librenms.org/Support/Performance/)
- 🔔 [Alerting](https://docs.librenms.org/Alerting/)
- 🗂️ [Device Groups](https://docs.librenms.org/Extensions/Device-Groups/)
- 🔍 [Auto Discovery](https://docs.librenms.org/Extensions/Auto-Discovery/)
- 🧩 [High Availability](https://docs.librenms.org/Support/High-Availability/)

---

<div align="center">

Built for **Ubuntu 24.04 LTS** · **MySQL** backend · Adapted from the [official LibreNMS docs](https://docs.librenms.org/Installation/Install-LibreNMS/)

</div>
