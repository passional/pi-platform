# Spec Staleness Audit — `.trellis/spec/{architecture,operations,security,testing}` + `guides/index.md`

Specs are dated 2026-08-20; `SKILL.md` (~2026-09-17) and the working tree diverge substantially. A structural refactor (`app/api/v1/module_*` → `app/modules/*`, `init_app.py` → `app/__init__.py`, removal of system-chat WS, transfer WS → SSE, health moved to `/monitor`) plus several security hardening fixes landed after publication. **Every one of the 12 target spec files contains at least one wrong claim; the three architecture files and three security files are the worst.**

Baseline code facts used throughout:
- `backend/app/api/v1/routers.py` — single route registry: `DOMAIN_CONTROLLERS: dict[str, list[APIRouter]]` (prefixes `/system`, `/monitor`, `/task`, `/ai`, `/generator`, `/common`).
- `backend/app/__init__.py` — app factory, `lifespan`, `register_routers` (imports `api_v1` + `dynamic_router.init_app`). **`backend/app/init_app.py` does not exist.**
- `backend/app/modules/` — first-party code (`ai`, `common`, `generator`, `monitor`, `system`, `task`). No `app/api/v1/module_*` tree.
- `backend/app/plugin/module_example/{plugin.toml,demo/controller.py}` — dynamic plugin example.

## 1. `architecture/`

### STALE / WRONG claims

`system-boundaries.md`
- **Wrong file/path model.** §2 says first-party routes live at `backend/app/api/v1/module_<area>/<feature>/` and are assembled via `backend/app/api/v1/module_system/__init__.py` with `system_router = APIRouter(prefix="/system")`; §2/§3 say `backend/app/init_app.py:register_routers` includes areas and that features live at `backend/app/api/v1/module_<area>/<feature>/`. Reality: features live under `backend/app/modules/<area>/<feature>/`; registration is the `DOMAIN_CONTROLLERS` dict in `backend/app/api/v1/routers.py`; `register_routers` is in `backend/app/__init__.py`. (§3 "Evidence" lines naming `backend/app/api/v1/module_system/dept/` are stale too.)
- **Wrong discovery failure behavior.** §4 matrix row "Plugin path is not importable or controller raises → Discovery logs and skips it". `backend/app/core/discover.py:96-99` now raises `RuntimeError` and **aborts startup** ("已中止启动"), and `check_route_conflicts()` (lines 112-139) also halts startup on duplicate routes. The docstring is explicit: fail-fast is intentional. This inverts the documented contract.
- **Wrong `plugin.toml` placement.** §3 says the module root (`<feature>/`) holds `plugin.toml`; in reality `plugin.toml` sits at the **area/package** level (`backend/app/modules/system/plugin.toml`), and dynamic discovery still ignores it. SKILL.md says "模块根放 plugin.toml" — ambiguous, but the spec's `<feature>` reading is wrong.
- **Test-path reference stale.** §6 cites `backend/tests/test_api_module_system.py`; that file no longer exists (tests are `conftest.py`, `test_main.py`, `test_migrations.py`).
- Still accurate: `ENABLED_MODULES` is README-only (only `README.md`/`README.en.md` mention it; absent from `setting.py`/`__init__.py`); `plugin.toml optional=true` has no runtime effect; docs `src/guide` + `src/en/guide` pair exists.

`http-and-realtime-contracts.md`
- **Realtime matrix largely stale.**
  - `/system/chat/ws` channel: **no longer exists.** `app/modules/system/` has no `chat` dir; `find backend/app -name ws_manager.py` returns nothing; no chat router in `routers.py`; `frontend/web/src/api/module_system/` has no `chat.ts`.
  - Storage transfer is now **SSE, not WebSocket**: `backend/app/modules/task/storage/transfer/controller.py:105` `@StorageTransferRouter.get("/stream", response_class=EventSourceResponse)`, prefix `/storage/transfer` under domain `/task` → public `/api/v1/task/storage/transfer/stream`. Owner files changed (`ws_manager.py` → `sse_manager.py`); frontend consumer exists now: `frontend/web/src/api/module_storage/transfer.ts:155` uses `createSSEClient` (`@utils/sse`).
  - Health SSE route moved: spec says `/common/health/stream`; actual `backend/app/modules/monitor/health/controller.py:30` `@HealthRouter.get("/stream")` under `/monitor` → `/api/v1/monitor/health/stream`.
- **AI WS carrier claim stale.** §3 says "current Web and App clients use query `?token=`" and "Do not describe the safer carrier as deployed until the Web client is changed and tested". `frontend/web/src/views/module_ai/chat/index.vue:103` now uses `new WebSocket(url, ["access_token", "access_token."+token])`. Only the uni-app client (`frontend/app/src/composables/useAiChat.ts:24`) still uses `?token=`.
- **AI WS auth detail incomplete.** Handshake now also enforces chat permission and closes with **4003** (`controller.py:219`), not only 4001.
- Still accurate: `ResponseSchema[T]` five fields (`backend/app/common/response.py:26-33`); `SuccessResponse`/`ErrorResponse`; 422 envelope; `items/total/page_no/page_size/has_next` (`frontend/web/src/types/global.d.ts:66-71`); App `PageResult.list` known mismatch (`frontend/app/src/types/global.d.ts:85-93`); the AI "stop is unreachable while generating" analysis (`controller.py:275-291` still `async for` inside the read loop).

`code-generation-and-routing.md`
- **Template output set stale.** §3 table lists `python/{controller,service,crud,schema,model,__init__}.py.jinja2` only. `Jinja2TemplateUtil.get_template_list()` (`jinja2_template_util.py:287-299`) now also registers `python/package_init.py.jinja2` and `python/plugin.toml.jinja2`.
- **Package `__init__.py` preview/ZIP claim wrong.** §3 says the package marker "is not part of preview or batch ZIP output; only the feature-level `__init__.py` comes from the registered template set". Now `package_init.py.jinja2` is a registered template mapping to `backend/{app/plugin}/{package}/__init__.py` (line 355) and therefore **is** in preview/ZIP; additionally local output creates a **module-level** `__init__.py` (`service.py:673-682`), which is not a registered template. The documented difference is inverted.
- **Docstring note now self-contradictory.** §3 says "The `generate_code` docstring still mentions an extra `business_path` ... executable code uses only package + module." The docstring still says `/{module_xxx}/{module_name}/{business_path}` (`service.py:630-631`), and `prepare_context` now computes a real `business_path` (line 172). The spec's conclusion (executable uses package+module) matches `_route_path = f"/{route_seg}/{_mn}"` (line 726) and `_component_path = f"{_pn}/{_mn}/index"` (line 727), so the mapping is right but the surrounding text is fragile/misleading.
- **Omitted endpoints.** §2 signature list omits `/generator/gencode/sync_db/preview/{table_name}` (`controller.py:155`) and `/list`, `/db/list`, `/import`, `/detail`, `/create`, `/delete`. `sync_db/preview` is a real added contract.
- Still accurate: permission set `module_example:demo:{query,detail,create,update,delete,patch,export,import,download}`; API base `/example/demo`; component path `module_example/demo/index`; menu route `/module_example/demo`; preview declared `ResponseSchema[GenTableOutSchema]` vs live path→content map (`controller.py:135`); `X-Skipped-Tables` (`controller.py:117`); partial-ZIP/all-failed semantics; overwrite-with-no-guard; file-before-menu-conflict; no dedicated generator tests.

### MISSING conventions (SKILL.md → architecture)
- **§8 Jinja2 `{% for %}`/`{% set %}` scoping pitfall** (schema.py.jinja2 validator-import bug) — belongs in `code-generation-and-routing.md` §"Write and failure behavior"/§6.
- **§3 "重启后端" invariant**: dynamic discovery `discover.py` runs only at startup, so generated/modified modules silently 404 until restart. Architecture already documents discovery; the *restart requirement + silent-failure mode* is absent.
- **§7 router/KeepAlive invariants as architecture-level contracts**: single-layer `KeepAlive` in `layouts/fa-page-content/index.vue`, cache key = leaf `path` (`routeViewCacheKey`), directories must keep `component: undefined` (`MenuProcessor.mapMenuNode`), `include` from worktab `opened`, `keepAliveExclude` eviction on `clearAll`. `system-boundaries.md` §"Web menu-to-component boundary" only covers `component_path` → glob (`route-loader.ts:24`).
- **§4/§2 `VITE_PORT=5180`, `VITE_APP_BASE_API=/api/v1`, WS/SSE endpoint `VITE_APP_WS_ENDPOINT`** — the realtime spec should note that the **SSE** consumers (`api/module_monitor/dashboard.ts:40`, `api/module_storage/transfer.ts:156`) also use `VITE_APP_WS_ENDPOINT`.

### CONTRADICTIONS
- Discovery failure semantics: spec "logs and skips" vs code "abort startup" (and SKILL.md §3 correctly says failure aborts/silently 404s).
- Web AI WS carrier: spec "Web uses query fallback" vs code subprotocol (SKILL.md §4 accurately says subprotocol).

**Assessment: ~75-85% of `architecture/` needs edits.** `system-boundaries.md` ~60% (paths/registration/discovery matrix), `http-and-realtime-contracts.md` ~70% (realtime matrix/carrier), `code-generation-and-routing.md` ~45%. `index.md` is fine.

## 2. `operations/`

### STALE / WRONG claims

`configuration-and-startup.md`
- **CLI `--env` defect is FIXED.** §2/§3 claim `backend/main.py` imports global `settings` before `ENVIRONMENT` is set and that the model computes `.env.None`. `backend/main.py` imports only `EnvironmentEnum`, sets `os.environ["ENVIRONMENT"] = env.value` inside each command **before** `from app.config.setting import settings` (run: line 34/37), and `get_settings()` defaults to `dev` (`setting.py:325`: `os.getenv('ENVIRONMENT', 'dev')`). Test-matrix rows "CLI `--env` without pre-set `ENVIRONMENT`" and the §7 Wrong example are stale.
- **Startup sequence stale.** §3 `lifespan` list: actual (`app/__init__.py:20-67`) is `InitializeData().init_db()` → `init_agno_tables()` → `redis_connect` → `ParamsService.init_cache` → `DictDataService.init_cache` → `SchedulerUtil.init_scheduler`. `init_agno_tables()` (AI session tables) is missing; the spec's ordering/description of step 1 ("runs metadata `create_all`") is incomplete.
- **Migration model changed.** §3 says `create_all` only, "Production schema changes require ... an explicit `upgrade` ... The versions directory currently contains only `__init__.py`, and the Docker command does not run Alembic. This is a current rollout gap." Reality (`app/scripts/initialize.py:60-136`): empty DB → `create_tables()` + `alembic stamp head`; existing DB → `command.upgrade(cfg, "head")`; **dev additionally autogenerates + applies migrations**, with a destructive-DROP guard. `docker/backend/Dockerfile` CMD is `python main.py run --env=prod`, so containers do migrate automatically. The gap framing is obsolete. (Versions dir still only `__init__.py` — that part is true.)
- **Seed path wrong.** §3 says seeds from `backend/sql/data/`; `SCRIPT_DIR = BASE_DIR / "sql"` (`path_conf.py`) and files are flat `backend/sql/*.json`.
- **Operation-log claim now false.** §"Secret and logging safety" says `OperationLogRoute` captures bodies "without field redaction" and persists login passwords/JWTs/API keys. `backend/app/core/router_class.py` now has `_SENSITIVE_KEYS` (lines 15-31) and `_redact_sensitive` applied to both request JSON (line 97), form fields (line 90) and **response JSON** (line 112). Direct contradiction (see security layer).
- Still accurate: case-sensitive settings, `extra="ignore"`, env-file-over-process precedence, `SECRET_KEY` dev default despite comment, user model `api_key` stored in Redis, default SECRET_KEY not rejected at startup.
- **Missing keys**: `WORKERS`, `TRUSTED_PROXY_HOPS`, `SESSION_MAX_LIFETIME_SECONDS`, `LOGIN_RATE_LIMIT_*`, `DATA_ENCRYPTION_KEY`/`_OLD_KEYS`, `SCHEDULER_ALLOW_CODE_EXEC`, `PASSWORD_MIN/MAX_LENGTH`, `PASSWORD_IMPORT_DEFAULT`, `CAPTCHA_MIN_VERIFY_SECONDS`, `REDIS_HEALTH_CHECK_INTERVAL`, `REDIS_DEFAULT_CACHE_TTL`, `GZIP_*`, `HTTPX_DEFAULT_TIMEOUT`, `CORS_EXPOSE_HEADERS`, `AUTOFETCH` (all in `backend/app/config/setting.py`). Also `backend/env/.env.example` lacks a `SECRET_KEY` entry even though its header comment tells users to set it.

`deployment-topology.md`
- **Backend image contract wrong.** §3 says the Dockerfile "copies only `requirements.txt`" and the `../backend:/home` mount "is therefore required". `docker/backend/Dockerfile` now has `COPY ./backend/ .`, and compose's backend `volumes` has `# - ../backend:/home` **commented out** (only `./backend/static/upload:/home/static/upload`). The whole "do not remove the mount" section, the error-matrix row, and §7 "Wrong/Correct" block are obsolete.
- **Health probes wrong (paths + names).** §2 table lists `/common/health/live`, `/common/health/ready`, `/common/health/check`. Actual: only `/monitor/health/check` (`controller.py:20`) and `/monitor/health/stream` (line 30). **There is no `/live` and no `/ready`** anywhere. Compose healthcheck already probes `http://localhost:8001/api/v1/monitor/health/check` with `X-Forwarded-Proto: https`, so the "Compose probes `/common/health`... there is no bare health handler" claim is doubly stale.
- **`deploy.sh` prune claim wrong.** §3 says it calls `docker system prune -a -f` in full deploy and `clean`. Actual `deploy.sh:116-121` cleanup calls only `docker image prune -f` and `docker builder prune -f`, with an explicit comment rejecting `-a`.
- **Realtime/replica evidence paths stale**: cites system-chat `ws_manager.py` and `module_storage/transfer/ws_manager.py`; both gone (system chat removed; transfer is `sse_manager.py` + `registry.py`). The *conclusion* (single backend replica; per-process APScheduler) remains valid.
- Still accurate: frontend artifacts + `deploy.sh` lacking build flags; `docker/README.md` documents unimplemented `--build-frontend`/`--skip-frontend`/`BUILD_WEB` (README lines 62-103, 179-184) and `NGINX_SERVER_NAME` (README line 63) while `nginx.conf` hard-codes `service.fastapiadmin.com`; `/docs` in README advertised as Swagger while Nginx `/` serves VitePress; MySQL/Redis host ports exposed by default; log rotation 10m×3.

### MISSING conventions (SKILL.md → operations)
- **§2 first-run/startup flow** ("复制 `.env.example` → `uv sync` → auto-migrate + `sql/*.json` seed → `pnpm dev`; default accounts `super`/`admin`/`user` = `123456`; sqlite zero-config; `uv run pytest`"): the spec has fragments but not the coherent first-run path, and the default-credentials/bootstrap contract is entirely absent.
- **§3 restart-after-codegen** and **§8 static-hosting `FRONTEND_DIST_DIR` pitfall** (`register_frontend` checks and mounts the same `path_conf.FRONTEND_DIST_DIR=backend/dist`; mismatch only fails on first request, visible in `backend/logs/fastapiadmin.log`) — belongs in deployment-topology.
- **§9 dist/deployment verification** (see Testing layer).
- **`DATABASE_TYPE` sqlite/postgres** support (spec lists the keys but not the zero-config sqlite path that SKILL.md §2 highlights).

### CONTRADICTIONS
- Compose/Dockerfile source-copy + bind mount (spec says mount required; code ships source and comments the mount).
- `deploy.sh` prune scope (spec: `system prune -a`; code: targeted image/builder prune, explicitly avoiding `-a`).
- Health probe existence (spec: bare `/common/health` 404s; code: compose probes a real `/api/v1/monitor/health/check`).
- Operation-log redaction (spec: none; code: redacts request+response).

**Assessment: ~65-75% of `operations/` needs edits.** `configuration-and-startup.md` ~60%, `deployment-topology.md` ~70%; `index.md` fine.

## 3. `security/`

### STALE / WRONG claims

`authentication-and-secrets.md`
- **Refresh-token rotation/replay is now implemented.** §4 says "Refresh ... does not compare the submitted token with the current stored refresh token. Therefore refresh issues replacement JWTs but does not revoke an older signed access/refresh JWT". `auth/service.py:449-462` compares `stored_refresh != refresh_token` and revokes the session on mismatch ("检测到疑似 refresh token 重放，已撤销会话"). Verification-matrix row and contract-testing text are stale.
- **Sliding-session claim wrong.** §4 says "Sliding access activity extends access/refresh-token keys but not the `USER_SESSION` key. A session still reaches its original session TTL". `dependencies.py:124-156` now renews `USER_SESSION` (the survival key) and the refresh key, bounded by `SESSION_MAX_LIFETIME_SECONDS` absolute cap (with `created_at` backfill and a "会话超过最大存活时长" 401). Contradiction.
- **Browser AI WS carrier stale.** §2 table says Web uses `?token=` fallback; `frontend/web/src/views/module_ai/chat/index.vue:103` uses the subprotocol. App still uses query.
- **System-chat channel gone.** §2 row "System-chat / transfer WebSocket | query `?token=`" — no such channel/module; transfer is SSE at `/api/v1/task/storage/transfer/stream`.
- Still accurate: HS256 small payload (`sub`, `is_refresh`, `exp`); `_authenticate` checks cached `user_status`, reloads SQL user filtering only `is_deleted` (no current `status` re-check — `dependencies.py:162-182`); logout decodes the body token independently (`auth/service.py:522-537`, controller passes body `payload`), no session-equality check; password PBKDF2-HMAC-SHA256 600k (`password_util.py:12-29`); schema length 6-128; `check_password_strength` unused; query-token exposure for system/transfer; token storage in localStorage/sessionStorage.

`authorization-and-data-scope.md`
- Still largely accurate: `AuthPermission` any-of semantics + superuser bypass + `*`/`*:*:*` endpoint config (`dependencies.py:192-234`); `permission.py` defines only scopes 1/2/3 (lines 16-18), no custom `5`; `_load_user_data_scopes` (lines 96-103) does **not** filter role `status`/`is_deleted`; department traversal swallows exceptions and falls back to current dept (lines 110-115); `WHITE_API_LIST_PATH` is demo-mode middleware surface, not an auth allowlist (`core/middlewares.py:49-60`); role/menu changes stale for existing sessions (permissions cached at login).
- Minor: §4 lists `/common/health` SSE as public; path is now `/monitor/health/stream`.

`untrusted-content-and-audit-data.md`
- **Operation-log exposure section now wrong.** §4 says values "are not redacted", lists "login form password and login response access/refresh tokens" as persisted, and gives Wrong example `oper_param["body"] = json.loads(payload.decode())`. Code does `oper_param["body"] = _redact_sensitive(json.loads(payload.decode()))` plus response redaction, form-field redaction, `_SENSITIVE_KEYS` covering `password/old_password/new_password/confirm_password/access_token/refresh_token/api_key/apikey/secret/client_secret/secret_key/authorization/token/captcha_key` (`router_class.py:15-41,90,97,112`). The "new code must redact" rule is already the current state.
- **`_PUBLIC_WRITE_PATHS` no longer exists** (no occurrence in `backend/`); the §4 bullet about it is dead.
- **Download confinement fixed.** §2 says download "uses a string `startswith` comparison rather than `Path.is_relative_to`". `backend/app/modules/common/file/service.py:52-56` uses `Path(...).resolve().is_relative_to(upload_root)` with a comment explicitly about sibling-prefix bypass. Matrix row and regression-test ask are stale.
- Still accurate: upload magic-byte validator warning-only returning `True` (`upload_util.py:216-233`); `check_file_size` trusts `file.size`; whole file read once before chunked write; SVG allowed; `NoticeCreateSchema.notice_content` Bleach-sanitized via `xss_util.sanitize_html`; workflow `builtin_nodes.py` bypass.
- Partially stale: captcha slider repro — `CaptchaService` now binds issuer fingerprint (IP+UA), enforces `CAPTCHA_MIN_VERIFY_SECONDS` minimum residence, and one-time consumes the key (`auth/service.py` CaptchaService docstring/`_matches`). Spec still frames it as "marks the supplied key verified... not proof of human challenge" without the new hardening.

### Cross-spec contradictions (security)
- **Login rate limiting: spec says absent, code has it.** "no backend limiter is registered" (`authentication-and-secrets.md` §4) — `auth/service.py:155-168` `_check_login_rate_limit` uses Redis `login_rate_limit:{ip}` with `LOGIN_RATE_LIMIT_WINDOW_SECONDS`/`LOGIN_RATE_LIMIT_MAX_ATTEMPTS` and is called at line 186. Additionally **Nginx now applies the zones**: `docker/nginx/nginx.conf` `location /api/v1 { limit_req zone=api_limit burst=60 nodelay; limit_conn conn_limit 100; }` — directly contradicting "declares `limit_req_zone`/`limit_conn_zone` but never applies `limit_req` or `limit_conn`".
- **Forget-password account-takeover claim wrong.** `POST /system/user/password/forget` is still unauthenticated (`user/controller.py:74-81`) but `UserService.forget_password` (`user/service.py:297-306`) **no longer changes the password**: it only logs and returns, explicitly "未登录调用方不得设置新密码" with anti-enumeration response. The §4 "account takeover exposure" claim is obsolete.
- Operation-log redaction (as in operations).

### MISSING conventions (SKILL.md → security)
- `v-hasPerm` (supports arrays) is now verified in code (`frontend/web/src/directives/permission/index.ts:10-21`, delegating to `useAuth().hasAuth`); the spec's authorization section mentions Web directives only in passing — no concrete contract for the directive/`hasAuth` API.
- `Sec-WebSocket-Protocol` handshake detail (`get_websocket_token` reads `access_token.<jwt>`, echoes subprotocol on `accept`; App query fallback) — partially present, but should be updated rather than removed.

**Assessment: ~65-75% of `security/` needs edits.** `authentication-and-secrets.md` ~60% (rate-limit, refresh rotation, sliding session, carrier, forget-password), `authorization-and-data-scope.md` ~25% (mostly accurate), `untrusted-content-and-audit-data.md` ~45% (operation-log section mostly obsolete, confinement fixed); `index.md` fine.

## 4. `testing/`

### STALE / WRONG claims

`backend-testing.md`
- **Test inventory wrong.** §1 lists `test_main.py` and `test_api_module_system.py`; actual `backend/tests/` = `conftest.py`, `test_main.py`, `test_migrations.py`. The spec then asserts "There are no dedicated backend security, WebSocket, generator, upload, or service/CRUD test files" — no longer literally true: `test_migrations.py` now owns a migration-consistency guard (create_all bootstrap ≡ metadata, no version-file leakage), which is a dedicated fixture-driven suite.
- Fixture description still accurate: `conftest.py` sets `DATABASE_TYPE=sqlite` before app import, one temp SQLite file, `AsyncMock` Redis with a module dict (lines 141-159), admin reset to `admin123` (line 188), session-scoped `_api_client`/`auth_headers`, `assert_route` semantics.
- §3's `assert_route` caution and async CRUD guidance remain valid.

`frontend-testing.md`
- **"The only current test file is `src/__tests__/smoke.spec.ts`" is wrong.** There are now two: `src/__tests__/smoke.spec.ts` and `src/__tests__/route-invariants.spec.ts`. The new file asserts the SKILL.md §7 router/KeepAlive invariants (directories have no `component`, single-layer KeepAlive) — so the spec's claim of "no component, router, store, HTTP, auth... coverage" understates reality.
- `vitest.config.ts` include/aliases described accurately; `pnpm test` = `vitest run`; `type-check` and `ts:check` both exist.
- App section accurate: `frontend/app/package.json` has no `test` script and no test files; `pnpm lint`/`type-check` present.

`contract-and-deployment-testing.md`
- **Deployment checks stale**: `docker compose` healthcheck now hits `/api/v1/monitor/health/check` (real); "direct `/common/health/live` and `/ready` plus their public equivalents" must become `/monitor/health/check` (no live/ready exist); "the backend image relies on the source bind mount" is false (`COPY ./backend/ .`); generator section still accurate (no generator tests) but its path list is under `module_generator` — check.
- Realtime table stale for system-chat (removed) and transfer (`/storage/transfer/ws` → `/task/storage/transfer/stream` SSE); AI stop limitation and close-before-accept for invalid auth still accurate for AI (4001/4003).
- REST/auth bullets: refresh rotation no longer "old signatures remain usable" (rotation + replay revocation); "user disable vs refresh" asymmetry still accurate; "operation logs are searched for sentinel passwords" is now a *regression guard on implemented redaction*, not a gap.
- Still accurate: `assert_route` not reusable for contracts; App ambient `list` mismatch; no CI workflow/E2E suite.

### MISSING (SKILL.md → testing)
- **§9 dist artifact verification** — no layer covers it. The `dist` contract (three artifacts: `frontend/web/dist` ← `pnpm build:prod`; `docker/nginx/web/dist` ← Nginx `alias /usr/share/nginx/html/web/dist`; `backend/dist` ← `app.frontend("/")`/`FRONTEND_DIST_DIR`; plus `docker/nginx/app/dist/build/h5`), the `rsync --delete` sync step, and "verify the built artifact (`grep -l "ai/chat/ws" docker/nginx/web/dist/js/*.js`, `grep -oE '.{0,60}readyState...'`) before touching source" belong in `contract-and-deployment-testing.md` (`operations/deployment-topology.md` for the artifact table).
  - **Caveat — SKILL.md §9 itself is currently inaccurate**: all four dist directories are **absent** in the working tree (`frontend/web/dist`, `docker/nginx/web/dist`, `backend/dist`, `docker/nginx/app/dist/build/h5`); `.gitignore` ignores `dist`, `docker/nginx/{web,app,docs}/dist`. So "仓库内存在多份预构建副本并直接被打进部署" does not hold for this checkout. Any spec edit should encode the *verification procedure* and the artifact/serving mapping, not the claim that prebuilt copies are committed.
- **§2 command table** (`uv run pytest`, `uv run main.py revision/upgrade`, `uv run ruff check .`) partly captured; `main.py revision/upgrade` subprocess testing and migration-guard test (`test_migrations.py`) should be cited.
- **§7 router/KeepAlive invariants** now have a real asserting test (`route-invariants.spec.ts`) — `frontend-testing.md` should document it as the canonical location.
- **§2 verification set** (`uv run pytest` / `pnpm ts:check` / `pnpm lint` / `pnpm test`) — fine but should note `route-invariants` and `test_migrations` specifically.

### CONTRADICTIONS
- Test-file inventories (backend `test_api_module_system.py`; frontend "only `smoke.spec.ts`") vs actual files.
- Deployment smoke expectations vs compose healthcheck path and Dockerfile source copy.

**Assessment: ~50-60% of `testing/` needs edits.** `backend-testing.md` ~35% (inventory + migration suite), `frontend-testing.md` ~40% (test inventory + route-invariants), `contract-and-deployment-testing.md` ~65% (deployment paths/image/rotation); `index.md` fine.

## 5. `guides/index.md`

- Link table resolves (`code-reuse-thinking-guide.md`, `cross-layer-thinking-guide.md` both exist); stated purpose ("checklists, not payloads/paths/commands") holds.
- **Minor gaps**: trigger list does not mention (a) router/KeepAlive caching invariants, (b) codegen "restart backend" requirement, (c) dist/deployment verification. Reference list omits `../backend/` and `../frontend/` specs and `SKILL.md` itself.
- No wrong claims. **~10-15% edit (additive).**

## Consolidated: SKILL.md content missing from these layers

| SKILL.md | Missing from | Suggested target spec file |
|---|---|---|
| §9 dist artifacts + `rsync --delete` + artifact-level grep verification | operations, testing | `operations/deployment-topology.md` (artifact table) + `testing/contract-and-deployment-testing.md` (procedure) |
| §7 KeepAlive single-layer / leaf-`path` cache key / directory `component: undefined` / worktab `keepAliveExclude` / WS handshake guard | architecture | `architecture/system-boundaries.md` (+ `testing/frontend-testing.md` to cite `route-invariants.spec.ts`) |
| §2 first-run flow, `sql/*.json` seed, default accounts `123456`, sqlite zero-config, `VITE_PORT=5180` | operations | `operations/configuration-and-startup.md` |
| §8 Jinja2 `{% for %}`/`{% set %}` scoping pitfall | architecture | `architecture/code-generation-and-routing.md` |
| §3 "restart backend after codegen" (discovery runs at startup only; silent 404 otherwise) | architecture | `architecture/code-generation-and-routing.md` / `system-boundaries.md` |
| §8 `register_frontend` `FRONTEND_DIST_DIR` same-path pitfall (fails on first request) | operations | `operations/deployment-topology.md` |

## Consolidated: SKILL.md claims that themselves need correction (found while cross-checking)

- §2/§4 says default seed passwords are bcrypt-verified; code uses **PBKDF2-HMAC-SHA256** (`backend/app/utils/password_util.py:12-29`).
- §9 says multiple prebuilt `dist` copies exist in-repo and are deployed; all four dist dirs are **absent** in this checkout and gitignored.
- §4 refers to `frontend/web/.env.development`; the repo only ships `.env.development.example` / `.env.production.example`.
- §2/§4 narrows `VITE_APP_WS_ENDPOINT` to the AI WebSocket; it is also used for the health SSE (`api/module_monitor/dashboard.ts:40`) and transfer SSE (`api/module_storage/transfer.ts:156`).
