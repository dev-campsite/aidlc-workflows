# CampSite System Architecture (Reference)

This is the detailed companion to [CLAUDE.md](../CLAUDE.md). It describes how the legacy CampSite PHP application actually works, not how a modern app ideally would.

**Evidence:** checked against CampSite `develop` @ `8e238a1345` on 2026-10-21. Items marked *(not verifiable from the repo)* come from internal infrastructure notes. Open questions are collected in [§12](#12-open-questions).

---

## 1. Shape of the application

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

- **Root `.htaccess-*` friendly URLs:** `/campers`, `/staff`, `/donors`, `/events`, `/apply`, `/info`, `/enroll`, `/camptoday`, `/family`, `/setup/<token>`, `/create/<token>`, each with an optional `/<initials>`. `admin/.htaccess-*` maps `/admin/camper/<id>` (and family, staff, alumni, guest, donor, group, authorized-pickup) and **auto-prepends `admin/inc/html_head.php` / appends `html_foot.php`** to every admin page.

---

## 2. Multi-tenancy: database per camp

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
- **Read replica:** pass `$useReaderHost = true` (3rd argument to `select`/`selectRow`/`selectField`, 4th to `selectObject(s)`/`getChildren`). About 87 call sites pass it. If a node has no reader host, the primary is used (see [§12](#12-open-questions)).
- **Which camp a request belongs to depends on the entry point:**
  - Admin: `$_SESSION['campInitials']`, set at login.
  - Parent portal: `$_SESSION['cf4pc_campInitials']`.
  - API: the `X-Camp-Initials` header.
  - Cron: loops over all camps.
  - Login pages and SSO: the subdomain.

  The subdomain is otherwise used for `isValidCamp()` and redirects.
- **Tenant isolation is by convention.** Each request's `$db` points at one camp DB. When code takes an ID from input, check that the record belongs to the session's camp, family, or admin scope. Because the global `$db` decides which DB objects read, switching it mid-request (`switchDb`, `new db(...)`) changes the context for everything created afterwards.

---

## 3. Authentication and sessions

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

---

## 4. Authorization

- `admin::getPermissionStrings()` builds `[ '<module.name>' . ucfirst('<permission.name>') => true, … ]` from the admin's `adminPermission` children and stores it in `$_SESSION['permissionStrings']`.
- **Page gate:** `html_head.php` requires `$page->getPermissionString()` (from `cf4_master.pages`) to be present. This covers only auto-prepended admin pages.
- **AJAX and API have no automatic gate.** Each action must check, for example: `if (empty($_SESSION['permissionStrings']['campersEdit'])) { … }`. The API checks `$authorizedAdmin->getPermissionStrings()` per route.
- There is no central helper; `hasFinanceAccess()` in `html_head.php` is the only one.
- Discovery work on the Laravel branch found that legacy permissions are often UI-gating rather than enforced at the mutation endpoint, and that `ajax/` has real gaps. Treat every endpoint you touch as needing its own check.

---

## 5. AJAX endpoint conventions

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

## 6. REST API (`api/v1/`)

- Slim 3.9.2. `index.php` lowercases the headers (for Zapier), authenticates, registers its own exception handler, then requires the route files:
  - `family`, `camper`, `staff`, `group`, `guest`, `donor`
  - `medication`, `report`, `transactions`, `enrollment`
  - `custom_value`, `form_value`, `bunk`, `division`
  - `entity_transportation`
- There are about 54 routes plus `/ping`; responses use `$response->withJson()`. Swagger is at `api/v1/docs/swagger.yaml`.
- Deployed to the `api` deployment group only; `AfterInstall.sh` removes `api/` from web and cron nodes.

---

## 7. Scheduled jobs (`cron/`)

- They are invoked **as HTTP requests**: EventBridge → Lambda → ALB → cron EC2 (`cron.campmanagement.com` is a special host in `isValidCamp()`). The schedule and crontab are **not in the repo**.
- Typical shape: `require '/var/www/classloader.php'`, `new db('cf4_master')`, loop over `SELECT * FROM camps` (skipping training, special, and locked camps), then `new db("cf4_{initials}")` per camp. Output is HTML; logs go to `/var/log/cronlog`.
- Flags come from `$_GET`. Only 5 of the 36 scripts implement `liveRun=yes`; the rest act immediately.
- ALB idle timeouts can cut long sweeps (see the comments in `cron/raise_web_capacity_for_enrollment_openings.php`).
- Lessons from the Laravel coexistence docs:
  - Never run two implementations of the same cron at once.
  - Don't assume a cron is idempotent.

---

## 8. Integrations (in code)

| Area        | Implementation                                                                                                                                                                                                                                                                                             |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Payments    | Gateways selected by string: `'CSI'` (`baseGatewayCSI`, `paymentGatewayCSI`, `eCheckPaymentGatewayCSI`), `'Stripe'` (`stripe/stripe-php` used inline with per-camp keys), `'Pyxis'` (`gateways/payment/PyxisPaymentGateway.php`), `'Payflow Pro'` (`baseGatewayPayflowPro`), `'Authorize.net'` (SDK 1.9.2) |
| Email / SMS | SendGrid 7.9 (+ `sendgrid_webhooks.php`), Twilio 5.42 (+ `twilio_callbacks.php`)                                                                                                                                                                                                                           |
| AWS         | SDK 3.x: S3, Rekognition, KMS, AutoScaling (enrollment capacity scaling), CloudFront signed URLs                                                                                                                                                                                                           |
| Other       | Slack (`notifySlack`), SmugMug, WHCC (+ webhooks), HubSpot, Google Sheets (`google/apiclient` 2.14), Datadog, Zendesk, Intercom, StatusPage, Cloudflare, New Relic, OneLogin SAML, Duo (DBA only), `illuminate/validation` 10 (`helpers/ValidatorFactory.php`)                                             |
| Documents   | wkhtmltopdf, pdftk (`mikehaertl/php-pdftk`), ghostscript, poppler, FPDF                                                                                                                                                                                                                                    |

QuickBooks has config constants but **no implementation**.

---

## 9. Schema and data

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

## 10. Testing, CI, deployment

- **Unit tests:** `tests/` (27 files), run with `phpunit.xml` (bootstrap `vendor/autoload.php`; excludes `tests/integration`). Many are specification mirrors with stub `db`/`camp` classes, which can pass while the real code differs.
- **Integration tests:** `tests/integration/` (4 files, enrollment/window features) with `phpunit-integration.xml`. They must run in a separate process, because of the `head()` conflict and the stub-class collisions. They need the Docker DB and **skip** when it is unreachable.
- **CI:** the Bitbucket `default` pipeline runs on every push. It runs `composer install` and the unit suite only.
- **Deploy:** the manual custom pipeline `deployCampSite(deploymentGroupName)` runs `git archive`, then the AWS CodeDeploy upload, then deploy. `appspec.yml` copies the repo to `/var/www` and the BunkLodgic script to `/opt/bunklodgic/`. `AfterInstall.sh` then:
  - moves `sql/database_real.sql` to `/var/sql` and deletes `sql/`, the dev htaccess, the Docker files, `config/`, `Makefile`, `docker-compose.yml`, and build/test config;
  - removes `cron/` and `api/` on `web` nodes, `api/` on `cron` nodes, and `cron/` on `api` nodes.
- `build.xml` (Ant: phpcs, pdepend) is not used by the pipeline.

---

## 11. Known limitations to design around

- `db` transactions are no-ops, so writes are not atomic.
- There is no CSRF protection in admin, and permissions are enforced per endpoint, if at all.
- `$db` is global, and objects bind to it at construction.
- Absolute `/var/www` and `/var/config` paths are hard-coded.
- `ep()` silently ignores missing log files.
- The SQL dialect must work on MySQL 5.6, 5.7, and 8.0.
- Unit tests are often mirrors, so a green suite is weak evidence (see [lessons-learned](lessons-learned.md)).
- Stray `composer.json.rej` / `composer.lock.rej` are committed and are not removed on deploy.

---

## 12. Open questions

These could not be verified from the repo. Don't present them as fact.

- How production `/var/config/` is provisioned and updated (the environment's config management is not in the repo).
- The exact production MySQL/Aurora versions and node layout (from infrastructure notes: MySQL 5.7 master with three replicas; Aurora 5.6 camp nodes).
- The cron schedule (EventBridge rules) and which cron scripts are actually scheduled.
- Whether `/opt/bunklodgic/` (copied by `appspec.yml`) is used. The code defaults to `/var/www/algorithm/…`, and the `scripts/provision_bunklodgic.sh` mentioned in `requirements.txt` is not in the repo.
- Production Memcached/ElastiCache configuration.
- Whether production nodes have `nodes.readerHost` set. About 87 call sites pass the replica flag, but the Laravel-branch risk register calls the reader-host code "dead". Without a reader host, the flag falls back to the primary.

### Production infrastructure *(not verifiable from the repo)*

| Role              | Notes                                                                   |
| ----------------- | ----------------------------------------------------------------------- |
| Web               | EC2 M5.xlarge, Ubuntu 18.04, Auto Scaling 2–48, behind Cloudflare → ALB |
| API               | EC2 M4.2xlarge + ALB (`api.campmanagement.com`)                         |
| Cron              | Lambda + EC2 C5.xlarge + ALB (`cron.campmanagement.com`)                |
| Staging / testing | `uat.staging`, `testing.campmanagement.com`                             |
| Other             | ElastiCache, S3, Route 53, Rekognition, Translate, OpenVPN, Rapid7      |
