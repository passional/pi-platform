# Frontend Spec Layer Audit — Staleness & Gap Report

Audited: all 7 files in `.trellis/spec/frontend/` (index, directory-structure, component-guidelines, hook-guidelines, state-management, type-safety, quality-guidelines), cross-checked against `SKILL.md` §1/2/4/7/9, the actual code under `frontend/web/src/`, `frontend/app/src/`, `frontend/web/package.json`, and adjacent spec dirs (`.trellis/spec/testing/`, `architecture/`, `operations/`).

Headline: the spec layer's **path/name claims are almost all still accurate** (~90%). The staleness is concentrated in two places: (1) the **test inventory** (misses the router-refactor test added in commits `24589ad9` / `740e334d`), and (2) **entire absence of SKILL §7 (KeepAlive/cache invariants) and §9 (dist artifact verification)** — the two highest-value bodies of knowledge added after 2026-08-20. There are **no references to the removed `NestedRouterParent`** anywhere in the spec (grep confirmed it exists only in `SKILL.md` and the new test), so the refactor did not invalidate any existing sentence — it just isn't captured.

## A. STALE / WRONG claims

| # | Spec file : line | Claim | Actual code evidence | Severity |
|---|---|---|---|---|
| 1 | `testing/frontend-testing.md:10-11` | "The only current test file is `src/__tests__/smoke.spec.ts`" | Two test files exist: `frontend/web/src/__tests__/route-invariants.spec.ts` (159 lines, added by commits `24589ad9` "remove nested parent component" + `740e334d` "add route invariant checks for deep skip rendering") and `smoke.spec.ts` (34 lines) | **High — factually wrong** |
| 2 | `frontend/quality-guidelines.md:52-54` | "The existing `frontend/web/src/__tests__/smoke.spec.ts` verifies importable runtime enums but is intentionally small." | Same: `route-invariants.spec.ts` is the current enforcement of the route-cache invariant and is unmentioned | **High — incomplete; omits the test that guards the router refactor** |
| 3 | `frontend/directory-structure.md:18` | `router/ # routes, dynamic menu loading, guards, refresh` | Actual `frontend/web/src/router/`: `MenuProcessor.ts`, `guards.ts`, `index.ts`, `refresh.ts`, `route-loader.ts`, `routes.ts`. The dynamic pipeline is `ComponentLoader` / `RouteTransformer` / `RouteRegistry` all inside `route-loader.ts` (SKILL §1 names "guards/MenuProcessor/RouteRegistry 动态路由管道"). The spec's one-liner predates/mis-describes the pipeline and never names `MenuProcessor` or `route-loader.ts` | Medium — stale description |
| 4 | `frontend/directory-structure.md:11-23` | Admin Web `src/` tree | Omits real top-level dirs used by SKILL §1: `directives/permission/` (exists, `v-hasPerm`), `locales/langs/` (exists, `zh.json`/`en.json`), plus `constants/`, `config/`, `enums/`, `mock/`, `assets/`, `App.vue` | Low-Medium — incomplete |
| 5 | `testing/frontend-testing.md:6-8` | "`vitest.config.ts` … maps `@`, `@utils`, and `@stores`" | `frontend/web/vitest.config.ts` maps `@`, `@views`, `@imgs`, `@icons`, `@utils`, `@stores`, `@plugins`, `@styles`, `@api`, `@fa_imgs` | Low — incomplete |
| 6 | `frontend/state-management.md:41` | App persist plugin "persists every store by ID except `temp` and `theme`" | `frontend/app/src/store/persist.ts` does exclude `['temp','theme']` (accurate as code description), but **no store with id `temp` exists** — store ids are `appConfig`, `appUserInfo`, `theme` (grep `defineStore(`). The `temp` exclusion is vestigial | Low — vestigial/confusing |
| 7 | `frontend/quality-guidelines.md:11` / `type-safety.md:12` | Run `pnpm type-check` | Both `type-check` (`vue-tsc --noEmit`) **and** `ts:check` (`vue-tsc --noEmit --skipLibCheck`) exist in `frontend/web/package.json`. Not wrong, but SKILL §2/§5 prescribe `pnpm ts:check`; the two are **not the same command** | Low — divergence from SKILL |

Claims that are **verified accurate** (spot-list, so you know what not to touch): `plugins/index.ts` order Store→Router→Directives→error-handler→terminal→i18n→CodeMirror; `components/modal/fa-dialog/index.vue` (typed props/emits, computed `modelValue`, `inheritAttrs:false` + attrs computed, `defineExpose`, Escape + Ctrl/Cmd+Enter); `FaTable`/`FaForm`/`FaSearchBar`/`FaDialog`/`FaDrawer` all exist; `layouts/fa-settings-panel/composables/` exists (4 files); `hooks/core/useTable.ts`, `hooks/index.ts` barrel (selective exports), `useTableColumns`/`useCrudDialog`/`useCrudForm`/`useLoading`/`useAuth`/`useImportExport`/`useConfirm` exist; `useTable` `AbortController` claim is precisely correct (`apiFn(requestParams)` with no signal; `currentController.signal.aborted` checked post-response; `cancelRequest()` only aborts the local controller); store barrel + `refreshAppCaches()`; setting persist `{ key:"setting", storage: localStorage }`; `utils/auth`; web tsconfig flags (`noUncheckedIndexedAccess`, `noImplicitOverride`, `isolatedModules`, `noEmit`); app `tsconfig.base.json` + `tsconfig.json` types; web ESLint `@typescript-eslint/no-explicit-any: "off"`; app has no `test` script; `eslint.config.mjs`/`.prettierrc.yaml`/`.stylelintrc.cjs`/`commitlint.config.cjs` all present; Prettier 2-space/double-quote/semi/100/trailingComma-es5; app `useListPage` `list` vs backend `PageResultSchema.items` vs web ambient `items` mismatch (all three confirmed: `backend/app/core/base_schema.py:124-133`, `frontend/app/src/types/global.d.ts:85-93`, `frontend/web/src/types/global.d.ts:66`); app `useAiChat.close()` + `closedByUser`; app `pages/mine/index.vue` empty catch comment; app `GlobalDialog.vue` MP-ALIPAY branch; `deptTreeType`/`searchSelectDataType` legacy interfaces in both `module_system/user.ts`.

## B. MISSING conventions (SKILL.md → suggested spec target)

**From SKILL §7 (KeepAlive 缓存与连接类资源):** none of this is in any spec file (grep for keepalive/include/exclude/route_name/worktab/readyState/max across `.trellis/spec/` returns only two vague sentences: `hook-guidelines.md:22,38`).

| SKILL §7 pitfall | Covered? | Suggested target |
|---|---|---|
| Single-layer route-view cache; `KeepAlive` exists only in `layouts/fa-page-content/index.vue`; cache key = leaf `path` (`routeViewCacheKey`) | No | new `frontend/routing-and-caching.md` (or `directory-structure.md` + `hook-guidelines.md`) |
| `include` derived from worktab `opened`; empty `opened` ⇒ `include` undefined ⇒ no prune | No | same |
| `keepAliveExclude` eviction in `worktab.store.ts clearAll()`, `removeKeepAliveExclude` on `openTab` | No | `state-management.md` (Avoid list) |
| Never add `:max` (LRU evicts still-open tabs) | No | `frontend/routing-and-caching.md` |
| Directory routes must keep `component: undefined` (`MenuProcessor.mapMenuNode` / `RouteTransformer.handleNormalRoute`); runtime fallback `warnInvalidRouteConfig` + dev warning in `fa-page-content` | No (nearest is `architecture/system-boundaries.md:59-70`, which only covers menu→component path resolution) | `frontend/routing-and-caching.md` + `architecture/code-generation-and-routing.md` |
| `defineOptions({ name })` **must equal backend menu `route_name`** for include/exclude matching | Partial — `component-guidelines.md:8` only says "name explicitly"; never states the equality requirement | `component-guidelines.md` |
| `onActivated`/`onDeactivated` mandatory for side-effectful pages; `onUnmounted` is not the sole cleanup point | Partial — `hook-guidelines.md:38-39` says "must consider both activated and deactivated states, as `useTable` does" | strengthen in `hook-guidelines.md` |
| WebSocket guard must be `if (ws && ws.readyState !== WebSocket.CLOSED) return` (blocking only `OPEN` leaks CONNECTING re-entry) | No (verified live at `frontend/web/src/views/module_ai/chat/index.vue:93`) | `hook-guidelines.md` / `type-safety.md` WebSocket section |
| Detach `onopen/onmessage/onerror/onclose` before `close()` | No | `hook-guidelines.md` |
| Duplicate-popup/duplicate-connection debugging path (grep the message string → include/exclude → login 404→replace race → `backend/sql/sys_menu.json`) | No | new `frontend/routing-and-caching.md` + `guides/` |

**From SKILL §9 (dist artifact verification):** entirely absent. Grep for `FRONTEND_DIST_DIR|backend/dist|rsync|build:prod|register_frontend` across `.trellis/spec/` returns **zero hits**.

| SKILL §9 element | Suggested target |
|---|---|
| Three dist copies + service entrypoints: `frontend/web/dist`, `docker/nginx/web/dist` (nginx alias `/usr/share/nginx/html/web/dist`, confirmed `docker/nginx/nginx.conf:104`), `backend/dist` (`path_conf.FRONTEND_DIST_DIR = BASE_DIR / "dist"`, confirmed `backend/app/config/path_conf.py:25`, mounted via `register_frontend` at `backend/app/__init__.py:127-129`); App H5 `docker/nginx/app/dist/build/h5` (`nginx.conf:111`) | `operations/deployment-topology.md` (it covers nginx dist generally at lines 51-56 and the "Missing frontend dist" row at 116, but never names the 3 copies or `backend/dist`) |
| `rsync -a --delete` requirement + `pnpm build:prod` | `operations/deployment-topology.md` |
| Artifact-feature grep verification (`grep -l "ai/chat/ws" docker/nginx/web/dist/js/*.js`, `readyState` guard check; property accesses and Chinese message strings survive minify) | `testing/frontend-testing.md` or `operations/deployment-topology.md` |
| "源码已修 ≠ 部署已修" rule / source-vs-dist mismatch first-response procedure | `operations/deployment-topology.md` |

**Also missing:** SKILL §4's route cache invariants and §1's `locales/langs/` i18n convention (`$t` + both `zh.json`/`en.json`) and `directives/permission/` are not in the frontend spec tree at all.

## C. CONTRADICTIONS (spec X vs SKILL Y)

1. **Type-check command** — spec: `pnpm type-check` (`frontend/quality-guidelines.md:11`, `type-safety.md:12`); SKILL §2/§5: `pnpm ts:check`. Both scripts exist but differ (`--skipLibCheck` only on `ts:check`). Pick one canonical command in the spec.
2. **Router pipeline description** — spec: "routes, dynamic menu loading, guards, refresh" (`directory-structure.md:18`); SKILL §1: "`routes.ts` 静态壳 + guards/MenuProcessor/RouteRegistry 动态路由管道"; SKILL §4 also adds the route-invariant checks. Spec omits `MenuProcessor`/`RouteRegistry`/`RouteTransformer` entirely.
3. **Test inventory** — spec (`frontend/quality-guidelines.md:52`, `testing/frontend-testing.md:10`): only `smoke.spec.ts`; code: `route-invariants.spec.ts` exists and is the executable contract for the router refactor.
4. **KeepAlive name matching** — spec (`component-guidelines.md:8`): "Name routed or keep-alive components explicitly with `defineOptions({ name: ... })`" (no linkage); SKILL §4: "`defineOptions({ name })` **必须与菜单的 route_name 一致**，否则页面缓存/缓存排除（exclude）会失灵". A genuine contradiction in strength/meaning, not just omission.

No contradiction on the removed `NestedRouterParent` — the spec never mentions it.

## D. Overall assessment

- **Accuracy of existing claims:** high. Of ~55 concrete path/name/config/behavior claims traced, ~50 verify exactly against current code (~90%). Only one is outright false (test inventory), one is a stale structural description (router pipeline), and three are incomplete enumerations.
- **Coverage of recent knowledge:** poor. SKILL §7 (10 sub-pitfalls) and §9 (4 sub-topics) are ~0% represented; the new route-invariant test is 0% represented.
- **Estimated edit need:** roughly **10-15% of existing lines** need edits/removals, **plus** ~2 new sections (or a new `frontend/routing-and-caching.md`) and ~6 additions to `operations/deployment-topology.md` / `testing/frontend-testing.md`. Treat the spec as "structurally correct but dangerously under-specified": an agent that reads only `.trellis/spec/frontend/` before touching routing, KeepAlive, or a frontend build will reproduce exactly the bugs SKILL §7/§9 were written to prevent.
- **Highest-priority fixes:** (1) fix the two stale test-inventory claims and document `route-invariants.spec.ts`; (2) add the §7 KeepAlive invariant block to `frontend/` (routing/caching); (3) add the §9 three-dist-copy + `rsync --delete` + artifact-grep contract to `operations/deployment-topology.md`; (4) state the `defineOptions name == route_name` rule in `component-guidelines.md`; (5) reconcile `ts:check` vs `type-check`.
