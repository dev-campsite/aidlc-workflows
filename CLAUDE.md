# CLAUDE.md — CampSite Technical Context for Code Generation

Read this at the top of every CampSite code conversation. This file encodes always-true facts about the stack, conventions, and architecture so generated code fits immediately.

Local development runs in **Docker Compose**; the old VirtualBox dev VM is retired. Before running, testing, or querying anything, read [Local Development Environment (Docker)](#local-development-environment-docker).

---

## Stack & Versions

| Component | Version                                        | Notes                                                                                                                          |
| --------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| PHP       | 8.3+                                           | `declare(strict_types=1)` in DB/exception classes. Typed constants in ExceptionHandler. Local: `php:8.3-apache` image.         |
| MySQL     | 5.7 / Aurora 5.6 (prod), 8.0.45 (local Docker) | PDO only (no raw `mysqli_*`). Reserved words like `rank` are backtick-quoted. Write SQL that runs on **both** 5.6/5.7 and 8.0. |
| jQuery    | 3.7.1 (legacy), 2.2.0 (BootSite/RAD)           | jQuery Migrate 1.4.1 loaded alongside 3.7.1.                                                                                   |
| Bootstrap | 3.3.5 + AdminLTE 2                             | Used in responsive ("BootSite") pages. Legacy pages use raw HTML/CSS.                                                          |
| Python    | 3.x stdlib-only                                | BunkLodgic placement algorithm at `algorithm/bunkLodgic_algorithm.py`.                                                         |

---

## Forbidden Patterns

**Never do these:**

- ❌ Call `mysqli_*()` directly or use raw string concatenation in SQL → Use `$db->select()` / `update()` / `delete()` wrapper instead.
- ❌ Hardcode table or column names → Use Active Record (AR) objects; let `CLASS_DICTIONARY` resolve metadata.
- ❌ Mix DB credentials or config into code → Credentials live in `/var/config/*.php` (see [Config & Environment](#config--environment)); fetch via constants only.
- ❌ Copy values out of `config/*.php` or `docker/certbot/cloudflare.ini` into code, commits, logs, AIDLC artifacts, or chat → Refer to constants by **name** only.
- ❌ Build queries without parameterized placeholders → Always `$db->select($sql, $params)` with `?` placeholders.
- ❌ Call database directly for multiple related rows → Use `getChildren()` or `getChildObjects()` on AR classes.
- ❌ Ignore the `useReaderHost` parameter for read-heavy queries → Route reads to replica: `$db->select($sql, $params, null, true)`.
- ❌ Write unquoted `rank` in SQL → `` `rank` `` is a reserved word since MySQL 8.0.2 (the local Docker DB); unquoted, it works in prod but fails locally. Always backtick it: `` ORDER BY `rank` ASC ``.
- ❌ Omit the 3rd `$className` argument on `selectObject()`/`selectObjects()` → It is required; missing it causes a fatal "Too few arguments" error.
- ❌ Omit the PK column from a `selectObjects()` query → The AR system uses the PK to instantiate each row; omitting it causes "record does not exist" failures.

---

## Naming Conventions

**Copy these patterns exactly:**

| Type               | Pattern                                 | Example                                                                                  |
| ------------------ | --------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Classes**        | lowerCamelCase                          | `campOptions`, `camperYear`, `bunkLodgicAlgorithmService`                                |
| **DB tables**      | lowerCamelCase plural                   | `campOptions`, `accrualPeriods`, `camperYears`; BunkLodgic tables prefixed `bunkLodgic*` |
| **Primary keys**   | `<ClassName>Id`                         | `campId`, `adminId`, `camperYearId`                                                      |
| **Class files**    | `classes/class.<ClassName>.php`         | `classes/class.camp.php`, `classes/class.camper.php`                                     |
| **AJAX endpoints** | `ajax/crud_<feature>.php`               | `ajax/crud_camper.php`, `ajax/crud_family.php`                                           |
| **Admin pages**    | Bare + optional `_m.php` for responsive | `admin/camper.php`, `admin/camper_m.php` (BootSite variant)                              |
| **Methods**        | lowerCamelCase                          | `getChildren()`, `addChild()`, `getAncestor()`                                           |
| **Indentation**    | Tabs                                    | Everywhere.                                                                              |

---

## _classDictionary.php Rules

`classes/_classDictionary.php` maps every AR class to its table, primary key, and parent relationships.

**Critical constraint — never cross databases in `parents`:**

```php
// WRONG: camp and campOption live in cf4_master, not cf4_dev
'bunkLodgicBoard' => [
    'table' => 'bunkLodgicBoards',
    'pk'    => 'bunkLodgicBoardId',
    'parents' => ['camp', 'campOption'],  // ❌ cross-db — breaks parent traversal silently
],

// CORRECT: only reference tables in the same (camp-level) database
'bunkLodgicBoard' => [
    'table' => 'bunkLodgicBoards',
    'pk'    => 'bunkLodgicBoardId',
    'parents' => [],  // campId scoping handled at the service layer
],
```

If a class has a `campId` FK column, leave `parents` as `[]`. The `camp` and `campOption` tables live in `cf4_master` and cannot be traversed as ORM parents.

---

## Permissions

Permission strings follow the pattern `core<FeatureName><Level>` (all camelCase with `core` prefix).

```php
// Check in PHP
if (in_array('coreBunkLodgicAdmin', $_SESSION['permissionStrings'])) { ... }
if (in_array('coreBunkLodgicEdit',  $_SESSION['permissionStrings'])) { ... }
if (in_array('coreBunkLodgicView',  $_SESSION['permissionStrings'])) { ... }
```

**Migration SQL must match PHP exactly** — if PHP checks `coreBunkLodgicAdmin`, the INSERT into `cf4_master.permissions` must also use `coreBunkLodgicAdmin`, not `bunkLodgicAdmin`.

---

## Core Architecture

### Active Record Pattern

Every database row is an object. All entities inherit from `activeRecord`.

```php
// Instantiate by PK — constructor does SELECT automatically
$camper = new camper(123);  // Looks up CLASS_DICTIONARY for table + pk field

// Access fields as properties
echo $camper->firstName;

// Update & delete
$camper->update(['firstName' => 'Alice']);
$camper->delete();

// Walk relationships
$family = $camper->getParent('family');        // Get parent object
$siblings = $camper->getChildren('camper');    // Get all child campers
$camp = $camper->getAncestor('camp');          // Walk up to 5 levels
```

### DB Wrapper (class.db.php)

Don't write SQL directly. Use the wrapper:

```php
// Read single row as assoc
$row = $db->selectRow($sql, $params);

// Read multiple rows
$rows = $db->select($sql, $params);

// Read scalar value
$count = $db->selectField('SELECT COUNT(*) FROM campers', []);

// Get AR objects directly — 3rd param (className) is REQUIRED
// SELECT must include the PK column; AR instantiation uses it
$campers = $db->selectObjects(
    'SELECT camperId, firstName FROM campers WHERE familyId = ?',
    [123],
    'camper'  // AR class matching the FROM table
);

// Single object variant — same rules
$board = $db->selectObject(
    'SELECT bunkLodgicBoardId, campId FROM bunkLodgicBoards WHERE bunkLodgicBoardId = ?',
    [7],
    'bunkLodgicBoard'
);

// Write
$lastId = $db->insert('INSERT INTO campers (familyId, firstName) VALUES (?, ?)', [123, 'Bob']);
$db->update('UPDATE campers SET firstName = ? WHERE camperId = ?', ['Carol', 456]);
```

**Key: Read/write routing**

```php
// Route reads to replica for heavy queries
$db->select($sql, $params, null, true);  // true = use readerHost
```

### Factory Pattern (class.cf.php)

Use `cf` for bulk lookups and object creation:

```php
$cf = new cf($db);

// Lookup or create
$camp = $cf->pkExistsGetObject('camp', 123);  // Returns object or false
$allStaff = $cf->getAllObjects('staffYear', 'staffYearId DESC');
$admin = $cf->getObjectFromFields('admin', ['email' => 'foo@example.com']);

// Insert & return object
$newCamper = $cf->addNewObject('camper', ['familyId' => 123, 'firstName' => 'Dave']);
```

---

## Key Files & Imports

Every request includes these via `classloader.php`:

```php
// Always available
$db              // PDO wrapper instance
$camp            // Current camp object
$campOptions     // Camp config (e.g., hasBootSite, modules)
$cf              // Factory helper
$admin           // On admin pages only
$family          // On parent portal only
```

**Where things live:**

- **Active Record classes**: `classes/class.*.php` (1,731+ files)
- **Global helpers**: `functions/functions.php` (1,700+ lines; key: `ep()` for logging)
- **Constants**: `useful_constants.php`, `classes/_classDictionary.php`
- **DB bootstrap**: `classes/class.activeRecord.php`, `classes/class.db.php`, `classes/class.cf.php`
- **Admin session**: `admin/inc/init.php` → sets `$_SESSION['adminId']` + `$_SESSION['campInitials']`
- **Parent portal session**: `p/campers/headers.php` → sets `$_SESSION['cf4pc_familyId']`

---

## Config & Environment

Application code loads config from the **absolute path `/var/config/`** (e.g. `require '/var/config/aes.php';`). That path always exists at runtime, but where it comes from depends on the environment:

| Environment                        | Where `/var/config/` comes from                                                                               |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Production / staging / testing EC2 | Provisioned on the server, outside the `/var/www/` deploy target; not shipped by CodeDeploy                   |
| Local Docker                       | Bind mount of the CampSite repo's **`./config/`** directory into the `web` container (`./config:/var/config`) |

So, locally, **edit `./config/<file>.php` on the host**. The change is live inside the container right away. A host-side PHP process cannot resolve `/var/config/...`, which is one more reason to run PHP inside the container.

| File                          | Contains                                                                                                                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `/var/config/db.php`          | `DB_HOST`, `DB_HOST_MASTER`, `DB_USERNAME`, `DB_PASSWORD`, `DB_USERNAME_ESCALATED`, `DB_PASSWORD_ESCALATED`. Locally both hosts are `db` (the Compose service name).                       |
| `/var/config/environment.php` | `ENVIRONMENT` ('development'/'testing'/'production'), `MAINTENANCE_MODE`, `MEMCACHED_SERVERS_DATA`, and the third-party API credentials (below). Locally `ENVIRONMENT` is `'development'`. |
| `/var/config/aes.php`         | `$aesKey` for credit card encryption                                                                                                                                                       |
| `/var/config/pk-*.pem`        | CloudFront URL-signing private key (path in `CLOUDFRONT_URL_SIGNING_PRIVATE_KEY_PATH`)                                                                                                     |

**Third-party credentials** (runtime constants defined in `environment.php`):

- `AWS_REGION`, `AWS_KEY`, `AWS_SECRET` (S3, Rekognition)
- `TWILIO_SID`, `TWILIO_TOKEN` (SMS)
- Plus Duo, Google (Maps/Places/Sheets), Zendesk, StatusPage, SmugMug, HubSpot, QuickBooks, WHCC, CSI, Froala, Intercom, Cloudflare

When code needs a new environment-specific value, add a `define()` to `environment.php` in **every** environment, local `./config/environment.php` included, and tell the developer that the server copies need updating too. Never write the value itself into application code.

**In-repo constants** (`useful_constants.php`):

- `ALLOWED_IMAGE_TYPES`, `US_STATES`, `PAYMENT_GATEWAYS`, etc.

---

## Local Development Environment (Docker)

Local development runs in Docker Compose and replaces the old VirtualBox VM. Everything is defined at the **root of the CampSite repo**:

| File                                    | Purpose                                                                  |
| --------------------------------------- | ------------------------------------------------------------------------ |
| `docker-compose.yml`                    | Service definitions (project name `campsite`)                            |
| `Makefile`                              | Day-to-day commands (`make bootstrap`, `make fresh`, `make db-shell`, …) |
| `docker/web/`                           | Web image: `Dockerfile`, `vhost.conf`, `php/99-campsite.ini`             |
| `docker/db/`                            | DB image: `Dockerfile`, `my.cnf`, `init/01-init.sql`                     |
| `docker/db-tool/prepare-sql.php`        | Rewrites the SQL dumps for MySQL 8 before import                         |
| `docker/phpmyadmin/`, `docker/certbot/` | phpMyAdmin config; Cloudflare DNS credentials for the SSL cert           |
| `.docker/tmp/`                          | Generated scratch output (git-ignored)                                   |

### Services

| Service      | Container             | Image                                      | What it runs                                                                                                              | Host ports           |
| ------------ | --------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `web`        | `dev-web`             | `docker/web/Dockerfile` (`php:8.3-apache`) | Apache + PHP 8.3, Composer 2, Python 3, Oracle `mysql`/`mysqldump` client, wkhtmltopdf, ghostscript, pdftk, poppler, sshd | 80, 443, 22          |
| `db`         | `dev-db`              | `docker/db/Dockerfile` (`mysql:8.0.45`)    | Both `cf4_master` and `cf4_dev`                                                                                           | 3306                 |
| `phpmyadmin` | `campsite-phpmyadmin` | `phpmyadmin:latest`                        | DB GUI, reverse-proxied by `web` at `/phpmyadmin/`                                                                        | none (internal only) |
| `certbot`    | `dev-certbot`         | `certbot/dns-cloudflare`                   | Issues/renews the wildcard `*.campmanagement.com` cert; run on demand                                                     | none                 |

PHP extensions in `web` include mysqli, pdo_mysql, gd, intl, bcmath, mbstring, zip, xsl, opcache, plus PECL igbinary, msgpack, memcached, mcrypt. Run `make health` to verify them.

**No Memcached server runs locally.** The `memcached` extension is installed and `MEMCACHED_SERVERS_DATA` points at `localhost`, but nothing is listening, so cache reads miss and the code falls back to the database. Caching behavior (for example the 24h camp-DB-host lookup) cannot be reproduced locally.

### Path mapping (host ↔ container)

| Host (CampSite repo)             | Container path                                | Notes                                                                                                     |
| -------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `./`                             | `/var/www` (web)                              | Live bind mount; host edits are immediate. Apache `DocumentRoot` and the `exec` working dir.              |
| `./config/`                      | `/var/config/` (web)                          | `db.php`, `environment.php`, `aes.php`, CloudFront key. See [Config & Environment](#config--environment). |
| `docker/web/vhost.conf`          | `/etc/apache2/sites-enabled/000-default.conf` | Bind mount. Uses `.htaccess-development` as `AccessFileName`; HTTP → HTTPS redirect.                      |
| `docker/web/php/99-campsite.ini` | `/usr/local/etc/php/conf.d/99-campsite.ini`   | **Baked in at build**: UTC timezone, 512M memory, `max_input_vars=10000`, `error_log=/var/log/phplog`     |
| `docker/db/my.cnf`               | `/etc/mysql/conf.d/99-custom.cnf` (db)        | **Baked in at build**: `sql_mode=NO_ENGINE_SUBSTITUTION` (non-strict, like prod)                          |
| volume `db_data`                 | `/var/lib/mysql` (db)                         | Survives `make down`; destroyed by `make reset-volumes`                                                   |
| volume `campsite_certs`          | `/etc/letsencrypt`                            | Shared by `certbot` (rw) and `web` (ro)                                                                   |

After changing a **baked-in** file or a Dockerfile, rebuild with `make up` (`docker compose up -d --build`). Files mounted from the repo need no rebuild. Apache config changes need `docker compose restart web`.

### Rules for the agent

- **Run PHP, Composer, PHPUnit, Python and `mysql` inside the `web` container**, from the CampSite repo root: `docker compose exec web <command>`. The host has neither the matching PHP build and extensions nor `/var/config`. Use `-T` when piping input or running non-interactively.
- **Edit files on the host.** The bind mount makes edits live, so there is no copy or deploy step.
- **DB hostname inside containers is `db`**, never `localhost` or `127.0.0.1`. `DB_HOST`/`DB_HOST_MASTER` are `db`, and the import rewrites `cf4_master.nodes.host` to `db`. From the host machine (a GUI client, for example) use `127.0.0.1:3306`.
- **Only one camp database exists locally: `cf4_dev`**, seeded from `sql/database_demo.sql`. `cf4_master` is seeded from `sql/cf4_master.sql`, and every `camps.subdomain` is set to the developer's `SUBDOMAIN`.
- **The DB root password is in the `db` container's environment.** Read it as `$MYSQL_ROOT_PASSWORD` inside the container; don't copy it into commands or files.
- **Never run `make db-reset`, `make fresh`, `make reset-volumes` or `make bootstrap` without asking.** They drop and re-import both databases and wipe local data.
- **App logs live inside the `web` container**, not on the host: `/var/log/phplog` (`ep()` default and PHP `error_log`) plus `pagelog`, `cronlog`, `stripelog`, `authorizelog`, `payflowprolog`, `pyxislog`. They are lost when the container is recreated.
- **MySQL 8 locally, MySQL 5.7 / Aurora 5.6 in prod.** Don't use 8.0-only SQL (CTEs `WITH …`, window functions `OVER (…)`, `JSON_TABLE`, `CHECK` constraints) in application code or migrations. A query that passes locally may still fail in prod.
- **Local schema differs slightly from the dumps.** `prepare-sql.php` converts any `varchar(n)` with `n > 16000` to `mediumtext` (MySQL 8 row-size limits) before import. Don't "fix" those columns back.

### Common commands

Run these from the CampSite repo root. `make` needs GNU make, for example from WSL or Git Bash with make installed.

```bash
# Stack lifecycle
make up                              # build + start all services
make down                            # stop (keeps DB volume)
make health                          # container status, PHP version, required extensions, DB list
make logs                            # follow all container logs

# First-time setup (or full rebuild): DESTRUCTIVE to local DB, so ask first
make ssl-create                      # one-time wildcard cert (needs docker/certbot/cloudflare.ini)
make bootstrap SUBDOMAIN=<you>       # ssl-check → up → wait-db → db-reset (drop + import + set subdomain)

# Shell / PHP / Python in the web container
docker compose exec web bash
docker compose exec web php -l classes/class.camper.php
docker compose exec web composer install
docker compose exec web python3 algorithm/bunkLodgic_algorithm.py < input.json

# Tests
docker compose exec web php vendor/bin/phpunit                                     # unit suite (phpunit.xml, no DB)
docker compose exec web php vendor/phpunit/phpunit/phpunit -c phpunit-integration.xml  # DB-backed suite (tests/integration/)

# Database
make db-shell                                                                      # interactive mysql as root
docker compose exec -T db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" cf4_dev' < sql/migrations/<file>.sql
docker compose exec -T db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT COUNT(*) FROM cf4_dev.campers"'

# Logs
docker compose exec web tail -f /var/log/phplog
```

### URLs & access

- App: `https://<SUBDOMAIN>.campmanagement.com` (for example `dma`). The developer's subdomain must resolve to the machine running Docker. TLS uses the Let's Encrypt wildcard cert in the `campsite_certs` volume.
- phpMyAdmin: `https://<SUBDOMAIN>.campmanagement.com/phpmyadmin/`, proxied through `web`. The `localhost:8081` URL printed by `make phpmyadmin` is stale because the port is not published.
- SSH: `web` runs `sshd` on port 22 with user `ubuntu` (in the `www-data` group), so a remote agent session can work in `/var/www` with `ssh ubuntu@<SUBDOMAIN>.campmanagement.com`. Host keys persist in the `ssh_host_keys` volume.

---

## Live Infrastructure

### Databases

| Database                         | Engine       | Instance                           | Role                                                                                                                       |
| -------------------------------- | ------------ | ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `cf4_master`                     | MySQL 5.7.26 | RDS db.r4.large (2 vCPU / 15 GB)   | Global registry: camps, admins, pages, routing, permissions                                                                |
| `cf4_master` replicas ×3         | MySQL 5.7.34 | RDS db.r4.2xlarge (8 vCPU / 61 GB) | Read replicas (Nodes 1, 4, 5)                                                                                              |
| `cf4_<initials>` (e.g. `cf4_fw`) | Aurora 5.6   | Staging node + Node 2 demo         | Per-camp application data                                                                                                  |
| `cf4_master` + `cf4_dev`         | MySQL 8.0.45 | Local Docker `db` container        | Local development; isolated from prod. See [Local Development Environment (Docker)](#local-development-environment-docker) |

### Compute

| Role                | Instance                             | Count               | Notes                           |
| ------------------- | ------------------------------------ | ------------------- | ------------------------------- |
| Web servers         | EC2 M5.xlarge (4 vCPU), Ubuntu 18.04 | 2–48 (Auto Scaling) | LAMP; `/var/www/` deploy target |
| REST API            | EC2 M4.2xlarge + ALB                 | Auto Scale          | `api.campmanagement.com`        |
| Cron                | Lambda + EC2 C5.xlarge + ALB         | Auto Scale          | `cron.campmanagement.com`       |
| Deploy HQ / Staging | EC2 M5.large                         | 1                   | `uat.staging`                   |
| Testing             | EC2 M5.large                         | 1                   | `testing.campmanagement.com`    |

### Traffic & DNS

```text
Client → Cloudflare → Application Load Balancer → EC2 Auto Scaling Group
    *.campmanagement.com  → camp web traffic
    api.campmanagement.com → REST API
    cron.campmanagement.com → cron Lambda
    campsite.link / campsite.mail → transactional
```

### Other Services

- **ElastiCache (Memcached)**: AR object cache + camp DB host lookup (24h TTL)
- **Amazon S3**: file/image storage; **Route 53**: DNS; **Rekognition**: image processing; **Amazon Translate**
- **OpenVPN 2.8** (T2.small): required for direct AWS network access
- **Rapid7** (M5.large): security scanner

---

## Database Architecture

### Two-Database Model

| Database                          | Role                                                          | Host                             |
| --------------------------------- | ------------------------------------------------------------- | -------------------------------- |
| `cf4_master`                      | Global registry: camps, admins, pages, routing, permissions   | Fixed `DB_HOST_MASTER`           |
| `cf4_<initials>` (e.g., `cf4_fw`) | Per-camp application data (campers, staff, enrollments, etc.) | Dynamic, cached in Memcached 24h |

**Why it matters for code:** Always instantiate AR objects with the correct camp's database. Use `$camp->dbName` or `$db->getDbName()`.

### Query Cache & Caching

- AR objects are cached in `$GLOBALS['arCache']` and `$GLOBALS['objectCache']`.
- Query results are cached in `$GLOBALS['queryCache']` (keyed by MD5 of SQL + params).
- Don't assume fresh DB state in a single request; use `unset()` to clear cache if needed.

---

## UI Layers: Legacy vs. BootSite

Two parallel front-ends coexist:

| Aspect      | Legacy                   | BootSite / RAD          |
| ----------- | ------------------------ | ----------------------- |
| File suffix | `.php`                   | `_m.php`                |
| Bootstrap   | None                     | 3.3.5 + AdminLTE 2      |
| jQuery      | 3.7.1 + UI 1.14.1        | 2.2.0 (CDN) + UI 1.11.4 |
| JS bundle   | `all.js.php?mode=legacy` | `all.js.php?mode=rad`   |

**Routing logic in `admin/inc/html_head.php`:**

- Check `cf4_master.pages.hasBootSite` flag
- Check `admin.hasBootSite` ('All' / 'Mobile' / off)
- Check `?legacy=1` GET param (force legacy)
- Pages named `bunkLodgic_*` → always BootSite

When creating new admin pages, state which layer(s) they target so Claude generates correct HTML/CSS/JS.

---

## Logging & Error Handling

**Primary logger:**

```php
ep($var, $logFile = null);  // Logs to /var/log/phplog or named file with UTC timestamp
```

**Global exception handler** (`exceptions/ExceptionHandler.php`):

- Registered via `spl_set_exception_handler()`
- Logs to `/var/log/phplog`
- Environment-aware output (dev vs. prod)

Locally, all `/var/log/*` files live **inside the `web` container**. Read them with `docker compose exec web tail -n 200 /var/log/phplog`.

**For alerts:**

```php
notifySlack($message, $webhookURL);  // Post to Slack webhook from functions.php
```

---

## APIs & Integrations

**Internal REST API** (`api/v1/`):

- Framework: Slim 3.9.2
- Auth: `X-Camp-Initials` header + API key hashed against `admins.APIKeyHash`
- Response: JSON via `$response->withJson()`
- Route files: `camper.php`, `family.php`, `staff.php`, `enrollment.php`, etc.

**AJAX endpoints** (`ajax/`):

- ~80+ `crud_*.php` files (no REST conventions)
- POST only, JSON responses, multiple actions via `$_POST['action']` parameter

**Major integrations:**

- SendGrid (email), Twilio (SMS), Stripe / Authorize.net / CSIPay (payments)
- AWS S3 (file storage), Slack (webhooks), OneLogin SAML (SSO)
- See `composer.json` / `composer.lock` in the CampSite repo for the full list and versions.

---

## Deployment & CI/CD

**Pipeline**: Bitbucket Pipelines → AWS CodeDeploy → `/var/www/` on EC2 instances

**Schema changes:**

- No ORM migration runner
- Apply raw SQL files via `class.dba.php` tooling
- Schema files in `sql/` (e.g., `cf4_master.sql`, `database_real.sql`); per-change migration scripts go in `sql/migrations/`, following the existing `YYYY-MM-DD-<description>.sql` naming
- Locally, apply a migration to the Docker DB with `docker compose exec -T db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" cf4_dev' < sql/migrations/<file>.sql`. Use `cf4_master` instead for permission or page rows. Keep `sql/database_demo.sql` / `sql/cf4_master.sql` in step when the change has to survive a `make db-reset`.

**Cron jobs** (`cron/~30 scripts`):

- Email dispatch, payment reminders, admin account expiry, SmugMug sync, etc.
- Run on dedicated cron instance (removed from web instances at deploy)

---

## BunkLodgic Module

BunkLodgic is a drag-and-drop cabin/bunk assignment feature. It runs exclusively in BootSite (`_m.php`) and has its own naming conventions.

### Table & Class Names

All tables follow the `bunkLodgic*` prefix (lowercase b):

| Table                               | AR Class                           |
| ----------------------------------- | ---------------------------------- |
| `bunkLodgicBoards`                  | `bunkLodgicBoard`                  |
| `bunkLodgicStagedCamperAssignments` | `bunkLodgicStagedCamperAssignment` |
| `bunkLodgicStagedStaffAssignments`  | `bunkLodgicStagedStaffAssignment`  |
| `bunkLodgicCriteriaConfigs`         | `bunkLodgicCriteriaConfig`         |
| `bunkLodgicCriteria`                | `bunkLodgicCriterion`              |
| `bunkLodgicLocks`                   | `bunkLodgicLock`                   |
| `bunkLodgicSnapshots`               | `bunkLodgicSnapshot`               |
| `bunkLodgicOverrides`               | `bunkLodgicOverride`               |

### Permissions

| String                | Grants                      |
| --------------------- | --------------------------- |
| `coreBunkLodgicAdmin` | Full admin including config |
| `coreBunkLodgicEdit`  | Move campers / staff        |
| `coreBunkLodgicView`  | Read-only view              |

### Python Algorithm

`algorithm/bunkLodgic_algorithm.py` — Python 3, stdlib-only placement optimizer. Called via PHP shell_exec from the service layer; communicates via stdin/stdout JSON. Do not add third-party Python dependencies.

### UI Conventions

Pages use Bootstrap 3 / AdminLTE 2 design language:

- Division headers: `#2980b9` (site staff blue)
- Board wrapper: `.box` panel style (`border-radius: 5px`, `box-shadow: 0 8px 42px 0 rgba(0,0,0,0.08)`)
- Buttons: Bootstrap 3 `.btn` sizing (`padding: 6px 14px`, `border-radius: 2px`, `font-size: 13px`)
- Font: `'Source Sans Pro', sans-serif` at 13px (AdminLTE default)

---

## When to Reference Deeper Docs

- **[Forbidden Patterns](#forbidden-patterns)** above: the anti-patterns Claude should avoid (raw `mysqli_*`, mixing concerns, hardcoded paths)
- **`docker-compose.yml`, `Makefile`, `docker/`** in the CampSite repo: authoritative for the local environment if this file and the Docker setup ever disagree
- **The codebase itself**: For questions on specific integrations, payment flows, or historical decisions; CampSite has 1,731+ classes — ask about the specific one
- **`classes/_classDictionary.php`**: Authoritative source for every class ↔ table ↔ PK ↔ parent relationship
- **`functions/functions.php`**: ~1,700-line global helper library; use `ep()` for logging, `notifySlack()` for alerts

---

## Code Review Checklist (For Claude)

When generating PHP code for CampSite:

- [ ] Using AR classes for DB access, not raw SQL?
- [ ] Queries parameterized with `?` placeholders?
- [ ] Naming matches lowerCamelCase / `<ClassName>Id` / `class.<Name>.php`?
- [ ] Correct database context (`$camp->dbName` for per-camp data)?
- [ ] Reads routed to replica if heavy (`useReaderHost = true`)?
- [ ] No raw `mysqli_*` or string concatenation in SQL?
- [ ] Secrets/config fetched from constants, not hardcoded? No config values copied into code, commits or artifacts?
- [ ] SQL portable across MySQL 5.6/5.7 (prod) and 8.0 (local Docker)? No CTEs, window functions or `JSON_TABLE`?
- [ ] Verified by running inside the `web` container (`docker compose exec web …`), not with host PHP?
- [ ] Appropriate UI layer (legacy `.php` or BootSite `_m.php`)?
- [ ] Errors logged via `ep()` or caught by global handler?
- [ ] `selectObject`/`selectObjects` has 3rd `$className` param AND PK column in SELECT?
- [ ] No `camp` or `campOption` listed in `parents` in `_classDictionary.php`?
- [ ] Permission strings use `core` prefix (e.g. `coreFeatureAdmin`)?
- [ ] Any SQL using `rank` as a column name has it backtick-quoted?

---

## Quick Reference: Common Tasks

```php
// Fetch a camper and their family
$camper = new camper(123);
$family = $camper->getParent('family');

// Find all campers in a family
$allCampers = $family->getChildren('camper');

// Insert a new family payment
$newPayment = $cf->addNewObject('transaction', [
    'familyId' => 456,
    'amount' => 99.99,
    'transactionType' => 'payment'
]);

// Query with read routing
$recentCampers = $db->select(
    'SELECT * FROM campers WHERE createdDate > ? ORDER BY createdDate DESC',
    [$startDate],
    null,
    true  // Use replica
);

// Log & notify
ep('Payment failed for family ' . $familyId, '/var/log/stripelog');
notifySlack("Payment alert: family $familyId", $webhookURL);
```
