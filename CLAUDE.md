# CLAUDE.md — CampSite Technical Context for Code Generation

Read this at the start of every CampSite conversation. It holds the high-priority facts and rules that generated code must respect. The detailed references live next to this file in [`campsite/`](campsite/); keep that folder with this file if you copy it into a project.

| Reference                                                          | Read it when…                                                                                   |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| [campsite/docker-local-dev.md](campsite/docker-local-dev.md)       | You run, test, query, or debug anything locally (Docker setup, commands, logs, troubleshooting) |
| [campsite/system-architecture.md](campsite/system-architecture.md) | You touch request bootstrap, tenancy, auth, AJAX/API/cron, integrations, deployment, or schema  |
| [campsite/lessons-learned.md](campsite/lessons-learned.md)         | You plan or review a non-trivial change, a migration, or the Laravel proof of concept           |

**Evidence baseline.** Statements here were checked against the CampSite repo's `develop` branch at commit `8e238a1345` and a running local Docker stack on 2026-10-21. Facts that come from outside the repo, such as production infrastructure, are marked *(not verifiable from the repo)*. If code and this file disagree, the code wins: say so, and flag the doc for correction.

---

## Environments at a glance

| Aspect    | Local development                                                                    | Deployed (production, staging, testing)                                                       |
| --------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Runtime   | Docker Compose: `web` (php:8.3-apache), `db` (MySQL 8.0.45), `phpmyadmin`, `certbot` | EC2 behind Cloudflare and an ALB; separate `web`, `cron` and `api` deployment groups          |
| Code path | CampSite repo bind-mounted at `/var/www`                                             | CodeDeploy copies the repo to `/var/www`                                                      |
| Config    | Repo `./config/` bind-mounted at `/var/config`                                       | `/var/config/` provisioned on the server; `AfterInstall.sh` deletes the repo's `config/` copy |
| Databases | `cf4_master` + a single camp DB, `cf4_dev`                                           | `cf4_master` + one `cf4_<initials>` DB per camp, spread across nodes                          |
| Memcached | Extension installed, **no server**, so every cache read misses                       | ElastiCache *(not verifiable from the repo)*                                                  |
| Cron      | No cron daemon; nothing runs on a schedule                                           | HTTP requests: EventBridge → Lambda → ALB → cron EC2                                          |

The VirtualBox dev VM is **retired**. Ignore any VM-era instructions, such as `192.168.56.x` hosts entries or a local Memcached.

`/var/config/*.php` is **not** VM-specific. It is the runtime config path everywhere: `classloader.php` loads `PROJECT_ROOT . '/../config/db.php'` and `environment.php` (from `/var/www`, that resolves to `/var/config`), and about 48 files `require '/var/config/aes.php'` directly. What differs is where the directory comes from (see the table above). Never infer production config values from the local `./config/` files.

---

## Critical rules

1. **Run code inside the `web` container**: `docker compose exec web <cmd>` from the CampSite repo root. The host has neither the PHP build nor `/var/config`. Edit files on the host; the bind mount makes edits live.
2. **Ask before anything destructive.** That covers `make bootstrap`, `make fresh`, `make db-reset`, `make reset-volumes`, and **bare `make`**, whose default goal is `bootstrap`. All of them drop and re-import both databases. Also ask before triggering crons, email/SMS sends, or payment calls: the local `config/` holds real third-party credentials.
3. **Never copy secrets.** Refer to config constants by name only. Never paste values from `config/*.php`, `docker/certbot/cloudflare.ini`, the `Makefile`, or `docker-compose.yml` into code, commits, logs, AI-DLC artifacts, or chat.
4. **Use the `db` wrapper with `?` placeholders.** No string-concatenated SQL and no raw `mysqli_*`.
5. **`$db` is a global that everything binds to.** `activeRecord` and `cf` both read `global $db` when constructed. Make sure it points at the right database (`new db($camp->dbName)` for camp data, `new db('cf4_master')` for global data) before you instantiate objects. Any SQL string containing `cf4_master` is routed to the master DB automatically.
6. **Transactions are no-ops.** `db::beginTransaction()`, `commit()`, and `rollBack()` are empty stubs, so multi-statement writes are **not atomic**. Design writes to be safe without atomicity, and say so in plans and reviews. Don't write code that relies on rollback.
7. **Check permissions in every AJAX endpoint.** Admin *pages* are gated automatically through the auto-prepended `admin/inc/html_head.php`, but `ajax/*.php` endpoints are not. The check is `!empty($_SESSION['permissionStrings']['<permissionString>'])`: `permissionStrings` is an associative `string => true` array, not a list.
8. **Don't assume CSRF protection.** There is no global CSRF layer, and admin AJAX has none. Parent-portal code uses `setCSRFToken`/`getCSRFToken`/`clearCSRFToken` from `p/inc/load_db.php`; use those helpers on new state-changing parent endpoints.
9. **Write SQL that runs on MySQL 5.6/5.7 (prod) and 8.0 (local).** Avoid CTEs, window functions, `JSON_TABLE`, and `CHECK` constraints. Always backtick `` `rank` ``: it has been reserved since MySQL 8.0.2, so unquoted it works in prod but fails locally.
10. **Schema changes are manual SQL.** Add a `sql/migrations/YYYY-MM-DD-<slug>.sql` script. It is run by hand against **every** `cf4_<initials>` DB (or `cf4_master`), and code that depends on it ships only after the rollout. There is no migration runner.
11. **Preserve existing behavior.** This is a large legacy multi-tenant app. Match the surrounding code's idiom, change incrementally, and verify the whole workflow, not just one method (see [lessons-learned](campsite/lessons-learned.md)).
12. **Laravel is not production.** The Laravel port exists only on the unmerged `feature/laravel-campsite` branch. Generate legacy PHP for CampSite unless told otherwise.

---

## Stack & Versions

| Component            | Version                                                                                             | Notes                                                                                                                                            |
| -------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| PHP                  | 8.3                                                                                                 | `php:8.3-apache` locally and in CI. `declare(strict_types=1)` appears only in some newer files (e.g. `exceptions/*`), **not** in `class.db.php`. |
| MySQL                | 5.7 (`cf4_master`) / Aurora 5.6 (camp DBs) in prod *(not verifiable from the repo)*; 8.0.45 locally | PDO via the `db` wrapper.                                                                                                                        |
| Slim                 | 3.9.2                                                                                               | REST API in `api/v1/`.                                                                                                                           |
| PHPUnit              | 11.5                                                                                                | Unit suite runs in Bitbucket CI; the integration suite is local only.                                                                            |
| jQuery               | 3.7.1 + Migrate 1.4.1 + UI 1.14.1 (legacy); 2.2.0 + UI 1.11.4 (BootSite)                            | BootSite libraries load from cdnjs.                                                                                                              |
| Bootstrap / AdminLTE | 3.3.5 / 2.3.3                                                                                       | BootSite (`_m.php`) pages only.                                                                                                                  |
| Python               | 3.x, stdlib only                                                                                    | BunkLodgic placement algorithm.                                                                                                                  |

---

## Naming Conventions

| Type               | Pattern                                                                           | Example / caveat                                                                                                             |
| ------------------ | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| AR classes         | lowerCamelCase, one per file `classes/class.<name>.php`, no namespace             | `camper`, `camperYear`; about 1,760 class files, autoloaded by name                                                          |
| Namespaced code    | `Campsite\{Interfaces,Helpers,Gateways}` → `interfaces/`, `helpers/`, `gateways/` |                                                                                                                              |
| DB tables          | Usually lowerCamelCase plural                                                     | Exceptions exist (e.g. `bunkLodgicCriteria`); always check the dictionary                                                    |
| Primary keys       | Usually `<className>Id`                                                           | **Not universal.** Most BunkLodgic tables use `id`, `bunkLodgicBoard` uses `boardId`, `bunkLodgicCriteria` uses `criteriaId` |
| AJAX endpoints     | `ajax/crud_<feature>.php`                                                         | 525 files                                                                                                                    |
| Admin pages        | `admin/<page>.php`, BootSite variant `admin/<page>_m.php`                         |                                                                                                                              |
| Permission strings | `<module.name>` + `ucfirst(<permission.name>)`                                    | `campersEdit`, `campersFinancialManage`, `coreAdmin`, `coreBunkLodgicAdmin`                                                  |
| Indentation        | Match the file                                                                    | Legacy files use tabs; newer service classes use 4 spaces                                                                    |

---

## _classDictionary.php

`classes/_classDictionary.php` defines `CLASS_DICTIONARY` as a **JSON-encoded string** that maps each AR class to `tableName`, `pkFieldName`, and `parents`. It is the authoritative source for class ↔ table ↔ PK ↔ parent; check it rather than guessing.

```php
"bunkLodgicBoard" => array(
    "tableName"   => "bunkLodgicBoards",
    "pkFieldName" => "boardId",
    "parents"     => array("session"),
),
"camp" => array(
    "tableName"   => "cf4_master.camps",   // cf4_master tables carry the db prefix
    "pkFieldName" => "campId",
    // ...
),
```

**Rule: a per-camp class must never list a `cf4_master` class (e.g. `camp`, `campOption`) in `parents`.** Parent traversal queries the current camp DB. Only `cf4_master.*` classes, such as `campNode` and `campHistory`, list `camp` as a parent. If a per-camp table has a `campId` column, scope by it in the service layer.

---

## Data Layer

```php
// Active Record: the constructor SELECTs by PK using CLASS_DICTIONARY
$camper = new camper(123);               // throws if the PK is non-numeric, the class is unknown, or the row is missing
$camper->update(['firstName' => 'Alice']);   // offsets datetime fields to UTC, clears arCache
$family   = $camper->getParent('family');
$campers  = $family->getChildren('camper');   // default ORDER BY <childPk> DESC; signature ($class, $order, $fields, $useReaderHost)
$camp     = $camper->getAncestor('camp');      // walks up to 5 levels
$newChild = $family->addChild('camper', ['firstName' => 'Dave']);  // sets the parent FK and created

// db wrapper (classes/class.db.php)
$rows  = $db->select($sql, $params, true);            // 3rd arg = use read replica
$row   = $db->selectRow($sql, $params, true);         // 3rd arg = replica; 4th = utf8mb4 connection
$value = $db->selectField($sql, $params, true);       // 3rd arg = replica
$objs  = $db->selectObjects($sql, $params, 'camper', true);  // 3rd = class (REQUIRED), 4th = replica
$obj   = $db->selectObject($sql, $params, 'camper');
$id    = $db->insert($sql, $params);                  // returns lastInsertId or false
$ok    = $db->update($sql, $params);                  // bool; same for delete()
$name  = $db->dbName;                                 // e.g. "cf4_fw" (there is no getDbName())

// cf factory: the constructor takes NO arguments and uses global $db
$cf    = new cf();
$obj   = $cf->pkExistsGetObject('camp', 123);         // instance method; object or false
$admin = cf::getObjectFromFields('admin', ['email' => $email]);   // static
$all   = cf::getAllObjects('staffYear', 'staffYearId DESC');      // static; parentless classes only
$row   = cf::addNewObject('someClass', ['field' => 'x']);        // static; parentless classes only
```

- `selectObject(s)` SQL **must include the PK column**, because objects are instantiated from it.
- Request-scoped caches: `$GLOBALS['arCache']`, `$GLOBALS['objectCache']`, and `$GLOBALS['queryCache']`. Every insert, update, or delete clears the whole query cache. Within one request you may read cached state.
- `getChildObjects()` and `db::getDbName()` **do not exist**. Don't generate calls to them.
- `useful_constants.php` arrays (`US_STATES`, `ALLOWED_IMAGE_TYPES`, `PAYMENT_GATEWAYS`, …) are **serialized strings**: `unserialize(US_STATES)`.

---

## Request bootstrap (summary)

`classloader.php` loads constants, registers `ExceptionHandler`, loads `/var/config/db.php` and `environment.php`, registers the autoloader, loads `functions/functions.php`, `libphutil.php`, and `vendor/autoload.php` **in that order** (the order is load-bearing), then calls `isValidCamp()`. That call returns 403 for `localhost`, and 404 for an unknown subdomain. It does **not** create `$db`, `$camp`, `$campOptions`, or `$cf`. The entry-point bootstraps do:

| Area                     | Bootstrap                                                               | Provides                                                                           |
| ------------------------ | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Admin pages + admin AJAX | `admin/inc/init.php`                                                    | `$camp` (from `$_SESSION['campInitials']`), `$campOptions`, `$db` (camp DB), `$cf` |
| Admin pages only         | `admin/inc/html_head.php` (auto-prepended by `admin/.htaccess-*`)       | `$admin`, the page permission gate, BootSite routing, page-view logging            |
| Parent portal            | `p/campers/headers.php` → `p/inc/classloader.php` + `p/inc/load_db.php` | `$camp`, `$campOptions`, `$db`, `$cf`, `$family`, CSRF helpers                     |
| REST API                 | `api/v1/index.php`                                                      | Camp from the `X-Camp-Initials` header; admin from the API key                     |
| Cron                     | `require '/var/www/classloader.php'`, then loop over `cf4_master.camps` | One `new db("cf4_{initials}")` per camp                                            |

Admin AJAX files that need `$admin` must create it themselves: `new admin($_SESSION['adminId'])`. Details are in [system-architecture.md](campsite/system-architecture.md).

---

## UI Layers: Legacy vs. BootSite

| Aspect        | Legacy                             | BootSite / RAD                                                         |
| ------------- | ---------------------------------- | ---------------------------------------------------------------------- |
| File          | `page.php`                         | `page_m.php`                                                           |
| CSS framework | None (custom CSS)                  | Bootstrap 3.3.5 + AdminLTE 2.3.3                                       |
| JS bundle     | `admin/inc/all.js.php?mode=legacy` | cdnjs jQuery 2.2.0 / UI 1.11.4 / Bootstrap, then `all.js.php?mode=rad` |

`bootSiteEnabled()` in `admin/inc/html_head.php` applies **only when `pages.hasBootSite` is true**. Within that:

- `?legacy=1` forces legacy and `?legacy=0` forces BootSite.
- Otherwise BootSite is used when `admin.hasBootSite` is `'All'`, or when it is `'Mobile'` on an iPhone/iPad/Android user agent.
- `bunkLodgic_*` pages and `camp_setup_questionnaire_m.php` use BootSite whenever the page has it enabled.

State which layer a new admin page targets. BootSite styling: Bootstrap 3 `.btn` and `.box` panels, `'Source Sans Pro'` 13px, division header blue `#2980b9`.

---

## BunkLodgic Module

- **AR classes** (all in `_classDictionary.php`; most use PK `id`):
  - Board, BoardSnapshot, BoardCriteriaConfig, BoardTemplate
  - Criteria, CriteriaConfig, CriteriaDefinition, CampCriteriaDefault, CampCriteriaSetting
  - Constraint, BunkConstraint
  - Lock, Override
  - StagedCamperAssignment, StagedStaffAssignment, StagedBunk
  - TemplateDivision, TemplateBunk, TemplateCriteriaConfig, TemplateConstraint
  - AlgorithmParam, AlgorithmDecision
  - Every name is prefixed `bunkLodgic` (e.g. `bunkLodgicBoardSnapshot` → table `bunkLodgicBoardSnapshots`). There is no `bunkLodgicCriterion` and no `bunkLodgicSnapshots` table. BunkConstraint, AlgorithmParam, and AlgorithmDecision have dictionary entries but no class file.
- **Services**: `classes/class.bunkLodgic*Service.php`. These are plain classes built with `__construct($db)`, not AR. They cover Algorithm, Assignment, Board, Copy, Criteria, Export, FieldRegistry, Legacy, Lock, Snapshot, and Template.
- **Permissions**: `coreBunkLodgicAdmin` (full + config), `coreBunkLodgicEdit` (move campers/staff), `coreBunkLodgicView` (read-only). Migration SQL inserting into `cf4_master.permissions` must produce exactly these strings.
- **Soft delete**: boards use `deleted IS NULL`.
- **Algorithm**: `algorithm/bunkLodgic_algorithm.py`, Python 3, stdlib only (don't add dependencies).
  - `bunkLodgicAlgorithmService` writes the JSON payload to a 0600 temp file and runs `python3 <script> <tmpFile>` via `shell_exec` under a per-board `flock`. It reads JSON from stdout.
  - The script path is env `BUNKLODGIC_SCRIPT_PATH`, defaulting to `/var/www/algorithm/bunkLodgic_algorithm.py`.

---

## Logging & Errors

- `ep($var, $logFile = null)` (`functions/functions.php`): with no file it writes to the PHP error log (`/var/log/phplog`) with a UTC timestamp. With a named file that doesn't exist, it silently returns `false`.
- `ExceptionHandler::register()` (called from `classloader.php`) uses `set_exception_handler`. When `ENVIRONMENT === 'development'` it renders a detailed page; otherwise it shows a generic error and logs. The API registers its own handler.
- `notifySlack($message, $webhookURL = <default>)`: the webhook argument is optional.
- Locally, logs live **inside the `web` container**: `docker compose exec web tail -n 200 /var/log/phplog`.

---

## Code Review Checklist (for Claude)

- [ ] DB access via AR or the `db` wrapper with `?` placeholders; no concatenated SQL.
- [ ] `selectObject(s)` has the class argument **and** the PK column in the SELECT.
- [ ] Replica flag in the right position (3rd for `select`/`selectRow`/`selectField`, 4th for `selectObject(s)`/`getChildren`).
- [ ] `$db` points at the right database before AR/`cf` objects are created.
- [ ] Multi-write logic is safe without transactions (they are no-ops).
- [ ] Every new or changed AJAX/API action checks `$_SESSION['permissionStrings'][...]` (or the API admin's permission strings) and the record's camp/family ownership.
- [ ] State-changing parent-portal endpoints use the CSRF helpers.
- [ ] No per-camp class lists a `cf4_master` class in `parents`.
- [ ] SQL is portable across MySQL 5.6/5.7 and 8.0; `` `rank` `` is backticked.
- [ ] Schema change shipped as a `sql/migrations/` script, with its rollout noted.
- [ ] No config values or secrets copied anywhere.
- [ ] Correct UI layer (legacy vs `_m.php`) and its jQuery version.
- [ ] Verified inside the `web` container, and through the real workflow in a browser where UI is involved.
