<div align="center">

# 📡 LibreNMS Deployment Journal
### Ubuntu 24.04 · MySQL · nginx · PHP 8.5 — Full install-to-production record

![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![MySQL](https://img.shields.io/badge/Database-MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.5-777BB4?style=for-the-badge&logo=php&logoColor=white)
![nginx](https://img.shields.io/badge/Webserver-nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Status](https://img.shields.io/badge/Status-Live-brightgreen?style=for-the-badge)

</div>

---

## 📖 What This Document Is

This is a complete record of standing up LibreNMS on server `anoq` — from the initial
install through every issue hit along the way, in the order they actually happened. It
exists so the same mistakes don't get repeated on the next server, and so a future "why is
X broken" moment can be matched against something that's already been solved once.

> 🖥️ **Server:** `anoq` (Ubuntu 24.04, `noble`) · **DB:** MySQL 8.0.46 · **PHP:** 8.5.10

---

## 📑 Table of Contents

1. [Part 1 — Base Installation](#part-1--base-installation)
2. [Part 2 — Issue: Database Access Denied (using password: YES)](#part-2--issue-database-access-denied-using-password-yes)
3. [Part 3 — Issue: Database Access Denied (using password: NO)](#part-3--issue-database-access-denied-using-password-no)
4. [Part 4 — Issue: Database Schema Missing](#part-4--issue-database-schema-missing)
5. [Part 5 — Issue: Scheduler / Python Cron Warnings](#part-5--issue-scheduler--python-cron-warnings)
6. [Part 6 — Issue: 502 Bad Gateway (the long one)](#part-6--issue-502-bad-gateway-the-long-one)
7. [Part 7 — Optional: Redis for Caching/Locking](#part-7--optional-redis-for-cachinglocking)
8. [✅ Final State](#-final-state)
9. [🧠 Lessons Learned](#-lessons-learned)

---

## Part 1 — Base Installation

Followed the official LibreNMS install process for Ubuntu 24.04, with one deliberate
change: **MySQL instead of MariaDB**. Ubuntu 24.04's own repos only ship PHP 8.3, so the
**Sury PHP repository** was added to get PHP 8.5 (LibreNMS requires 8.4+).

<details>
<summary>📦 <strong>Click to expand — full install command sequence</strong></summary>

```bash
# 1. Update and add the Sury PHP repo
apt update && apt upgrade -y
apt install -y lsb-release ca-certificates curl
curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
dpkg -i /tmp/debsuryorg-archive-keyring.deb
echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
apt update

# 2. Install all required packages (MySQL edition)
apt install acl curl fping git mysql-server mysql-client mtr-tiny nginx-full nmap \
php8.5-cli php8.5-curl php8.5-fpm php8.5-gd php8.5-gmp php8.5-mbstring php8.5-mysql php8.5-snmp php8.5-xml php8.5-zip \
python3-command-runner python3-dotenv python3-pip python3-psutil python3-pymysql python3-redis python3-setuptools python3-systemd \
rrdtool snmp snmpd traceroute unzip whois

# 3. Create the librenms system user
useradd librenms -d /opt/librenms -M -r -s "$(which bash)"

# 4. Download LibreNMS
cd /opt
git clone https://github.com/librenms/librenms.git

# 5. Set file permissions
chown -R librenms:librenms /opt/librenms
chmod 771 /opt/librenms
setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/

# 6. Install PHP dependencies via Composer
su - librenms
./scripts/composer_wrapper.php install --no-dev
exit

# 7. Set the timezone in both php.ini files and at the OS level
# (edited date.timezone in /etc/php/8.5/fpm/php.ini and /etc/php/8.5/cli/php.ini)
timedatectl set-timezone Etc/UTC

# 8. Configure MySQL
# Added under [mysqld] in /etc/mysql/mysql.conf.d/mysqld.cnf:
#   innodb_file_per_table=1
#   lower_case_table_names=0
systemctl enable mysql
systemctl restart mysql
mysql -u root
```
```sql
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
```
```bash
# 9. Configure PHP-FPM pool
cp /etc/php/8.5/fpm/pool.d/www.conf /etc/php/8.5/fpm/pool.d/librenms.conf
# Edited: [www] -> [librenms], user/group -> librenms, listen -> /run/php-fpm-librenms.sock

# 10. Configure nginx
# Created /etc/nginx/sites-enabled/librenms.vhost pointing at
# fastcgi_pass unix:/run/php-fpm-librenms.sock
rm /etc/nginx/sites-enabled/default
systemctl reload nginx
systemctl restart php8.5-fpm

# 11-16. Firewall (skipped, ufw inactive), lnms CLI symlink, snmpd config,
# cron job, systemd polling scheduler, log rotation
ln -s /opt/librenms/lnms /usr/bin/lnms
cp /opt/librenms/misc/lnms-completion.bash /etc/bash_completion.d/
cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
# edited community string in snmpd.conf
curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
chmod +x /usr/bin/distro
systemctl enable snmpd && systemctl restart snmpd
cp /opt/librenms/dist/librenms.cron /etc/cron.d/librenms
cp /opt/librenms/dist/librenms-scheduler.service /opt/librenms/dist/librenms-scheduler.timer /etc/systemd/system/
systemctl enable librenms-scheduler.timer
systemctl start librenms-scheduler.timer
cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```
</details>

**Result of first `validate.php` run:** multiple failures — this is where the real
troubleshooting begins.

---

## Part 2 — Issue: Database Access Denied (using password: YES)

### Symptom
```
[FAIL]  The database credentials are incorrect.
SQLSTATE[HY000] [1045] Access denied for user 'librenms'@'localhost' (using password: YES)
```

### Diagnosis
`.env`'s `DB_PASSWORD` didn't match the password actually set on the MySQL user — likely
because the web installer generated a different random value than what was set manually in
Part 1, Step 8.

### Fix
```bash
sudo grep DB_PASSWORD /opt/librenms/.env
mysql -u root -e "ALTER USER 'librenms'@'localhost' IDENTIFIED BY 'P@ssw0rd'; FLUSH PRIVILEGES;"
sudo chown librenms:librenms /opt/librenms/.env
sudo chmod 640 /opt/librenms/.env
```

**Result:** error changed from `(using password: YES)` to `(using password: NO)` — progress,
but a new symptom.

---

## Part 3 — Issue: Database Access Denied (using password: NO)

### Symptom
```
SQLSTATE[HY000] [1045] Access denied for user 'librenms'@'localhost' (using password: NO)
```
The "NO" here was the key clue — LibreNMS wasn't sending a *wrong* password anymore, it was
sending **none at all**.

### Root Cause
```bash
$ sudo grep DB_PASSWORD /opt/librenms/.env
#DB_PASSWORD=
```
The line was **commented out** (`#` prefix) — so `.env` never actually defined a password
for LibreNMS to send.

### Fix
```bash
# Uncommented and set the value in .env:
DB_PASSWORD=P@ssw0rd

mysql -u root -e "ALTER USER 'librenms'@'localhost' IDENTIFIED BY 'P@ssw0rd'; FLUSH PRIVILEGES;"
sudo -u librenms php /opt/librenms/validate.php
```

**Result:** `Database | MySQL 8.0.46-0ubuntu0.24.04.4` — connected successfully for the
first time.

---

## Part 4 — Issue: Database Schema Missing

### Symptom
```
DB Schema | No Schema (0)
Exception: SQLSTATE[42S02]: Base table or view not found: 1146 Table 'librenms.devices' doesn't exist
```
Same error repeated for `notifications`, `cache`, and `cache_locks` tables when `./daily.sh`
was run.

### Root Cause
The database existed and was reachable, but was **empty** — the Laravel migrations that
create LibreNMS's actual tables had never been run.

### Fix
```bash
cd /opt/librenms
sudo -u librenms php artisan migrate --force
```
> `--force` is required because Laravel otherwise refuses to run migrations when it doesn't
> think it's in a local/dev environment.

**Result:**
```
DB Schema | 2026_09_13_120000_close_stuck_device_outages (400)
[OK]    Database Schema is current
[OK]    Database schema correct
```

---

## Part 5 — Issue: Scheduler / Python Cron Warnings

### Symptom
```
[FAIL]  Python wrapper cron entry is not present
[FAIL]  Scheduler is not running
```

### Fix
```bash
# Re-install the systemd timer/service pair
sudo cp /opt/librenms/dist/librenms-scheduler.service /opt/librenms/dist/librenms-scheduler.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable librenms-scheduler.timer
sudo systemctl start librenms-scheduler.timer

# Re-install the full cron file (restores the Python wrapper entry)
sudo cp /opt/librenms/dist/librenms.cron /etc/cron.d/librenms
sudo systemctl restart cron
```

### Verification
```bash
sudo systemctl status librenms-scheduler.timer   # active (waiting)
sudo systemctl status librenms-scheduler.service # inactive (dead), status=0/SUCCESS — correct for a oneshot job
sudo systemctl list-timers --all | grep librenms # confirms ~1 min cadence
```

> 💡 **Note:** a timer showing `active (waiting)` and its paired `.service` showing
> `inactive (dead)` right after a successful run is **normal**, not broken — the service
> only "wakes up" once a minute, runs, and goes back to sleep.

**Result:** both `[FAIL]`s cleared; `[OK] Python poller wrapper is polling`.

---

## Part 6 — Issue: 502 Bad Gateway (the long one)

This was the biggest chain of the whole deployment — several *different* root causes
surfaced one after another under the same symptom.

### 6.1 — PHP-FPM pool never actually configured

**Symptom:** `curl -I http://localhost/` → `502 Bad Gateway`

**Diagnosis:**
```bash
$ ls -la /run/php-fpm-librenms.sock
ls: cannot access '/run/php-fpm-librenms.sock': No such file or directory

$ grep -n '^user' /etc/php/8.5/fpm/pool.d/librenms.conf
28:user = www-data
```
The `librenms.conf` pool file was still an **unedited copy** of the default `www.conf` —
Step 9 of the original install had been copied but never actually edited.

**Fix:**
```bash
sudo nano /etc/php/8.5/fpm/pool.d/librenms.conf
# [www] -> [librenms]
# user = www-data -> user = librenms
# group = www-data -> group = librenms
# listen = ... -> listen = /run/php-fpm-librenms.sock
sudo php-fpm8.5 -t
sudo systemctl restart php8.5-fpm
```
**Verified fixed:** socket appeared, `curl` returned non-502 at this point.

---

### 6.2 — nginx vhost file wiped to a single line

**Symptom:** 502 returned again later, and `nginx -t` failed with:
```
"fastcgi_pass" directive is not allowed here in /etc/nginx/sites-enabled/librenms.vhost:1
```

**Diagnosis:**
```bash
$ cat -n /etc/nginx/sites-enabled/librenms.vhost
     1  fastcgi_pass unix:/run/php-fpm-librenms.sock;
```
The entire file had been reduced to a single orphaned line — almost certainly from an
interrupted editor session (a leftover `.swp`/`.swo` file was also spotted in the PHP-FPM
pool directory around the same time, suggesting a save/editor mishap was an ongoing risk
during this session).

**Fix:** rewrote the file completely from scratch using `tee` + heredoc (no editor involved,
so no risk of a repeat):
```bash
sudo tee /etc/nginx/sites-enabled/librenms.vhost > /dev/null << 'EOF'
server {
 listen      80;
 server_name anoq.mainnet.com;
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
EOF
sudo nginx -t
```

---

### 6.3 — nginx service left in `failed` state

**Symptom:** `sudo systemctl reload nginx` → `nginx.service is not active, cannot reload.`

**Diagnosis:** nginx had attempted an auto-restart *while* the vhost file was still the
broken single-line version (6.2), failed, and systemd marked the service `failed`. A
`failed` unit doesn't recover on its own from `reload` — it needs a proper `start`.

**Fix:**
```bash
sudo systemctl start nginx
sudo systemctl status nginx --no-pager -l   # confirmed active (running)
```

---

### 6.4 — Duplicate nginx site definition in `conf.d`

**Symptom:** 502 persisted even with a correct `sites-enabled/librenms.vhost` and nginx
running. `nginx -t` also printed:
```
[warn] conflicting server name "anoq.mainnet.com" on 0.0.0.0:80, ignored
```

**Diagnosis:**
```bash
$ grep -rn "server_name" /etc/nginx/sites-enabled/ /etc/nginx/conf.d/ /etc/nginx/nginx.conf
/etc/nginx/sites-enabled/librenms.vhost:3: server_name anoq.mainnet.com;
/etc/nginx/conf.d/librenms.conf:3: server_name anoq.mainnet.com;
```
A **second, separate config file** existed at `/etc/nginx/conf.d/librenms.conf` — a full
duplicate of the vhost, but still pointing at the old broken socket path
(`/run/php8.5-fpm.sock`). Since `conf.d/*.conf` loads before `sites-enabled/*` in nginx's
default include order, this stale duplicate was the one actually being served, silently
overriding every fix made to `sites-enabled/librenms.vhost`.

**Fix:**
```bash
sudo rm /etc/nginx/conf.d/librenms.conf
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost/
```

**Result:** `curl` finally returned a real response instead of `502`. Confirmed shortly
after by nginx's error log showing successful `fastcgi://unix:/run/php-fpm-librenms.sock`
connections for real page loads (`/install/checks`, `/validate`, `/login`,
`/health/metric=mempool`, `/search/...`).

> 🧵 **Six-part thread summary for 502:** wrong pool config → fixed → vhost file wiped →
> rewritten → nginx left in failed state → started → duplicate conf.d file silently
> overriding the fix → deleted. Each fix was real and necessary, but a *different* cause was
> hiding behind the same symptom each time.

---

## Part 7 — Optional: Redis for Caching/Locking

### Symptom (non-blocking)
```
[WARN]  database is in use for locking. Set CACHE_STORE=redis.
```
`redis-server` was already installed and running on `127.0.0.1:6379`; LibreNMS was just
using MySQL for its cache/lock storage instead, which works but is slower.

### Fix
```bash
# In /opt/librenms/.env:
CACHE_STORE=redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

cd /opt/librenms
sudo -u librenms php artisan config:clear
sudo -u librenms php /opt/librenms/validate.php
```

---

## ✅ Final State

```
[OK]    Database Connected
[OK]    Database Schema is current
[OK]    SQL Server meets minimum requirements
[OK]    lower_case_table_names is enabled
[OK]    MySQL engine is optimal
[OK]    Database and column collations are correct
[OK]    Database schema correct
[OK]    MySQL and PHP time match
[OK]    Active pollers found
[OK]    Dispatcher Service not detected
[OK]    Locks are functional
[OK]    Python poller wrapper is polling
[OK]    No errors found with polling frequencies.
[OK]    rrd_dir is writable
[OK]    rrdtool version ok
```

LibreNMS is reachable over HTTP at `http://anoq.mainnet.com/` (and by server IP), logged in,
polling at least one device, and the scheduler/cron are both confirmed running on their
expected cadence. A handful of PHP deprecation notices remain in `nginx/error.log`
(`LoadUserPreferences.php`, backtick-operator warnings) — these are cosmetic upstream code
issues tied to running a `26.9.0-dev` build against PHP 8.5, not configuration problems, and
don't block page loads.

---

## 🧠 Lessons Learned

- **"Access denied" error text matters.** `(using password: YES)` vs `(using password: NO)`
  point to two completely different problems — a wrong password vs. no password being sent
  at all (e.g. a commented-out `.env` line).
- **An empty database still "connects."** `validate.php` can report a healthy DB connection
  while every table is missing — always confirm `php artisan migrate` was actually run, not
  just that credentials work.
- **A `[FAIL]` on a timer/scheduler can be stale.** Re-run the check a few seconds later
  before assuming it's actually broken — a oneshot systemd service legitimately shows
  `inactive (dead)` between runs.
- **502 Bad Gateway can stack multiple unrelated causes.** Don't stop investigating after
  the first fix if the symptom returns — check the *current* error log and config each time,
  since a new/different cause can produce the exact same HTTP status.
- **`nginx -T` (capital T) shows the fully-loaded, merged config** — invaluable for spotting
  a duplicate or stale file overriding the one you think is active.
- **Avoid interrupted editor sessions on live config files.** Prefer `tee ... << 'EOF'`
  heredocs for one-shot, atomic config writes when reliability matters more than
  interactivity — it removes the risk of a half-saved file entirely.
