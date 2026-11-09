# CLAUDE.md — CampSite Technical Context for Code Generation

Read this at the start of every CampSite conversation. It is the single source of CampSite context for AI-DLC and Claude Code. It describes how the legacy CampSite PHP application actually works, not how a modern application ideally would.

**Evidence baseline.** Statements here were checked against the CampSite repo's `develop` branch at commit `8e238a1345` and a running local Docker stack (Windows 11, Docker Desktop 29.7 with the WSL2 backend, `desktop-linux` context) on 2026-10-21. Labels used throughout:

- *(not verifiable from the repo)*: the fact comes from internal infrastructure notes.
- *(Windows-verified)*: tested on Windows only.
- *(unverified)*: not tested at all.

If code and this file disagree, the code wins: say so, and flag this file for correction.

**Contents:**

1. [Environments at a glance](#environments-at-a-glance)
2. [Critical rules](#critical-rules)
3. [Stack & Versions](#stack--versions)
4. [Naming Conventions](#naming-conventions)
5. [_classDictionary.php](#_classdictionaryphp)
6. [Data Layer](#data-layer)
7. [Application Structure & Request Bootstrap](#application-structure--request-bootstrap)
8. [Multi-Tenancy](#multi-tenancy-database-per-camp)
9. [Authentication](#authentication-and-sessions)
10. [Authorization](#authorization)
11. [AJAX](#ajax-endpoints)
12. [REST API](#rest-api-apiv1)
13. [Cron](#scheduled-jobs-cron)
14. [UI Layers](#ui-layers-legacy-vs-bootsite)
15. [BunkLodgic](#bunklodgic-module)
16. [Integrations](#integrations)
17. [Logging & Errors](#logging--errors)
18. [Schema & Data](#schema-and-data)
19. [Testing, CI & Deployment](#testing-ci-and-deployment)
20. [Local Development (Docker)](#local-development-environment-docker)
21. [Known Limitations](#known-limitations)
22. [Lessons Learned](#lessons-learned)
23. [Code Review Checklist](#code-review-checklist-for-claude)
24. [Open Questions](#open-questions)

---

## Environments at a glance

| Aspect     | Local development                                                                    | Deployed (production, staging, testing)                                                       |
| ---------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Runtime    | Docker Compose: `web` (php:8.3-apache), `db` (MySQL 8.0.45), `phpmyadmin`, `certbot` | EC2 behind Cloudflare and an ALB; separate `web`, `cron` and `api` deployment groups          |
| Code path  | CampSite repo bind-mounted at `/var/www`                                             | CodeDeploy copies the repo to `/var/www`                                                      |
| Config     | Repo `./config/` bind-mounted at `/var/config`                                       | `/var/config/` provisioned on the server; `AfterInstall.sh` deletes the repo's `config/` copy |
| Databases  | `cf4_master` + a single camp DB, `cf4_dev`                                           | `cf4_master` + one `cf4_<initials>` DB per camp, spread across nodes                          |
| Memcached  | Extension installed, **no server**, so every cache read misses                       | ElastiCache *(not verifiable from the repo)*                                                  |
| Cron       | No cron daemon; nothing runs on a schedule                                           | HTTP requests: EventBridge → Lambda → ALB → cron EC2                                          |
| htaccess   | `.htaccess-development`                                                              | `.htaccess-production`                                                                        |
| Reached by | Hosts-file entry → 127.0.0.1                                                         | Cloudflare → ALB                                                                              |

The VirtualBox dev VM is **retired**. Ignore any VM-era instructions, such as `192.168.56.x` hosts entries, or a MySQL and Memcached running on the VM itself. The only remaining traces are commented-out hosts-file entries and the "VirtualBox" comment in `docker/web/php/99-campsite.ini`.

`/var/config/*.php` is **not** VM-specific. It is the runtime config path everywhere: `classloader.php` loads `PROJECT_ROOT . '/../config/db.php'` and `environment.php` (from `/var/www`, that resolves to `/var/config`), and about 48 files `require '/var/config/aes.php'` directly. What differs is where the directory comes from (see the table above). **Never infer production config values from the local `./config/` files.**

---

## Critical rules

1. **Run code inside the `web` container**: `docker compose exec web <cmd>` from the CampSite repo root. The host has neither the PHP build nor `/var/config`. Edit files on the host; the bind mount makes edits live.
2. **Ask before anything destructive.** That covers `make bootstrap`, `make fresh`, `make db-reset`, `make db-import`, `make reset-volumes`, and **bare `make`**, whose default goal is `bootstrap`. All of them drop and re-import both databases. Also ask before triggering crons, email/SMS sends, or payment calls: the local `config/` holds real third-party credentials.
3. **Never copy secrets.** Refer to config constants by name only. Never paste values from `config/*.php`, `docker/certbot/cloudflare.ini`, the `Makefile`, or `docker-compose.yml` into code, commits, logs, AI-DLC artifacts, or chat.
4. **Use the `db` wrapper with `?` placeholders.** No string-concatenated SQL and no raw `mysqli_*`.
5. **`$db` is a global that everything binds to.** `activeRecord` and `cf` both read `global $db` when constructed. Make sure it points at the right database (`new db($camp->dbName)` for camp data, `new db('cf4_master')` for global data) before you instantiate objects. Any SQL string containing `cf4_master` is routed to the master DB automatically.
6. **Transactions are no-ops.** `db::beginTransaction()`, `commit()`, and `rollBack()` are empty stubs, so multi-statement writes are **not atomic**. Design writes to be safe without atomicity, and say so in plans and reviews. Don't write code that relies on rollback.
7. **Check permissions in every AJAX endpoint.** Admin *pages* are gated automatically through the auto-prepended `admin/inc/html_head.php`, but `ajax/*.php` endpoints are not. The check is `!empty($_SESSION['permissionStrings']['<permissionString>'])`: `permissionStrings` is an associative `string => true` array, not a list. Also check that the record belongs to the session's camp, family, or admin scope.
8. **Don't assume CSRF protection.** There is no global CSRF layer, and admin AJAX has none. Parent-portal code uses `setCSRFToken`/`getCSRFToken`/`clearCSRFToken` from `p/inc/load_db.php`; use those helpers on new state-changing parent endpoints.
9. **Write SQL that runs on MySQL 5.6/5.7 (prod) and 8.0 (local).** Avoid CTEs, window functions, `JSON_TABLE`, and `CHECK` constraints. Always backtick `` `rank` ``: it has been reserved since MySQL 8.0.2, so unquoted it works in prod but fails locally.
10. **Schema changes are manual SQL.** Add a `sql/migrations/YYYY-MM-DD-<slug>.sql` script. It is run by hand against **every** `cf4_<initials>` DB (or `cf4_master`), and code that depends on it ships only after the rollout. There is no migration runner.
11. **Preserve existing behavior.** This is a large legacy multi-tenant app. Match the surrounding code's idiom, change incrementally, and verify the whole workflow, not just one method (see [Lessons Learned](#lessons-learned)).
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
- **Read replica:** about 87 call sites pass the flag. If a node has no reader host, the primary is used (see [Open Questions](#open-questions)).
- **Request-scoped caches:** `$GLOBALS['arCache']`, `$GLOBALS['objectCache']`, and `$GLOBALS['queryCache']`. Every insert, update, or delete clears the whole query cache. Within one request you may read cached state.
- `getChildObjects()` and `db::getDbName()` **do not exist**. Don't generate calls to them.
- Other `db` methods: `selectMultiField`, `selectMap`, `selectExists`, `switchDb($host, $name)`. `exec()` logs and **exits** on any write error.
- `useful_constants.php` arrays (`US_STATES`, `ALLOWED_IMAGE_TYPES`, `PAYMENT_GATEWAYS`, …) are **serialized strings**: `unserialize(US_STATES)`.

---

## Application Structure & Request Bootstrap

- **No front controller.** Apache serves `.php` files directly, and friendly URLs are `RewriteRule`s in `.htaccess-development` / `.htaccess-production`. Absolute paths such as `/var/www/...` and `/var/config/...` are hard-coded throughout.
- **Main areas:**

| Path                      | What it is                                                                                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `admin/`                  | Camp staff back office (legacy `x.php` + BootSite `x_m.php`)                                                                                                                 |
| `ajax/`                   | 525 `crud_*.php` endpoints called from admin and portal pages                                                                                                                |
| `p/`                      | Public portals: `campers` (parents), `staff`, `donors`, `events`, `today`, plus RFI and registration pages                                                                   |
| `api/v1/`                 | REST API (Slim 3.9.2)                                                                                                                                                        |
| `cron/`                   | 36 scheduled scripts + `cron/dbIntegrityTests/`                                                                                                                              |
| `dba/`                    | Internal CampSite staff console (Duo login), e.g. camps, builds, payment gateways, `dba/tools/`                                                                              |
| `sso/`                    | SAML SSO (OneLogin `php-saml` 2.10.7)                                                                                                                                        |
| `mobile/`                 | jQuery Mobile pages (auto-prepends `admin/inc/mobile_head.php`)                                                                                                              |
| `classes/`                | About 1,760 `class.<name>.php` files: AR models, services, gateways                                                                                                          |
| `functions/functions.php` | About 2,700-line global helper library (`ep()`, `notifySlack()`, `isValidCamp()`, …)                                                                                         |
| `algorithm/`              | BunkLodgic Python placement algorithm                                                                                                                                        |
| Root scripts              | Webhooks and callbacks (`sendgrid_webhooks.php`, `twilio_callbacks.php`, `whcc_webhooks.php`, `charge2FA_webhook.php`, `google-oauth-callback.php`), onboarding, unsubscribe |

- **Root `.htaccess-*` friendly URLs:** `/campers`, `/staff`, `/donors`, `/events`, `/apply`, `/info`, `/enroll`, `/camptoday`, `/family`, `/setup/<token>`, `/create/<token>`, each with an optional `/<initials>`.
- **`admin/.htaccess-*`** maps `/admin/camper/<id>` (and family, staff, alumni, guest, donor, group, authorized-pickup). It also **auto-prepends `admin/inc/html_head.php` / appends `html_foot.php`** to every admin page.

**`classloader.php` does not create the request globals.** In order, it:

1. loads constants and registers `ExceptionHandler`;
2. loads `/var/config/db.php` and `environment.php`;
3. registers the autoloader;
4. loads `functions/functions.php`, `libphutil.php`, and `vendor/autoload.php`, **in that order** (the order is load-bearing);
5. calls `isValidCamp()`, which returns 403 for `localhost`, and 404 for an unknown subdomain.

It does **not** create `$db`, `$camp`, `$campOptions`, or `$cf`. The entry-point bootstraps do:

| Area                     | Bootstrap                                                               | Provides                                                                           |
| ------------------------ | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Admin pages + admin AJAX | `admin/inc/init.php`                                                    | `$camp` (from `$_SESSION['campInitials']`), `$campOptions`, `$db` (camp DB), `$cf` |
| Admin pages only         | `admin/inc/html_head.php` (auto-prepended)                              | `$admin`, the page permission gate, BootSite routing, page-view logging            |
| Parent portal            | `p/campers/headers.php` → `p/inc/classloader.php` + `p/inc/load_db.php` | `$camp`, `$campOptions`, `$db`, `$cf`, `$family`, CSRF helpers                     |
| REST API                 | `api/v1/index.php`                                                      | Camp from the `X-Camp-Initials` header; admin from the API key                     |
| Cron                     | `require '/var/www/classloader.php'`, then loop over `cf4_master.camps` | One `new db("cf4_{initials}")` per camp                                            |

Admin AJAX files that need `$admin` must create it themselves: `new admin($_SESSION['adminId'])`.

---

## Multi-Tenancy: Database per Camp

- `cf4_master` is the global registry. Notable tables:
  - `camps`, `nodes`, `campNodes`
  - `campOptions`, `campOptionsMeta*`, `campModules`, `modules`
  - `pages`, `publicFacingPages`, `permissions`, `permissionCategories`
  - `dbas`, `campGroups`, `campGroupCamps`, `loginHistories`
  - `timezones`, `countries`, `currencies`, `languages`
- **`admins` and `adminPermissions` are per-camp tables**, not in `cf4_master`.
- Each camp has its own database, `cf4_<initials>`. `camp::__construct` sets `$camp->dbName = "cf4_{$initials}"`.
- `new db($dbName)` resolves the host as follows:
  - `cf4_master` uses `DB_HOST_MASTER`.
  - Any other DB looks up `md5("dbHostFor<db>")` / `md5("dbReaderHostFor<db>")` in Memcached. On a miss, it queries `nodes JOIN campNodes JOIN camps` and caches the result for 24h.
  - It rejects non-`cf4_*` names, and shows `maintenance.php` when the node has `inMaintenance`.
- **Which camp a request belongs to depends on the entry point:**
  - Admin: `$_SESSION['campInitials']`, set at login.
  - Parent portal: `$_SESSION['cf4pc_campInitials']`.
  - API: the `X-Camp-Initials` header.
  - Cron: loops over all camps.
  - Login pages and SSO: the subdomain.

  The subdomain is otherwise used for `isValidCamp()` and redirects.
- **Tenant isolation is by convention.** Each request's `$db` points at one camp DB. When code takes an ID from input, check that the record belongs to the session's camp, family, or admin scope. Because the global `$db` decides which DB objects read, switching it mid-request (`switchDb`, `new db(...)`) changes the context for everything created afterwards.

---

## Authentication and Sessions

| Actor                | Flow                                                                                                                                                                                                                                                                      | Session keys                                                                                            |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Camp admin           | `admin/login.php` → `admin/login_submit.php`: camp by subdomain, admin by email, `password_verify`; locks after 10 failed attempts; forces a reset after 90 days; `admin::login()`; TOTP 2FA (`RobThree\Auth\TwoFactorAuth`, `admins.secret2FA`) via `admin/2fa_auth.php` | `adminId`, `campInitials`, `timezone`, `email`, `has2FA`, `2faVerified`, `permissionStrings`, `isAgent` |
| Parent               | `p/campers/login_submit.php` → `family::login()`; parent TOTP 2FA in `p/campers/2fa_auth_m*.php`                                                                                                                                                                          | `cf4pc_campInitials`, `cf4pc_familyId`, `cf4pc_parent`, `cf4pc_email`, `has2FA`, `2faVerified`, …       |
| SSO                  | `sso/` SAML, enabled per camp by `campOptions->enableOkta` (IdP in `oktaEntityId`)                                                                                                                                                                                        | Same keys as admin login                                                                                |
| CampSite staff (DBA) | `dba/login_submit.php` with **Duo Universal**; Duo is used only here                                                                                                                                                                                                      | `dba/inc/init.php`                                                                                      |
| API client           | `X-Camp-Initials` + `X-API-Key`, checked with `password_verify` against active admins' `APIKeyHash`                                                                                                                                                                       | Stateless; `$authorizedAdmin`                                                                           |

- `google-oauth-callback.php` is **not** a login method. It links a logged-in admin to Google Sheets (`integrations/google/`).
- `admin/inc/init.php` checks the session, loads `$camp`/`$campOptions`/`$db`/`$cf`, exits if `lockAllAccess`, and redirects to 2FA when it is unverified. On an expired session it **dies with plain text**, even for AJAX.
- Admin pages get `$admin` from `html_head.php`. `html_head.php` also logs page views and logs out inactive admins.
- The session name in Docker is `PHPSESSID2` (`99-campsite.ini`); no app code depends on it.

---

## Authorization

- `admin::getPermissionStrings()` builds `[ '<module.name>' . ucfirst('<permission.name>') => true, … ]` from the admin's `adminPermission` children and stores it in `$_SESSION['permissionStrings']`.
- **Page gate:** `html_head.php` requires `$page->getPermissionString()` (from `cf4_master.pages`) to be present. This covers only auto-prepended admin pages.
- **AJAX and API have no automatic gate.** Each action must check, for example: `if (empty($_SESSION['permissionStrings']['campersEdit'])) { … }`. The API checks `$authorizedAdmin->getPermissionStrings()` per route.
- There is no central helper; `hasFinanceAccess()` in `html_head.php` is the only one.
- **Migration SQL must match PHP exactly.** If PHP checks `coreBunkLodgicAdmin`, the `cf4_master.permissions` rows must produce exactly that string.
- Discovery work on the Laravel branch found that legacy permissions are often UI-gating rather than enforced at the mutation endpoint, and that `ajax/` has real gaps. Treat every endpoint you touch as needing its own check.

---

## AJAX Endpoints

- Most files start with `require "/var/www/admin/inc/init.php";`. Others use `dba/inc/init.php` (29 files) or a portal `p/*/headers.php`.
- Input conventions:
  - `$params = $_POST;` (about 290 files)
  - `$params = $_POST ?: $_GET;` (about 90)
  - `$params = $_GET;` (about 10)

  It is **not POST-only**.
- Dispatch is `if ($params['action'] == '…') { … }` blocks.
- Output is usually `echo json_encode($results)` with an `errors` array and **no Content-Type header**; some actions return HTML.
- CSRF: admin AJAX has none. Parent-side endpoints use the `setCSRFToken`/`getCSRFToken`/`clearCSRFToken` helpers (`p/inc/load_db.php`), and `load_db.php` also sends a report-only CSP header.
- About 44 files also `require '/var/config/aes.php'` for card encryption.

---

## REST API (`api/v1/`)

- Slim 3.9.2. `index.php` lowercases the headers (for Zapier), authenticates, registers its own exception handler, then requires the route files:
  - `family`, `camper`, `staff`, `group`, `guest`, `donor`
  - `medication`, `report`, `transactions`, `enrollment`
  - `custom_value`, `form_value`, `bunk`, `division`
  - `entity_transportation`
- There are about 54 routes plus `/ping`; responses use `$response->withJson()`. Swagger is at `api/v1/docs/swagger.yaml`.
- Deployed to the `api` deployment group only; `AfterInstall.sh` removes `api/` from web and cron nodes.

---

## Scheduled Jobs (`cron/`)

- They are invoked **as HTTP requests**: EventBridge → Lambda → ALB → cron EC2 (`cron.campmanagement.com` is a special host in `isValidCamp()`). The schedule and crontab are **not in the repo**.
- Typical shape: `require '/var/www/classloader.php'`, `new db('cf4_master')`, loop over `SELECT * FROM camps` (skipping training, special, and locked camps), then `new db("cf4_{initials}")` per camp. Output is HTML; logs go to `/var/log/cronlog`.
- Flags come from `$_GET`. Only 5 of the 36 scripts implement `liveRun=yes`; the rest act immediately when requested.
- ALB idle timeouts can cut long sweeps (see the comments in `cron/raise_web_capacity_for_enrollment_openings.php`).
- Never run two implementations of the same cron at once, and don't assume a cron is idempotent.

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

`bootSiteRedirectUrl()` rewrites `x.php` ↔ `x_m.php` and redirects. State which layer a new admin page targets. BootSite styling: Bootstrap 3 `.btn` and `.box` panels, `'Source Sans Pro'` 13px, division header blue `#2980b9`.

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
- **Permissions**: `coreBunkLodgicAdmin` (full + config), `coreBunkLodgicEdit` (move campers/staff), `coreBunkLodgicView` (read-only).
- **Soft delete**: boards use `deleted IS NULL`.
- **Algorithm**: `algorithm/bunkLodgic_algorithm.py`, Python 3, stdlib only (don't add dependencies).
  - `bunkLodgicAlgorithmService` writes the JSON payload to a 0600 temp file and runs `python3 <script> <tmpFile>` via `shell_exec` under a per-board `flock`. It reads JSON from stdout.
  - The script path is env `BUNKLODGIC_SCRIPT_PATH`, defaulting to `/var/www/algorithm/bunkLodgic_algorithm.py`.

---

## Integrations

| Area        | Implementation                                                                                                                                                                                                                                                                                             |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Payments    | Gateways selected by string: `'CSI'` (`baseGatewayCSI`, `paymentGatewayCSI`, `eCheckPaymentGatewayCSI`), `'Stripe'` (`stripe/stripe-php` used inline with per-camp keys), `'Pyxis'` (`gateways/payment/PyxisPaymentGateway.php`), `'Payflow Pro'` (`baseGatewayPayflowPro`), `'Authorize.net'` (SDK 1.9.2) |
| Email / SMS | SendGrid 7.9 (+ `sendgrid_webhooks.php`), Twilio 5.42 (+ `twilio_callbacks.php`)                                                                                                                                                                                                                           |
| AWS         | SDK 3.x: S3, Rekognition, KMS, AutoScaling (enrollment capacity scaling), CloudFront signed URLs                                                                                                                                                                                                           |
| Other       | Slack (`notifySlack`), SmugMug, WHCC (+ webhooks), HubSpot, Google Sheets (`google/apiclient` 2.14), Datadog, Zendesk, Intercom, StatusPage, Cloudflare, New Relic, OneLogin SAML, Duo (DBA only), `illuminate/validation` 10 (`helpers/ValidatorFactory.php`)                                             |
| Documents   | wkhtmltopdf, pdftk (`mikehaertl/php-pdftk`), ghostscript, poppler, FPDF                                                                                                                                                                                                                                    |

QuickBooks has config constants but **no implementation**. For package versions, see `composer.json` / `composer.lock`.

---

## Logging & Errors

- `ep($var, $logFile = null)` (`functions/functions.php`): with no file it writes to the PHP error log (`/var/log/phplog`) with a UTC timestamp. With a named file that doesn't exist, it silently returns `false`.
- `ExceptionHandler::register()` (called from `classloader.php`) uses `set_exception_handler`. When `ENVIRONMENT === 'development'` it renders a detailed page; otherwise it shows a generic error and logs. The API registers its own handler.
- `notifySlack($message, $webhookURL = <default>)`: the webhook argument is optional.
- Log files: `phplog`, `pagelog`, `cronlog`, `stripelog`, `authorizelog`, `payflowprolog`, `pyxislog`, all in `/var/log/`. Locally they live **inside the `web` container** (see [Working in the containers](#working-in-the-containers)).

---

## Schema and Data

- **No migration runner.** Changes ship as hand-run SQL in `sql/migrations/YYYY-MM-DD-<slug>.sql`; there is one so far. Its header documents the procedure:
  - Run it manually against every camp DB.
  - It is not idempotent.
  - App code ships only after every DB is done.

  `dba/tools/query_all.php` can run a query across all camp DBs.
- `classes/class.dba.php` is the AR model for internal staff users, **not** a schema tool.
- **SQL dumps** in `sql/`:

| File                        | Contents                | Generated by                      | Used for                                      |
| --------------------------- | ----------------------- | --------------------------------- | --------------------------------------------- |
| `cf4_master.sql`            | Master DB               | `admin/create_cf4_master_sql.php` | Local import                                  |
| `database_real.sql`         | Blank camp schema       | `admin/create_real_sql.php`       | Blank restores; moved to `/var/sql` on deploy |
| `database_demo.sql`         | Demo camp schema + data | `admin/create_demo_sql.php`       | Local `cf4_dev`                               |
| `database_demo_options.sql` | Demo camp options       |                                   | With the demo dump                            |

  If a schema change must exist in new camps or locally, update the relevant dump too.

- Datetimes are stored in **UTC**. `activeRecord::update()`/`addChild()` and `cf::addNewObject()` run `offsetDatetimeFields()`. Don't compare datetime columns to `NOW()` in SQL, because the session timezone is not guaranteed to be UTC.

---

## Testing, CI and Deployment

- **Unit tests:** `tests/` (27 files), run with `phpunit.xml` (bootstrap `vendor/autoload.php`; excludes `tests/integration`). Many are specification mirrors with stub `db`/`camp` classes, which can pass while the real code differs.
- **Integration tests:** `tests/integration/` (4 files, enrollment/window features) with `phpunit-integration.xml`.
  - They must run in a separate process, because of the `head()` conflict and the stub-class collisions.
  - They boot `classloader.php` as `cron.campmanagement.com`, so they need the Docker DB, and they **skip** rather than fail when it is unreachable. Check the skip count.
- **CI:** the Bitbucket `default` pipeline runs on every push. It runs `composer install` and the unit suite only.
- **Deploy:** the manual custom pipeline `deployCampSite(deploymentGroupName)` runs `git archive`, then the AWS CodeDeploy upload, then deploy. `appspec.yml` copies the repo to `/var/www` and the BunkLodgic script to `/opt/bunklodgic/`. `AfterInstall.sh` then:
  - moves `sql/database_real.sql` to `/var/sql` and deletes `sql/`, the dev htaccess, the Docker files, `config/`, `Makefile`, `docker-compose.yml`, and build/test config;
  - removes `cron/` and `api/` on `web` nodes, `api/` on `cron` nodes, and `cron/` on `api` nodes.
- `build.xml` (Ant: phpcs, pdepend) is not used by the pipeline.

---

## Local Development Environment (Docker)

Everything is defined at the **root of the CampSite repo**: `docker-compose.yml`, `Makefile`, and `docker/`. **macOS was not tested**; Windows-only and untested items are labeled.

### Prerequisites

| Tool                                        | Notes                                                                                                                                                                                                                                  |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Docker with Compose v2 (`docker compose …`) | *(Windows-verified)* Docker Desktop with the WSL2 backend. On macOS, Docker Desktop is the presumed equivalent *(unverified)*.                                                                                                         |
| GNU make                                    | *(Windows-verified)* GNU Make 4.4.1 installed natively via winget (`ezwinports.make`), runnable from Git Bash or PowerShell; WSL also works. On macOS the Xcode Command Line Tools ship a `make` *(unverified against this Makefile)*. |
| Free host ports **80, 443, 22, 3306**       | Published by `web` and `db`. A host SSH server on port 22 (common on macOS with Remote Login, or Windows OpenSSH Server) or a local MySQL on 3306 conflicts.                                                                           |
| `docker/certbot/cloudflare.ini`             | Cloudflare DNS credentials for the wildcard cert. Already tracked in the repo; never copy its contents.                                                                                                                                |
| A hosts-file entry for your subdomain       | See [URLs, DNS, and access](#urls-dns-and-access). Without it, `<sub>.campmanagement.com` resolves to **production** Cloudflare.                                                                                                       |

CPU architecture: the web `Dockerfile` downloads wkhtmltopdf by `${TARGETARCH}`, so arm64 (Apple Silicon) appears intended. Only `x86_64` was verified.

### Files and services

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

**PHP runtime.** PHP extensions include mysqli, pdo_mysql, gd, intl, bcmath, mbstring, zip, xsl, and OPcache, plus igbinary, msgpack, memcached, and mcrypt built from source. The ini settings are UTC, `memory_limit=512M`, `max_input_vars=10000`, and `error_log=/var/log/phplog`.

**Apache.** Modules: rewrite, headers, ssl, unique_id, proxy, proxy_http. `vhost.conf` sets `AccessFileName .htaccess-development` and redirects HTTP to HTTPS.

**MySQL.** `my.cnf` sets `sql_mode=NO_ENGINE_SUBSTITUTION` (non-strict), `innodb_strict_mode=0`, and `max_connections=10000`.

### Path mapping (host ↔ container)

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

### Setup, run, rebuild

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
2. Run `prepare-sql.php`, which converts `varchar(n>16000)` to `mediumtext` in copies under `.docker/tmp/sql/`. Don't "fix" those columns back.
3. Import `sql/cf4_master.sql` into `cf4_master` and `sql/database_demo.sql` into `cf4_dev`.
4. Set **every** `cf4_master.camps.subdomain` to `SUBDOMAIN`, and `cf4_master.nodes.host` to `db`.

### Working in the containers

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
- **The BunkLodgic algorithm** takes a JSON **file path** argument, not stdin: `docker compose exec web python3 algorithm/bunkLodgic_algorithm.py <payload.json>`.
- **Crons** do not run locally (no cron daemon). Treat requesting one as running it live (see [Scheduled Jobs](#scheduled-jobs-cron)), and ask first.

### URLs, DNS, and access

- **App:** `https://<SUBDOMAIN>.campmanagement.com/`. Public DNS for `*.campmanagement.com` points at production Cloudflare, so you need a hosts-file entry `127.0.0.1 <SUBDOMAIN>.campmanagement.com`. *(Windows-verified: `C:\Windows\System32\drivers\etc\hosts`. On macOS the file is `/etc/hosts` (unverified for this stack).)* Without the entry you are on **production**.
- `isValidCamp()` returns **403 for `localhost`/`127.0.0.1`** and **404 for a subdomain not in `cf4_master.camps`**, so always browse by subdomain.
- **phpMyAdmin:** `https://<SUBDOMAIN>.campmanagement.com/phpmyadmin/`.
- **SSH:** `web` runs sshd on host port 22 as user `ubuntu` (in `www-data`): `ssh ubuntu@<SUBDOMAIN>.campmanagement.com`, working in `/var/www`.

### Troubleshooting (verified behavior)

| Symptom                                                  | Cause                                                                                                  | Fix                                                                               |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| `make health` shows `unique_id: MISSING`                 | `unique_id` is an Apache module, not a PHP extension; the check can never pass                         | Ignore, or confirm with `docker compose exec web apache2ctl -M \| grep unique_id` |
| OPcache reported missing in ad-hoc checks                | The CLI reports it as `Zend OPcache`                                                                   | `php -r 'var_dump(extension_loaded("Zend OPcache"));'`                            |
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

## Known Limitations

- `db` transactions are no-ops, so writes are not atomic.
- There is no CSRF protection in admin, and permissions are enforced per endpoint, if at all.
- `$db` is global, and objects bind to it at construction.
- Absolute `/var/www` and `/var/config` paths are hard-coded.
- `ep()` silently ignores missing log files.
- The SQL dialect must work on MySQL 5.6, 5.7, and 8.0.
- Unit tests are often mirrors, so a green suite is weak evidence (see [Lessons Learned](#lessons-learned)).
- Stray `composer.json.rej` / `composer.lock.rej` are committed and are not removed on deploy.

---

## Lessons Learned

Reusable guidance from the Laravel migration hackathon and recent AI-DLC units. Each lesson cites its source so it can be re-checked. Sources are paths on `origin/feature/laravel-campsite` unless noted otherwise.

### Status: what is and isn't Laravel

- **Production CampSite is the legacy PHP application.** Nothing Laravel is merged into `develop` or `master` (checked 2026-10-21 with `git merge-base --is-ancestor`), and no deployment path exists for it.
- **The Laravel proof of concept lives only on `origin/feature/laravel-campsite`.**
  - **Layout:** a separate Laravel 12 app in `laravel-campsite/`, with its own Docker stack (8 services).
  - **Databases:** it shares the legacy databases (`cf4_master` + camp DBs) without schema changes. Its own sessions, cache and jobs live in a separate `campsite_app` DB.
  - **Login:** it has its own login, not shared with legacy sessions; an SSO bridge is backlog.
  - **Tenancy:** ADR-1 defines it (`aidlc-docs/inception/application-design/architecture-decision-records.md`). Subdomain → `camps.initials` → node lookup → a `tenant` connection, with models split into `Master` and `Tenant`.
  - **Docs:** they disagree on ports and routing. Treat that branch's docs as a proof of concept, not as a description of CampSite.
- **The other hackathon branches are legacy PHP, not Laravel:** `hackathon/ask-campsite` (AI report builder) and `hackathon/bunk-buddy-AI`.
- **The NOW-28 / NOW-29 "modern profile" work is also legacy PHP** (`_x` pages), on feature branches.

When asked for CampSite code, generate legacy PHP that follows this file. Laravel code belongs only in work explicitly scoped to `laravel-campsite/`.

### Context before code

- **Read the legacy source before porting or changing a behavior; the schema doesn't capture the business rules.** Write paths must have their rules re-verified rather than "assumed to be covered by the DB schema alone" (`aidlc-docs/inception/application-design/legacy-laravel-coexistence.md`).
- **Sample-based discovery misses things.** A validation pass over the first discovery added 8 new risks (#22–29). A headline finding (87% of files lacking a permission check) turned out to be "mostly a false negative" for `admin/`, which is gated by auto-prepend, but "a real, standing gap" for `ajax/` (`aidlc-docs/inception/reverse-engineering/matrix-validation-notes.md`). Confirm the mechanism before drawing per-file conclusions.

### Preserve behavior and compatibility

- **Keep legacy quirks unless the change is explicitly scoped.** The Shop port has separate sections for "Legacy quirks deliberately preserved" and "Deliberate differences" (`aidlc-docs/construction/shop-module/code/shop-module-summary.md`). Do the same: list what you keep and what you change, and why.
- **Don't flip risky security settings as a side effect.** SAML `strict` was "deliberately NOT changed" because flipping it could cause "a production incident" (`aidlc-docs/legacy-vs-laravel-audit.md`).

### Tenancy and isolation

- **Every new model must target the right database** (master vs tenant), or "it will silently query the wrong database" (`aidlc-docs/inception/application-design/laravel-target-architecture.md`). In legacy code the equivalent is: make sure the global `$db` is the right DB before creating objects.
- **Fail closed on unknown tenants.** Legacy `isValidCamp()` returns 404 for an unknown subdomain. The POC's local configuration has a default dev camp (`CAMPSITE_DEV_CAMP_INITIALS`); a default-tenant fallback like that must never reach a deployed environment.

### Authorization and ownership

- **Check permissions at the mutation endpoint, not only in the UI.** Legacy checks are "mostly UI-gating rather than enforced at the mutation endpoint", and admin has "no CSRF protection anywhere" (`aidlc-docs/legacy-vs-laravel-audit.md`).
- **A flow that is enrollable is not necessarily enforced.** Lockout and 2FA defects in the port: 2FA "was enrollable but never enforced", and the lockout counter was "checked but never persisted" (same audit). Test the enforcement path, not just the setup path.

### Configuration assumptions

- **Production config "lives only on running instances".** Local `config/` says nothing about production values (same audit; `AfterInstall.sh` deletes the repo copy on deploy).
- **`campmanagement.com` is the live production domain.** Without a hosts entry, a local test signs into production (same audit, "The hosts entry is not optional").
- **htaccess differs per environment.** A missing `QSA` flag meant "production would have kept discarding the query string" (NOW-28, commit `ddd35b039c`). Check `.htaccess-production` whenever you change `.htaccess-development`.

### Test the whole workflow

- **A green suite can be meaningless.** In NOW-29, a test checked a copy of the function, so "the test could not have failed" (`aidlc-docs/NOW-29-process-retrospective.md` on `origin/feature/NOW-29-view-as-parent`). Test the real code path; mirror only what cannot be loaded.
- **Browser passes find what other tools miss.** In NOW-29, "curl and unit tests both passed" while the guard failed silently on parent pages (same retrospective). In NOW-28, axe reported zero violations while Lighthouse caught a broken skip link, and styling was "correct by luck" because a Tailwind `@source` pointed at deleted files (commit `339abfe499`).
- **A failing check is often a wrong test.** In this project "a failing check has more often meant a wrong test than wrong code", which can only be told apart by going back to the legacy source (`aidlc-docs/construction/build-and-test/build-and-test-summary.md`).
- **Test across subdomains.** For cross-subdomain flows such as OAuth, test on at least two subdomains (`test/CAMP-6710-takeaways.md`, on `develop`).

### Findings: separate verified defects from recommendations

- **Assess every finding.** "A finding raised is not a finding assessed": attach a reachability verdict with the evidence (NOW-29 retrospective).
- **Give every "cannot" a subject:** who cannot, and why (same).
- **Track assumptions.** A standing assumption needs an owner and a stage by which it is tested or downgraded (same).
- **Label speed claims honestly.** The "278x faster" forms result was not a framework benchmark, and legacy was "Not measured" (`aidlc-docs/construction/forms-module/code/forms-module-summary.md`). State what was and was not measured.

### Incremental change with explicit criteria

- **Turn repeated defects into a count.** When the same defect repeats N times, make the test "a count that must reach zero" (NOW-29 retrospective).
- **Never run two implementations of a cron at once,** and roll schema changes out per camp DB before shipping dependent code (coexistence doc; `sql/migrations/` header on `develop`).

### Applying this in AI-DLC stages

| Stage                    | Apply                                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Inception / requirements | State the tenant scope (camp, family, admin), the permission strings involved, and which UI layer is targeted. Read the legacy files end-to-end, not by grep. |
| Design                   | List preserved quirks vs intended differences. Say how writes stay consistent without transactions. Name the DB (master or camp) for every table touched.     |
| Code generation          | Follow the [Critical rules](#critical-rules); add endpoint-level permission, ownership, and CSRF checks where state changes.                                  |
| Build & test             | Run inside the `web` container and exercise the real workflow in a browser on your subdomain. Report skipped tests, not just passes.                          |
| Review                   | Classify each finding as a **verified defect** (with reproduction or evidence) or a **recommendation**. Never mix them in one list without labels.            |

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

---

## Open Questions

These could not be verified from the repo. Don't present them as fact.

- How production `/var/config/` is provisioned and updated (the environment's config management is not in the repo).
- The exact production MySQL/Aurora versions and node layout (from infrastructure notes: MySQL 5.7 master with three replicas; Aurora 5.6 camp nodes).
- The cron schedule (EventBridge rules) and which cron scripts are actually scheduled.
- Whether `/opt/bunklodgic/` (copied by `appspec.yml`) is used. The code defaults to `/var/www/algorithm/…`, and the `scripts/provision_bunklodgic.sh` mentioned in `requirements.txt` is not in the repo.
- Production Memcached/ElastiCache configuration.
- Whether production nodes have `nodes.readerHost` set. About 87 call sites pass the replica flag, but the Laravel-branch risk register calls the reader-host code "dead". Without a reader host, the flag falls back to the primary.
- Whether the Docker setup works unchanged on macOS (Apple Silicon and Intel).

### Production infrastructure *(not verifiable from the repo)*

| Role              | Notes                                                                   |
| ----------------- | ----------------------------------------------------------------------- |
| Web               | EC2 M5.xlarge, Ubuntu 18.04, Auto Scaling 2–48, behind Cloudflare → ALB |
| API               | EC2 M4.2xlarge + ALB (`api.campmanagement.com`)                         |
| Cron              | Lambda + EC2 C5.xlarge + ALB (`cron.campmanagement.com`)                |
| Staging / testing | `uat.staging`, `testing.campmanagement.com`                             |
| Other             | ElastiCache, S3, Route 53, Rekognition, Translate, OpenVPN, Rapid7      |
