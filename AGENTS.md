# AGENTS.md

Infinitum：Next.js App Router 新闻聚合系统。  
默认解释语言：中文。

## UI presentation preference

- 保持页面和配置界面简洁，优先使用字段标题、控件和必要的操作反馈。
- 不额外添加解释性辅助段落、提示卡片或冗长说明；只有确实影响操作理解的信息才展示。
- 能通过界面结构表达的规则，不重复写成帮助文案。

## Overview

流水线：RSS 拉取 → 全文抽取（必要时）→ AI 分析/翻译/摘要 → 聚类去重 → 公开展示 + Admin 管理。

## Commands

```bash
# Dev
npm run dev              # Next.js dev (localhost:3000)
npm run worker           # 后台任务 worker

# Database
npm run db:setup         # 初始化/升级 dev SQLite
npm run db:test:setup    # 重置并初始化 test DB
npm run prisma:generate
# Docker/production SQLite uses the generated schema snapshot and idempotent
# runtime schema upgrades; no Prisma migration history is shipped or executed.

# Test / lint / build
npm test                 # 全量测试 + coverage（会重置 test DB）
npm run test:watch
vitest run tests/integration/feed-api.test.ts   # 单文件示例
npm run lint
npm run build
```

### 按改动面建议的验证命令

按改动面选择，至少跑相关子集；大改动再跑全量：

| 改动面 | 建议命令 |
|---|---|
| Feed 查询/时间窗/缓存 | `vitest run tests/integration/feed-api.test.ts tests/unit/feed-range.test.ts` |
| Ingestion / 任务 | `vitest run tests/integration/ingestion-service.test.ts tests/integration/background-task-service.test.ts` |
| DB schema / setup | `npm run schema:generate && vitest run tests/integration/sqlite-setup.test.ts` |
| 宽改动 / 收工 | `npm test` 与/或 `npm run lint` |

## Layout

```text
src/app/           App Router 页面与 Route Handlers
  page.tsx         公开展示首页
  login/           管理登录
  admin/           管理台（content / settings / monitor）
  api/feed/        公共 feed API
  api/admin/       管理 API（需 session）
  api/ingest/      拉取触发
src/lib/
  feed/            公共 feed 查询层（service 为主入口）
  ingestion/       RSS → DB 管线
  clusters/        聚类
  tasks/           后台任务（service / worker / scheduler / handlers）
  items/           单条再生/重分析
  settings/        配置 CRUD + AI 配置
  ai/              OpenAI 兼容客户端
  tags/ events/ aggregation/  标签、事件、聚合等
prisma/schema.prisma
scripts/generate-sqlite-schema.mjs # compile current Prisma schema to SQLite DDL snapshot
scripts/setup-sqlite.mjs     # SQLite schema snapshot/runtime setup and small legacy upgrades
config/infinitum.config.json   # 首次导入模板；运行时配置在 DB
tests/{unit,integration}/
```

## Layer order

`src/lib/*` → `src/app/api/*` → UI 组件。  
共享参数解析 / DTO 组装抽到 helpers；Feed 读路径用 `src/lib/feed/service.ts`，不要绕过直接玩缓存。

## Hard constraints（禁忌）

1. **Feed 时间过滤用 `items.createdAt`，不要改成 `publishedAt`**  
   语义是「系统入库时间窗」，不是源站发布时间。
2. **影响公共 feed 的写操作必须 `invalidateFeedCache()`**（`src/lib/feed/cache.ts`）。
3. **Schema 只改 `prisma/schema.prisma`，并同步维护 `scripts/setup-sqlite.mjs` 的升级逻辑和测试**；Docker 和本地初始化基于生成的 schema snapshot 幂等执行，不使用 Prisma migration history。
4. **后台任务有双路径**：Admin 同步执行 + worker 队列；改 handler 两边都要考虑。
5. **Ingestion 优化优先总耗时**，不是盲目加并发；减少串行步骤与无效 DB 写。
6. **改 feed 查询时核对**：cluster/单条混排、分组计数、标题搜索、时区日界。
7. **高频写**：批处理 / 节流 / 阶段末 flush，避免逐条狂写。

## Env

`.env` 至少需要：

- `DATABASE_URL`（如 `file:./prisma/dev.db`）
- `ADMIN_PASSWORD`
- `ADMIN_SESSION_SECRET`

运行时配置在 DB（源、黑名单、AI、prompt、调度等）；`config/infinitum.config.json` 仅首次导入。

## Admin

- Login: `http://localhost:3000/login`
- Admin: `http://localhost:3000/admin`
- Settings: `http://localhost:3000/admin/settings`
- Monitor: `http://localhost:3000/admin/monitor`

Session：HTTP-only cookie，由 `ADMIN_SESSION_SECRET` 签名。

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **infinitum**. Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> Index stale? Run `node .gitnexus/run.cjs analyze --index-only` from the project root — it auto-selects an available runner. No `.gitnexus/run.cjs` yet? `npx gitnexus analyze --index-only` (npm 11 crash → `npm i -g gitnexus`; #1939).

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows. For regression review, compare against the default branch: `detect_changes({scope: "compare", base_ref: "main"})`.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `query({search_query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `context({name: "symbolName"})`.
- For security review, `explain({target: "fileOrSymbol"})` lists taint findings (source→sink flows; needs `analyze --pdg`).

## Never Do

- NEVER edit a function, class, or method without first running `impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `rename` which understands the call graph.
- NEVER commit changes without running `detect_changes()` to check affected scope.

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/infinitum/context` | Codebase overview, check index freshness |
| `gitnexus://repo/infinitum/clusters` | All functional areas |
| `gitnexus://repo/infinitum/processes` | All execution flows |
| `gitnexus://repo/infinitum/process/{name}` | Step-by-step execution trace |

<!-- gitnexus:end -->
