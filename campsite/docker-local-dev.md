# CampSite Local Development (Docker)

Local development runs in Docker Compose; the VirtualBox dev VM is retired. Everything is defined at the **root of the CampSite repo**: `docker-compose.yml`, `Makefile`, and `docker/`.

**Evidence:** checked against CampSite `develop` @ `8e238a1345` and a running stack on Windows 11 (Docker Desktop 29.7, WSL2 backend, `desktop-linux` context) on 2026-10-21. **macOS was not tested.** Items marked *(Windows-verified)* or *(unverified)* say so explicitly.

---

## 1. Prerequisites

| Tool                                        | Notes                                                                                                                                                                                                                                  |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Docker with Compose v2 (`docker compose …`) | *(Windows-verified)* Docker Desktop with the WSL2 backend. On macOS, Docker Desktop is the presumed equivalent *(unverified)*.                                                                                                         |
| GNU make                                    | *(Windows-verified)* GNU Make 4.4.1 installed natively via winget (`ezwinports.make`), runnable from Git Bash or PowerShell; WSL also works. On macOS the Xcode Command Line Tools ship a `make` *(unverified against this Makefile)*. |
| Free host ports **80, 443, 22, 3306**       | Published by `web` and `db`. A host SSH server on port 22 (common on macOS with Remote Login, or Windows OpenSSH Server) or a local MySQL on 3306 conflicts.                                                                           |
| `docker/certbot/cloudflare.ini`             | Cloudflare DNS credentials for the wildcard cert. Already tracked in the repo; never copy its contents.                                                                                                                                |
| A hosts-file entry for your subdomain       | See [§6](#6-urls-dns-and-access). Without it, `<sub>.campmanagement.com` resolves to **production** Cloudflare.                                                                                                                        |

CPU architecture: the web `Dockerfile` downloads wkhtmltopdf by `${TARGETARCH}`, so arm64 (Apple Silicon) appears intended. Only `x86_64` was verified.

---

## 2. Files and services

| File                                    | Purpose                                                                            |
| --------------------------------------- | ---------------------------------------------------------------------------------- |
| `docker-compose.yml`                    | Services (project name `campsite`)                                                 |
| `Makefile`                              | Lifecycle, DB import/reset, and SSL commands                                       |
| `docker/web/`                           | `Dockerfile` (php:8.3-apache), `vhost.conf`, `php/99-campsite.ini`                 |
| `docker/db/`                            | `Dockerfile` (mysql:8.0.45), `my.cnf`, `init/01-init.sql` (creates the `cf4` user) |
| `docker/db-tool/prepare-sql.php`        | Rewrites the SQL dumps for MySQL 8 before import                                   |
| `docker/phpmyadmin/config.user.inc.php` | phpMyAdmin behind `/phpmyadmin/`                                                   |
| `.docker/tmp/`                          | Generated SQL (git-ignored)                                                        |

| Service      | Container             | What it runs                                                                                                                                  | Host ports  |
| ------------ | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| `web`        | `dev-web`             | Apache + PHP 8.3, Composer 2, Python 3, the Oracle `mysql`/`mysqldump` 8.0.45 client, wkhtmltopdf 0.12.6.1, ghostscript, pdftk, poppler, sshd | 80, 443, 22 |
| `db`         | `dev-db`              | MySQL 8.0.45 with `cf4_master` and `cf4_dev`                                                                                                  | 3306        |
| `phpmyadmin` | `campsite-phpmyadmin` | phpMyAdmin, reverse-proxied by `web`                                                                                                          | none        |
| `certbot`    | `dev-certbot`         | Issues and renews `*.campmanagement.com` via Cloudflare DNS; run on demand                                                                    | none        |

All services share Compose's default network; containers reach each other by service name (`db`, `phpmyadmin`).

**PHP runtime.** PHP extensions include mysqli, pdo_mysql, gd, intl, bcmath, mbstring, zip, xsl, and OPcache, plus igbinary, msgpack, memcached, and mcrypt built from source. The ini settings are UTC, `memory_limit=512M`, `max_input_vars=10000`, `error_log=/var/log/phplog`, and `session.name=PHPSESSID2`; no app code depends on the session name.

**Apache.** Modules: rewrite, headers, ssl, unique_id, proxy, proxy_http. `vhost.conf` sets `AccessFileName .htaccess-development` and redirects HTTP to HTTPS.

**MySQL.** `my.cnf` sets `sql_mode=NO_ENGINE_SUBSTITUTION` (non-strict), `innodb_strict_mode=0`, and `max_connections=10000`.

---

## 3. Path mapping (host ↔ container)

| Host (CampSite repo)             | Container                                             | Live?                                                   |
| -------------------------------- | ----------------------------------------------------- | ------------------------------------------------------- |
| `./`                             | `/var/www` (web; DocumentRoot and `exec` working dir) | Bind mount, live                                        |
| `./config/`                      | `/var/config` (web)                                   | Bind mount, live                                        |
| `docker/web/vhost.conf`          | `/etc/apache2/sites-enabled/000-default.conf`         | Bind mount; needs `docker compose restart web`          |
| `docker/web/php/99-campsite.ini` | `/usr/local/etc/php/conf.d/99-campsite.ini`           | **Baked in**: `make up` to rebuild                      |
| `docker/db/my.cnf`               | `/etc/mysql/conf.d/99-custom.cnf` (db)                | **Baked in**: `make up` to rebuild                      |
| `docker/db/init/`                | `/docker-entrypoint-initdb.d` (db)                    | Runs only when `db_data` is first created               |
| volume `db_data`                 | `/var/lib/mysql`                                      | Survives `make down`; destroyed by `make reset-volumes` |
| volume `campsite_certs`          | `/etc/letsencrypt` (rw in certbot, ro in web)         | Persistent                                              |
| volume `ssh_host_keys`           | `/ssh-host-keys`                                      | Persistent sshd host keys                               |

`/var/config` is the same path production uses; only the source differs. Production provisions it on the server, and `AfterInstall.sh` deletes the repo's `config/` (and `docker/`, `Makefile`, `docker-compose.yml`) on deploy. **Local `config/` values say nothing about production values.**

---

## 4. Setup, run, rebuild

Run everything from the CampSite repo root.

```bash
# First time only (or when the cert volume is gone)
make ssl-create                    # wildcard cert via Cloudflare DNS, then restarts web

# First time, or to rebuild the databases. DESTRUCTIVE: drops cf4_master and cf4_dev.
make bootstrap SUBDOMAIN=<you>     # ssl-check → up → wait-db → db-reset → db-import

# Day to day
make up                            # docker compose up -d --build
make down                          # stop; keeps db_data
make health                        # ps, PHP version, extension check, SHOW DATABASES
make logs                          # follow all container logs
docker compose restart web         # after vhost.conf changes
```

What `db-reset` / `db-import` do:

1. Drop and create `cf4_master` and `cf4_dev`.
2. Run `prepare-sql.php`, which converts `varchar(n>16000)` to `mediumtext` in copies under `.docker/tmp/sql/`.
3. Import `sql/cf4_master.sql` into `cf4_master` and `sql/database_demo.sql` into `cf4_dev`.
4. Set **every** `cf4_master.camps.subdomain` to `SUBDOMAIN`, and `cf4_master.nodes.host` to `db`.

**Destructive targets:** `bootstrap`, `fresh` (down → up → db-reset), `db-reset`, `db-import`, `reset-volumes`, and **bare `make`** (`.DEFAULT_GOAL := bootstrap`). An agent must ask before running any of them.

---

## 5. Working in the containers

```bash
docker compose exec web bash                                   # shell (runs as root)
docker compose exec -u www-data web <cmd>                      # run as Apache's user when the output must stay writable by Apache
docker compose exec web php -l classes/class.camper.php
docker compose exec web composer install

# Tests
docker compose exec web php vendor/bin/phpunit                                          # unit suite (phpunit.xml, no DB)
docker compose exec web php vendor/phpunit/phpunit/phpunit -c phpunit-integration.xml   # DB-backed suite, tests/integration/

# Database. The root password is in the db container's environment; never copy it.
make db-shell
docker compose exec -T db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" cf4_dev' < sql/migrations/<file>.sql
docker compose exec -T db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW DATABASES"'

# Logs (inside web, not on the host; lost when the container is recreated)
docker compose exec web tail -n 200 /var/log/phplog
```

- **DB host inside containers is `db`.** `DB_HOST`/`DB_HOST_MASTER` are `db`, and the camp DB host comes from `cf4_master.nodes.host`, also `db`. From the host machine, use `127.0.0.1:3306`.
- **Log files** in the `web` container's `/var/log/`: `phplog` (`ep()` default and PHP `error_log`), `pagelog`, `cronlog`, `stripelog`, `authorizelog`, `payflowprolog`, `pyxislog`. `ep()` to a file that does not exist silently does nothing.
- **Integration tests** boot `classloader.php` as `cron.campmanagement.com` and need the Docker DB. They **skip** rather than fail when no DB is reachable, so check the skip count.
- **The BunkLodgic algorithm** takes a JSON **file path** argument, not stdin: `docker compose exec web python3 algorithm/bunkLodgic_algorithm.py <payload.json>`.
- **Crons** do not run locally (no cron daemon). In deployed environments they are HTTP requests. They read flags from `$_GET`, and only 5 of the 36 scripts have a `liveRun=yes` dry-run switch; the rest act immediately. Local `config/` holds real third-party credentials (e.g. Twilio), so ask before triggering a cron or any email/SMS/payment path.

---

## 6. URLs, DNS, and access

- **App:** `https://<SUBDOMAIN>.campmanagement.com/`. Public DNS for `*.campmanagement.com` points at production Cloudflare, so you need a hosts-file entry `127.0.0.1 <SUBDOMAIN>.campmanagement.com`. *(Windows-verified: `C:\Windows\System32\drivers\etc\hosts`. On macOS the file is `/etc/hosts` (unverified for this stack).)* Without the entry you are on **production**.
- `classloader.php` → `isValidCamp()` returns **403 for `localhost`/`127.0.0.1`** and **404 for a subdomain not in `cf4_master.camps`**, so always browse by subdomain.
- **phpMyAdmin:** `https://<SUBDOMAIN>.campmanagement.com/phpmyadmin/`.
- **SSH:** `web` runs sshd on host port 22 as user `ubuntu` (in `www-data`): `ssh ubuntu@<SUBDOMAIN>.campmanagement.com`, working in `/var/www`.

---

## 7. Troubleshooting (verified behavior)

| Symptom                                                  | Cause                                                                                                  | Fix                                                                               |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| `make health` shows `unique_id: MISSING`                 | `unique_id` is an Apache module, not a PHP extension; the check can never pass                         | Ignore, or confirm with `docker compose exec web apache2ctl -M \| grep unique_id` |
| `make health` shows OPcache missing in ad-hoc checks     | The CLI reports it as `Zend OPcache`                                                                   | `php -r 'var_dump(extension_loaded("Zend OPcache"));'`                            |
| `make phpmyadmin` prints `http://localhost:8081`         | Stale message; the port is not published                                                               | Use `/phpmyadmin/` on your subdomain                                              |
| `make help` lists `make ssh`                             | No such target                                                                                         | `docker compose exec web bash` or SSH as above                                    |
| `make bootstrap` stops with "SSL certificates not found" | Empty `campsite_certs` volume                                                                          | `make ssl-create`                                                                 |
| 403 in the browser                                       | Browsing `localhost`                                                                                   | Use `https://<SUBDOMAIN>.campmanagement.com`                                      |
| 404 for every page                                       | Subdomain not in `cf4_master.camps`                                                                    | Re-import with the right `SUBDOMAIN` (destructive; ask first)                     |
| You are on the live site                                 | No hosts-file entry                                                                                    | Add `127.0.0.1 <SUBDOMAIN>.campmanagement.com`                                    |
| "Page not found" for a new page after a DB reset         | A reset rebuilds `cf4_master` from `sql/cf4_master.sql`, so `pages` rows you inserted by hand are gone | Re-insert them, or add them to the dump/seed if they must survive resets          |
| Port bind error on `make up`                             | A host service already uses 80/443/22/3306                                                             | Stop that service                                                                 |
| Caching behavior can't be reproduced                     | No Memcached server locally                                                                            | Expected; reason about it, or test in a deployed environment                      |
| Files created by CLI commands not writable by Apache     | `docker compose exec` runs as root                                                                     | Use `-u www-data`                                                                 |
| `ini`/`my.cnf` change has no effect                      | Those files are baked into the image                                                                   | `make up` (rebuild)                                                               |

**Recommendation (unverified for this stack):** on Windows, Docker's general guidance is that bind mounts from the Windows filesystem (especially OneDrive-synced folders) are slower than from the WSL filesystem. Consider cloning into WSL if file I/O is slow.

---

## 8. Local vs. legacy VM vs. deployed

| Aspect     | Local Docker (current)                           | Legacy VM (retired)         | Deployed EC2                                                              |
| ---------- | ------------------------------------------------ | --------------------------- | ------------------------------------------------------------------------- |
| Config     | `./config` → `/var/config`                       | `/var/config` on the VM     | `/var/config` on the server *(provisioning not in the repo)*              |
| DB         | MySQL 8.0.45 container; `cf4_master` + `cf4_dev` | Local MySQL                 | MySQL 5.7 master + Aurora 5.6 camp nodes *(not verifiable from the repo)* |
| Memcached  | None                                             | Local Memcached             | ElastiCache *(not verifiable from the repo)*                              |
| Cron       | Not scheduled                                    | —                           | EventBridge → Lambda → ALB → cron EC2                                     |
| htaccess   | `.htaccess-development`                          | —                           | `.htaccess-production`                                                    |
| Reached by | Hosts file → 127.0.0.1                           | Hosts file → `192.168.56.x` | Cloudflare → ALB                                                          |

The legacy VM details come only from leftover hosts-file entries and the "VirtualBox" comment in `99-campsite.ini`.
