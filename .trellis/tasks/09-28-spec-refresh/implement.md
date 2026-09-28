# Implement plan — spec refresh

事实来源优先级：**当前代码 > `research/canonical-facts.md` > 各审计报告 > SKILL.md**。所有层统一使用 canonical-facts.md 的术语与路径，保证跨层一致。审计中标注 "Still accurate" 的内容不得改动。Spec 正文用英文。

## Stage 1 — 分层更新 `.trellis/spec/`（按过时程度排序）

- [x] 1.1 `architecture/`（~75-85%）：`system-boundaries.md`（modules/ 模型、routers.py 注册、discover fail-fast、plugin.toml 位置、测试引用）；`http-and-realtime-contracts.md`（删 /system/chat/ws、transfer→SSE、health→/monitor、AI WS subprotocol 载体 + 4003、VITE_APP_WS_ENDPOINT 范围）；`code-generation-and-routing.md`（模板集 +package_init/plugin.toml、preview/ZIP 反转、sync_db/preview 端点、Jinja2 作用域坑、重启后端不变量、§7 路由缓存契约引用）。依据：`research/arch-ops-sec-testing-audit.md` §1。
- [x] 1.2 `backend/`（~45-55%）：`directory-structure.md` 重写（modules/ 模型 + 单注册表 + app/__init__.py + plugin 约定 + core/ 清单）；`logging-guidelines.md` 脱敏段改为现状（_redact_sensitive 请求+响应+表单）；`database-guidelines.md`（initialize 迁移模型、sql/*.json 路径、storage 路径、chat commit 声明删除）；`error-handling.md`（仅 3 处 dept 路径）；`quality-guidelines.md`（测试清单、restart、Jinja2、权限标识格式、uv run 命令）；`index.md` 引用路径。依据：`research/backend-audit.md`。
- [x] 1.3 `operations/`（~65-75%）：`configuration-and-startup.md`（--env 已修复、lifespan 顺序 + init_agno_tables、迁移模型、seed 路径、缺失配置键、首跑流程/默认账号/sqlite 零配置）；`deployment-topology.md`（Dockerfile COPY、bind mount 注释、健康探针只有 check/stream、deploy.sh prune 范围、§9 三份 dist 产物表 + rsync --delete + register_frontend 坑）。依据：`research/arch-ops-sec-testing-audit.md` §2。
- [x] 1.4 `security/`（~65-75%）：`authentication-and-secrets.md`（refresh 轮换+重放撤销、滑动会话+绝对上限、登录限流 Redis+nginx、WS 载体、系统聊天通道删除、PBKDF2）；`untrusted-content-and-audit-data.md`（脱敏已实现、_PUBLIC_WRITE_PATHS 删除、下载收敛已修复、captcha 加固、保留仍存在的缺口）；`authorization-and-data-scope.md`（仅 /monitor/health/stream 路径 + v-hasPerm 契约补充）。依据：同上 §3。
- [x] 1.5 `testing/`（~50-60%）：`backend-testing.md`（清单 conftest/test_main/test_migrations、迁移守卫）；`frontend-testing.md`（两个测试文件、route-invariants 为路由不变量的 canonical 测试、vitest 别名全集）；`contract-and-deployment-testing.md`（健康检查路径、镜像 COPY、SSE 化、refresh 轮换、脱敏回归守卫、§9 产物验证流程）。依据：同上 §4。
- [x] 1.6 `frontend/`（~10-15% + 覆盖缺口）：新增 `routing-and-caching.md`（SKILL §7 十大坑全量，route-invariants.spec.ts 为可执行契约）；`quality-guidelines.md` + `type-safety.md`（测试清单、ts:check 统一）；`directory-structure.md`（router 管道描述、补 directives/locales 等目录）；`component-guidelines.md`（name == route_name 强约束）；`hook-guidelines.md`（onActivated/onDeactivated 强制、WS 守卫 !== CLOSED、先摘回调再 close）；`state-management.md`（keepAliveExclude 驱逐机制、temp 为 vestigial 注记）；`index.md` 加入新文件链接。依据：`research/frontend-audit.md`。
- [x] 1.7 `guides/`（增补）：`index.md` 触发清单 + 路由缓存/codegen 重启/dist 验证三项、引用列表补 backend/frontend/SKILL.md；`cross-layer-thinking-guide.md:52` 的 `init_app.py` 引用改为 `app/__init__.py`。
- [x] 1.8 各层 `index.md` 的 Pre-Development Checklist / Quality Check 与新文件对齐。

## Stage 2 — SKILL.md 修正（仅 4 处，其余不动）

- [x] 2.1 §2 首次启动第 4 条：bcrypt → PBKDF2-HMAC-SHA256（600k）。
- [x] 2.2 §9 首段："仓库内存在多份预构建副本并直接被打进部署" → 改为部署链路存在三份产物目录（均 gitignore），构建后必须同步（保留流程与案例）。
- [x] 2.3 §4 环境变量行：`.env.development` → `.env.development.example`（本地复制为 `.env.development`）。
- [x] 2.4 §2/§4 `VITE_APP_WS_ENDPOINT`：补充"健康/传输 SSE 也使用该端点"。

## Stage 3 — 一致性校验（本地）

- [x] 3.1 AC1 grep：`grep -rnE 'api/v1/module_|init_app\.py|ws_manager|test_api_module_system|NestedRouterParent|remountOnFullPath|/common/health|system/chat/ws|rate_limiter|deploy\.bat|utils/socket|chat\.store|fa-chat-window|dashboard/screen|_PUBLIC_WRITE_PATHS|sql/data' .trellis/spec/` → 0 命中或仅历史标注行。
- [x] 3.2 AC2 路径抽样：每层 ≥5 条 `backend/`、`frontend/` 路径 `test -e` 核验。
- [x] 3.3 AC5：`python3 ./.trellis/scripts/get_context.py --mode packages` 正常；7 个 index.md 链接的文件全部存在。
- [x] 3.4 交叉抽查：canonical-facts.md 关键条目（DOMAIN_CONTROLLERS、sse_manager、/monitor/health/check、PBKDF2、rsync --delete）逐条 grep 确认已写入对应层。

## Stage 4 — 同步回 registry（specs 仓库）

- [x] 4.1 `rsync -a --delete .trellis/spec/ /tmp/fa-specs/.trellis/spec/`（克隆已在 /tmp/fa-specs，main 分支）。
- [x] 4.2 `git -C /tmp/fa-specs diff --stat` 人工审查（应只有 .trellis/spec/ 变化）。
- [x] 4.3 commit（message 注明基线推进 859b162d → 1cd12c72 与主要修订面）+ `git push origin main`。
- [x] 4.4 本地 hash 基线重建：`trellis update -f`，随后 `trellis update --dry-run` 确认 spec 全部 Unchanged（AC6）。

## 回滚点

- Stage 1-2 前：本地 spec 有导入前备份（`.trellis/.backup-2026-09-28T06-38-46/`、`/tmp/trellis-spec-backup-20260928/`）；SKILL.md 受 pi-platform git 跟踪，可 `git checkout -- SKILL.md`。
- Stage 4 后：specs 仓库可 `git revert`；本地可重跑 `trellis update -f` 回拉。

## 执行方式

Stage 1 各层相互独立、由 canonical-facts.md 保证一致性 → 按层并行派发 `trellis-implement` 子代理（每层一个，prompt 以 `Active task:` 开头并指定必读 research 文件）；Stage 2-4 由主会话执行；Stage 3 校验派 `trellis-check`。

## Validation commands

```bash
grep -rnE 'api/v1/module_|init_app\.py|ws_manager|test_api_module_system|NestedRouterParent|/common/health|system/chat/ws' .trellis/spec/   # expect no hits
python3 ./.trellis/scripts/get_context.py --mode packages
trellis update --dry-run   # after Stage 4: spec files all Unchanged
```
