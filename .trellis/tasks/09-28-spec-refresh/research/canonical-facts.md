# Canonical facts — current code state @ HEAD 1cd12c72 (2026-09-17)

Use this sheet as the single terminology/path source when rewriting any spec layer, so all layers stay consistent. Everything here was verified against code by the audits (see sibling reports for line-level evidence).

## Backend structure & routing
- App factory + lifespan + registration: `backend/app/__init__.py` (`create_app()`, `lifespan()`, `register_exceptions/register_middlewares/register_routers/register_static/register_docs/register_frontend`). **`backend/app/init_app.py` does not exist.**
- First-party business code: `backend/app/modules/{ai,common,generator,monitor,system,task}/<feature>/{controller,service,crud,model,schema}.py`. Canonical example slice: `backend/app/modules/system/dept/`.
- Plugins (dynamically discovered): `backend/app/plugin/module_example/{plugin.toml,demo/controller.py}`. `plugin.toml` lives at the **area/package** level, present for first-party areas too (`app/modules/system/plugin.toml` etc.), ignored by discovery at runtime.
- Route registration: **single explicit table** `DOMAIN_CONTROLLERS` in `backend/app/api/v1/routers.py` ("唯一路由事实来源"), domain prefixes `/system`, `/monitor`, `/task`, `/ai`, `/generator`, `/common`. `backend/app/api/` contains only `__init__.py` + `v1/{__init__.py,routers.py}`.
- Plugin discovery `backend/app/core/discover.py` runs **only at startup** and is **fail-fast**: unimportable plugin/controller raises `RuntimeError` and aborts startup; `check_route_conflicts()` halts on duplicate routes. Consequence: after codegen or adding a module you **must restart the backend** (dev reload does not re-run discovery; new modules silently 404 otherwise).
- `backend/app/core/` inventory highlights: `database.py`, `base_crud.py`, `base_model.py`, `base_schema.py`, `redis_crud.py`, `security.py`, `permission.py`, `dependencies.py`, `exceptions.py`, `router_class.py` (`OperationLogRoute`), `ap_scheduler.py`, `sse_manager.py` (renamed from `ws_manager.py`), `middlewares.py`, `logger.py`, `validator.py`, `discover.py`.
- Storage module: `backend/app/modules/task/storage/{browse,node,transfer,workflow,core}`; file module: `backend/app/modules/common/file/`; health: `backend/app/modules/monitor/health/controller.py`.
- Encryption helper: `backend/app/utils/crypto_util.py`.

## HTTP / realtime contracts
- Response envelope: `ResponseSchema[T]` (5 fields), `SuccessResponse`/`ErrorResponse` in `backend/app/common/response.py`; pagination `items/total/page_no/page_size/has_next` (web ambient type); **App (uni-app) ambient type still uses `list` — known mismatch, record as-is**.
- AI chat WebSocket: `/api/v1/ai/chat/ws`; **Web client uses `Sec-WebSocket-Protocol` carrier** `["access_token", "access_token."+jwt]` (`views/module_ai/chat/index.vue:103`); App client still `?token=` query. Handshake auth `core/dependencies.py:websocket_authenticate`; close codes 4001 (auth) and **4003 (permission)**.
- Storage transfer realtime is **SSE**: `GET /api/v1/task/storage/transfer/stream` (`EventSourceResponse`); frontend consumer `api/module_storage/transfer.ts` via `createSSEClient` (`@utils/sse`).
- Health: `GET /api/v1/monitor/health/check` and SSE `GET /api/v1/monitor/health/stream`. **No `/live`, no `/ready`, no `/common/health/*`, no `/system/chat/ws` anymore.**
- `VITE_APP_WS_ENDPOINT` is used by AI WS **and** the SSE consumers (`api/module_monitor/dashboard.ts`, `api/module_storage/transfer.ts`).
- Permission identifiers: `模块:资源:操作` with `module_` prefix in backend code (`module_system:dept:query`, `module_ai:chat:*`, `module_monitor:dashboard:query`); used via `Security(AuthPermission([...]))`. (SKILL.md §4's `sys:user:add` example is itself stale — do not copy it.)

## Auth / security current state
- Password hashing: **PBKDF2-HMAC-SHA256, 600k iterations** (`backend/app/utils/password_util.py`). NOT bcrypt. `check_password_strength` exists but unused.
- Refresh tokens: rotation **with replay detection** — mismatch vs stored refresh revokes the session (`auth/service.py:449-462`).
- Sessions: sliding activity renews `USER_SESSION` + refresh key, bounded by `SESSION_MAX_LIFETIME_SECONDS` absolute cap (`dependencies.py:124-156`).
- Login rate limiting: Redis `login_rate_limit:{ip}` with `LOGIN_RATE_LIMIT_WINDOW_SECONDS` / `LOGIN_RATE_LIMIT_MAX_ATTEMPTS` (`auth/service.py:155-168`); nginx applies `limit_req zone=api_limit burst=60 nodelay` + `limit_conn conn_limit 100` on `/api/v1`. Global `core/rate_limiter.py` / `fastapi-limiter` was added then **removed** — do not describe it as existing.
- Forget-password endpoint `POST /system/user/password/forget` is unauthenticated but **no longer sets a password** (logs + anti-enumeration response only).
- Operation log: `OperationLogRoute` redacts request JSON body, form fields, **and response JSON body** via `_SENSITIVE_KEYS`/`_redact_sensitive` (`core/router_class.py:15-41,90,97,112`) → `******`.
- File download confinement: `Path.resolve().is_relative_to(upload_root)` (`modules/common/file/service.py:52-56`) — the old `startswith` weakness is fixed.
- Captcha: `CaptchaService` binds issuer fingerprint (IP+UA), enforces `CAPTCHA_MIN_VERIFY_SECONDS`, one-time key consumption.
- Still-open gaps (keep in spec): upload magic-byte validator warns only and returns True; `check_file_size` trusts client `file.size`; SVG allowed; `_load_user_data_scopes` doesn't filter role status/is_deleted; permissions cached at login (role/menu changes stale for live sessions); logout doesn't compare session equality; App-side `?token=` query exposure.

## Migrations / startup / seed
- `backend/app/scripts/initialize.py` runs on **every** startup: empty DB → `create_tables()` + `alembic stamp head` + seed; existing DB → `command.upgrade(cfg, "head")`; **DEV additionally autogenerates + applies migrations and deletes/aborts revisions containing destructive `op.drop_*`/`DROP`**; self-heals zombie `alembic_version` rows.
- Seed data: flat `backend/sql/*.json` (menu/role/user/dept/dict/param). **Not `sql/data/`.**
- Lifespan order (`app/__init__.py:20-67`): `InitializeData().init_db()` → `init_agno_tables()` (AI session tables) → `redis_connect` → `ParamsService.init_cache` → `DictDataService.init_cache` → `SchedulerUtil.init_scheduler`.
- CLI: `uv run main.py run --env=dev` / `revision -m "..."` / `upgrade --env=dev`; `main.py` sets `os.environ["ENVIRONMENT"]` **before** importing settings; `get_settings()` defaults to `dev`. The old `--env`/`.env.None` defect is fixed.
- `DATABASE_TYPE` supports mysql / postgres / **sqlite (zero-config local start)**.
- Docker backend CMD: `python main.py run --env=prod` → containers auto-migrate.
- Default seed accounts: `super` / `admin` / `user`, password `123456`.

## Codegen
- Generator module `backend/app/modules/generator/gencode/`; templates `backend/templates/{python,ts,vue}/*.jinja2`.
- Registered python templates now include `package_init.py.jinja2` and `plugin.toml.jinja2` (in preview/ZIP), plus feature-level `controller/service/crud/schema/model/__init__`; local output also writes a module-level `__init__.py` (not a registered template).
- Route/component mapping: `_route_path = /{package}/{module}`, `_component_path = {package}/{module}/index`.
- Jinja2 pitfall: `{% for %}` + `{% set %}` accumulator booleans leak scope (for-block isolation) — compute with filters once, e.g. `columns | selectattr('python_type','equalto','date') | list | length > 0` (real bug: missing validator import in generated `schema.py`).
- `sync_db/preview/{table_name}` endpoint exists (`controller.py:155`); `X-Skipped-Tables` header on import.

## Frontend web (Vue3)
- Router dir: `frontend/web/src/router/{index,routes,guards,refresh,MenuProcessor,route-loader}.ts`; dynamic pipeline `ComponentLoader`/`RouteTransformer`/`RouteRegistry` live in `route-loader.ts`; static shell in `routes.ts`.
- **`NestedRouterParent` and `RouteMeta.remountOnFullPath` are removed** — multi-level menus deep-skip to the leaf via RouterView depth; directory routes must keep `component: undefined` (`MenuProcessor.mapMenuNode`, `RouteTransformer.handleNormalRoute`).
- KeepAlive invariants (SKILL §7, enforced by `src/__tests__/route-invariants.spec.ts`): single KeepAlive only in `layouts/fa-page-content/index.vue`; cache key = leaf route `path` (`routeViewCacheKey`); `include` from worktab `opened` (empty ⇒ undefined ⇒ no prune); eviction via `keepAliveExclude` in `worktab.store.ts clearAll()` + `removeKeepAliveExclude` on `openTab`; never add `:max`; `defineOptions({ name })` **must equal menu `route_name`**; matching is by component name (unified in `2d393247`).
- Side-effectful pages: implement `onActivated`/`onDeactivated`; `onUnmounted` is not the sole cleanup point. WS guard pattern: `if (ws && ws.readyState !== WebSocket.CLOSED) return`; detach handlers before `close()`.
- Views/API naming inconsistency (record as-is): `api/module_storage/**` vs `views/module_task/storage/**`.
- Deleted (do not reference): `utils/socket/`, `chat.store.ts`, `fa-chat-window`, `views/module_system/chat`, `views/dashboard/screen/**`, `components/forms/fa-transfer`, `api/module_monitor/resource.ts`.
- Commands: `pnpm ts:check` (vue-tsc --noEmit --skipLibCheck) is the SKILL-prescribed check; `pnpm type-check` (no --skipLibCheck) also exists — spec should name `ts:check` canonical and note the difference. `pnpm lint`, `pnpm test` (vitest run), `pnpm build:prod`.
- Tests: `src/__tests__/smoke.spec.ts` + `src/__tests__/route-invariants.spec.ts`. vitest aliases: `@, @views, @imgs, @icons, @utils, @stores, @plugins, @styles, @api, @fa_imgs`.
- i18n: `locales/langs/zh.json` + `en.json` both required; permission directive `directives/permission/` (`v-hasPerm`, array-capable, delegates to `useAuth().hasAuth`).

## Backend tests
- `backend/tests/` = `conftest.py`, `test_main.py`, `test_migrations.py` (migration-consistency guard: create_all bootstrap ≡ metadata, no version-file leakage). `test_api_module_system.py` deleted; `assert_route` helper has zero call sites (still defined `conftest.py:240`).

## Ops / deployment
- `deploy.sh` at repo root (tracked); no `deploy.bat`, no `docker/deploy.sh`. Cleanup does `docker image prune -f` + `docker builder prune -f` only — explicitly NOT `system prune -a`. No longer git-pulls; loads `docker/.env`; `PROJECT_NAME="FastapiAdmin"`.
- `docker/backend/Dockerfile` does `COPY ./backend/ .`; compose backend bind mount `../backend:/home` is **commented out** (only upload dir mounted).
- Compose healthcheck probes `http://localhost:8001/api/v1/monitor/health/check` with `X-Forwarded-Proto: https`.
- nginx: `service.fastapiadmin.com` hard-coded (`NGINX_SERVER_NAME` documented in docker/README but unimplemented); `/web` alias `/usr/share/nginx/html/web/dist`; `/app` → `docker/nginx/app/dist/build/h5`; `limit_req`/`limit_conn` **applied** on `/api/v1`.
- dist artifacts (all gitignored, absent in fresh checkout): `frontend/web/dist` (build output), `docker/nginx/web/dist` (nginx /web), `backend/dist` (`path_conf.FRONTEND_DIST_DIR`, served by `register_frontend` at `app/__init__.py:127-129`), `docker/nginx/app/dist/build/h5`. Sync: `pnpm build:prod` then `rsync -a --delete` to the other two. Verification: grep artifact strings (e.g. `grep -l "ai/chat/ws" docker/nginx/web/dist/js/*.js`; `readyState` guard) — property accesses and Chinese message strings survive minify.
- `register_frontend` pitfall: check and mount must use the same `FRONTEND_DIST_DIR`; mismatch (`check_dir=False`) fails only on **first request**, diagnosable via `backend/logs/fastapiadmin.log`.
- Backend deps: version `3.2.0`; `pydantic[email]>=2.12.5`, `pydantic-settings>=2.6.1,<3`, `py-ip2region==3.0.4`, storage SDKs (cryptography/paramiko/boto3/obs/oss/cos). `frontend/web/package.json` unchanged since 08-15.
- Loguru: rotation `00:00`, retention 30 days, audit log `backend/logs/fastapiadmin.log`, APScheduler forced WARNING. Ruff: line-length 200, double quotes, `FAST/F/E/W/I/B/C4/UP`, fix=true.

## SKILL.md's own 4 inaccuracies (fix in SKILL.md; never copy into spec)
1. §2/§4 "bcrypt" → PBKDF2-HMAC-SHA256 (`password_util.py`).
2. §9 "仓库内存在多份预构建副本" → dist dirs are gitignored/absent in fresh checkout; keep the procedure, drop the committed-copies claim (adjust wording to "部署链路存在三份产物目录，构建后必须同步").
3. §4 `frontend/web/.env.development` → repo ships `.env.development.example` / `.env.production.example` only.
4. §2/§4 `VITE_APP_WS_ENDPOINT` scope → AI WS **and** health/transfer SSE consumers.
(Note: SKILL.md §4's `sys:user:add` example vs backend `module_*` reality — flag to user, out of the approved 4-fix scope unless trivially adjacent.)
