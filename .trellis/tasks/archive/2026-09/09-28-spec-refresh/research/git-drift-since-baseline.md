# Code drift since the ~2026-08-20 spec baseline

Repo `/home/zzhou/coding/pi-platform`, branch `master`, HEAD `1cd12c72` (2026-09-17 20:57 +0800).
Spec-baseline commit (last on master ≤ 2026-08-20): **`859b162d`** (Merge PR #507, 2026-08-19).

- Commits since 2026-08-10: **47**; since 2026-08-15: **42**; after the baseline: **`859b162d..HEAD` = 37** (2026-09-03 → 2026-09-17; dead gap 2026-08-20 → 2026-09-02 with no commits on master).
- The local `.trellis/spec/` was refreshed on 2026-09-28 (registry import) but its *content* still describes the pre-Sept code state.

## Themed change groups

### Theme A — Backend module restructure `api/v1/module_*` → `app/modules/<domain>/<feature>` (largest spec-breaking change)
Commits: `d470c7eb`, `92081815`, `e340053e` (2026-09-06), groundwork in `3f472d1e` (09-03), `9e80f970` (09-06).
- Old `backend/app/api/v1/module_{system,ai,monitor,generator,storage,common,task,workflow}/**` deleted; new roots `backend/app/modules/{ai,common,generator,monitor,system,task}/<feature>/{controller,service,crud,schema,model}.py`.
- `backend/app/api/v1/{ai,common,file,generator,monitor,system,task,workflow}.py` aggregators deleted; replaced by single `backend/app/api/v1/routers.py` ("唯一路由事实来源", `DOMAIN_CONTROLLERS` map). `backend/app/api/` now only `__init__.py` + `v1/{__init__.py,routers.py}`.
- `backend/app/init_app.py` **deleted** (`e340053e`); lifespan now in `backend/app/__init__.py`.
- Plugin convention: `backend/app/plugin/module_example/demo/` is the plugin example.
- Impacts: **backend (all), architecture (all 3), security, operations, testing**.

### Theme B — Storage / workflow module rename + deletions
Commits: `3f472d1e` (09-03), `9e80f970` (09-06), `a8d29d4a` (09-07), `8d7efb6c` (09-12).
- Backend: `modules/workflow/**` → `modules/task/storage/{browse,node,transfer,workflow,core}`; `modules/file/**` → `modules/common/file/**`; task split into `task/cronjob/{job,node}` + `task/storage/*`; `module_monitor/resource/**` **deleted**.
- New: `backend/app/modules/`, `backend/app/utils/crypto_util.py`, `backend/data/ip2region/`, `backend/templates/`, `backend/sql/*.json`.
- Frontend: `views/module_storage/**` → `views/module_task/storage/**` (~10 files, `a8d29d4a`); API layer went `api/module_task/workflow/*` → `api/module_storage/*` (`9e80f970`) — **so `api/module_storage` and `views/module_task/storage` now disagree (existing code inconsistency; record as-is)**; `components/forms/fa-transfer/index.vue` deleted; `api/module_monitor/resource.ts` deleted.
- Impacts: **backend/directory-structure, database-guidelines, architecture, frontend/directory-structure, testing**.

### Theme C — WebSocket → SSE migration, chat consolidation
Commit: `d470c7eb` (09-06).
- `backend/app/core/ws_manager.py` → **renamed `backend/app/core/sse_manager.py`**; `grep ws_manager backend/app` = zero hits.
- `module_system/chat/**` collapsed into `backend/app/modules/ai/chat/{controller,crud,schema,service}.py` (no `ws_manager.py`); internal-chat WS manager gone.
- Frontend: `utils/socket/index.ts` **deleted** → `utils/sse/index.ts`; `store/modules/chat.store.ts` **deleted**; `layouts/fa-chat-window/index.vue` **deleted**; `views/module_system/chat/index.vue` and `api/module_system/chat.ts` **deleted**.
- Impacts: **architecture/http-and-realtime-contracts, operations/deployment-topology, security/authentication-and-secrets, frontend/state-management, frontend/hook-guidelines**.

### Theme D — Rate limiting added then removed + security hardening
Commits: `a8d29d4a` (09-07, adds `core/rate_limiter.py` + `fastapi-limiter` + hardening), `8cb5f178` (09-07, removes it).
- Net: `core/rate_limiter.py` **deleted**; `fastapi-limiter` removed from `pyproject.toml` + `requirements.txt`; "移除全局 API 与 WS 限流" + proxy-config fix. (Note: login rate limiting via Redis in `auth/service.py` and nginx `limit_req` still exist — see security audit.)
- Impacts: **security, operations, architecture**.

### Theme E — Frontend router / KeepAlive rewrite
Commits: `2d393247`, `24589ad9`, `740e334d` (09-14), merges `1f8757f8` (#528), `39767fd0` (#529), `0cf2e24b` (#530).
- `24589ad9` "remove nested parent component": removes `NestedRouterParent` export and `ROUTE_COMPONENT_NESTED_PARENT` from `frontend/web/src/router/routes.ts`; deletes `RouteMeta.remountOnFullPath` from `types/router/index.ts`; drops `component: NestedRouterParent` from `Dashboard`/`Fastlink` parents. Multi-level menus skip straight to the leaf via `RouterView` depth.
- `2d393247`: unifies KeepAlive matching by **component `name`** across `layouts/fa-page-content/index.vue`, `router/routes.ts`, and 17 view components.
- `740e334d`: adds `frontend/web/src/__tests__/route-invariants.spec.ts` (+159 lines) proving deep-skip rendering; touches `fa-page-content/index.vue`, `router/route-loader.ts`, `vitest.config.ts`. The test file is the only remaining place `NestedRouterParent` appears (historical comment).
- `frontend/web/src/router/` today: `MenuProcessor.ts, guards.ts, index.ts, refresh.ts, route-loader.ts, routes.ts`.
- Impacts: **frontend/component-guidelines, hook-guidelines, directory-structure, architecture/code-generation-and-routing, testing/frontend-testing**.

### Theme F — Data-screen (大屏) removal + plumbing
Commit: `8d7efb6c` (09-12).
- **Deleted** `frontend/web/src/views/dashboard/screen/index.vue` + ~33 `modules/*.vue` files.
- Also touched: `components/base/fa-back-to-top/index.vue`, `hooks/core/useCommon.ts`, `layouts/fa-page-content/index.vue`, `router/guards.ts`, `router/routes.ts`, `store/modules/worktab.store.ts`; backend `common/dataclasses.py`, `config/{path_conf,setting}.py`, `modules/system/auth/*`, `scripts/initialize.py`, `utils/ip_local_util.py`.
- Impacts: **frontend/directory-structure, architecture, operations**.

### Theme G — Features/bugfixes (lower spec impact)
- `f15fdb86` (09-12): ticket stats API + server-IP optimization — `modules/monitor/server/service.py`, `modules/system/ticket/*`.
- `14428bff` (09-07): pull-to-refresh on work/notice/AI-model pages.
- `8b63042e` (09-10): generator python plugin template + frontend static path — `gencode/jinja2_template_util.py`, `backend/app/__init__.py`.
- `f12a4009` (09-13): non-Integer PK model generation/import fix — `jinja2_template_util.py` + `views/module_generator/gencode/utils/createTableVisualPresets.ts`.
- `8291c1a4` (09-13): list-page cache/refresh (`hooks/core/useTable.ts`).
- `594c3f63` (09-07): `el-tabs` controlled-binding fix.
- `0cf2e24b` (09-17, PR #530): `backend/alembic.ini` trimmed 74 lines + `views/module_task/storage/node/index.vue`.

### Theme H — Docker / deploy
- `bddb177d` (09-07): **`deploy.bat` deleted**, `deploy.sh` moved → `docker/deploy.sh`.
- `37b7b8e0` (09-07): `.dockerignore` excludes upload dirs; `docker/deploy.sh` moved **back to repo-root `deploy.sh`**.
- `a8d29d4a` (09-07): `deploy.sh` modified (`PROJECT_NAME="FastapiAdmin"`, `WORK_DIR`, loads `docker/.env`, no longer git-pulls).
- Current: root `deploy.sh` exists (tracked); `docker/deploy.sh` and `deploy.bat` do **not** exist. `docker/{backend/Dockerfile,docker-compose.yaml,nginx/nginx.conf,redis/,mysql/}` present.
- Impacts: **operations/deployment-topology, configuration-and-startup, testing/contract-and-deployment-testing**.

### Theme I — Dependencies
- `backend/pyproject.toml` version `2.0.0` → **`3.2.0`** (`e340053e`); `pydantic[email]>=2.12.5`, `pydantic-settings>=2.6.1,<3` (`3f472d1e`); `py-ip2region==3.0.4` added (`8d7efb6c`); storage SDKs `cryptography==50.0.0, paramiko==5.0.0, boto3==1.43.72, esdk-obs-python==3.26.6, alibabacloud-oss-v2==1.3.2, cos-python-sdk-v5==1.9.44` (`001d1bdf`, just before baseline); `fastapi-limiter` added then removed.
- `frontend/web/package.json` **unchanged** since 2026-08-15. `frontend/app/package.json` changed in `61336936` (08-16).
- Impacts: **operations/configuration-and-startup, architecture**.

### Theme J — Tests
- Added: `backend/tests/test_migrations.py` (`e340053e`); `frontend/web/src/__tests__/route-invariants.spec.ts` (`740e334d`).
- Deleted: `backend/tests/test_api_module_system.py` (`d470c7eb`).
- Modified: `backend/tests/conftest.py` (3f472d1e, d470c7eb, e340053e), `test_main.py`.
- Impacts: **testing/backend-testing, testing/frontend-testing** — spec's "only test file is smoke.spec.ts" is now false.

### Theme K — Docs / requirements (mostly non-code)
- `45a00334` (09-06) adds `REQUIREMENTS.md` (SaaS multi-tenant requirements); `30dfc709` updates `CONTRIBUTING.md`; `e340053e` updates READMEs; `e39a369b` adds a dev-guide doc.
- Working tree dirty: `REQUIREMENTS.md` is **deleted but uncommitted** (` D REQUIREMENTS.md`); `.agents/`, `.claude/`, `.codex/`, `.gitattributes`, `.trellis/` untracked.

## Deleted / renamed patterns specs may still describe as existing

Backend (all still referenced by specs):
- `backend/app/api/v1/module_*/**` — whole tree gone; cited in `backend/*` (5 files), `architecture/*` (3 files), `security/*`, `operations/*`, `guides/*`. New path: `backend/app/modules/<domain>/<feature>/`.
- `module_system/dept/*` (spec's canonical slice) → `backend/app/modules/system/dept/*`.
- `module_common/health/controller.py` → `backend/app/modules/monitor/health/controller.py`.
- `module_storage/**` (incl. `core/encrypt.py`, `transfer/{engine,ws_manager}.py`) → `backend/app/modules/task/storage/**`; encryption helper now `backend/app/utils/crypto_util.py`.
- `module_system/chat/{controller,service,ws_manager}.py` → gone; AI chat at `backend/app/modules/ai/chat/*` (no ws_manager).
- `backend/app/core/ws_manager.py` → `backend/app/core/sse_manager.py`.
- `backend/app/init_app.py` (referenced by `system-boundaries.md:12,31,82`, `directory-structure.md:7,48,94`, `logging-guidelines.md:22`, `configuration-and-startup.md:86`, `cross-layer-thinking-guide.md:52`) — deleted; lifespan now `backend/app/__init__.py`.
- `module_monitor/resource/**` — deleted entirely.
- `core/rate_limiter.py` — added then deleted.
- `tests/test_api_module_system.py` — deleted.

Frontend:
- `router/routes.ts` exports `NestedRouterParent`, `ROUTE_COMPONENT_NESTED_PARENT` — removed (`24589ad9`); `RouteMeta.remountOnFullPath` removed.
- `utils/socket/index.ts` deleted → `utils/sse/index.ts`.
- `store/modules/chat.store.ts`, `layouts/fa-chat-window/index.vue`, `views/module_system/chat/index.vue`, `api/module_system/chat.ts` — all deleted (`http-and-realtime-contracts.md:81` still cites the last one).
- `views/module_storage/**` → `views/module_task/storage/**`; `api/module_storage/**` still exists (api/views naming inconsistent).
- `views/dashboard/screen/**` (~34 files) — deleted.
- `components/forms/fa-transfer/index.vue` — deleted (`components/others/fa-transfer` copy remains).
- `api/module_monitor/resource.ts` — deleted.

Root/deploy:
- `deploy.bat` deleted; `docker/deploy.sh` never survives (moved back to root `deploy.sh`; path valid, contents/behavior changed).
- `REQUIREMENTS.md` — in git but deleted in working tree (uncommitted).

Staleness ranking for the refresh: **backend/architecture (module paths + init_app.py) > architecture/http-and-realtime-contracts + operations/deployment-topology (WS→SSE, ws_manager) > frontend router/KeepAlive specs > testing/frontend-testing (test inventory) > operations/deployment-topology (deploy.sh/dockerignore) > security (rate limiting removed, module paths)**.
