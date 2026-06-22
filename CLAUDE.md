# CLAUDE.md — CampSite Technical Context for Code Generation

Read this at the top of every CampSite code conversation. This file encodes always-true facts about the stack, conventions, and architecture so generated code fits immediately.

For anti-patterns and counter-examples, see `BAD_EXAMPLES.md`.

---

## Stack & Versions

| Component | Version | Notes |
|-----------|---------|-------|
| PHP | 8.3+ | `declare(strict_types=1)` in DB/exception classes. Typed constants in ExceptionHandler. |
| MySQL | 5.7 / 8.x | PDO only (no raw `mysqli_*`). Reserved words like `rank` are backtick-quoted. |
| jQuery | 3.7.1 (legacy), 2.2.0 (BootSite/RAD) | jQuery Migrate 1.4.1 loaded alongside 3.7.1. |
| Bootstrap | 3.3.5 + AdminLTE 2 | Used in responsive ("BootSite") pages. Legacy pages use raw HTML/CSS. |
| Python | 3.x stdlib-only | BunkLodgic placement algorithm at `algorithm/bunkLodgic_algorithm.py`. |

---

## Forbidden Patterns

**Never do these:**

- ❌ Call `mysqli_*()` directly or use raw string concatenation in SQL → Use `$db->select()` / `update()` / `delete()` wrapper instead.
- ❌ Hardcode table or column names → Use Active Record (AR) objects; let `CLASS_DICTIONARY` resolve metadata.
- ❌ Mix DB credentials or config into code → Credentials live in `/var/config/db.php` (outside repo); fetch via constants only.
- ❌ Build queries without parameterized placeholders → Always `$db->select($sql, $params)` with `?` placeholders.
- ❌ Call database directly for multiple related rows → Use `getChildren()` or `getChildObjects()` on AR classes.
- ❌ Ignore the `useReaderHost` parameter for read-heavy queries → Route reads to replica: `$db->select($sql, $params, null, true)`.
- ❌ Write unquoted `rank` in SQL → `` `rank` `` is a reserved word in MySQL 5.7+; always backtick it: `` ORDER BY `rank` ASC ``.
- ❌ Omit the 3rd `$className` argument on `selectObject()`/`selectObjects()` → It is required; missing it causes a fatal "Too few arguments" error.
- ❌ Omit the PK column from a `selectObjects()` query → The AR system uses the PK to instantiate each row; omitting it causes "record does not exist" failures.

---

## Naming Conventions

**Copy these patterns exactly:**

| Type | Pattern | Example |
|------|---------|---------|
| **Classes** | lowerCamelCase | `campOptions`, `camperYear`, `bunkLodgicAlgorithmService` |
| **DB tables** | lowerCamelCase plural | `campOptions`, `accrualPeriods`, `camperYears`; BunkLodgic tables prefixed `bunkLodgic*` |
| **Primary keys** | `<ClassName>Id` | `campId`, `adminId`, `camperYearId` |
| **Class files** | `classes/class.<ClassName>.php` | `classes/class.camp.php`, `classes/class.camper.php` |
| **AJAX endpoints** | `ajax/crud_<feature>.php` | `ajax/crud_camper.php`, `ajax/crud_family.php` |
| **Admin pages** | Bare + optional `_m.php` for responsive | `admin/camper.php`, `admin/camper_m.php` (BootSite variant) |
| **Methods** | lowerCamelCase | `getChildren()`, `addChild()`, `getAncestor()` |
| **Indentation** | Tabs | Everywhere. |

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

## Config & Environment Variables

**Server-side (outside repo, NOT in version control):**

File | Contains
-----|----------
`/var/config/db.php` | `DB_HOST_MASTER`, `DB_USERNAME`, `DB_PASSWORD`, `MEMCACHED_SERVERS_DATA`, `DB_USERNAME_ESCALATED`
`/var/config/environment.php` | `ENVIRONMENT` ('development'/'testing'/'production'), `MAINTENANCE_MODE`
`/var/config/aes.php` | `$aesKey` for credit card encryption

**Third-party credentials** (runtime constants):
- `AWS_REGION`, `AWS_KEY`, `AWS_SECRET` (S3, Rekognition)
- `TWILIO_SID`, `TWILIO_TOKEN` (SMS)

**In-repo constants** (`useful_constants.php`):
- `ALLOWED_IMAGE_TYPES`, `US_STATES`, `PAYMENT_GATEWAYS`, etc.

---

## Live Infrastructure

### Databases

| Database | Engine | Instance | Role |
|----------|--------|----------|------|
| `cf4_master` | MySQL 5.7.26 | RDS db.r4.large (2 vCPU / 15 GB) | Global registry: camps, admins, pages, routing, permissions |
| `cf4_master` replicas ×3 | MySQL 5.7.34 | RDS db.r4.2xlarge (8 vCPU / 61 GB) | Read replicas (Nodes 1, 4, 5) |
| `cf4_<initials>` (e.g. `cf4_fw`) | Aurora 5.6 | Staging node + Node 2 demo | Per-camp application data |
| Dev VM | Local MySQL + Memcached | — | Local development; isolated from prod |

### Compute

| Role | Instance | Count | Notes |
|------|----------|-------|-------|
| Web servers | EC2 M5.xlarge (4 vCPU), Ubuntu 18.04 | 2–48 (Auto Scaling) | LAMP; `/var/www/` deploy target |
| REST API | EC2 M4.2xlarge + ALB | Auto Scale | `api.campmanagement.com` |
| Cron | Lambda + EC2 C5.xlarge + ALB | Auto Scale | `cron.campmanagement.com` |
| Deploy HQ / Staging | EC2 M5.large | 1 | `uat.staging` |
| Testing | EC2 M5.large | 1 | `testing.campmanagement.com` |

### Traffic & DNS

```
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

| Database | Role | Host |
|----------|------|------|
| `cf4_master` | Global registry: camps, admins, pages, routing, permissions | Fixed `DB_HOST_MASTER` |
| `cf4_<initials>` (e.g., `cf4_fw`) | Per-camp application data (campers, staff, enrollments, etc.) | Dynamic, cached in Memcached 24h |

**Why it matters for code:** Always instantiate AR objects with the correct camp's database. Use `$camp->dbName` or `$db->getDbName()`.

### Query Cache & Caching

- AR objects are cached in `$GLOBALS['arCache']` and `$GLOBALS['objectCache']`.
- Query results are cached in `$GLOBALS['queryCache']` (keyed by MD5 of SQL + params).
- Don't assume fresh DB state in a single request; use `unset()` to clear cache if needed.

---

## UI Layers: Legacy vs. BootSite

Two parallel front-ends coexist:

| Aspect | Legacy | BootSite / RAD |
|--------|--------|-----------------|
| File suffix | `.php` | `_m.php` |
| Bootstrap | None | 3.3.5 + AdminLTE 2 |
| jQuery | 3.7.1 + UI 1.14.1 | 2.2.0 (CDN) + UI 1.11.4 |
| JS bundle | `all.js.php?mode=legacy` | `all.js.php?mode=rad` |

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
- See ENVIRONMENT.md for full list & versions.

---

## Deployment & CI/CD

**Pipeline**: Bitbucket Pipelines → AWS CodeDeploy → `/var/www/` on EC2 instances

**Schema changes:**
- No ORM migration runner
- Apply raw SQL files via `class.dba.php` tooling
- Schema files in `sql/` (e.g., `cf4_master.sql`, `database_real.sql`)

**Cron jobs** (`cron/~30 scripts`):
- Email dispatch, payment reminders, admin account expiry, SmugMug sync, etc.
- Run on dedicated cron instance (removed from web instances at deploy)

---

## BunkLodgic Module

BunkLodgic is a drag-and-drop cabin/bunk assignment feature. It runs exclusively in BootSite (`_m.php`) and has its own naming conventions.

### Table & Class Names

All tables follow the `bunkLodgic*` prefix (lowercase b):

| Table | AR Class |
|-------|----------|
| `bunkLodgicBoards` | `bunkLodgicBoard` |
| `bunkLodgicStagedCamperAssignments` | `bunkLodgicStagedCamperAssignment` |
| `bunkLodgicStagedStaffAssignments` | `bunkLodgicStagedStaffAssignment` |
| `bunkLodgicCriteriaConfigs` | `bunkLodgicCriteriaConfig` |
| `bunkLodgicCriteria` | `bunkLodgicCriterion` |
| `bunkLodgicLocks` | `bunkLodgicLock` |
| `bunkLodgicSnapshots` | `bunkLodgicSnapshot` |
| `bunkLodgicOverrides` | `bunkLodgicOverride` |

### Permissions

| String | Grants |
|--------|--------|
| `coreBunkLodgicAdmin` | Full admin including config |
| `coreBunkLodgicEdit` | Move campers / staff |
| `coreBunkLodgicView` | Read-only view |

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

- **BAD_EXAMPLES.md**: Real anti-patterns Claude should avoid (e.g., raw `mysqli_*`, mixing concerns, hardcoded paths)
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
- [ ] Secrets/config fetched from constants, not hardcoded?
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
