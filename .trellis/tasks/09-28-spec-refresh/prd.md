# Spec refresh: 对齐 2026-09 代码现状并同步回 registry

## Background

- `.trellis/spec/` 于 2026-09-28 从 registry `gh:passional/fastapiadmin-trellis-specs/.trellis/spec` 导入（registry 发布于 2026-08-20，内容基线为 pi-platform commit `859b162d`，2026-08-19 / PR #507）。
- 基线之后 master 有 37 个提交（2026-09-03 → 09-17，HEAD `1cd12c72`），核心变更：后端模块大重构（`app/api/v1/module_*` → `app/modules/<域>/<功能>/`、`init_app.py` 删除、路由注册收敛到 `api/v1/routers.py` 的 `DOMAIN_CONTROLLERS`）、WS→SSE 迁移、router/KeepAlive 重写（`NestedRouterParent` 移除、新增 route-invariants 测试）、多项安全修复落地（操作日志脱敏、refresh 轮换、登录限流、下载路径收敛）、部署脚本/Dockerfile 变更、测试清单变化。
- 4 份并行审计（见 `research/`）确认：12 个抽查文件全部含错误声明。各层需修改比例：architecture ~75-85%、operations ~65-75%、security ~65-75%、testing ~50-60%、backend ~45-55%、frontend ~10-15%（另有 SKILL §7/§9 两大覆盖缺口）、guides ~10-15%（增补）。

## Goals

1. `.trellis/spec/` 全部 7 层对齐 HEAD `1cd12c72` 的代码现状。**事实冲突裁决顺序：当前代码 > SKILL.md > 既有 spec**。
2. 将 SKILL.md §7（KeepAlive/缓存不变量）、§9（dist 产物验证）及其他缺失约定（codegen 重启、Jinja2 作用域坑、首跑流程、权限标识格式等）补入对应 spec 层。
3. 修正 SKILL.md 自身 4 处不准确（bcrypt→PBKDF2、"仓库内存在预构建 dist 副本"表述、`.env.development` 文件名、`VITE_APP_WS_ENDPOINT` 适用范围）。
4. 更新后的 spec 同步回 `passional/fastapiadmin-trellis-specs` main 分支（直接 push），随后本地 `trellis update -f` 重建 hash 基线。

## Non-goals

- 不修改任何业务代码（SKILL.md 除外，且仅限上述 4 处修正）。
- 不新增 spec 层；层内可新增文件（如 `frontend/routing-and-caching.md`）。
- 不将 `.trellis/` 提交进 pi-platform 主仓库（另行决策）。
- 不重写审计中已验证准确的内容（如 backend 异常映射表、frontend FaDialog/useTable 契约、error-handling 主体）。

## Requirements

- R1: spec 语言保持英文（与 registry 现有文档一致）。
- R2: 删除/修正所有指向已不存在路径的引用：`app/api/v1/module_*`、`app/init_app.py`、`core/ws_manager.py`、`tests/test_api_module_system.py`、`NestedRouterParent`、`RouteMeta.remountOnFullPath`、`/common/health/{live,ready,check,stream}`、`/system/chat/ws`、`module_storage/transfer/ws_manager.py`、`rate_limiter.py`、`deploy.bat`、`utils/socket/`、`chat.store.ts`、`fa-chat-window`、`views/module_system/chat`、`views/dashboard/screen`、`_PUBLIC_WRITE_PATHS`、`sql/data/`。历史沿革如需提及必须显式标注为历史。
- R3: "安全缺口"类声明按当前代码改写为现状描述（脱敏已实现、refresh 轮换+重放撤销已实现、登录限流已存在、下载收敛已修复、忘记密码接口不再改密码）。确实仍存在的缺口（如 upload magic-byte 仅告警、SVG 放行、权限缓存会话内不刷新等）保留。
- R4: 新增内容优先落在审计建议的目标文件（见各 research 报告的 MISSING 表）。
- R5: SKILL.md §9 写入 spec 时按代码事实校正：dist 目录当前不存在且被 gitignore，spec 应描述"构建→rsync --delete 同步→产物 grep 验证"的流程契约，而非"仓库内已有预构建副本"。
- R6: 每层 index.md 的链接与文件清单同步骤更新；改动后全库引用路径必须真实存在。
- R7: 同步 registry 时保持仓库结构（`.trellis/spec/` 前缀），commit message 说明基线推进（859b162d → 1cd12c72）。

## Acceptance Criteria

- [ ] AC1: `grep -rE 'api/v1/module_|init_app\.py|ws_manager|test_api_module_system|NestedRouterParent|remountOnFullPath|/common/health|system/chat/ws|rate_limiter|deploy\.bat|utils/socket|chat\.store|fa-chat-window|dashboard/screen|_PUBLIC_WRITE_PATHS|sql/data' .trellis/spec/` 无命中（或仅命中标注为历史上下文的行，逐条人工确认）。
- [ ] AC2: 所有 spec 文件中出现的 `backend/...`、`frontend/...` 代码路径抽样核验存在（每层 ≥5 条）。
- [ ] AC3: SKILL §7 十大坑可在 `frontend/`（routing/caching 相关）逐条找到；§9 产物验证流程可在 `operations/deployment-topology.md` + `testing/contract-and-deployment-testing.md` 找到；codegen 重启 + Jinja2 坑可在 `architecture/code-generation-and-routing.md` 找到；首跑流程/默认账号/sqlite 零配置可在 `operations/configuration-and-startup.md` 找到。
- [ ] AC4: SKILL.md 4 处修正完成，其余内容不动。
- [ ] AC5: `python3 ./.trellis/scripts/get_context.py --mode packages` 正常输出且 7 层 index 可解析。
- [ ] AC6: specs 仓库 main 收到更新 commit；本地随后执行 `trellis update -f` 后再次 `trellis update --dry-run` 显示 spec 全部 Unchanged（hash 基线一致）。

## Design note

跳过 `design.md`：本任务是纯文档/规范更新，无架构决策与代码边界变更；4 份审计报告 + `research/canonical-facts.md` 即设计输入。执行计划见 `implement.md`。

## Notes

- 审计中确认准确、明确"无需修改"的声明清单保存在各 research 报告内，实现者不得顺手改动。
- 前端存在既有代码不一致（`api/module_storage/` vs `views/module_task/storage/`），spec 如实记录该现状并标注，不要"纠正"成任一侧。
