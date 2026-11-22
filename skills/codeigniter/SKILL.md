# CodeIgniter 4 System Architecture Standard

## Part 0: Philosophy — Simple vs. Easy

> _"Conflating [Simple and Easy] is why we're drowning in complexity."_ — **Rich Hickey** (_"Simple Made Easy"_, 2011)

| Attribute         | Simple (Standard)                                                | Easy (Anti-Pattern)                                    |
| :---------------- | :--------------------------------------------------------------- | :----------------------------------------------------- |
| **Root / Nature** | _Simplex_ ("one fold/braid"); objective structure                | _Adjacens_ ("nearby/at hand"); subjective convenience  |
| **Focus**         | Single deterministic responsibility; disentangled                | Familiarity, proximity, copy-paste, global shortcuts   |
| **Cost**          | Deliberate upfront design, strict contracts, refactoring         | Zero initial friction; complected/braided dependencies |
| **Outcome**       | Long-term maintainability, deterministic debugging, linear scale | Rapid start, cascading failures, high defect debt      |

### CI4 Paradigm Mappings

- **Easy (FORBIDDEN):** `$this->request->getPost()` or `session()` inside Services/Models; inline `$db->table()` queries in Controllers/Views; `$var ?? 'default'` masking missing View payloads.
- **Simple (REQUIRED):** Typed scalars, DTOs, and Entities passed into HTTP-agnostic Services; DB access strictly inside constructor-injected Models; explicit View data contracts failing immediately on missing or malformed keys.
- **Core Mandate:** Disentanglement MUST NOT be sacrificed for developer comfort, brevity, or convenience.

---

## Part 1: Core Meta-Rules

- **Universal Scope:** This standard SHALL apply unconditionally to EVERY file, class, trait, helper, entity, view, pager, command, filter, migration, seeder, config, cell, decorator, and test. Zero exceptions.
- **Project-Agnostic:** Specifications and reference examples MUST use generic placeholders (`DomainController`, `DomainService`, `DomainEntity`, `DomainModel`, `domain_id`, `[ModuleName]`).
- **Deterministic Language (RFC 2119):** Use ONLY `MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, and `FORBIDDEN`. Permissive or ambiguous words (_may_, _should_, _generally_, _recommended_, _best practice_, _optional_, _permitted_, _approved_, _mandatory_) are **FORBIDDEN**.
- **Zero Fallbacks & Zero Suppression:** Unvalidated defaults, null-coalescing chains (`??`) masking missing keys or dependencies, `env('KEY', 'default')`, `old('field', '')`, blind casts (`(int)`, `(string)`, `(float)`, `(bool)`, `(array)`) on `mixed` or `array|string|null`, `Referer`-dependent `redirect()->back()`, unchecked `service()` / `model()` / `session()` / `$this->validator` / `$file->move()` / `$db->transCommit()`, silent exception swallows, PHP's `@` error-suppression operator, and JS `|| ''` fallbacks or silent no-ops are **FORBIDDEN**. Operations MUST succeed deterministically or fail explicitly.
- **Pre-Task Alignment:** Agents and engineers MUST load and align with this standard before any design, refactor, or implementation.
- **Runtime Verification & Log Audit:** Engineers MUST execute active self-verification across all routes (automated tests, CLI, `curl`, local HTTP) and inspect `writable/logs/*.log` for zero warnings or errors before declaring completion.
- **Static Analysis & Style Gate:** CI and pre-commit hooks MUST enforce **PHPStan Level 8** (with `phpstan-strict-rules` and checked-exception rules, failing on untyped boundaries, dynamic properties, blind casts, and unhandled checked exceptions), **PHPCS (`PSR-12`)** across all pure PHP directories (`app/`, `tests/`), a dedicated View template lint check for mixed PHP/HTML files (`Views/`, `Pagers/`), and an automated pattern check rejecting forbidden fallback constructs and non-RFC keywords.

---

## Part 2: Folder Structure & Modular Architecture

Enforces decoupled **Modular MVC-S** under `app/Modules/`. Root `app/` is reserved strictly for shared core infrastructure (base layouts, base controllers, core filters, strict view helpers, global events, core home routing).

### 2.1 Module Layout (`app/Modules/[ModuleName]/`)

Create ONLY directories containing files, EXCEPT `Config/Routes.php` and `Config/Services.php`, which are **REQUIRED** for every module. `Database/Migrations/` is **REQUIRED** for every module that owns persistent database tables:

- `Commands/` (Spark CLI) | `Config/{Routes.php,Services.php}` | `Controllers/` (HTTP orchestration) | `Database/{Migrations/,Seeds/}` | `Entities/` (type-safe records) | `Helpers/` (`[name]_helper.php`) | `Libraries/` (Services, DTOs, domain exceptions—exclusive home of business logic) | `Models/` (DB access) | `Views/` (escaped Bootstrap 5.3 templates).

### 2.2 Registration Protocol

- **Scaffold:** Create the exact module layout manually or via project scaffolding (`php spark make:module` does not exist in core CodeIgniter 4).
- **Namespace (`app/Config/Autoload.php`):** Register `'App\Modules\[ModuleName]' => APPPATH . 'Modules/[ModuleName]'` in `$psr4` (enables `php spark migrate --all` and `Config\Modules` discovery).
- **Services (`Config/Services.php`):** Extend `CodeIgniter\Config\BaseService`. Define a typed static factory method accepting nullable injected dependencies and `bool $getShared = true` (e.g., `domainService(?DomainModel $domainModel = null, bool $getShared = true): DomainService`). When `$getShared` is `true`, resolve via `static::getSharedInstance('domainService', $domainModel)` and assert `instanceof DomainService` (throwing `RuntimeException` on mismatch). When `$getShared` is `false`, resolve default dependencies via `model(DomainModel::class)` + `instanceof` assertion (throwing `RuntimeException` on failure) and instantiate the service. Cross-module shared services MUST reside in `app/Config/Services.php`. Consumers MUST resolve services via `service('domainService')` paired with an `instanceof` guard (or `Services::domainService()`).
- **Routes (`Config/Routes.php`):** Wrap inside `$routes->group('domain', ['namespace' => 'App\Modules\[ModuleName]\Controllers'], static function (RouteCollection $routes): void { ... });`.
- **Helpers:** Global helper autoloading in `app/Config/Autoload.php` is **FORBIDDEN**. Declare per-controller (or in `BaseController::$helpers` for core shared helpers) using CI4 `FileLocator` syntax: `protected $helpers = ['strict_view', 'form'];` (plus `'App\Modules\[ModuleName]\domain'` when the module provides a `Helpers/domain_helper.php`; specifying `Helpers\` inside the helper name is **FORBIDDEN**).

---

## Part 3: Layer Responsibilities

### 3.1 Controllers (`Controllers/`)

- **MUST** extend `CodeIgniter\Controller` or `App\Controllers\BaseController` and declare explicit native return types: `string|ResponseInterface` for HTML view/redirect actions and `ResponseInterface` for redirect, JSON, and file download actions (`DownloadResponse` and `RedirectResponse` implement `ResponseInterface`).
- **MUST** resolve Services in `__construct()` via `service(...)` paired with an explicit `instanceof` assertion (throwing `RuntimeException` on mismatch). Accessing `$this->request`, `$this->response`, or `session()` inside `__construct()` is **FORBIDDEN** (request context is injected later via `initController()`).
- **MUST** validate all HTTP input (POST/PUT/PATCH bodies, JSON payloads via `getJSON()`, query parameters, route parameters, and required headers) via the Validation service. Controllers MUST assert `$this->validator !== null` (throwing `RuntimeException` if null) and retrieve validated input ONLY via `$this->validator->getValidated()` with explicit type checks (`getPost()`, `getGet()`, and `getJSON()` without prior validation are **FORBIDDEN**). Route parameters MUST be validated explicitly (e.g., `$domainId <= 0` MUST fail with HTTP `400`).
- **MUST** validate authenticated actor identity from `session()` strictly (`is_int($raw)` or `is_string($raw) && ctype_digit($raw)` with value `> 0`); unauthenticated requests MUST fail deterministically with HTTP `401 Unauthorized` (or explicit redirect to the named login route for browser flows).
- **MUST** declare `string|ResponseInterface` for HTML view actions or `ResponseInterface` for redirect, JSON, and file download actions, passing pre-formatted view payloads (`<= 5` lines of view logic) and all 7 SEO metadata keys to every HTML view: `pageTitle`, `metaDescription`, `metaKeywords`, `canonicalUrl`, `robotsTag` (`index, follow` for public pages; `noindex, follow` for private/authenticated pages), `metaImage`, and `schemaJson` (`array|null`). Data-rendering views MUST additionally receive `'data'`.
- **FORBIDDEN:** DB queries/Query Builder, `new` Model/DB instantiation, business logic, direct filesystem operations (`is_file()`, `unlink()`, `mkdir()`), inline HTML, `redirect()->back()`, and leaking raw `$e->getMessage()` in HTTP or flash responses.

### 3.2 Services (`Libraries/`)

- **MUST** house 100% of business rules, external APIs, persistence workflows, transactions, and filesystem/storage operations.
- **MUST** inject Models/sub-services via `__construct()`, declare all properties with explicit visibility (`private`/`protected`) and native types at class top (dynamic properties **FORBIDDEN**), and obtain DB connections via `$this->domainModel->db` asserted as `instanceof BaseConnection` (or constructor-injected `BaseConnection`). Calling `\Config\Database::connect()` or instantiating `new` Models inside methods or loops is **FORBIDDEN**.
- **MUST** wrap multi-write mutations in deterministic transactions (Section 4.9) and return `Entity`, `array<int, Entity>`, typed DTOs, typed scalars, or `void`. On failure, Services MUST throw typed domain or runtime exceptions (returning error arrays such as `['status' => 'error', ...]` is **FORBIDDEN**).
- **FORBIDDEN:** Superglobals (`$_POST`, `$_GET`, `$_FILES`, `$_SERVER`, `$_COOKIE`, `$_REQUEST`, `$_ENV`), `session()`, `RequestInterface`, `ResponseInterface`, HTTP redirects, and session flash data.
- **Topology:** Strictly vertical (`Controller -> MainService -> Model` or `Controller -> OrchestratorService -> SubService -> Model`). Triangular ping-pong (Controller calling both MainService and its internal SubService) is **FORBIDDEN**; MainService MUST act as Facade. Same-tier Brother-Services (`ModuleAService` <-> `ModuleBService`) MUST NOT call each other directly (route cross-module orchestration through `OrchestratorService`).

### 3.3 Models (`Models/`)

- **MUST** extend `CodeIgniter\Model` and define `$table`, `$primaryKey`, `$returnType = DomainEntity::class`, `$allowedFields` (**FORBIDDEN** to omit), `$useTimestamps = true`, `$dateFormat = 'datetime'`, `$createdField`, `$updatedField`, and `$deletedField`. Stateful tables MUST set `$useSoftDeletes = true`; immutable append-only audit/log tables MUST set `$useSoftDeletes = false` and omit `deleted_at`.
- **CI4 Inheritance Exemption:** Overridden untyped properties inherited from `CodeIgniter\Model` (`$table`, `$primaryKey`, `$returnType`, `$allowedFields`, `$useTimestamps`, `$dateFormat`, `$createdField`, `$updatedField`, `$deletedField`, `$useSoftDeletes`, `$useAutoIncrement`) MUST NOT declare native PHP types (which trigger a PHP Fatal Error against CI4's untyped parent declarations) and MUST instead carry `@var` PHPDoc annotations with descriptions.
- **FORBIDDEN:** Model-level `$casts` (conflicts with Entity casts), business workflows, direct invocation from Views or Controllers, and importing dev-only `Faker\Generator` into production Models (test factories belong under `tests/_support/`).

### 3.4 Entities (`Entities/`)

- **MUST** extend `CodeIgniter\Entity\Entity` and represent all persistent data across layer boundaries (passing raw associative arrays across layers is **FORBIDDEN**).
- **MUST** define `$casts` (e.g., `'id' => 'integer'`, `'domain_id' => 'integer'`, `'owner_id' => 'integer'`, `'value' => 'string'`, `'is_active' => 'boolean'`, `'metadata' => '?json-array'`), following the CI4 untyped-parent-property exemption (`@var` PHPDoc **REQUIRED**; native PHP types omitted on `$casts` and `$dates`). `DECIMAL` and monetary columns MUST cast as `'string'` (`'float'` casts are **FORBIDDEN**).
- **MUST** list timestamp fields in `$dates` (returned as `\CodeIgniter\I18n\Time|null`) and annotate every virtual/column attribute plus `$dates` via one-tag-per-line `@property` PHPDoc on the class. Typed accessors/mutators (e.g., `setPassword(string $pass): static`) MUST declare explicit parameter and return types.
- **FORBIDDEN:** DB queries, service dependencies, or business workflows.

### 3.5 Views (`Views/`)

- **MUST** open with `<?php` and `declare(strict_types=1);` followed by a top-level multiline `@var` PHPDoc block typing `\CodeIgniter\View\View $this` and every variable supplied to the template.
- **Layout & Structure:** Module page views MUST immediately follow the PHP header with `<?= $this->extend('layouts/default') ?>` and wrap markup in `section('content')`, `section('styles')`, `section('scripts')`, and `section('nav_items')`. Reusable partials (`app/Views/partials/*`) and pager templates (`app/Views/Pagers/*`) omit `$this->extend(...)`.
- **Complexity & Escaping:** Maximum conditional (`if`/`elseif`/`else`) nesting depth is `1` (exclusive of outer `foreach` data iteration); maximum consecutive raw PHP block is `5` lines; all dynamic scalar output MUST be escaped via `esc($v, 'html'|'attr'|'js')` and validated via `strict_view_helper` functions (blind casts and `old('field', '')` fallbacks are **FORBIDDEN**).
- **FORBIDDEN:** DB calls, Service/Model instantiation, and raw unvalidated access to `session()`, `old()`, or `lang()`.

### 3.6 Helpers (`Helpers/`)

- Pure stateless functions in `[helper_name]_helper.php`, loaded via Controller `$helpers`. Every function MUST declare explicit parameter and return types. Business logic, DB queries, and mutable global state are **FORBIDDEN**.

---

## Part 4: Database Schema & Management Standards

- **4.1 Engine/Charset:** `ENGINE=InnoDB`, `CHARACTER SET utf8mb4`, `COLLATE utf8mb4_unicode_ci` (`latin1` and 3-byte `utf8` **FORBIDDEN**), `ROW_FORMAT=DYNAMIC`.
- **4.2 Keys:** Surrogate PK `id` (business PKs such as email, slug, or tax ID **FORBIDDEN**): `INT UNSIGNED` or `BIGINT UNSIGNED` (omitting legacy MySQL integer display-width constraints such as `'constraint' => 11`), or `BINARY(16)` UUIDv7/ULID (36-char string UUIDs **FORBIDDEN**). `BINARY(16)` PKs MUST set `$useAutoIncrement = false` on the Model, cast deterministically on the Entity, and use explicit UUID/hex route constraints rather than `(:num)`. FKs MUST match the parent PK type, length, and unsigned attribute.
- **4.3 Column Types:** Use the narrowest exact fit: `TINYINT`/`SMALLINT`/`INT`/`BIGINT` (`UNSIGNED` for IDs and counts); `DECIMAL(precision,scale)` for monetary and exact numeric values (`FLOAT`/`DOUBLE` in SQL and `float` casts in PHP Entities/Services **FORBIDDEN**); `TINYINT(1)` with strict `0|1` default (`"Y"/"N"/"true"/"false"` **FORBIDDEN**); `DATETIME`/`TIMESTAMP` for temporals (UNIX `INT` or string dates **FORBIDDEN**); `CHAR(len)` fixed and `VARCHAR(N)` bounded (`TEXT`/`JSON` for unbounded payloads or structured metadata only).
- **4.4 Integrity:** Explicit `'null' => false` (`NOT NULL`) on all columns unless explicit nullable semantics are required; `UNIQUE` constraints via `addUniqueKey()`; explicit `addForeignKey($field, $table, $targetField, $onUpdate, $onDelete)` specifying both `ON UPDATE` (`CASCADE` or `RESTRICT`) and `ON DELETE` (`RESTRICT` or `CASCADE`; orphan records **FORBIDDEN**).
- **4.5 Indexes:** `addKey` is **REQUIRED** on FKs (`[entity]_id`), multi-state status columns (`status` with >= 3 distinct states), composite filter keys (e.g., `['domain_id', 'is_active']`), and temporal sort/range columns (`created_at`). `addUniqueKey` is **REQUIRED** on unique lookup columns (`email`, `slug`, `hash`). Composite keys MUST be ordered: highest selectivity -> equality -> range/sort. Single-column low-selectivity boolean indexes (e.g., standalone `addKey('is_active')`) and volatile per-request indexes (`last_seen_at`) are **FORBIDDEN**.
- **4.6 Naming:** `snake_case` tables and columns (`camelCase` and `kebab-case` **FORBIDDEN**); plural table names; pivot tables named as alphabetical singular nouns (`domain_user`); PK `id`; FK `[parent]_id`; booleans prefixed `is_`/`has_`/`must_`; timestamps suffixed `*_at`.
- **4.7 Lifecycle/Soft Deletes:** Raw SQL `DELETE` on core entities is **FORBIDDEN**. Stateful tables MUST include `created_at DATETIME NOT NULL`, `updated_at DATETIME NULL`, and `deleted_at DATETIME NULL` (`NULL` = active). Immutable audit/log tables MUST omit `updated_at` and `deleted_at` and set `$useSoftDeletes = false`.
- **4.8 Querying:** `SELECT *` is **FORBIDDEN**—all queries (including Model `find()`, `first()`, `findAll()`, and `paginate()`) MUST specify explicit columns via `->select(...)` prior to execution. Raw `$db->query()` is **FORBIDDEN** except for multi-CTE or window-function aggregate reports that Query Builder cannot express; such queries MUST use parameterized bindings and carry an inline `@rationale` PHPDoc comment explaining why Query Builder cannot express them.
- **4.9 Transactions:** Multi-write mutations (> 1 DB write) MUST use deterministic manual transactions (`$db->transBegin()`, `$db->transCommit()`, and `if ($db->transDepth > 0) { $db->transRollback(); }` inside `catch (\Throwable $e)`), verifying the boolean return values of `transBegin()` and `transCommit()` (throwing `RuntimeException` on `false`), logging `'critical'`, and rethrowing `$e`. Using `transException(true)->transStart()` / `transComplete()` is **FORBIDDEN** due to sticky `transException` state across shared connections.
- **4.10 Migrations & Seeders:**
  - Schema changes MUST occur strictly via Migrations (manual DDL **FORBIDDEN**). Active local development defects MUST revise the initial creation migration + `php spark migrate:refresh --all`; deployed production MUST use non-breaking forward migrations.
  - Local development schema unification merges alters into the base `createTable()` and purges redundant staged alters; deployed production base migrations MUST NOT be mutated.
  - Migrations are DDL-only (DML `INSERT`/`UPDATE`/`DELETE` **FORBIDDEN**). `down()` MUST revert `up()` in reverse-dependency order wrapped in `$this->db->disableForeignKeyChecks()` followed by `try { ... } finally { $this->db->enableForeignKeyChecks(); }`.
  - Seeders (`php spark db:seed MainSeeder`) MUST be declarative and idempotent with all FKs populated (`for`/`while`/`%`/`rand()` **FORBIDDEN**).
  - Provisioning sequence: `php spark migrate --all` -> (if local dev conflict) `php spark migrate:refresh --all` -> `php spark db:seed MainSeeder` -> `php spark migrate:status`.

---

## Part 5: Code Quality, Typing & Documentation

- **Format & Strictness:** All pure PHP files (`app/`, `tests/`, excluding mixed PHP/HTML `Views/` and `Pagers/`) MUST comply with **PSR-12** (120-character maximum line length). Every PHP file MUST begin with `declare(strict_types=1);` as the first statement, followed by one blank line and `namespace` (for class/config files).
- **PHPDoc:** Every PHPDoc tag MUST occupy its own line (packing multiple tags onto one line is **FORBIDDEN**). Classes MUST declare `@package`, `@author`, and `@since`. Methods MUST declare `@param` (with descriptions), `@return`, and `@throws` for every thrown or propagated exception. Properties MUST declare a `@var` PHPDoc type/description AND native PHP types—EXCEPT overridden untyped CI4 parent properties (`CodeIgniter\Model`, `CodeIgniter\Entity\Entity`, `CodeIgniter\Controller::$helpers`, `CodeIgniter\CLI\BaseCommand`, `CodeIgniter\Test\DatabaseTestTrait`), which MUST omit native types and carry `@var` PHPDoc descriptions.
- **Naming:** `PascalCase` classes; `camelCase` methods, properties, and variables; `snake_case` DB columns and array keys. Leading-underscore method prefixes (e.g., `_privateHelper()`) are **FORBIDDEN** (`PSR-12` violation); enforce visibility via `private`/`protected` keywords, grouped under `// --- Helper Methods ---` BEFORE public callers.
- **Return Contracts:** Explicit native return types are **REQUIRED** on all functions and methods; PHPStan generics (e.g., `array<int, DomainEntity>`) are **REQUIRED** on all array parameters and returns.

---

## Part 6: Request, Response & Protocol

- **6.1 Routing:** `public bool $autoRoute = false;` MUST be set in `app/Config/Routing.php`. Route closures are **FORBIDDEN**. Every route MUST be named (`'as' => ...`) and resolved via `url_to()` (hardcoded URIs **FORBIDDEN**).
- **6.2 PRG (`POST`/`PUT`/`PATCH`/`DELETE`):** Controllers MUST NOT return `view()` from mutating requests. Validate -> assert `$this->validator !== null` -> `$this->validator->getValidated()` -> Entity -> Service -> flash -> `redirect()->to(url_to(...))`. On validation error or `\Throwable`, log `'error'` (for exceptions) and `redirect()->to(url_to(...))` to an explicit named route (`redirect()->back()` is **FORBIDDEN**). Controllers MUST NOT leak `$e->getMessage()`.
- **6.3 AJAX Envelope:** `['status' => 'success'|'error'|'validation_error', 'message' => string, 'result' => mixed, 'errors' => array<string, string>, 'csrf_token' => csrf_hash()]` paired with deterministic HTTP status codes (`200`/`201` on `success`, `400`/`422` on `validation_error`, `401`/`403`/`404`/`429`/`500` on `error`).
- **6.4 SSE:** Set `Content-Type: text/event-stream`, `Cache-Control: no-cache`, and `X-Accel-Buffering: no`; call `session_write_close()` before the stream loop; emit `csrf_hash()` in the initial packet; flush each packet with `if (ob_get_level() > 0) { ob_flush(); } flush();`; terminate cleanly when `connection_aborted() !== 0`.

---

## Part 7: Security Protocols

- **7.1 CSRF:** Globally enabled in `app/Config/Security.php` AND registered in `app/Config/Filters.php` (`$globals['before'] = ['csrf']`). Forms MUST use `form_open()` or `form_open_multipart()` (duplicate `csrf_field()` **FORBIDDEN**). All JSON responses MUST include `csrf_token`. Layout `<meta name="csrf-token">` and `<meta name="csrf-header">` tags are **REQUIRED**; client JavaScript MUST throw an `Error` if the CSRF `<meta>` element or token string is missing or invalid.
- **7.2 Validation:** Validate all input strictly; assert `$this->validator !== null`; retrieve exclusively via `$this->validator->getValidated()`.
- **7.3 Secrets:** Store secrets ONLY in `.env` (`.gitignore`d); read ONLY via `strict_env('KEY')` asserting `is_string($val) && $val !== ''` and throwing `RuntimeException` when unset (`env('KEY', 'default')` and hardcoded secrets are **FORBIDDEN**). `service('settings')` MUST NOT be used unless `codeigniter4/settings` is installed and its return value is verified via `instanceof`.
- **7.4 Throttling:** Apply `App\Filters\ThrottleFilter` (`throttle` alias in `app/Config/Filters.php`), e.g., `'filter' => 'throttle:5,60'`, to auth, payment, and heavy endpoints. Filter arguments MUST be validated as positive digit strings (`ctype_digit` and `> 0`, throwing `InvalidArgumentException`). Throttled requests MUST deterministically return HTTP `429 Too Many Requests` (`ResponseInterface`) with a `Retry-After` header (JSON envelope for AJAX/`Accept: application/json`, explicit `429` error body for browser requests; `redirect()->back()` is **FORBIDDEN**).
- **7.5 XSS & URL Safety:** Escape ALL dynamic scalar values, URLs, tokens, and dates via `esc($v, 'html'|'attr'|'js')`. Dynamic URLs rendered into `href` or `src` attributes MUST be validated for safe schemes (`http`, `https`, or relative `/`, rejecting `//`, `javascript:`, `data:`, and `vbscript:`) BEFORE `esc($url, 'attr')`. Exempt ONLY structural helpers: `form_open()`, `form_open_multipart()`, `form_close()`, `$this->include()`, `$this->renderSection()`, `csp_script_nonce()`, `csp_style_nonce()`, and `json_encode($schemaJson, JSON_HEX_TAG | JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT | JSON_THROW_ON_ERROR)` inside `<script type="application/ld+json">` (`JSON_HEX_TAG` prevents `</script>` breakout XSS).

---

## Part 8: Native File Handling, Storage & Streaming

- **Prohibited:** `$_FILES`, `header()`, `file_get_contents()`, `readfile()`.
- **8.1 Isolation & Ephemeral Staging:** `WRITEPATH` (`writable/tmp/`) is reserved strictly for ephemeral upload/download staging; `FCPATH` (`public/`) is reserved strictly for public static assets. Persistent local storage of uploaded user documents and binary/resource handles in session are **FORBIDDEN**. On upload, Services MUST offload staged files to cloud object storage, persist the `cloud_locator`, validated `mime_type`, and `extension` inside the record's DB `metadata` within a transaction, and destroy the staged file via explicit `unlink($stagedPath)` (`@unlink()` **FORBIDDEN**; failure MUST throw `RuntimeException` and log `'critical'`). On download, Services MUST verify authorization, fetch the remote object into an ephemeral `WRITEPATH . 'tmp/downloads/'` file, and register a worker-safe `Events::on('post_system', ...)` cleanup listener (removing the listener after execution so it runs once per request) that executes `unlink($tempPath)` and logs `'critical'` if deletion fails. Create staging directories via `mkdir($dir, 0755, true)` with explicit error checks.
- **8.2–8.4 Uploads:** Narrow `$this->request instanceof IncomingRequest` in the Controller; retrieve single uploads via `$this->request->getFile('key')` and multiple uploads via `getFileMultiple('key')`. Validate in Controller (`uploaded`, `max_size`, `mime_in`, `ext_in`; and `is_image`, `max_dims` exclusively on image-only endpoints). In the Service, assert `$file instanceof UploadedFile && $file->isValid() && ! $file->hasMoved()` (throwing `RuntimeException` with `$file->getErrorString()`, cleaning up prior files in multi-file loops), inspect bytes via `$file->getMimeType()` against an explicit allowlist (client extension or headers MUST NOT be trusted), generate `$file->getRandomName()` (`getClientName()` is **FORBIDDEN** for storage paths), and assert `$file->move($dir, $name) === true`.
- **8.5 Downloads:** Controllers MUST delegate authorization and ephemeral staging to the Service (`DocumentDownload` DTO), then stream via `$this->response->download($dto->tempPath, null, true)` (`CodeIgniter\HTTP\DownloadResponse` handles MIME, `Content-Length`, `Content-Disposition`, `Content-Transfer-Encoding: binary`, chunked streaming, and buffer cleansing; chain `->setFileName($dto->downloadName)` and throw `RuntimeException` if `download()` returns `null`). Download endpoints MUST sit behind `auth` routes and enforce actor ownership/permissions and record status (`401` unauthenticated, `403` unauthorized, `404` missing record/asset) before streaming.
- **8.6 Upload Views:** MUST use `form_open_multipart(url_to(...))` (raw `<form enctype=...>` **FORBIDDEN**) with Bootstrap 5.3 `.is-invalid`/`.invalid-feedback` bindings.

---

## Part 9: Error Handling, Logging & Environment

- **Exceptions:** Services MUST throw typed exceptions (`DomainNotFoundException`, `DomainAuthorizationException`, `RuntimeException`) on failure. Controllers MUST catch typed exceptions (`404` / `403`) and `\Throwable` (`500` or PRG error redirect), log context, and return sanitized messages (Controllers MUST NOT expose raw `$e->getMessage()`). Production errors MUST render `app/Views/errors/html/production.php`.
- **Logging:** Use `log_message('critical'|'error'|'info', $msg, $context)` (`critical`: DB, disk, or ephemeral cleanup failure; `error`: runtime, transaction, or upload failure; `info`: business milestones).
- **Environments:** `development` (`CI_ENVIRONMENT=development`, `display_errors=1`, Toolbar On); `testing` (`testing`, `0`, Off); `production` (`production`, `0`, Off). Committing `d()`, `dd()`, `die()`, `var_dump()`, or `print_r()` is **FORBIDDEN**.

---

## Part 10: Frontend Blueprints & Bootstrap 5.3

- **Framework:** Use native Bootstrap 5.3+ utilities and components (`.container`, `.container-fluid`, `.row`, `.col-*`, `.vstack`, `.hstack`, `.gap-*`, `.form-floating`, `.btn-primary`, `.btn-outline-secondary`, `.btn-outline-danger`/`.btn-danger`, `.hstack.gap-2`). Only `--theme-*` Bootstrap CSS custom property mappings and `:focus-visible` rules inside `<style <?= csp_style_nonce() ?>>` in the base layout SHALL be used for custom styling; all other custom component CSS, inline `style=""`, hardcoded hex colors, and third-party CSS frameworks are **FORBIDDEN**.
- **Themes & Layouts:** Set `<html data-bs-theme="light|dark">`. Page views MUST extend `layouts/default`; the layout MUST NOT couple to module routes (inject module navigation via `renderSection('nav_items')`).
- **SEO & JSON-LD:** Render `pageTitle`, `metaDescription`, `metaKeywords`, `canonicalUrl`, `robotsTag`, `metaImage`, and `schemaJson`. Serialize `$schemaJson` ONLY via `json_encode($schemaJson, JSON_HEX_TAG | JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT | JSON_THROW_ON_ERROR)` (manual JSON concatenation or `esc(..., 'js')` is **FORBIDDEN**).

---

## Part 11: Bootstrap 5.3 Pagination Template

Register `'bs_full' => 'App\Views\Pagers\bs_full'` in `app/Config/Pager.php`. File `app/Views/Pagers/bs_full.php` (**REQUIRED** standard):

```php
<?php

declare(strict_types=1);

use CodeIgniter\Pager\PagerRenderer;
use CodeIgniter\View\View;

/**
 * @var View $this
 * @var PagerRenderer $pager
 */
$pager->setSurroundCount(2);
?>
<nav aria-label="<?= esc(strict_lang('Pager.pageNavigation'), 'attr') ?>">
    <ul class="pagination pagination-sm justify-content-center flex-wrap mb-0">
        <?php if ($pager->hasPreviousPage()) : ?>
            <li class="page-item">
                <a href="<?= esc(safe_url(strict_string($pager->getFirst(), 'Pager first URI')), 'attr') ?>"
                   class="page-link"
                   aria-label="<?= esc(strict_lang('Pager.first'), 'attr') ?>">
                    <span aria-hidden="true">&laquo;&laquo;</span>
                </a>
            </li>
            <li class="page-item">
                <a href="<?= esc(safe_url(strict_string($pager->getPreviousPage(), 'Pager previous URI')), 'attr') ?>"
                   class="page-link"
                   aria-label="<?= esc(strict_lang('Pager.previous'), 'attr') ?>">
                    <span aria-hidden="true">&laquo;</span>
                </a>
            </li>
        <?php else : ?>
            <li class="page-item disabled"><span class="page-link" aria-hidden="true">&laquo;&laquo;</span></li>
            <li class="page-item disabled"><span class="page-link" aria-hidden="true">&laquo;</span></li>
        <?php endif ?>
        <?php foreach ($pager->links() as $link) : ?>
            <li class="page-item <?= strict_bool($link['active'], 'Pager link active') ? 'active' : '' ?>"
                <?= strict_bool($link['active'], 'Pager link active') ? 'aria-current="page"' : '' ?>>
                <a href="<?= esc(safe_url(strict_string($link['uri'], 'Pager link URI')), 'attr') ?>"
                   class="page-link"><?= esc(strict_string($link['title'], 'Pager link title'), 'html') ?></a>
            </li>
        <?php endforeach ?>
        <?php if ($pager->hasNextPage()) : ?>
            <li class="page-item">
                <a href="<?= esc(safe_url(strict_string($pager->getNextPage(), 'Pager next URI')), 'attr') ?>"
                   class="page-link"
                   aria-label="<?= esc(strict_lang('Pager.next'), 'attr') ?>">
                    <span aria-hidden="true">&raquo;</span>
                </a>
            </li>
            <li class="page-item">
                <a href="<?= esc(safe_url(strict_string($pager->getLast(), 'Pager last URI')), 'attr') ?>"
                   class="page-link"
                   aria-label="<?= esc(strict_lang('Pager.last'), 'attr') ?>">
                    <span aria-hidden="true">&raquo;&raquo;</span>
                </a>
            </li>
        <?php else : ?>
            <li class="page-item disabled"><span class="page-link" aria-hidden="true">&raquo;</span></li>
            <li class="page-item disabled"><span class="page-link" aria-hidden="true">&raquo;&raquo;</span></li>
        <?php endif ?>
    </ul>
</nav>
```

---

## Part 12: Tooling, CLI & Publisher

- **Spark Commands:** Extend `CodeIgniter\CLI\BaseCommand` in `app/Commands/` or module `Commands/`, defining `$group`, `$name`, `$description`, `$usage`, `$arguments`, `$options` (untyped per CI4 parent exemption, with `@var` PHPDoc), and `run(array $params): int` catching `\Throwable` and returning `\EXIT_SUCCESS` or `\EXIT_ERROR`.
- **Prompts & Signals:** Use `CLI::prompt()` / `CLI::promptByKey()` with validation callbacks. Daemons MUST use `CodeIgniter\CLI\SignalTrait`, invoke `$this->registerSignals()`, loop on `while (! $this->shouldTerminate())`, and call `$this->unregisterSignals()` inside `finally` for deterministic `SIGINT`/`SIGTERM` cleanup.
- **Assets (`Publisher`):** Deploy vendor assets to `public/assets/vendor/` via `CodeIgniter\Publisher\Publisher`. Manual `public/` copy-paste and runtime external CDN links in layouts are **FORBIDDEN**.

---

## Part 13: Advanced Architecture

- **View Cells & Decorators:** Reusable data-fetching UI widgets MUST be implemented as `CodeIgniter\View\Cells\Cell` (`view_cell(...)`); service calls inside raw views are **FORBIDDEN**. Global output transforms MUST implement `CodeIgniter\View\ViewDecoratorInterface` in `Config\View::$decorators`.
- **Events:** Non-blocking secondary actions (emails, metrics) MUST trigger via `CodeIgniter\Events\Events::trigger('eventName', $payload)`.
- **CSP & CORS:** Enable `public bool $CSPEnabled = true;` in `app/Config/App.php` and configure strict CSP rules in `app/Config/ContentSecurityPolicy.php` using `<?= csp_script_nonce() ?>` and `<?= csp_style_nonce() ?>` exclusively (`{csp-script-nonce}` placeholder strings are **FORBIDDEN**). Manage CORS via `CodeIgniter\Filters\Cors` (`cors` alias, `app/Config/Cors.php`); wildcard `*` origins in production are **FORBIDDEN**.
- **Worker Mode:** Under long-lived worker runtimes (FrankenPHP, RoadRunner, Swoole via a dedicated worker adapter package), services MUST NOT store mutable request state in static properties; controllers MUST be instantiated per request and request-scoped services MUST be reset via `Services::reset()` or non-shared factories.

---

## Part 14: Testing Architecture

- **Suite & Clock:** PHPUnit (`./vendor/bin/phpunit`) is **REQUIRED** for every module, service, model, controller, filter, and command (`php spark test` does not exist in core CI4). Extend `CodeIgniter\Test\CIUnitTestCase`, call `parent::setUp()` / `parent::tearDown()`, freeze time with `Time::setTestNow(...)`, and restore with `Time::setTestNow()`.
- **Database (`DatabaseTestTrait`):** Declare `protected $migrate = true;` and `protected $namespace = 'App\Modules\[ModuleName]';` (or `null` / `array<int, string>` when core `App` migrations are also required); auto-rollback keeps the test DB clean; assert state via `seeInDatabase()`, `dontSeeInDatabase()`, and `seeNumRecords()`.
- **Data (`Fabricator`):** Use `CodeIgniter\Test\Fabricator` with test-only faker subclasses under `tests/_support/Fakers/` implementing `fake(Generator &$faker): array`. Production `Models/` MUST NOT import `Faker\Generator` (`--no-dev` safe). Raw SQL `INSERT` statements in tests are **FORBIDDEN**.
- **HTTP (`FeatureTestTrait`):** Execute `$this->get()`, `$this->post()`, `$this->put()`, and `$this->delete()` using `url_to(...)` (passing `csrf_token() => csrf_hash()` on mutations, and `withSession([...])` for authenticated flows); assert `TestResponse` (`assertStatus()`, `assertOK()`, `assertRedirectTo()`, `assertSessionHas()`, `assertSee()`).
- **Mocks:** Inject test doubles via `\Config\Services::injectMock('name', $mock)` and reset deterministically in `tearDown()` via `\Config\Services::reset()`.

---

## Part 15: Boot & Deployment SOP

- **Local Boot:** `composer install` -> `php spark migrate --all` -> `php spark db:seed MainSeeder`.
- **Production Deploy:** (1) `CI_ENVIRONMENT = production`, `display_errors = 0`; (2) `composer install --no-dev --optimize-autoloader --no-interaction`; (3) `php spark optimize`; (4) `php spark migrate --all`; (5) Set web server document root strictly to `/public` (block `/app`, `/system`, `/writable`, `/vendor`; grant web user write access exclusively to `writable/`).

---

## Part 16: Endpoint Verification & Log-Auditing Loop

1. Verify every route in `php spark routes` via headless browser or `curl -i` (engineers MUST NOT pass `-L`, so intermediate status codes and PRG `Location` headers are verified explicitly; test mutating `POST`/`PUT`/`PATCH`/`DELETE` routes with valid CSRF tokens; assuming template correctness without execution is **FORBIDDEN**).
2. Inspect `writable/logs/*.log` simultaneously for exceptions, DB warnings, or PHP notices.
3. On any failure: analyze stack trace -> fix per MVC-S -> purge `writable/logs/*.log` (preserving `.gitkeep` and `index.html`) -> re-test -> repeat until every route passes with zero log entries.

---

## Appendix: Canonical Reference Implementation

### 1. `app/Config/Autoload.php`

```php
<?php

declare(strict_types=1);

namespace Config;

use CodeIgniter\Config\AutoloadConfig;

/**
 * Application autoload configuration.
 *
 * @package Config
 * @author  Senior Developer
 * @since   1.0.0
 */
class Autoload extends AutoloadConfig
{
    /**
     * PSR-4 namespace mappings.
     *
     * @var array<string, string|list<string>>
     */
    public $psr4 = [
        APP_NAMESPACE              => APPPATH,
        'App\Modules\[ModuleName]' => APPPATH . 'Modules/[ModuleName]',
    ];
}
```

### 2. `app/Modules/[ModuleName]/Config/Services.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Config;

use App\Modules\[ModuleName]\Libraries\DomainService;
use App\Modules\[ModuleName]\Models\DomainModel;
use CodeIgniter\Config\BaseService;
use RuntimeException;

/**
 * Module service container definitions.
 *
 * @package App\Modules\[ModuleName]\Config
 * @author  Senior Developer
 * @since   1.0.0
 */
class Services extends BaseService
{
    /**
     * Resolves the DomainService instance.
     *
     * @param DomainModel|null $domainModel Optional model dependency for testing or overrides.
     * @param bool             $getShared   Whether to return the shared container instance.
     *
     * @return DomainService
     *
     * @throws RuntimeException When the model or shared service cannot be resolved.
     */
    public static function domainService(
        ?DomainModel $domainModel = null,
        bool $getShared = true
    ): DomainService {
        if ($getShared) {
            $service = static::getSharedInstance('domainService', $domainModel);
            if (! $service instanceof DomainService) {
                throw new RuntimeException('Failed to resolve shared DomainService instance.');
            }

            return $service;
        }

        if ($domainModel === null) {
            $resolved = model(DomainModel::class);
            if (! $resolved instanceof DomainModel) {
                throw new RuntimeException('Failed to resolve DomainModel instance.');
            }
            $domainModel = $resolved;
        }

        return new DomainService($domainModel);
    }
}
```

### 3. `app/Modules/[ModuleName]/Config/Routes.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Config;

use CodeIgniter\Router\RouteCollection;

/** @var RouteCollection $routes */
$routes->group(
    'domain',
    ['namespace' => 'App\Modules\[ModuleName]\Controllers'],
    static function (RouteCollection $routes): void {
        $routes->get(
            'record',
            'DomainController::index',
            ['as' => 'domain.record.index', 'filter' => 'auth']
        );
        $routes->get(
            'record/new',
            'DomainController::new',
            ['as' => 'domain.record.new', 'filter' => 'auth']
        );
        $routes->post(
            'record',
            'DomainController::create',
            ['as' => 'domain.record.create', 'filter' => ['auth', 'throttle:5,60']]
        );
        $routes->post(
            'file/upload',
            'DomainController::upload',
            ['as' => 'domain.file.upload', 'filter' => ['auth', 'throttle:10,60']]
        );
        $routes->get(
            'file/download/(:num)',
            'DomainController::download/$1',
            ['as' => 'domain.file.download', 'filter' => ['auth', 'throttle:30,60']]
        );
    }
);
```

### 4. `app/Filters/ThrottleFilter.php`

```php
<?php

declare(strict_types=1);

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\IncomingRequest;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use CodeIgniter\Throttle\ThrottlerInterface;
use InvalidArgumentException;
use RuntimeException;

/**
 * Deterministic rate-limiting filter returning HTTP 429 on limit exhaustion.
 *
 * @package App\Filters
 * @author  Senior Developer
 * @since   1.0.0
 */
class ThrottleFilter implements FilterInterface
{
    /**
     * Enforces request throttling before controller dispatch.
     *
     * @param RequestInterface        $request   Current incoming HTTP request.
     * @param array<int, string>|null $arguments Filter arguments [limit, seconds].
     *
     * @return RequestInterface|ResponseInterface|string|null
     *
     * @throws InvalidArgumentException When filter arguments are missing or non-positive integers.
     * @throws RuntimeException         When throttler or response services cannot be resolved.
     */
    public function before(
        RequestInterface $request,
        $arguments = null
    ): RequestInterface|ResponseInterface|string|null {
        if (! $request instanceof IncomingRequest) {
            return null;
        }
        if ($arguments === null || count($arguments) < 2) {
            throw new InvalidArgumentException('ThrottleFilter requires throttle:[limit],[seconds].');
        }
        if (! ctype_digit($arguments[0]) || ! ctype_digit($arguments[1])) {
            throw new InvalidArgumentException('ThrottleFilter arguments must be positive integers.');
        }

        $capacity = (int) $arguments[0];
        $seconds  = (int) $arguments[1];
        if ($capacity <= 0 || $seconds <= 0) {
            throw new InvalidArgumentException('ThrottleFilter limit and window must exceed zero.');
        }

        $throttler = service('throttler');
        if (! $throttler instanceof ThrottlerInterface) {
            throw new RuntimeException('Throttler service is unavailable.');
        }

        $key = 'throttle_' . hash('sha256', $request->getIPAddress() . '|' . $request->getRoutePath());
        if ($throttler->check($key, $capacity, $seconds) === false) {
            $response = service('response');
            if (! $response instanceof ResponseInterface) {
                throw new RuntimeException('Response service is unavailable.');
            }

            $response->setStatusCode(429, 'Too Many Requests')
                ->setHeader('Retry-After', (string) $seconds);

            if ($request->isAJAX() || str_contains($request->getHeaderLine('Accept'), 'application/json')) {
                return $response->setJSON([
                    'status'     => 'error',
                    'message'    => 'Too many requests. Please try again later.',
                    'result'     => null,
                    'errors'     => [],
                    'csrf_token' => csrf_hash(),
                ]);
            }

            return $response->setBody('Too many requests. Please try again later.');
        }

        return null;
    }

    /**
     * Post-controller hook (no-op).
     *
     * @param RequestInterface        $request   Current HTTP request.
     * @param ResponseInterface       $response  Generated HTTP response.
     * @param array<int, string>|null $arguments Filter arguments.
     *
     * @return ResponseInterface|null
     */
    public function after(
        RequestInterface $request,
        ResponseInterface $response,
        $arguments = null
    ): ?ResponseInterface {
        return null;
    }
}
```

### 5. `app/Modules/[ModuleName]/Database/Migrations/`

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Database\Migrations;

use CodeIgniter\Database\Migration;

/**
 * Creates the parent domains reference table.
 *
 * @package App\Modules\[ModuleName]\Database\Migrations
 * @author  Senior Developer
 * @since   1.0.0
 */
class CreateDomainsTable extends Migration
{
    /**
     * Applies the migration schema changes.
     *
     * @return void
     */
    public function up(): void
    {
        $this->forge->addField([
            'id' => [
                'type'           => 'INT',
                'unsigned'       => true,
                'auto_increment' => true,
                'null'           => false,
            ],
            'slug' => [
                'type'       => 'VARCHAR',
                'constraint' => 120,
                'null'       => false,
            ],
            'created_at' => [
                'type' => 'DATETIME',
                'null' => false,
            ],
            'updated_at' => [
                'type' => 'DATETIME',
                'null' => true,
            ],
            'deleted_at' => [
                'type' => 'DATETIME',
                'null' => true,
            ],
        ]);
        $this->forge->addKey('id', true);
        $this->forge->addUniqueKey('slug');
        $this->forge->createTable('domains', true, [
            'ENGINE'        => 'InnoDB',
            'CHARACTER SET' => 'utf8mb4',
            'COLLATE'       => 'utf8mb4_unicode_ci',
            'ROW_FORMAT'    => 'DYNAMIC',
        ]);
    }

    /**
     * Reverts the migration schema changes.
     *
     * @return void
     */
    public function down(): void
    {
        $this->db->disableForeignKeyChecks();
        try {
            $this->forge->dropTable('domains', true);
        } finally {
            $this->db->enableForeignKeyChecks();
        }
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Database\Migrations;

use CodeIgniter\Database\Migration;

/**
 * Creates the domain_table schema with strict foreign keys and indexes.
 *
 * @package App\Modules\[ModuleName]\Database\Migrations
 * @author  Senior Developer
 * @since   1.0.0
 */
class CreateDomainTable extends Migration
{
    /**
     * Applies the migration schema changes.
     *
     * @return void
     */
    public function up(): void
    {
        $this->forge->addField([
            'id' => [
                'type'           => 'INT',
                'unsigned'       => true,
                'auto_increment' => true,
                'null'           => false,
            ],
            'domain_id' => [
                'type'     => 'INT',
                'unsigned' => true,
                'null'     => false,
            ],
            'owner_id' => [
                'type'     => 'INT',
                'unsigned' => true,
                'null'     => false,
            ],
            'value' => [
                'type'       => 'DECIMAL',
                'constraint' => '12,2',
                'default'    => '0.00',
                'null'       => false,
            ],
            'is_active' => [
                'type'       => 'TINYINT',
                'constraint' => 1,
                'default'    => 0,
                'null'       => false,
            ],
            'metadata' => [
                'type' => 'TEXT',
                'null' => true,
            ],
            'created_at' => [
                'type' => 'DATETIME',
                'null' => false,
            ],
            'updated_at' => [
                'type' => 'DATETIME',
                'null' => true,
            ],
            'deleted_at' => [
                'type' => 'DATETIME',
                'null' => true,
            ],
        ]);
        $this->forge->addKey('id', true);
        $this->forge->addKey('domain_id');
        $this->forge->addKey('owner_id');
        $this->forge->addKey(['domain_id', 'is_active']);
        $this->forge->addKey(['domain_id', 'owner_id']);
        $this->forge->addKey('created_at');
        $this->forge->addForeignKey('domain_id', 'domains', 'id', 'CASCADE', 'RESTRICT');
        $this->forge->createTable('domain_table', true, [
            'ENGINE'        => 'InnoDB',
            'CHARACTER SET' => 'utf8mb4',
            'COLLATE'       => 'utf8mb4_unicode_ci',
            'ROW_FORMAT'    => 'DYNAMIC',
        ]);
    }

    /**
     * Reverts the migration schema changes.
     *
     * @return void
     */
    public function down(): void
    {
        $this->db->disableForeignKeyChecks();
        try {
            $this->forge->dropTable('domain_table', true);
        } finally {
            $this->db->enableForeignKeyChecks();
        }
    }
}
```

### 6. `Entities/DomainEntity.php` & `Models/DomainModel.php` (plus `DomainsModel.php`)

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Entities;

use CodeIgniter\Entity\Entity;
use CodeIgniter\I18n\Time;

/**
 * Persistent domain record entity.
 *
 * @package App\Modules\[ModuleName]\Entities
 * @author  Senior Developer
 * @since   1.0.0
 *
 * @property int                       $id
 * @property int                       $domain_id
 * @property int                       $owner_id
 * @property string                    $value
 * @property bool                      $is_active
 * @property array<string, mixed>|null $metadata
 * @property Time|null                 $created_at
 * @property Time|null                 $updated_at
 * @property Time|null                 $deleted_at
 */
class DomainEntity extends Entity
{
    /**
     * Attribute cast definitions.
     *
     * @var array<string, string>
     */
    protected $casts = [
        'id'        => 'integer',
        'domain_id' => 'integer',
        'owner_id'  => 'integer',
        'value'     => 'string',
        'is_active' => 'boolean',
        'metadata'  => '?json-array',
    ];

    /**
     * Timestamp fields mutated to Time instances.
     *
     * @var array<int, string>
     */
    protected $dates = ['created_at', 'updated_at', 'deleted_at'];
}
```

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Models;

use App\Modules\[ModuleName]\Entities\DomainEntity;
use CodeIgniter\Database\BaseConnection;
use CodeIgniter\Model;

/**
 * Persistence gateway for domain_table records.
 *
 * @package App\Modules\[ModuleName]\Models
 * @author  Senior Developer
 * @since   1.0.0
 *
 * @property BaseConnection $db
 */
class DomainModel extends Model
{
    /**
     * Target database table name.
     *
     * @var string
     */
    protected $table = 'domain_table';

    /**
     * Primary key column name.
     *
     * @var string
     */
    protected $primaryKey = 'id';

    /**
     * Entity class hydrated on queries.
     *
     * @var class-string<DomainEntity>
     */
    protected $returnType = DomainEntity::class;

    /**
     * Mass-assignable columns.
     *
     * @var array<int, string>
     */
    protected $allowedFields = ['domain_id', 'owner_id', 'value', 'is_active', 'metadata'];

    /**
     * Enables automatic timestamp management.
     *
     * @var bool
     */
    protected $useTimestamps = true;

    /**
     * Database datetime format string.
     *
     * @var string
     */
    protected $dateFormat = 'datetime';

    /**
     * Creation timestamp column.
     *
     * @var string
     */
    protected $createdField = 'created_at';

    /**
     * Update timestamp column.
     *
     * @var string
     */
    protected $updatedField = 'updated_at';

    /**
     * Soft-delete timestamp column.
     *
     * @var string
     */
    protected $deletedField = 'deleted_at';

    /**
     * Enables soft deletes on stateful records.
     *
     * @var bool
     */
    protected $useSoftDeletes = true;
}
```

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Models;

use CodeIgniter\Database\BaseConnection;
use CodeIgniter\Model;

/**
 * Persistence gateway for parent domains records.
 *
 * @package App\Modules\[ModuleName]\Models
 * @author  Senior Developer
 * @since   1.0.0
 *
 * @property BaseConnection $db
 */
class DomainsModel extends Model
{
    /**
     * Target database table name.
     *
     * @var string
     */
    protected $table = 'domains';

    /**
     * Primary key column name.
     *
     * @var string
     */
    protected $primaryKey = 'id';

    /**
     * Return type for parent domain rows.
     *
     * @var string
     */
    protected $returnType = 'object';

    /**
     * Mass-assignable columns.
     *
     * @var array<int, string>
     */
    protected $allowedFields = ['slug'];

    /**
     * Enables automatic timestamp management.
     *
     * @var bool
     */
    protected $useTimestamps = true;

    /**
     * Database datetime format string.
     *
     * @var string
     */
    protected $dateFormat = 'datetime';

    /**
     * Creation timestamp column.
     *
     * @var string
     */
    protected $createdField = 'created_at';

    /**
     * Update timestamp column.
     *
     * @var string
     */
    protected $updatedField = 'updated_at';

    /**
     * Soft-delete timestamp column.
     *
     * @var string
     */
    protected $deletedField = 'deleted_at';

    /**
     * Enables soft deletes.
     *
     * @var bool
     */
    protected $useSoftDeletes = true;
}
```

### 7. `Libraries/DocumentDownload.php`, Typed Exceptions & `Libraries/DomainService.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Libraries;

/**
 * Immutable DTO representing an authorized ephemeral document download.
 *
 * @package App\Modules\[ModuleName]\Libraries
 * @author  Senior Developer
 * @since   1.0.0
 */
final readonly class DocumentDownload
{
    /**
     * Constructs the download descriptor.
     *
     * @param string $tempPath     Absolute path to the ephemeral staging file in WRITEPATH.
     * @param string $downloadName Safe client-facing filename with verified extension.
     */
    public function __construct(
        public string $tempPath,
        public string $downloadName
    ) {
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Libraries;

use RuntimeException;

/**
 * Thrown when a requested domain record or cloud object does not exist.
 *
 * @package App\Modules\[ModuleName]\Libraries
 * @author  Senior Developer
 * @since   1.0.0
 */
class DomainNotFoundException extends RuntimeException
{
}
```

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Libraries;

use RuntimeException;

/**
 * Thrown when an actor lacks authorization for a domain resource.
 *
 * @package App\Modules\[ModuleName]\Libraries
 * @author  Senior Developer
 * @since   1.0.0
 */
class DomainAuthorizationException extends RuntimeException
{
}
```

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Libraries;

use App\Modules\[ModuleName]\Entities\DomainEntity;
use App\Modules\[ModuleName]\Models\DomainModel;
use CodeIgniter\Database\BaseConnection;
use CodeIgniter\Events\Events;
use CodeIgniter\HTTP\Files\UploadedFile;
use RuntimeException;
use Throwable;

/**
 * Business logic orchestrator for domain records and document storage.
 *
 * @package App\Modules\[ModuleName]\Libraries
 * @author  Senior Developer
 * @since   1.0.0
 */
class DomainService
{
    /**
     * Allowed MIME types mapped to canonical file extensions.
     *
     * @var array<string, string>
     */
    private const ALLOWED_MIME_EXTENSIONS = [
        'application/pdf' => 'pdf',
        'image/jpeg'      => 'jpg',
        'image/png'       => 'png',
    ];

    /**
     * Injected domain persistence model.
     *
     * @var DomainModel
     */
    protected DomainModel $domainModel;

    /**
     * Constructs the DomainService instance.
     *
     * @param DomainModel $domainModel Injected domain record model.
     */
    public function __construct(DomainModel $domainModel)
    {
        $this->domainModel = $domainModel;
    }

    // --- Helper Methods ---

    /**
     * Asserts and returns the active BaseConnection instance.
     *
     * @return BaseConnection
     *
     * @throws RuntimeException When the underlying DB connection is invalid.
     */
    private function requireConnection(): BaseConnection
    {
        $db = $this->domainModel->db;
        if (! $db instanceof BaseConnection) {
            throw new RuntimeException('DomainModel database connection is not a BaseConnection.');
        }

        return $db;
    }

    /**
     * Ensures a staging directory exists with 0755 permissions.
     *
     * @param string $dir Absolute directory path under WRITEPATH.
     *
     * @return void
     *
     * @throws RuntimeException When directory creation fails.
     */
    private function ensureDirectory(string $dir): void
    {
        if (! is_dir($dir) && ! mkdir($dir, 0755, true) && ! is_dir($dir)) {
            throw new RuntimeException('Failed to create staging directory: ' . $dir);
        }
    }

    // --- Public API ---

    /**
     * Persists a new DomainEntity inside an explicit database transaction.
     *
     * @param DomainEntity $entity Validated entity to persist.
     *
     * @return DomainEntity
     *
     * @throws RuntimeException When insertion, retrieval, or commit fails.
     * @throws Throwable        When an unexpected persistence error occurs.
     */
    public function createRecord(DomainEntity $entity): DomainEntity
    {
        $db = $this->requireConnection();
        if (! $db->transBegin()) {
            throw new RuntimeException('Failed to begin database transaction.');
        }

        try {
            $insertedId = $this->domainModel->insert($entity, true);
            if (! is_int($insertedId) && ! is_string($insertedId)) {
                throw new RuntimeException('Database record insertion failed.');
            }

            /** @var DomainEntity|null $savedEntity */
            $savedEntity = $this->domainModel
                ->select('id, domain_id, owner_id, value, is_active, metadata, created_at, updated_at, deleted_at')
                ->find($insertedId);
            if (! $savedEntity instanceof DomainEntity) {
                throw new RuntimeException('Failed to retrieve newly created domain entity.');
            }

            if (! $db->transCommit()) {
                throw new RuntimeException('Database transaction commit failed.');
            }

            return $savedEntity;
        } catch (Throwable $e) {
            if ($db->transDepth > 0) {
                $db->transRollback();
            }
            log_message(
                'critical',
                'DomainService::createRecord failed: ' . $e->getMessage(),
                ['entity' => $entity->toArray(), 'trace' => $e->getTraceAsString()]
            );
            throw $e;
        }
    }

    /**
     * Retrieves active and inactive domain records for listing views.
     *
     * @return array<int, DomainEntity>
     */
    public function getAllRecords(): array
    {
        /** @var array<int, DomainEntity> $records */
        $records = $this->domainModel
            ->select('id, domain_id, owner_id, value, is_active, created_at')
            ->findAll();

        return $records;
    }

    /**
     * Validates, stages, offloads to cloud storage, persists metadata, and purges local staging.
     *
     * @param UploadedFile $file     Uploaded file instance.
     * @param int          $domainId Target domain identifier (> 0).
     * @param int          $actorId  Authenticated actor identifier (> 0).
     *
     * @return string Cloud storage locator URI.
     *
     * @throws DomainNotFoundException      When the target record does not exist.
     * @throws DomainAuthorizationException When the actor does not own the record.
     * @throws RuntimeException             When file validation, move, transaction, or unlink fails.
     * @throws Throwable                    When an unexpected processing error occurs.
     */
    public function processUploadedDocument(UploadedFile $file, int $domainId, int $actorId): string
    {
        if (! $file->isValid() || $file->hasMoved()) {
            throw new RuntimeException('Invalid uploaded file: ' . $file->getErrorString());
        }

        $mimeType = $file->getMimeType();
        if (! isset(self::ALLOWED_MIME_EXTENSIONS[$mimeType])) {
            throw new RuntimeException('Disallowed MIME type detected: ' . $mimeType);
        }
        $extension = self::ALLOWED_MIME_EXTENSIONS[$mimeType];

        /** @var DomainEntity|null $record */
        $record = $this->domainModel
            ->select('id, domain_id, owner_id, is_active, metadata')
            ->where('domain_id', $domainId)
            ->first();
        if (! $record instanceof DomainEntity) {
            throw new DomainNotFoundException('Target domain record not found.');
        }
        if ($record->owner_id !== $actorId) {
            throw new DomainAuthorizationException('Actor is not authorized to upload for this record.');
        }

        $dir = WRITEPATH . 'tmp/uploads/' . $domainId . '/';
        $this->ensureDirectory($dir);

        $secureName = $file->getRandomName();
        if (! $file->move($dir, $secureName)) {
            throw new RuntimeException('Failed to stage uploaded file: ' . $file->getErrorString());
        }
        $stagedPath = $dir . $secureName;

        $db = $this->requireConnection();
        if (! $db->transBegin()) {
            if (is_file($stagedPath) && ! unlink($stagedPath)) {
                log_message('critical', 'Failed to purge staged file after transBegin failure', [
                    'staged_path' => $stagedPath,
                    'domain_id'   => $domainId,
                ]);
            }
            throw new RuntimeException('Failed to begin upload metadata transaction.');
        }

        try {
            $remoteLocator             = 'cloud_bucket://domains/' . $domainId . '/' . $secureName;
            $metadata                  = is_array($record->metadata) ? $record->metadata : [];
            $metadata['cloud_locator'] = $remoteLocator;
            $metadata['mime_type']     = $mimeType;
            $metadata['extension']     = $extension;

            if (! $this->domainModel->update($record->id, ['metadata' => $metadata])) {
                throw new RuntimeException('Failed to persist document metadata.');
            }
            if (! $db->transCommit()) {
                throw new RuntimeException('Failed to commit upload metadata transaction.');
            }
            if (! unlink($stagedPath)) {
                log_message('critical', 'Failed to purge ephemeral upload staging', [
                    'staged_path' => $stagedPath,
                    'domain_id'   => $domainId,
                ]);
                throw new RuntimeException('Failed to purge ephemeral upload staging: ' . $stagedPath);
            }

            log_message('info', 'DomainService::processUploadedDocument completed', [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
                'locator'   => $remoteLocator,
            ]);

            return $remoteLocator;
        } catch (Throwable $e) {
            if ($db->transDepth > 0) {
                $db->transRollback();
            }
            if (is_file($stagedPath) && ! unlink($stagedPath)) {
                log_message('critical', 'Failed to purge staged file during rollback', [
                    'staged_path' => $stagedPath,
                    'domain_id'   => $domainId,
                ]);
            }
            log_message(
                'error',
                'DomainService::processUploadedDocument failed: ' . $e->getMessage(),
                ['domain_id' => $domainId, 'actor_id' => $actorId, 'trace' => $e->getTraceAsString()]
            );
            throw $e;
        }
    }

    /**
     * Authorizes actor access and materializes an ephemeral download file with shutdown cleanup.
     *
     * @param int $domainId Target domain identifier (> 0).
     * @param int $actorId  Authenticated actor identifier (> 0).
     *
     * @return DocumentDownload
     *
     * @throws DomainNotFoundException      When the record or document metadata is missing.
     * @throws DomainAuthorizationException When the record is inactive or owned by another actor.
     * @throws RuntimeException             When ephemeral download staging fails.
     */
    public function prepareAuthorizedDownload(int $domainId, int $actorId): DocumentDownload
    {
        /** @var DomainEntity|null $record */
        $record = $this->domainModel
            ->select('id, domain_id, owner_id, is_active, metadata')
            ->where('domain_id', $domainId)
            ->first();
        if (! $record instanceof DomainEntity) {
            throw new DomainNotFoundException('Domain record does not exist.');
        }
        if ($record->owner_id !== $actorId || ! $record->is_active) {
            throw new DomainAuthorizationException('Actor unauthorized or record inactive.');
        }

        $meta = $record->metadata;
        if (
            ! is_array($meta)
            || ! isset($meta['cloud_locator'], $meta['extension'])
            || ! is_string($meta['cloud_locator'])
            || ! is_string($meta['extension'])
        ) {
            throw new DomainNotFoundException('Document metadata is missing from domain record.');
        }

        if (! in_array($meta['extension'], self::ALLOWED_MIME_EXTENSIONS, true)) {
            throw new RuntimeException('Stored document extension is invalid.');
        }

        $dir = WRITEPATH . 'tmp/downloads/';
        $this->ensureDirectory($dir);

        $tempPath = $dir . 'dl_' . bin2hex(random_bytes(16)) . '.' . $meta['extension'];
        $listener = null;
        $listener = static function () use ($tempPath, &$listener): void {
            if (is_callable($listener)) {
                Events::removeListener('post_system', $listener);
            }
            if (is_file($tempPath) && ! unlink($tempPath)) {
                log_message('critical', 'Failed to purge ephemeral download file', ['temp_path' => $tempPath]);
            }
        };
        Events::on('post_system', $listener);

        // Example stub: Cloud SDK adapter fetches $meta['cloud_locator'] bytes into $tempPath.
        if (file_put_contents($tempPath, $meta['cloud_locator']) === false) {
            Events::removeListener('post_system', $listener);
            throw new RuntimeException('Failed to write ephemeral download file.');
        }

        return new DocumentDownload($tempPath, 'domain_export_' . $domainId . '.' . $meta['extension']);
    }

    /**
     * Synchronizes inactive domain records in a deterministic transaction.
     *
     * @param bool $isDryRun When true, counts candidates without mutating database state.
     *
     * @return int Number of candidate or synchronized records.
     *
     * @throws RuntimeException When transaction or update fails.
     * @throws Throwable        When an unexpected synchronization error occurs.
     */
    public function syncRecords(bool $isDryRun): int
    {
        /** @var array<int, DomainEntity> $pending */
        $pending = $this->domainModel
            ->select('id, domain_id, is_active')
            ->where('is_active', 0)
            ->findAll();

        if ($isDryRun || $pending === []) {
            return count($pending);
        }

        $db = $this->requireConnection();
        if (! $db->transBegin()) {
            throw new RuntimeException('Failed to begin sync transaction.');
        }

        try {
            $count = 0;
            foreach ($pending as $record) {
                if (! $this->domainModel->update($record->id, ['is_active' => 1])) {
                    throw new RuntimeException('Failed to activate domain record ID: ' . $record->id);
                }
                $count++;
            }

            if (! $db->transCommit()) {
                throw new RuntimeException('Failed to commit sync transaction.');
            }

            log_message('info', 'DomainService::syncRecords activated records', ['count' => $count]);

            return $count;
        } catch (Throwable $e) {
            if ($db->transDepth > 0) {
                $db->transRollback();
            }
            log_message('critical', 'DomainService::syncRecords failed: ' . $e->getMessage(), [
                'trace' => $e->getTraceAsString(),
            ]);
            throw $e;
        }
    }
}
```

### 8. `app/Modules/[ModuleName]/Controllers/DomainController.php`

```php
<?php

declare(strict_types=1);

namespace App\Modules\[ModuleName]\Controllers;

use App\Controllers\BaseController;
use App\Modules\[ModuleName]\Entities\DomainEntity;
use App\Modules\[ModuleName]\Libraries\DomainAuthorizationException;
use App\Modules\[ModuleName]\Libraries\DomainNotFoundException;
use App\Modules\[ModuleName]\Libraries\DomainService;
use CodeIgniter\HTTP\DownloadResponse;
use CodeIgniter\HTTP\Files\UploadedFile;
use CodeIgniter\HTTP\IncomingRequest;
use CodeIgniter\HTTP\ResponseInterface;
use CodeIgniter\Router\Exceptions\RouterException;
use RuntimeException;
use Throwable;

/**
 * HTTP orchestration controller for domain records and authorized documents.
 *
 * @package App\Modules\[ModuleName]\Controllers
 * @author  Senior Developer
 * @since   1.0.0
 */
class DomainController extends BaseController
{
    /**
     * Controller-scoped helper list.
     *
     * @var array<int, string>
     */
    protected $helpers = ['strict_view', 'form'];

    /**
     * Injected DomainService facade.
     *
     * @var DomainService
     */
    protected DomainService $domainService;

    /**
     * Constructs the DomainController and resolves stateless service facade.
     *
     * @throws RuntimeException When DomainService cannot be resolved.
     */
    public function __construct()
    {
        $service = service('domainService');
        if (! $service instanceof DomainService) {
            throw new RuntimeException('DomainService could not be resolved.');
        }

        $this->domainService = $service;
    }

    // --- Helper Methods ---

    /**
     * Resolves and strictly validates the authenticated actor ID from session.
     *
     * @return int|null Positive actor ID, or null when unauthenticated.
     *
     * @throws RuntimeException When the session user_id value is malformed.
     */
    private function resolveAuthenticatedActorId(): ?int
    {
        $raw = session('user_id');
        if ($raw === null) {
            return null;
        }
        if (is_int($raw) && $raw > 0) {
            return $raw;
        }
        if (is_string($raw) && ctype_digit($raw) && (int) $raw > 0) {
            return (int) $raw;
        }

        throw new RuntimeException('Malformed session user_id value.');
    }

    /**
     * Asserts that the active validation instance is initialized.
     *
     * @return array<string, string>
     *
     * @throws RuntimeException When validator service is uninitialized.
     */
    private function requireValidationErrors(): array
    {
        if ($this->validator === null) {
            throw new RuntimeException('Validation service was not initialized.');
        }

        /** @var array<string, string> $errors */
        $errors = $this->validator->getErrors();

        return $errors;
    }

    /**
     * Asserts validator initialization and returns validated input.
     *
     * @return array<string, mixed>
     *
     * @throws RuntimeException When validator service is uninitialized.
     */
    private function requireValidatedInput(): array
    {
        if ($this->validator === null) {
            throw new RuntimeException('Validation service was not initialized.');
        }

        /** @var array<string, mixed> $validated */
        $validated = $this->validator->getValidated();

        return $validated;
    }

    // --- Public Actions ---

    /**
     * Displays the domain record listing view.
     *
     * @return string|ResponseInterface
     *
     * @throws RouterException When the canonical route name cannot be resolved.
     */
    public function index(): string|ResponseInterface
    {
        return view('App\Modules\[ModuleName]\Views\record\index', [
            'pageTitle'       => 'Domain Records',
            'metaDescription' => 'Review active domain records.',
            'metaKeywords'    => 'domain, record, index',
            'canonicalUrl'    => url_to('domain.record.index'),
            'robotsTag'       => 'noindex, follow',
            'metaImage'       => base_url('assets/images/og-default.jpg'),
            'schemaJson'      => null,
            'data'            => $this->domainService->getAllRecords(),
        ]);
    }

    /**
     * Displays the record creation form view.
     *
     * @return string|ResponseInterface
     *
     * @throws RouterException When the canonical route name cannot be resolved.
     */
    public function new(): string|ResponseInterface
    {
        return view('App\Modules\[ModuleName]\Views\record\new', [
            'pageTitle'       => 'Create Domain Record',
            'metaDescription' => 'Submit a new domain entry.',
            'metaKeywords'    => 'domain, record, new',
            'canonicalUrl'    => url_to('domain.record.new'),
            'robotsTag'       => 'noindex, follow',
            'metaImage'       => base_url('assets/images/og-default.jpg'),
            'schemaJson'      => null,
        ]);
    }

    /**
     * Validates and persists a new domain record via PRG.
     *
     * @return ResponseInterface
     *
     * @throws RouterException  When target redirect route resolution fails.
     * @throws RuntimeException When validated input keys or session state are malformed.
     */
    public function create(): ResponseInterface
    {
        $actorId = $this->resolveAuthenticatedActorId();
        if ($actorId === null) {
            return $this->response->setStatusCode(401, 'Unauthorized');
        }

        if (! $this->validate(['domain_id' => 'required|is_natural_no_zero', 'value' => 'required|decimal'])) {
            return redirect()->to(url_to('domain.record.new'))
                ->withInput()
                ->with('errors', $this->requireValidationErrors());
        }

        $valid = $this->requireValidatedInput();
        if (! is_string($valid['domain_id']) || ! ctype_digit($valid['domain_id']) || ! is_string($valid['value'])) {
            throw new RuntimeException('Validated input types do not match contract.');
        }

        $domainId = (int) $valid['domain_id'];
        try {
            $this->domainService->createRecord(new DomainEntity([
                'domain_id' => $domainId,
                'owner_id'  => $actorId,
                'value'     => $valid['value'],
                'is_active' => false,
                'metadata'  => ['source' => 'web_admin'],
            ]));

            return redirect()->to(url_to('domain.record.index'))
                ->with('success', 'Record created successfully.');
        } catch (Throwable $e) {
            log_message('error', 'DomainController::create failed: ' . $e->getMessage(), [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
                'trace'     => $e->getTraceAsString(),
            ]);

            return redirect()->to(url_to('domain.record.new'))
                ->withInput()
                ->with('error', 'Failed to save record. Please try again.');
        }
    }

    /**
     * Validates and processes an uploaded domain document via PRG.
     *
     * @return ResponseInterface
     *
     * @throws RouterException  When target redirect route resolution fails.
     * @throws RuntimeException When request type or validated input is malformed.
     */
    public function upload(): ResponseInterface
    {
        $actorId = $this->resolveAuthenticatedActorId();
        if ($actorId === null) {
            return $this->response->setStatusCode(401, 'Unauthorized');
        }

        if (! $this->request instanceof IncomingRequest) {
            throw new RuntimeException('Upload action requires IncomingRequest.');
        }

        $rules = [
            'domain_id' => 'required|is_natural_no_zero',
            'document'  => [
                'label' => 'Domain Document',
                'rules' => [
                    'uploaded[document]',
                    'max_size[document,5120]',
                    'mime_in[document,application/pdf,image/jpeg,image/png]',
                    'ext_in[document,pdf,jpg,jpeg,png]',
                ],
            ],
        ];
        if (! $this->validate($rules)) {
            return redirect()->to(url_to('domain.record.index'))
                ->withInput()
                ->with('errors', $this->requireValidationErrors());
        }

        $valid = $this->requireValidatedInput();
        if (! is_string($valid['domain_id']) || ! ctype_digit($valid['domain_id'])) {
            throw new RuntimeException('Validated domain_id does not match contract.');
        }
        $domainId = (int) $valid['domain_id'];

        $file = $this->request->getFile('document');
        if (! $file instanceof UploadedFile || ! $file->isValid() || $file->hasMoved()) {
            return redirect()->to(url_to('domain.record.index'))
                ->with('error', 'The uploaded file is invalid or could not be processed.');
        }

        try {
            $this->domainService->processUploadedDocument($file, $domainId, $actorId);

            return redirect()->to(url_to('domain.record.index'))
                ->with('success', 'File uploaded and processed successfully.');
        } catch (DomainAuthorizationException $e) {
            log_message('error', 'DomainController::upload unauthorized: ' . $e->getMessage(), [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
            ]);

            return $this->response->setStatusCode(403, 'Forbidden');
        } catch (DomainNotFoundException $e) {
            log_message('error', 'DomainController::upload not found: ' . $e->getMessage(), [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
            ]);

            return $this->response->setStatusCode(404, 'Not Found');
        } catch (Throwable $e) {
            log_message('error', 'DomainController::upload failed: ' . $e->getMessage(), [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
                'trace'     => $e->getTraceAsString(),
            ]);

            return redirect()->to(url_to('domain.record.index'))
                ->with('error', 'File processing failed. Please try again.');
        }
    }

    /**
     * Streams an authorized domain document from ephemeral staging.
     *
     * @param int $domainId Positive domain identifier from route parameter.
     *
     * @return ResponseInterface
     *
     * @throws RuntimeException When session actor identity is malformed.
     */
    public function download(int $domainId): ResponseInterface
    {
        $actorId = $this->resolveAuthenticatedActorId();
        if ($actorId === null) {
            return $this->response->setStatusCode(401, 'Unauthorized');
        }
        if ($domainId <= 0) {
            return $this->response->setStatusCode(400, 'Bad Request');
        }

        try {
            $dto      = $this->domainService->prepareAuthorizedDownload($domainId, $actorId);
            $download = $this->response->download($dto->tempPath, null, true);
            if (! $download instanceof DownloadResponse) {
                throw new RuntimeException('Failed to initialize DownloadResponse.');
            }

            return $download->setFileName($dto->downloadName);
        } catch (DomainNotFoundException $e) {
            log_message('error', 'DomainController::download not found: ' . $e->getMessage(), [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
            ]);

            return $this->response->setStatusCode(404, 'Not Found');
        } catch (DomainAuthorizationException $e) {
            log_message('error', 'DomainController::download forbidden: ' . $e->getMessage(), [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
            ]);

            return $this->response->setStatusCode(403, 'Forbidden');
        } catch (Throwable $e) {
            log_message('error', 'DomainController::download failed: ' . $e->getMessage(), [
                'domain_id' => $domainId,
                'actor_id'  => $actorId,
                'trace'     => $e->getTraceAsString(),
            ]);

            return $this->response->setStatusCode(500, 'Internal Server Error');
        }
    }
}
```

### 9. `app/Helpers/strict_view_helper.php`, `app/Views/layouts/default.php` & `app/Views/partials/flash_messages.php`

```php
<?php

declare(strict_types=1);

if (! function_exists('strict_string')) {
    /**
     * Asserts a value is a non-empty string.
     *
     * @param mixed  $value   Candidate scalar value.
     * @param string $context Diagnostic label on failure.
     *
     * @return string
     *
     * @throws \RuntimeException When the value is not a non-empty string.
     */
    function strict_string(mixed $value, string $context): string
    {
        if (! is_string($value) || $value === '') {
            throw new \RuntimeException('Expected non-empty string for ' . $context);
        }

        return $value;
    }
}

if (! function_exists('strict_bool')) {
    /**
     * Asserts a value is strictly boolean.
     *
     * @param mixed  $value   Candidate value.
     * @param string $context Diagnostic label on failure.
     *
     * @return bool
     *
     * @throws \RuntimeException When the value is not boolean.
     */
    function strict_bool(mixed $value, string $context): bool
    {
        if (! is_bool($value)) {
            throw new \RuntimeException('Expected boolean for ' . $context);
        }

        return $value;
    }
}

if (! function_exists('strict_lang')) {
    /**
     * Resolves a localized string via lang(), failing on missing or array keys.
     *
     * @param string $line Translation key.
     *
     * @return string
     *
     * @throws \RuntimeException When lang() does not return a non-empty string.
     */
    function strict_lang(string $line): string
    {
        return strict_string(lang($line), 'lang(' . $line . ')');
    }
}

if (! function_exists('strict_old_string')) {
    /**
     * Retrieves flashed scalar old() input, rendering empty string only when absent (null).
     *
     * @param string $key Form field name.
     *
     * @return string
     *
     * @throws \RuntimeException When old() contains a non-string payload.
     */
    function strict_old_string(string $key): string
    {
        $value = old($key);
        if ($value === null) {
            return '';
        }
        if (! is_string($value)) {
            throw new \RuntimeException('Expected string old input for field: ' . $key);
        }

        return $value;
    }
}

if (! function_exists('strict_flash_string')) {
    /**
     * Retrieves an optional session flash string, rejecting non-string payloads.
     *
     * @param string $key Session flash key.
     *
     * @return string|null
     *
     * @throws \RuntimeException When the flash payload exists but is not a string.
     */
    function strict_flash_string(string $key): ?string
    {
        if (! session()->has($key)) {
            return null;
        }
        $value = session($key);
        if (! is_string($value)) {
            throw new \RuntimeException('Expected string flash payload for key: ' . $key);
        }

        return $value;
    }
}

if (! function_exists('strict_flash_errors')) {
    /**
     * Retrieves session validation error messages as a strict string list.
     *
     * @param string $key Session flash errors key.
     *
     * @return array<int|string, string>
     *
     * @throws \RuntimeException When errors payload is not an array of strings.
     */
    function strict_flash_errors(string $key = 'errors'): array
    {
        if (! session()->has($key)) {
            return [];
        }
        $raw = session($key);
        if (! is_array($raw)) {
            throw new \RuntimeException('Expected array flash payload for key: ' . $key);
        }

        $validated = [];
        foreach ($raw as $field => $message) {
            if (! is_string($message)) {
                throw new \RuntimeException('Malformed flash error entry for field: ' . (string) $field);
            }
            $validated[$field] = $message;
        }

        return $validated;
    }
}

if (! function_exists('strict_field_error')) {
    /**
     * Retrieves a single field validation error from session flash data.
     *
     * @param string $field Field name.
     *
     * @return string|null
     *
     * @throws \RuntimeException When the stored error is not a string.
     */
    function strict_field_error(string $field): ?string
    {
        $value = session('errors.' . $field);
        if ($value === null) {
            return null;
        }
        if (! is_string($value)) {
            throw new \RuntimeException('Malformed validation error for field: ' . $field);
        }

        return $value;
    }
}

if (! function_exists('safe_url')) {
    /**
     * Validates that a URL uses http, https, or a single-slash relative path.
     *
     * @param string $url Target URL string.
     *
     * @return string
     *
     * @throws \InvalidArgumentException When an unsafe or unsupported URI scheme is supplied.
     */
    function safe_url(string $url): string
    {
        if ($url === '') {
            throw new \InvalidArgumentException('URL cannot be empty.');
        }
        if (str_starts_with($url, '/') && ! str_starts_with($url, '//')) {
            return $url;
        }

        $scheme = parse_url($url, PHP_URL_SCHEME);
        if (! is_string($scheme) || ! in_array(strtolower($scheme), ['http', 'https'], true)) {
            throw new \InvalidArgumentException('Disallowed URL scheme in: ' . $url);
        }

        return $url;
    }
}

if (! function_exists('strict_env')) {
    /**
     * Retrieves a required non-empty environment variable, throwing when unset or empty.
     *
     * @param string $key Environment variable name.
     *
     * @return string
     *
     * @throws \RuntimeException When the environment variable is missing or empty.
     */
    function strict_env(string $key): string
    {
        $value = env($key);
        if (! is_string($value) || $value === '') {
            throw new \RuntimeException('Required environment variable is missing or empty: ' . $key);
        }

        return $value;
    }
}
```

```php
<?php

declare(strict_types=1);

use CodeIgniter\View\View;

/**
 * @var View                      $this
 * @var string                    $pageTitle
 * @var string                    $metaDescription
 * @var string                    $metaKeywords
 * @var string                    $robotsTag
 * @var string                    $canonicalUrl
 * @var string                    $metaImage
 * @var array<string, mixed>|null $schemaJson
 */
$csrfTokenName = csrf_token();
if (! is_string($csrfTokenName) || $csrfTokenName === '') {
    throw new \RuntimeException('CSRF token name is unavailable.');
}
?>
<!DOCTYPE html>
<html lang="en" data-bs-theme="light">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="csrf-token" content="<?= esc(csrf_hash(), 'attr') ?>">
    <meta name="csrf-header" content="<?= esc(csrf_header(), 'attr') ?>">
    <title><?= esc($pageTitle, 'html') ?></title>
    <meta name="description" content="<?= esc($metaDescription, 'attr') ?>">
    <meta name="keywords" content="<?= esc($metaKeywords, 'attr') ?>">
    <meta name="robots" content="<?= esc($robotsTag, 'attr') ?>">
    <link rel="canonical" href="<?= esc(safe_url($canonicalUrl), 'attr') ?>">
    <meta property="og:type" content="website">
    <meta property="og:title" content="<?= esc($pageTitle, 'attr') ?>">
    <meta property="og:description" content="<?= esc($metaDescription, 'attr') ?>">
    <meta property="og:url" content="<?= esc(safe_url($canonicalUrl), 'attr') ?>">
    <meta property="og:image" content="<?= esc(safe_url($metaImage), 'attr') ?>">
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="<?= esc($pageTitle, 'attr') ?>">
    <meta name="twitter:description" content="<?= esc($metaDescription, 'attr') ?>">
    <meta name="twitter:image" content="<?= esc(safe_url($metaImage), 'attr') ?>">
    <link rel="stylesheet"
          href="<?= esc(safe_url(base_url('assets/vendor/bootstrap/css/bootstrap.min.css')), 'attr') ?>">
    <?php if ($schemaJson !== null) : ?>
        <script type="application/ld+json">
            <?= json_encode(
                $schemaJson,
                JSON_HEX_TAG | JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT | JSON_THROW_ON_ERROR
            ) ?>
        </script>
    <?php endif ?>
    <style <?= csp_style_nonce() ?>>
        :root,
        [data-bs-theme="light"],
        [data-bs-theme="dark"] {
            --theme-bg-canvas: var(--bs-body-bg);
            --theme-card-bg: var(--bs-card-bg);
            --theme-text-primary: var(--bs-body-color);
            --theme-border-color: var(--bs-border-color);
            --theme-accent: var(--bs-primary);
        }
        :focus-visible {
            outline: 2px solid var(--theme-accent) !important;
            outline-offset: 3px !important;
        }
    </style>
    <?= $this->renderSection('styles') ?>
</head>
<body class="bg-body-tertiary text-body min-vh-100 d-flex flex-column">
    <header class="navbar navbar-expand-lg bg-body border-bottom">
        <div class="container">
            <a class="navbar-brand fw-bold" href="<?= esc(safe_url(base_url('/')), 'attr') ?>">Application Portal</a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse"
                    data-bs-target="#navContent" aria-controls="navContent"
                    aria-expanded="false" aria-label="Toggle navigation">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navContent">
                <ul class="navbar-nav me-auto mb-2 mb-lg-0">
                    <?= $this->renderSection('nav_items') ?>
                </ul>
            </div>
        </div>
    </header>
    <main class="container my-4 flex-grow-1">
        <?= $this->include('partials/flash_messages') ?>
        <?= $this->renderSection('content') ?>
    </main>
    <footer class="bg-body border-top py-3 mt-auto">
        <div class="container text-center text-body-secondary small">
            &copy; <?= esc(date('Y'), 'html') ?> Modular Architecture.
        </div>
    </footer>
    <script src="<?= esc(safe_url(base_url('assets/vendor/bootstrap/js/bootstrap.bundle.min.js')), 'attr') ?>"></script>
    <script <?= csp_script_nonce() ?>>
        window.getCSRFToken = function () {
            const meta = document.querySelector('meta[name="csrf-token"]');
            if (!meta) {
                throw new Error('Missing meta[name="csrf-token"] element.');
            }
            const token = meta.getAttribute('content');
            if (typeof token !== 'string' || token.length === 0) {
                throw new Error('CSRF meta token content is empty or invalid.');
            }
            return token;
        };
        window.updateCSRFToken = function (token) {
            if (typeof token !== 'string' || token.length === 0) {
                throw new Error('Cannot update CSRF token with an empty or non-string value.');
            }
            const meta = document.querySelector('meta[name="csrf-token"]');
            if (!meta) {
                throw new Error('Missing meta[name="csrf-token"] element.');
            }
            meta.setAttribute('content', token);
            document.querySelectorAll('input[name="<?= esc($csrfTokenName, 'js') ?>"]').forEach(function (input) {
                input.value = token;
            });
        };
    </script>
    <?= $this->renderSection('scripts') ?>
</body>
</html>
```

```php
<?php

declare(strict_types=1);

use CodeIgniter\View\View;

/** @var View $this */
$successMessage = strict_flash_string('success');
$errorMessage   = strict_flash_string('error');
$errorList      = strict_flash_errors('errors');
?>
<?php if ($successMessage !== null) : ?>
    <div class="alert alert-success alert-dismissible fade show" role="alert">
        <?= esc($successMessage, 'html') ?>
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
    </div>
<?php endif ?>
<?php if ($errorMessage !== null) : ?>
    <div class="alert alert-danger alert-dismissible fade show" role="alert">
        <?= esc($errorMessage, 'html') ?>
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
    </div>
<?php endif ?>
<?php if ($errorList !== []) : ?>
    <div class="alert alert-danger alert-dismissible fade show" role="alert">
        <ul class="mb-0">
            <?php foreach ($errorList as $error) : ?>
                <li><?= esc($error, 'html') ?></li>
            <?php endforeach ?>
        </ul>
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
    </div>
<?php endif ?>
```

### 10. Module Views (`Views/record/{index,new}.php`) & Upload Form Snippet

```php
<?php

declare(strict_types=1);

use App\Modules\[ModuleName]\Entities\DomainEntity;
use CodeIgniter\View\View;

/**
 * @var View                    $this
 * @var string                  $pageTitle
 * @var string                  $metaDescription
 * @var string                  $metaKeywords
 * @var string                  $robotsTag
 * @var string                  $canonicalUrl
 * @var string                  $metaImage
 * @var array<string, mixed>|null $schemaJson
 * @var array<int, DomainEntity>  $data
 */
?>
<?= $this->extend('layouts/default') ?>
<?= $this->section('nav_items') ?>
    <li class="nav-item">
        <a class="nav-link" href="<?= esc(safe_url(url_to('domain.record.index')), 'attr') ?>">Records</a>
    </li>
<?= $this->endSection() ?>
<?= $this->section('content') ?>
<div class="vstack gap-4">
    <div class="d-flex justify-content-between align-items-center">
        <h1 class="h3 mb-0">Domain Records</h1>
        <a href="<?= esc(safe_url(url_to('domain.record.new')), 'attr') ?>" class="btn btn-primary">Create Record</a>
    </div>
    <div class="card shadow-sm border">
        <div class="card-body p-0">
            <?php if ($data === []) : ?>
                <div class="p-4 text-center text-body-secondary">No domain records found.</div>
            <?php else : ?>
                <div class="table-responsive">
                    <table class="table table-hover table-striped align-middle mb-0">
                        <thead class="table-light">
                            <tr>
                                <th scope="col" class="ps-3">ID</th>
                                <th scope="col">Domain Ref</th>
                                <th scope="col">Value</th>
                                <th scope="col">Status</th>
                                <th scope="col" class="pe-3 text-end">Created</th>
                            </tr>
                        </thead>
                        <tbody>
                            <?php foreach ($data as $record) : ?>
                                <?php
                                if ($record->created_at === null) {
                                    throw new \RuntimeException('Persisted DomainEntity missing created_at.');
                                }
                                ?>
                                <tr>
                                    <td class="ps-3"><?= esc((string) $record->id, 'html') ?></td>
                                    <td><?= esc((string) $record->domain_id, 'html') ?></td>
                                    <td>$<?= esc($record->value, 'html') ?></td>
                                    <td>
                                        <?php if ($record->is_active) : ?>
                                            <span class="badge bg-success-subtle text-success-emphasis
                                                         border border-success-subtle">Active</span>
                                        <?php else : ?>
                                            <span class="badge bg-warning-subtle text-warning-emphasis
                                                         border border-warning-subtle">Inactive</span>
                                        <?php endif ?>
                                    </td>
                                    <td class="pe-3 text-end text-body-secondary small">
                                        <?= esc($record->created_at->toLocalizedString('yyyy-MM-dd HH:mm'), 'html') ?>
                                    </td>
                                </tr>
                            <?php endforeach ?>
                        </tbody>
                    </table>
                </div>
            <?php endif ?>
        </div>
    </div>
</div>
<?= $this->endSection() ?>
```

```php
<?php

declare(strict_types=1);

use CodeIgniter\View\View;

/**
 * @var View                      $this
 * @var string                    $pageTitle
 * @var string                    $metaDescription
 * @var string                    $metaKeywords
 * @var string                    $robotsTag
 * @var string                    $canonicalUrl
 * @var string                    $metaImage
 * @var array<string, mixed>|null $schemaJson
 */
$domainIdError = strict_field_error('domain_id');
$valueError    = strict_field_error('value');
?>
<?= $this->extend('layouts/default') ?>
<?= $this->section('nav_items') ?>
    <li class="nav-item">
        <a class="nav-link" href="<?= esc(safe_url(url_to('domain.record.index')), 'attr') ?>">Records</a>
    </li>
<?= $this->endSection() ?>
<?= $this->section('content') ?>
<div class="row justify-content-center">
    <div class="col-12 col-md-8 col-lg-6">
        <div class="card shadow-sm border">
            <div class="card-header bg-body py-3">
                <h1 class="h4 card-title mb-0">Create Domain Record</h1>
            </div>
            <div class="card-body p-4">
                <?= form_open(safe_url(url_to('domain.record.create'))) ?>
                <div class="form-floating mb-3">
                    <input type="number" name="domain_id" id="domainId" placeholder="1029"
                           class="form-control <?= $domainIdError !== null ? 'is-invalid' : '' ?>"
                           value="<?= esc(strict_old_string('domain_id'), 'attr') ?>" required>
                    <label for="domainId">Domain ID</label>
                    <?php if ($domainIdError !== null) : ?>
                        <div class="invalid-feedback"><?= esc($domainIdError, 'html') ?></div>
                    <?php endif ?>
                </div>
                <div class="form-floating mb-4">
                    <input type="number" name="value" step="0.01" id="recordValue" placeholder="100.00"
                           class="form-control <?= $valueError !== null ? 'is-invalid' : '' ?>"
                           value="<?= esc(strict_old_string('value'), 'attr') ?>" required>
                    <label for="recordValue">Monetary Value</label>
                    <?php if ($valueError !== null) : ?>
                        <div class="invalid-feedback"><?= esc($valueError, 'html') ?></div>
                    <?php endif ?>
                </div>
                <div class="hstack gap-2 justify-content-end">
                    <a href="<?= esc(safe_url(url_to('domain.record.index')), 'attr') ?>"
                       class="btn btn-outline-secondary">Cancel</a>
                    <button type="submit" class="btn btn-primary">Save Record</button>
                </div>
                <?= form_close() ?>
            </div>
        </div>
    </div>
</div>
<?= $this->endSection() ?>
```

```php
<?php

declare(strict_types=1);

use CodeIgniter\View\View;

/** @var View $this */
$uploadDomainError = strict_field_error('domain_id');
$documentError     = strict_field_error('document');
?>
<?= form_open_multipart(safe_url(url_to('domain.file.upload'))) ?>
    <div class="mb-3">
        <label for="domainIdInput" class="form-label">Domain ID</label>
        <input class="form-control <?= $uploadDomainError !== null ? 'is-invalid' : '' ?>"
               type="number" id="domainIdInput" name="domain_id"
               value="<?= esc(strict_old_string('domain_id'), 'attr') ?>" required>
        <?php if ($uploadDomainError !== null) : ?>
            <div class="invalid-feedback"><?= esc($uploadDomainError, 'html') ?></div>
        <?php endif ?>
    </div>
    <div class="mb-3">
        <label for="domainDocument" class="form-label">Upload Document</label>
        <input class="form-control <?= $documentError !== null ? 'is-invalid' : '' ?>"
               type="file" id="domainDocument" name="document" required>
        <?php if ($documentError !== null) : ?>
            <div class="invalid-feedback"><?= esc($documentError, 'html') ?></div>
        <?php endif ?>
    </div>
    <button type="submit" class="btn btn-primary">Submit Upload</button>
<?= form_close() ?>
```

### 11. `app/Commands/DomainSyncCommand.php`

```php
<?php

declare(strict_types=1);

namespace App\Commands;

use App\Modules\[ModuleName]\Libraries\DomainService;
use CodeIgniter\CLI\BaseCommand;
use CodeIgniter\CLI\CLI;
use RuntimeException;
use Throwable;

/**
 * Synchronizes domain catalog records via Spark CLI.
 *
 * @package App\Commands
 * @author  Senior Developer
 * @since   1.0.0
 */
class DomainSyncCommand extends BaseCommand
{
    /**
     * Spark command group name.
     *
     * @var string
     */
    protected $group = 'Domain';

    /**
     * Spark command invocation name.
     *
     * @var string
     */
    protected $name = 'domain:sync';

    /**
     * Command summary description.
     *
     * @var string
     */
    protected $description = 'Synchronizes domain catalog records.';

    /**
     * Command CLI usage pattern.
     *
     * @var string
     */
    protected $usage = 'domain:sync [options]';

    /**
     * Positional CLI arguments.
     *
     * @var array<string, string>
     */
    protected $arguments = [];

    /**
     * Supported CLI options.
     *
     * @var array<string, string>
     */
    protected $options = [
        '--dry-run' => 'Simulates synchronization without database modification.',
    ];

    /**
     * Executes the domain record synchronization workflow.
     *
     * @param array<int|string, mixed> $params Parsed CLI arguments and options.
     *
     * @return int
     */
    public function run(array $params): int
    {
        $isDryRun = array_key_exists('dry-run', $params) || CLI::getOption('dry-run') !== null;
        CLI::write('Executing domain sync...', 'yellow');

        try {
            $service = service('domainService');
            if (! $service instanceof DomainService) {
                throw new RuntimeException('Failed to resolve DomainService.');
            }

            $processed = $service->syncRecords($isDryRun);
            CLI::write('Domain sync finished. Processed: ' . $processed, 'green');

            return \EXIT_SUCCESS;
        } catch (Throwable $e) {
            log_message('error', 'DomainSyncCommand failed: ' . $e->getMessage(), [
                'trace' => $e->getTraceAsString(),
            ]);
            CLI::error('Domain sync failed.');

            return \EXIT_ERROR;
        }
    }
}
```

### 12. `tests/_support/Fakers/DomainFabricatorModel.php` & `tests/Controllers/DomainControllerTest.php`

```php
<?php

declare(strict_types=1);

namespace Tests\Support\Fakers;

use App\Modules\[ModuleName]\Models\DomainModel;
use Faker\Generator;

/**
 * Test-only Fabricator model providing deterministic DomainEntity fixtures.
 *
 * @package Tests\Support\Fakers
 * @author  Senior Developer
 * @since   1.0.0
 */
class DomainFabricatorModel extends DomainModel
{
    /**
     * Generates synthetic row data for Fabricator tests.
     *
     * @param Generator $faker Faker generator instance.
     *
     * @return array<string, int|string>
     */
    public function fake(Generator &$faker): array
    {
        return [
            'domain_id' => 1,
            'owner_id'  => 10,
            'value'     => number_format($faker->randomFloat(2, 10, 5000), 2, '.', ''),
            'is_active' => 0,
        ];
    }
}
```

```php
<?php

declare(strict_types=1);

namespace Tests\Controllers;

use App\Modules\[ModuleName]\Models\DomainsModel;
use CodeIgniter\I18n\Time;
use CodeIgniter\Router\RouteCollection;
use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\DatabaseTestTrait;
use CodeIgniter\Test\Fabricator;
use CodeIgniter\Test\FeatureTestTrait;
use Config\Services;
use RuntimeException;
use Tests\Support\Fakers\DomainFabricatorModel;

/**
 * Integration tests covering DomainController, filters, and persistence.
 *
 * @package Tests\Controllers
 * @author  Senior Developer
 * @since   1.0.0
 */
class DomainControllerTest extends CIUnitTestCase
{
    use DatabaseTestTrait;
    use FeatureTestTrait;

    /**
     * Enables automatic test database migrations.
     *
     * @var bool
     */
    protected $migrate = true;

    /**
     * Restricts test migrations to the target module namespace.
     *
     * @var string|null
     */
    protected $namespace = 'App\Modules\[ModuleName]';

    /**
     * Configures frozen test clock and module routes.
     *
     * @return void
     *
     * @throws RuntimeException When route service is unavailable.
     */
    protected function setUp(): void
    {
        parent::setUp();
        Time::setTestNow('2026-11-13 10:00:00');

        $routes = service('routes');
        if (! $routes instanceof RouteCollection) {
            throw new RuntimeException('RouteCollection service unavailable.');
        }

        $routes->group(
            'domain',
            ['namespace' => 'App\Modules\[ModuleName]\Controllers'],
            static function (RouteCollection $routes): void {
                $routes->get('record', 'DomainController::index', ['as' => 'domain.record.index']);
                $routes->get('record/new', 'DomainController::new', ['as' => 'domain.record.new']);
                $routes->post('record', 'DomainController::create', ['as' => 'domain.record.create']);
                $routes->get(
                    'file/download/(:num)',
                    'DomainController::download/$1',
                    ['as' => 'domain.file.download']
                );
            }
        );
    }

    /**
     * Resets frozen clock and mocked services.
     *
     * @return void
     */
    protected function tearDown(): void
    {
        Time::setTestNow();
        Services::reset();
        parent::tearDown();
    }

    /**
     * Verifies index route renders the empty state cleanly.
     *
     * @return void
     */
    public function testIndexViewRendersSuccessfully(): void
    {
        $response = $this->withSession(['user_id' => 10])->get(url_to('domain.record.index'));
        $response->assertStatus(200);
        $response->assertSee('Domain Records', 'h1');
        $response->assertSee('No domain records found.');
    }

    /**
     * Verifies valid record creation persists data and redirects deterministically.
     *
     * @return void
     *
     * @throws RuntimeException When DomainsModel or fixture insertion fails.
     */
    public function testCreateSavesToDatabaseAndRedirects(): void
    {
        $domainsModel = model(DomainsModel::class);
        if (! $domainsModel instanceof DomainsModel) {
            throw new RuntimeException('Failed to resolve DomainsModel.');
        }

        $domainId = $domainsModel->insert(['slug' => 'core-domain'], true);
        if (! is_int($domainId)) {
            throw new RuntimeException('Failed to insert parent domain fixture.');
        }

        $response = $this->withSession(['user_id' => 10])->post(url_to('domain.record.create'), [
            'domain_id'    => (string) $domainId,
            'value'        => '520.45',
            csrf_token()   => csrf_hash(),
        ]);

        $response->assertRedirectTo(url_to('domain.record.index'));
        $response->assertSessionHas('success');
        $this->seeInDatabase('domain_table', [
            'domain_id' => $domainId,
            'owner_id'  => 10,
            'value'     => '520.45',
            'is_active' => 0,
        ]);
    }

    /**
     * Verifies missing input fails validation and redirects to the creation form.
     *
     * @return void
     */
    public function testCreateFailsValidationOnMissingInput(): void
    {
        $response = $this->withSession(['user_id' => 10])->post(url_to('domain.record.create'), [
            csrf_token() => csrf_hash(),
        ]);

        $response->assertRedirectTo(url_to('domain.record.new'));
        $response->assertSessionHas('errors');
        $this->seeNumRecords(0, 'domain_table');
    }

    /**
     * Verifies unauthorized download fails deterministically with HTTP 403.
     *
     * @return void
     *
     * @throws RuntimeException When DomainsModel or fixture insertion fails.
     */
    public function testDownloadDeniesUnauthorizedActorWith403(): void
    {
        $domainsModel = model(DomainsModel::class);
        if (! $domainsModel instanceof DomainsModel) {
            throw new RuntimeException('Failed to resolve DomainsModel.');
        }

        $domainId = $domainsModel->insert(['slug' => 'restricted-domain'], true);
        if (! is_int($domainId)) {
            throw new RuntimeException('Failed to insert parent domain fixture.');
        }

        $fabricator = new Fabricator(DomainFabricatorModel::class);
        $fabricator->setOverrides(['domain_id' => $domainId, 'owner_id' => 10, 'is_active' => 1]);
        $fabricator->create();

        $response = $this->withSession(['user_id' => 99])->get(url_to('domain.file.download', $domainId));
        $response->assertStatus(403);
    }
}
```
