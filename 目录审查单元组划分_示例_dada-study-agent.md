## 附录 A：dada-study-agent 建议单元组（实例）

> 项目专属附录；换项目时删除。基于 `git ls-files` 实测（跟踪文件 319，`AGENTS.md` 32 份）。**组按模块边界切，不按文件数切**；文件数列仅用于判断是否再拆。**建议起点，用户可调整。**

| 组 | 目录 | 划分理由（模块边界） | 模块类型 → 区块集合 | 文件 | 已有文档 |
| --- | --- | --- | --- | --- | --- |
| G-backend-core | `backend/auth` `backend/config` | 身份、权限矩阵与凭据加载同属安全边界，其他模块都依赖 | 认证/权限 + 配置 → C1-C4 + 业务规则 + 对外合同表 | 5+7 | 是 |
| G-backend-app | `backend/app` | 组合根：路由、中间件、装配、启动，唯一把各模块接起来的地方 | 集成/组合根 → C1-C4 + 对外合同表 + 端到端数据流 + 状态清单 | 7 | 是 |
| G-backend-agent | `backend/agent` | 意图→固定 action 的受控映射层，LLM 与业务之间唯一的翻译边界 | 领域服务 → C1-C4 + 业务规则 + 状态清单 + 对外合同表 | 10 | 是 |
| G-backend-data | `backend/store` `backend/settings` `backend/memory` `backend/session` | 后端自有 SQLite 与其上的读写服务，共享库文件与 schema 事实 | 持久化 → C1-C4 + 数据资产表 + 业务规则 | 4+5+1+4 | 是 |
| G-backend-integration | `backend/providers` `backend/llm` `backend/voice` `backend/notify` `backend/gateway` `backend/dada` | 都是对进程外部的适配层（provider/核心 CLI/TTS/通知），边界同源 | 集成/适配器 → C1-C4 + 对外合同表 + 端到端数据流 + 已知限制 | 4+4+5+4+4+0 | 是 |
| G-backend-tests | `backend/tests` | 后端断言层，需与 `backend/**` 实现对照才可判读 | 测试 → C1-C4 + 已知限制 | 56 | 是 |
| G-mcp-deploy | `mcp` `packaging` `deploy` `devtools` `scripts` | 对外暴露面与交付形态：MCP 固定 action、打包、部署、开发工具 | 集成/适配器 + 配置 → C1-C4 + 对外合同表 + 已知限制 | 2+5+5+7+1 | 部分是 |
| G-root-tests | `tests`（`test_outline_deploy_retirement.py` + `promptfoo/`） | 跨模块金标准与 irreversible 数据集，不属任何单一模块 | 测试 → C1-C4 + 已知限制 | 10 | 否 |
| G-web-root | `web` 根（`AGENTS.md`/`DELIVERY.md`/`index.html`/`package*.json`/`playwright.config.ts`/`tsconfig.json`/`vite.config.ts`/`vitest.config.ts`/.gitignore）+ `web/src` 根（`App.tsx`/`main.tsx`/`settingsReturnTo.ts` 及 3 个测试） | Web 壳与路由：`App.tsx` 是角色门与路由装配的唯一权威 | 页面/路由 → C1-C4 + 端到端数据流 + 状态清单 | 10+6 | 是 |
| G-web-api | `web/src/api` | same-origin REST 客户端与形状校验，页面与服务端之间唯一的合同层 | API 客户端 → C1-C4 + 对外合同表 + 端到端数据流 | 17 | 是 |
| G-web-chat | `web/src/chat` | provider-first 工作区：聊天与模型服务配置共存于同一装配根 | 页面/路由 → C1-C4 + 端到端数据流 + 状态清单 | 24 | 是 |
| G-web-child | `web/src/child` `web/e2e` | 孩子端唯一入口与其端到端验收同源（`/child/chat`） | 页面/路由 + 测试 → C1-C4 + 端到端数据流 + 状态清单 + 已知限制 | 11+4 | 是 |
| G-web-parent | `web/src/parent` | 家庭管理（设备、成员、纲要、虚拟设备）同属家长操作面 | 页面/路由 → C1-C4 + 端到端数据流 + 状态清单 | 23 | 是 |
| G-web-shared | `web/src/components` `web/src/styles` `web/src/style-samples` `web/src/mock` | 跨端共享展示与视觉 token；样板与 legacy mock 只作旁证 | 纯展示 + 配置 → C1-C4 + 对外合同表（token） | 9+5+12+2 | 是 |
| G-root-cross | 根 8 个跟踪文件 + `docs/`（25） | 仓库级规则、打包入口与设计文档，横切全部模块 | 横切 → C1-C4 + 对外合同表 + 已知限制 | 33 | 是 |

## 附录 B：dada-study-agent 全局配置与已知事实（实例）

> 把上面「全局配置」段的占位符替换为下列值即可。

```text
REVIEW_ROOT       = D:\0-development\projects\dada-study-agent
REVIEW_SCOPE      = 全盘（按 4.0 exclude）
PROJECT_DOC       = AGENTS.md          # CLAUDE.md 只是指向 AGENTS.md 的指针，不作为模块文档
PROJECT_RULES     = auto-discover      # 根 AGENTS.md 五条核心不变量 + docs/ 设计文档为最高优先级
WORKER_AGENT      = <你的 Orca agent id，如 opencode / codex>
REVIEW_REPORT_DIR = D:\0-development\projects\dada-study-agent\.tmp\code-reviews   # 已在 .gitignore:38
RUN_OBJECTIVE     = dada-study-agent 全目录代码审查与模块知识库重建
base ref          = bin448482/dada-unified-family-ui（HEAD 60c63fd）
```

**必须登记、不得自行处置的事实**

1. **base 不是 `main`/`master`**：主工作树在 `bin448482/dada-unified-family-ui`（HEAD `60c63fd`）；所有审查 worktree 固定从该 ref 起。
2. 已有 5 个 sibling worktree（`authenticated-browser-acceptance`、`backend-chat-baseline`、`parent-chat-context-fix`、`virtual-child-chat-stability`、`virtual-device-atomic-design`）：`--name` 必须带 `review-<shortId>-` 前缀避免冲突，且不得复用它们做审查。
3. 主树 8 条未提交改动（5 个已跟踪截图被删 + 3 个未跟踪新截图）全在 `temp_snapshot/`：该目录**排除审查**，只在 Baseline 如实记录，worker 不得触碰。
4. `project_code_promts/` 是本提示词自身，排除审查范围（否则会审查审查规则）。
5. `*.sqlite3`（`backend.sqlite3`、`memory.sqlite3`）只登记存在，禁止读取；`opencode.json` 含模型配置，只登记存在，禁止读取/修改；`build/` 为未跟踪构建产物，排除。
6. `backend/dada/` 现仅剩 `AGENTS.md`（`*.py` 已在 `c312373` 全部删除）→ 按 8.6 只登记退役建议。
7. 6 份 `web/src/**/AGENTS.md` 含指向 `.tmp/code-reviews/<具体 run>` 的易腐指针（`api`、`child`、`components`、`mock`、`parent`、`styles`）→ 按 8.5 清理并逐条登记。
8. `web/src/api/AGENTS.md` 头部写死审查 worktree 路径 `D:\0-development\orca\workspaces\dada-study-agent\review-web-api` 与旧 HEAD → 属易腐快照，按 8.5 处理。
9. **`web/src/chat/AGENTS.md` 已在准备阶段重做完成，作为本轮格式基准**：只写符号锚点、不写行号、不写审查结论，23/23 文件全覆盖。worker 以它为基准，不要退回散文体。
