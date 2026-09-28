# Backend spec layer audit — `.trellis/spec/backend/`

Verified against `backend/` at commit `1cd12c72` (2026-09-17) and `/home/zzhou/coding/pi-platform/SKILL.md`.

Note: `.trellis/spec/` is **untracked in git**, so the "published 2026-08-20" date is a filesystem timestamp only — there is no committed history to diff against.

## 1. STALE / WRONG claims

### Root cause / highest impact: the whole directory model predates the `modules/` refactor

`directory-structure.md` and `index.md` describe an `app/api/v1/module_<area>/<feature>/` layout. The actual layout is `app/modules/<area>/<feature>/` with a single explicit router table.

| # | Spec file | Claim | Reality (evidence) |
|---|---|---|---|
| 1 | `directory-structure.md:8-9` | "`backend/app/init_app.py` owns lifespan setup and the registration order for exceptions, middleware, routers, static assets, docs, and the optional built web application." | **No `app/init_app.py` exists.** App factory is `backend/app/__init__.py` — `create_app()` + `register_exceptions/register_middlewares/register_routers/register_static/register_docs/register_frontend`, and `lifespan()`. |
| 2 | `directory-structure.md:12-24` (tree) | `app/api/v1/` = "first-party API modules"; no `modules/` node in the tree | `backend/app/api/v1/` contains only `__init__.py` + `routers.py` (2655 bytes). First-party slices are under `backend/app/modules/{ai,common,generator,monitor,system,task}/`. Tree also omits `templates/`, `sql/`, `dist/`, `logs/`, `data/`. |
| 3 | `directory-structure.md:18` | `core/` = "DB/session, auth, CRUD, middleware, logging" | Understates: also `redis_crud.py`, `ap_scheduler.py`, `sse_manager.py`, `permission.py`, `base_schema.py`, `validator.py`, `security.py`, `discover.py`. |
| 4 | `directory-structure.md:28-30` | Slices "below the `backend/app/api/v1/` tree in a `module_<area>/<feature>` slice" | Slices are `backend/app/modules/<area>/<feature>/`; `module_` prefix is used only by **plugins**. |
| 5 | `directory-structure.md:42` | "`backend/app/api/v1/module_system/dept/` demonstrates the complete flow." | Actual: `backend/app/modules/system/dept/{controller,service,crud,model,schema}.py`. Same wrong path in `index.md:18-20`, `error-handling.md:21`, `error-handling.md:75`, `database-guidelines.md:33`, `database-guidelines.md:61`. |
| 6 | `directory-structure.md:47-50` | "Area routers are assembled explicitly in each `module_*/__init__.py`, then in `backend/app/init_app.py`… `module_system/__init__.py` mounts `DeptRouter` below `/system`" | `backend/app/modules/system/__init__.py` is a docstring only. Registration is a **single explicit table** `DOMAIN_CONTROLLERS` in `backend/app/api/v1/routers.py`, included by `app/__init__.py:register_routers`. |
| 7 | `directory-structure.md:89` | Avoid "manual imports of plugin routers in `init_app.py`" | Wrong filename (see #1); plugin routers are auto-discovered, first-party routers are manually imported in `api/v1/routers.py`. |
| 8 | `directory-structure.md:93-99` (Reference files) | Lists `backend/app/init_app.py`, `backend/app/api/v1/module_system/__init__.py`, `.../dept/*.py` | 4 of 7 listed paths do not exist. Valid: `main.py`, `app/core/discover.py`, `app/plugin/module_example/demo/controller.py`. |
| 9 | `database-guidelines.md:33-34` | "storage tables below `backend/app/api/v1/module_storage/`" | No such directory. Storage lives at `backend/app/modules/task/storage/`. |
| 10 | `database-guidelines.md:20-21` | "Explicit commits exist in long-running/background flows such as storage transfer **and chat**" | Storage transfer: confirmed (`app/modules/task/storage/transfer/engine.py:169,243,254,299,316,328,340,377`). Chat: **no `commit()` anywhere under `backend/app/modules/ai/`**. Also unlisted: `app/modules/system/log/service.py:151`. Half-wrong. |
| 11 | `database-guidelines.md:87-88` | "`scripts/initialize.py` creates tables and seed data on first startup. Schema changes intended for established installations use Alembic." | Incomplete/outdated. `app/scripts/initialize.py` **always** runs `command.upgrade(alembic_cfg, "head")` on every startup; on an empty DB it `create_tables()` + `stamp head`; in `ENVIRONMENT == DEV` it auto-generates a migration, applies it, and **deletes + aborts any revision containing destructive `op.drop_*`/`DROP`**; it also self-heals zombie `alembic_version` rows. The spec's framing ("Alembic is only for established installations") is wrong. |
| 12 | `database-guidelines.md:45-58` | Claims about `base_model.py`, `schema.py`, CRUD, `validator.py`, `search_to_dict()` | **Accurate** (verified `dept/schema.py`, `dept/service.py:47`, `core/validator.py`, `core/base_model.py`). No edit needed beyond the wrong path in #5. |
| 13 | `logging-guidelines.md:58-66` | "the operation route captures non-file form/JSON requests and JSON responses **without field redaction**. Existing login/refresh/logout, AI model configuration, and system/AI chat write routes can therefore persist passwords, returned JWTs, refresh tokens, API keys, or message content in `sys_operation_log`. Fix or explicitly exclude/redact these paths…" | **Directly contradicted by current code.** `backend/app/core/router_class.py` defines `_SENSITIVE_KEYS` (password, token, access_token, refresh_token, api_key, secret, authorization, …) and `_redact_sensitive()`, applied to the JSON body (`payload` → `_redact_sensitive(json.loads(...))`), form fields, and the **JSON response body** (`response_data = ..._redact_sensitive(json.loads(...))`), replaced with `******`. The header/code comments explicitly cite token leakage on login/refresh. The credential-leak claim is fixed; only the free-text message-content caveat may survive. |
| 14 | `quality-guidelines.md:62-64` | "`assert_route` … is useful for broad route coverage in `test_api_module_system.py`" | `backend/tests/test_api_module_system.py` **does not exist**. `backend/tests/` = `conftest.py`, `test_main.py`, `test_migrations.py`. `assert_route` has **zero call sites** (grep: only defined at `tests/conftest.py:240`). The spec also never mentions `test_migrations.py`, the create_all-vs-metadata drift guard. (Same stale claim in `.trellis/spec/testing/backend-testing.md:17`.) |
| 15 | `quality-guidelines.md:51-53` | "Prefer … over code copied from generated templates under `backend/templates/`" | Path is valid (`backend/templates/{python,ts,vue}/*.jinja2`), but omits the generator module `app/modules/generator/gencode/` and the mandatory restart behavior (see §2). |

Claims that are **correct and need no edit** (sampled): `main.py` Typer commands `run/revision/upgrade`; `db_getter` transaction boundary + flush-not-commit contract (`core/base_crud.py:34`); `ModelMixin`/`UserMixin` semantics and the `__table_args__`-replaces-mixin-indexes subtlety; CRUD `get_or_404/exists/count/include_deleted/preload` joinedload-vs-selectinload/auto `created_by,updated_by`; `Permission` data-scope condition (`base_crud.py:321-325`); the full `handle_exception()` mapping table including the 400-not-503 SQLAlchemy branch and the `IntegrityError` "connect"/"connection" special case (`core/exceptions.py`, note it actually returns HTTP **403** with a 503 constant); Loguru setup (rotation `00:00`, retention 30, audit log at `backend/logs/fastapiadmin.log`, APScheduler forced to WARNING); Ruff config (line-length 200, indent 4, double quotes, `FAST/F/E/W/I/B/C4/UP`, `fix=true`); CONTRIBUTING `feature/`/`bugfix/` + PR-to-`dev`; test fixtures (temp SQLite, mocked Redis, reduced lifespan, shared `TestClient`, `auth_headers`).

## 2. MISSING conventions (present in SKILL.md, absent from backend spec)

| Convention | SKILL.md source | Suggested target spec file |
|---|---|---|
| **`app/modules/<模块>/<功能>/` is the normal business placement; `app/plugin/` is for pluggable/example modules — same structure, both have `plugin.toml`** | §3 (L86-88, L95), §1 | `directory-structure.md` (rewrite "Business slices"/"Plugins") |
| **`app/api/v1/routers.py` is the single route registration point** | §3 (L89), §5 | `directory-structure.md` ("Routers" section) |
| **Permission identifier format `模块:资源:操作`, used as `Security(AuthPermission(["module_system:dept:query"]))`;** code only ever uses `module_<area>:<res>:<action>` (grep: `module_ai:chat:*`, `module_system:dept:*`, `module_monitor:dashboard:query`, …); **no `sys:*` strings exist in the backend** | §3 (L90), §4 (L103), §5 (L120) | `directory-structure.md` or a new auth/permission section |
| **`plugin.toml` lives at the module root (name/title/version/description); present on first-party modules too** (`app/modules/system/plugin.toml`, `app/modules/{ai,common,generator,monitor,task}/plugin.toml`, `app/plugin/module_example/plugin.toml`) | §3 (L88, L95) | `directory-structure.md` |
| **`app/modules/generator/gencode/` + `backend/templates/{python,ts,vue}/*.jinja2` code generator; after generating/modifying code you MUST restart the backend — `discover.py` only runs at startup, dev reload misses new modules and fails silently** | §3 (L96), §5 (L116) | `directory-structure.md` (plugins) + `quality-guidelines.md`; there is also a sibling `.trellis/spec/architecture/code-generation-and-routing.md` that should own the detail |
| **`scripts/initialize.py` auto-runs migrations on startup and imports `backend/sql/*.json` seed data (menu/role/user/dept/dict/param); dev auto-generates + applies migrations, destructive DROPs are guarded** | §1 (L29), §2 (L76) | `database-guidelines.md` ("Migrations and initialization") |
| **Jinja2 pitfall: `{% for %}` + `{% set %}` accumulator flags leak scope — use a filter expression instead** (`templates/python/schema.py.jinja2` already uses `columns \| selectattr('python_type','equalto','date') \| list \| length > 0`) | §8 (L140) | `quality-guidelines.md` or code-generation spec |
| **`register_frontend` check and mount must use the same `path_conf.FRONTEND_DIST_DIR` (`backend/dist`); mismatch only explodes on first request, visible only in `backend/logs/fastapiadmin.log`** | §8 (L139) | `directory-structure.md` / operations spec |
| **`core/` also houses `ap_scheduler` (scheduler) and `sse_manager`** | §1 (L28) | `directory-structure.md` |
| **`uv run main.py run --env=dev` startup command** (spec only documents `revision`/`upgrade`) | §2 (L54) | `quality-guidelines.md` / `index.md` |
| **Operation-log audit redaction behavior (`_SENSITIVE_KEYS` / `_redact_sensitive`, request + response)** — spec should document the *current* behavior, not the old gap | §8-adjacent (security/audit specs) | `logging-guidelines.md` ("Sensitive data") |
| **`test_migrations.py` create_all-vs-metadata drift guard** | not in SKILL, but it is the only other test | `quality-guidelines.md` ("Tests") |

Also missing from the spec (frontend-adjacent but cross-layer): WebSocket handshake auth via `Sec-WebSocket-Protocol` (`core/dependencies.py:websocket_authenticate`, `WS_TOKEN_SUBPROTOCOL`), and the multi-copy `dist/` deployment artifact whose backend copy is `backend/dist` served by `register_frontend`.

## 3. CONTRADICTIONS

| Topic | Spec says | SKILL.md says | Code says |
|---|---|---|---|
| Slice placement | `app/api/v1/module_<area>/<feature>/` (`directory-structure.md:28-30`) | `app/modules/<模块>/<功能>/` (§3 L86-87) | `backend/app/modules/system/dept/` → **SKILL wins** |
| Route registration | per-`module_*/__init__.py` + `app/init_app.py` (`directory-structure.md:47-50`) | `app/api/v1/routers.py` (§3 L89) | `backend/app/api/v1/routers.py` `DOMAIN_CONTROLLERS` → **SKILL wins** |
| App factory file | `app/init_app.py` (`directory-structure.md:8`, `logging-guidelines.md:23`) | `app/__init__.py` (§8 L139) | `backend/app/__init__.py:create_app` → **SKILL wins** |
| Permission literal | (silent) | §3 `module_ai:chat:query` (matches code) vs §4 `sys:user:add` (does **not** match code) | `module_system:dept:query`, `module_ai:chat:query` → SKILL §3 wins, **SKILL §4's `sys:user:add` example is itself stale** |
| Broad route test | `tests/test_api_module_system.py` (`quality-guidelines.md:63`) | not mentioned | file absent → **spec wrong** (and `testing/backend-testing.md` repeats it) |
| Operation-log redaction | "without field redaction … can persist passwords/JWTs/API keys" (`logging-guidelines.md:60-64`) | no explicit claim | `core/router_class.py` redacts request + response → **spec wrong** |
| `plugin.toml` | "optional metadata" for plugins only (`directory-structure.md:74`) | module-root file for both `modules/` and `plugin/` (§3 L88, L95) | plugin.toml under every area + the plugin → **SKILL more complete** |

Note: the sibling `.trellis/spec/architecture/system-boundaries.md` (L26, 31, 48, 55, 62, 83) repeats the **same stale** `app/api/v1/module_*` + `app/init_app.py` model, so the staleness is not isolated to `backend/` — the architecture, testing, and security specs share it.

## 4. Overall assessment

**Roughly 45-55% of concrete factual claims require edits**, unevenly distributed:

- `directory-structure.md` — **~75-80% stale**. Every path/registration/plugin-placement claim needs a rewrite; effectively the wrong architecture.
- `database-guidelines.md` — **~40% stale**. Mostly wrong paths (2), the `chat` commit claim, and the initialize/migration framing; the model/schema/CRUD/query-operator guidance is accurate and high quality.
- `error-handling.md` — **~25% stale**. Only the 3 dept paths are wrong; the exception-mapping table and transaction guidance match `core/exceptions.py` exactly.
- `logging-guidelines.md` — **~15-20% stale**, but the stale part is the load-bearing "Sensitive data" section, which now asserts the opposite of reality and would actively mislead a developer into a needless "fix".
- `quality-guidelines.md` — **~15-20% stale**. Toolchain/ruff/CONTRIBUTING/test-fixture claims are all correct; the nonexistent `test_api_module_system.py` reference and the missing restart/Jinja2/migration-guard conventions are the gaps.
- `index.md` — **~30% stale** (3 referenced paths, 2 of them the wrong reference slice).

Recommendation ranking: (1) rewrite `directory-structure.md` against `app/modules/` + `api/v1/routers.py` + `app/__init__.py`; (2) fix the `logging-guidelines.md` redaction claim; (3) update `database-guidelines.md` initialization/migration section to document startup auto-migration and `sql/*.json`; (4) fix `test_api_module_system.py` + add `test_migrations.py`, restart requirement, Jinja2 scope pitfall, permission-string format, and code-generator locations.
