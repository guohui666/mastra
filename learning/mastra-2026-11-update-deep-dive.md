# Mastra 2026-11 更新梳理(10-24 → 11-18)

> 基准:`cabb887412`(2026-10-24)→ `a6a5b78085`(2026-11-18),upstream `mastra-ai/mastra` main。
> 共 **1591 个提交,336 个 feat**。feat 提交分布:factory 89、playground-ui 49、core 30、factory-ui 27、ui 25、observability 22、connect 10、mcp-apps-components 10、mcp 7、mastracode 7。
> 本文配套 HTML 讲解页:`.archify/` 下最新一期(旧版为 `architecture-mastra-framework-20261024-120117/`,基于 10 月代码)。

## 0. 版本跨度

| 包 | 10-24 | 11-18 |
|---|---|---|
| @mastra/core | 1.80.0-alpha.2 | **1.85.0** |
| @mastra/cli | 1.36.0-alpha.2 | 1.40.1 |
| @mastra/memory | 1.39.1-alpha.0 | 1.41.2 |
| @mastra/mcp | 2.6.0-alpha.0 | 2.10.0 |
| @mastra/server / deployer | 1.80.0-alpha.2 | 1.85.0 |
| @mastra/playground(-ui) | 1.36(-alpha.2)/66.0-alpha.2 | 1.40.1 / 70.1.0 |
| @mastra/rag | 2.7.0 | 2.8.1 |
| @mastra/connect | 1.2.0-alpha.0 | 1.4.1 |
| @mastra/platform | (新增) | 0.1.1 |
| @mastra/mcp-apps-components | (新增) | 0.2.0 |

---

## 1. Knowledge:跨会话共享知识图(core@1.82.0 引入,alpha)

**解决什么**:Memory 只装一段对话内的记忆;Knowledge 装**超出单次对话、需要跨线程/跨 agent/人机共享**的信息——政策、产品文档、约定、runbook、其他 agent 的学习成果。

**模型**(代码:`packages/core/src/knowledge/`):
- **page**:一个主题页,markdown 正文,可嵌套(`org:acme#policies/refund` = `org:acme` scope 下 `policies/refund` 页)
- **note**:附在页面上的短事实,带作者署名,不用重写页面即可追加
- **link**:页面间引用 `[[org:acme#policies/refund]]`,构成图而非文档列表
- **scope**:`org:acme` / `team:acme:support` 等隔离分区,由 server 声明每次 run 可持有哪些 scope
- **role**:viewer/contributor/editor/owner,由 scope 声明中的规则逐工具调用推导,不落库

**本月新增 API**(`packages/core/src/knowledge/index.ts`):
- `knowledge.movePage()` — 移动页面不丢 notes/pins(#30631)
- `knowledge.reembed(options)` — 对既有内容重跑向量化(`reembed.ts`,#30603)
- pinned 页面经 **signals** 进入 agent 上下文,编辑后运行中的 agent 下一轮即拿到新版本(`pinned.ts`、`pinned-state-processor.ts`)

**启用形态**:
```ts
const knowledge = new Knowledge({ storage });
// tools: true 给 agent 搜索/读/写工具;注册到 Mastra + KNOWLEDGE_ROUTES 后可在 Studio 浏览
const agent = new Agent({ name: 'a', instructions: '...', model, knowledge, tools: true });
```

**边界**:alpha,API 会变;无内置 importer(平台侧提供);scope 间内容可能经多 scope agent 泄漏;暂不支持过期清理。

文档:`docs/src/content/en/docs/knowledge/`(overview、load-your-content、search、pinned-context、scopes-and-access、agents-and-tools、patterns)+ reference 7 页。

---

## 2. Mastra Connect + @mastra/platform:托管集成与托管运行时

### 2.1 Connect(`packages/connect`,v1.4.1,Public Beta)

**解决什么**:不用自己写 OAuth、存凭证、刷新 token——把 Mastra Platform 项目上挂载的第三方连接(GitHub/Slack/Linear/Notion/Stripe/Snowflake…)直接变成 agent 的 toolset。凭证只存平台侧,工具调用走 Platform proxy,由平台注入凭证、刷新 token、审计计数。

**三种消费形态**:
1. `tools()` → `ToolsResolver`(`packages/connect/src/tools.ts:777`)— 直接 `new Agent({ tools })`;
   - `providers` 支持 allowlist 数组或 Record(`allowTools`/`disallowTools` glob、`requireApproval`、`connectionId` 钉连接)
   - 多连接自动注入 `<integrationId>__list_connections` 工具让 agent 按显示名选账号
   - catalog 中 `capabilities.mcp: true` 的集成经 `/v2/connections/:id/mcp` 动态发现 MCP 工具
2. `channels()` — Slack/Telegram/Discord/Teams 聊天通道凭证
3. `getCredential()` — 直接拿凭证调 provider 原生 SDK

**连接生命周期**:开发者 `mastra connect add linear` 走授权流(平台经 Nango 存凭证),或终端用户经 `userConnectRoutes()` 的 `/connect` 路由组自助连接(自带渲染页 `user-connect-page.ts`,按请求级 `resourceId` 隔离每个最终用户)。连接属于 organization、link 到 project;agent 每次运行按当前活跃连接动态解析工具,**连接增删免重启生效**。

**本月新 API**:
- `ConnectClient`(#30320,`src/connect-client.ts:129`)— platform `/v2` 集成 API 的 typed 客户端;`withResource(resourceId)` 派生最终用户作用域客户端
- `EventsClient`(`src/events.ts:203`)— SSE 订阅 `events.mastra.ai`,`subscribe(topic, { cursorStore, onGap })` 返回 AsyncIterable,断线自动重连 + cursor 恢复
- headless 连接表单(#30336,`packages/playground-ui/src/domains/connect/headless-connect.ts`)— Studio 与平台 dashboard 共用,直接驱动 Nango 前端 SDK
- Code Mode 集成(#29583,`src/code-mode.ts`)— `ConnectCodeMode { tools, instructions }`,基于 core `createCodeMode()`;函数声明超 ~10k token 自动切 `search_functions` 搜索模式

**重要边界**:客户端包开源(Apache-2.0),但**运行时强依赖 Mastra Platform 云**——凭证注入/OAuth 刷新在平台侧(Nango),默认端点 platform.mastra.ai / events.mastra.ai,不可自托管。仓库内 checked-in provider 32 个(`packages/connect/src/providers/`),attio/clay 等仅存于平台 catalog,经 MCP 动态发现。

### 2.2 @mastra/platform(v0.1.1,experimental)

core 同一抽象的 **hosted 替身**,一行切换托管:
- `PlatformMemory` extends core `MastraMemory` → 走 hosted memory HTTP API
- `PlatformKnowledge` extends core `Knowledge` → 委托 `knowledge-api.mastra.ai`
- `PlatformBrowser` extends StagehandBrowser → 按需开 Browserbase 会话(或直传已有 `cdpUrl`)
- `PlatformSandbox`/`PlatformFilesystem`(re-export `@mastra/platform-workspace`,direct-exec/e2b/private-net 多执行后端)

`new Agent({ memory: new PlatformMemory() })` 即换托管;README 定义了与 core+本地 PG 的 parity 验证矩阵。

文档:`docs/src/content/en/docs/mastra-platform/connect.mdx`;integrations 目录共 **168 个 .mdx**(本月新增约 147:vectors 20、documents 8、auth 9、channels 8、crm 3、email-calendar 3 等)。

---

## 3. Agent 能力杂项(core)

- **Skills 按搜索发现**(#26316,`packages/core/src/skills/agent-skills-resolver.ts`):不再把全部 skills 塞进 prompt 列表,agent 主动搜索定位,省 token
- **Scorer judge 获得完整 agent 上下文**(#29387):judge 拿 agent instructions + run context,评分更贴场景
- **`REPROCESS_PART_KEY`**(#29939,`packages/core/src/processors/stream-reprocess.ts`):自定义 output processor 可把一次输出拆成两个 chunk 发出
- **DynamicArgument 拿到认证用户**(#28612):工具动态参数回调可按用户定制
- **Chat SDK 日志走 Mastra logger**(#29392)
- mssql 后端支持 Observational Memory(#30716)

## 4. mastracode(TUI/CLI)

- 批准 plan 即作为 goal 启动(#28037)
- picker/输入框鼠标支持(#29701)、Shift+方向键/双击三击选文本(#29994)
- Mistral Voxtral 语音转文字 provider(#29363)
- 3–5 词自动会话标题(#30881);导出 `runTui` 供外部程序启动 TUI(#30371)
- `MASTRACODE_AGENT_STREAMS=off` 开关(#30020)
- Linear issue 与 PR Fixes 行联动(#30112)

## 5. 文档站新章节

- **docs/knowledge/**(7 页指南)+ reference/knowledge/*(7 页)
- **docs/mastra-platform/**:deploy(environments/github/regions/server/studio)、access-tokens、team-members、connect、knowledge、memory、observability
- **reference/server/builtin-mcp-tools.mdx**(内置 MCP 端点工具参考)
- **reference/processors/skills-processor.mdx**、**reference/signals/ring-signals.mdx**
- agents/resumable-streams.mdx、classifier.mdx
- 新模型 provider 页一批(zro、zerogpu、watt 等)

---

## 6. 核心运行时:storage v2、durable 执行与恢复

### 6.1 Storage v2 for workflow runs(#28299,experimental)

**解决什么**:v1 每次写入**重写整个 run 快照**,长流程/循环的写入量随步数爆炸;v2 改为 run header + 每步一条 entry,只写增量。

- Schema:`packages/core/src/storage/constants.ts`(`WORKFLOW_RUNS_SCHEMA`/`WORKFLOW_RUN_ENTRIES_SCHEMA`,标 `@experimental`);journal 层 `packages/core/src/workflows/run-journal.ts`
- 适配器:`stores/pg/src/storage/domains/workflows/index.ts`、`stores/libsql/...`(各 +990 行);共享测试套件 `stores/_test-utils/src/domains/workflows/run-journal.ts`
- 启用:`createWorkflow({ id: 'w', options: { storageVersion: 2 } })`(`packages/core/src/workflows/types.ts:698`)
- **限制**:仅 default/evented 引擎 + in-memory/@mastra pg/libsql;不能与 `pruneSnapshot` 同用;同步 `state`/`getStepResult` 会 throw,必须 `await getState()` / `getStepResultAsync()`

### 6.2 Evented workflows + durable agents 上 v2(#29370,experimental)

**解决什么**:evented 引擎每条 step 消息携带全量状态 + 历史 step 结果 + 父 run 副本,流量随步数**平方增长**。上 v2 后官方数据:16 次工具调用的 agent run 发布 98MB→11MB、写盘 75MB→3.1MB、耗时 4.1s→2.6s。

- 代码:`packages/core/src/workflows/evented/`(execution-engine/workflow/step-executor)、`packages/core/src/agent/durable/`(`create-durable-agent.ts`、`create-evented-agent.ts`)
- 用法:`import { createWorkflow } from '@mastra/core/workflows/evented'`;agent 侧 `createDurableAgent({ ..., storageVersion: 2 })`
- 两版均新增 `getState(keys?)` / `updateState(partial)`(增量合并,并行分支互不覆盖)。默认仍 v1

### 6.3 全 Agent 可恢复流(#28424,稳定)

**解决什么**:断线重连/第二读者跟随 run 以前是 durable agent 专属;现在任何 Agent 的 run 都发布到 `agent.stream.<runId>` topic,带 replay 缓存。

- 代码:`packages/core/src/agent/streams/config.ts`、`agent.observe()`(`packages/core/src/agent/agent.ts:12582`)、`abortRunStream()`(`agent.ts:10658`)
- 用法:`new Agent({ ..., durable: true })` → `const obs = await agent.observe(runId)`;`obs.abort()` 中止。⚠️ PR 里的 `streams` 选项在当前 main 已**并入 `durable`**(`agent.ts:1128` 有迁移警告)
- 注册到 Mastra 的 agent 默认开;独立 agent 默认关

### 6.4 崩溃恢复(稳定,默认 auto)

**工作流侧**(#30281):`startWorkers()` 起常驻恢复循环,多实例**只有一个 leader**(`run-owners` 表 `::recovery-leader` 行,租约续期,实例死亡 TTL 后别人接管)。配置:

```ts
new Mastra({ recovery: { workflows: 'auto', // 默认
  workflowConcurrency: 5, workflowGraceMs: 2000, workflowScanIntervalMs: 60000 } })
```

无人持有租约且超 graceMs 才重启;evented run 不走此循环(靠 broker 重投递)。

**Agent 侧**(#30330):人工审批(`approveToolCall`/`resumeStream` 等)落库后进程死掉,决策不丢——决策+claim 持久化,任意进程可接手:`await agent.recoverResume({ runId })`(`agent.ts` ~:8557;文档 `docs/.../human-in-the-loop.mdx:605`)。接管条件:claim 后 60s 未启动,或租约 60s 未续。⚠️ 副作用工具需幂等(恢复会重跑被恢复的工具调用)。另有 `recovery.durableAgents: 'auto'`(默认 off)。

### 6.5 v2 结果 steps 懒加载(#29362,experimental)

`const { result } = await run.start(); const steps = await result.steps();` — v2 的 `steps` 是按需读取的异步 loader,不再整块读回内存。default 引擎上 run 曾 resume/restart/delete 则抛 `WORKFLOW_STORAGE_V2_STEPS_STALE`。代码:`packages/core/src/workflows/run-journal.ts`、`default.ts`、`run-storage.ts`。

### 6.6 有界一次性调度(#25131,稳定)

- `runAt` 一次性触发;`cron` + `endAt` 到期自动转新状态 **`completed`**(原子 check-and-set,多实例只触发一次)
- 声明式:`createWorkflow({ id: 'w', schedule: { runAt: new Date('2026-12-01') } })` 或 `{ cron: '0 9 * * *', endAt }`;命令式 `mastra.createSchedule({ scheduleName, runAt })`
- 代码:`packages/core/src/schedules/schedules.ts`;各 store 新增 `run_at`/`end_at` 列;`list()` 默认隐藏 completed

### 6.7 Temporal suspend/resume(#27020,0.x)

`workflows/temporal/src/run.ts` — `await run.resume({ step: 'approval', resumeData: { approved: true } })`;step 可 `suspend(...)` 并在 Temporal runtime 内持久等待 `mastra.resume` 信号(`workflow.ts` 生成 signal handler + `condition()`)。注意 Temporal 无独立 suspended 状态(UI 显示 running/waiting)。

---

## 7. 内置 MCP endpoint + MCP Apps

### 7.1 内置 MCP endpoint(#29929)

**解决什么**:不用自己 `new MCPServer` 手动注册 agent/workflow 工具——server 直接把整个 Mastra 实例暴露为 MCP 服务。

- 路由:`packages/server/src/server/mcp/route.ts:170`,Streamable HTTP(stateless),默认 `/api/mcp`,`server.mcp.path` 可改
- 开关(`config.ts:57`):默认 `enabled: 'dev'` — `mastra dev` 开放(带 DNS-rebinding 防护);**生产构建仅当配了 `server.auth` 才开**,否则 404;`enabled: true` 强开
- 需装可选 peer `@mastra/mcp`(未装 501,首次请求动态 import)
- 与自建 MCPServer 的关系:内置服务器不进 `mastra.mcpServers`,id `'mastra'` 保留;每请求临时实例,工具目录按调用者权限现场过滤

```ts
new Mastra({ agents: {...}, server: { mcp: { toolsets: ['agents','workflows','tools'] } } })
// 客户端接入:claude mcp add --transport http mastra http://localhost:4111/api/mcp
```

客户端可用 `?toolsets=agents,tools`、`?readonly=1` 或 `X-MCP-Toolsets` header **只收窄不放宽**。

### 7.2 OAuth discovery + audience 校验(#29930)

`packages/server/src/server/mcp/oauth.ts`:配了 `server.auth` 后,未带 token 返回 401 + `WWW-Authenticate: Bearer resource_metadata=...`(RFC 9728);元数据挂 `/.well-known/oauth-protected-resource`。`server.mcp.oauth.resource` 同时启用 token `aud` 校验——为别的 API 签发的 token 不能重放到 `/api/mcp`。注意 provider 差异:MastraJwtAuth/Auth0 自带 audience 检查,Okta/Clerk/Supabase 不检查;`requireAudience: true` 会拒绝无法检查的 provider。

### 7.3 toolset 目录(基础 3 组 + 扩展 3 组,#29931)

工具目录构建于 `packages/server/src/server/mcp/catalogue.ts`(每个内置 HTTP 路由自带 `route.mcp.name`,toolset 包装成同名 MCP 工具):

| toolset | 工具 | 条件 |
|---|---|---|
| agents | list/get/call_agent(返回 threadId 供续聊) | — |
| workflows | list/get/start/get_run/resume/cancel | — |
| tools | `tool_<id>`,超 enumLimit 折叠为 list/get/call | — |
| memory | 9 个(list_threads/get_thread/.../working_memory) | 有 agent 配 memory |
| observability | 12 个 query/aggregate + 6 个 analytics | 存储有 observability domain |
| stored_agents | 5 个(list_available_models/create/update...) | editor + agents domain,含模型策略校验 |

### 7.4 packages/mcp-apps-components:预构建 MCP Apps

**解决什么**:让 MCP 客户端(如 Claude)里能直接**渲染**交互界面——App = 自包含单页 HTML(`McpAppResource = { name, description, html }`),iframe 无网络权限,全部数据经 `tools/call` 回服务器取。

- App 源码 `src/apps/*`:greeting、observability-widget、trace-viewer、observability-explorer
- 注册:`new MCPServer({ appResources: { 'ui://mastra/greeting': greetingApp } })`;工具声明 `mcp: { _meta: { ui: { resourceUri } } }` + outputSchema;App 专用数据工具加 `visibility: ['app']` 只给 App 不给模型
- dashboard 渲染器(#29321)用 Studio 同款组件画 KPI/时序/表格;草稿 schema `createObservabilityDraftSchemas`(#29832)支持"模型写 dashboard → 保存";存储域 `packages/core/src/storage/domains/studio/dashboards/`

### 7.5 内置 endpoint × MCP Apps 联动(#30244)

装了 `@mastra/mcp-apps-components` 即生效(`server.mcp.apps: false` 关):`get_trace` 打开 Studio trace timeline、`query_traces` 打开 observability explorer、`analyze_metrics` 开 Metrics tab;模型只收短文本摘要,App 收全量数据;支持 "Open in Studio"。不支持 Apps 的客户端(终端 Claude Code)拿到纯 JSON,行为不变。Studio 的 MCP Servers 页顶部新增 "Mastra (built-in)" 卡片(#29932),附各客户端接入片段。

### 7.6 其他

`MCPServer` 新增 `maxRequestBodySize`(#30411,`packages/mcp/src/server/server.ts:269`)。

---

## 8. Mastra Factory:看板式 agent 工作流(本月最大特性)

**是什么**:`@mastra/factory` v0.30.0(`mastracode/factory/`)——"Mastra Software Factory" 的可复用**后端库**(非独立服务):把 GitHub issue/PR、Dependabot、Vanta、incident.io、GitLab、Jira、Slack 等外部事件自动变成看板卡片,由 code-agent 按阶段(triage→plan→build→review)自主推进。产品形态 = `mastracode/factory`(后端库)+ `factory-ui`(React)+ `mastracode/web`(宿主)+ `sdk`(agent-controller 共享行为)。

**领域模型**(`src/storage/domains/work-items/base.ts:545`,三表拆分):
- `factory_work_items` — 工作项身份(标题/父链接)
- `factory_work_item_sources` — 外部来源(issue/PR 可共享一张卡)
- `factory_board_items` — 看板放置(board/phases/sessions/过程戳)
- `Card = BoardItemRow + workItem + externalSource`
- board:有序 phases;phase 带 `kind`(resting/working/terminal)、`runner({ agentId, skill, goal, ... })`、`exitTool`、`nextPhases`(**拓扑即迁移白名单**)。board 现存数据库,`defineBoard()`/`boards` 选项已弃用,改 `factory.upsertBoard()`
- session:`Card.sessions[context] = { sessionId, branch, threadId }` — 每个上下文(triage/work/review/自定义)独立 session,各带 sandbox 和 branch;Planning 与 Building 共享 `work` session

**跑的是 agent 不是 workflow**:`runner.agentId` 必须是 `code-agent`(`src/boards/define-board.ts:92`);跨阶段编排靠 `nextPhases` + exit tool + goal(`runner.goal` 用 session 模型做 judge 判 done/continue/waiting,最多 maxRuns 次,Work 的 Building 上限 20)。每进程并发 `maxInFlight=10` + 2 个 interactive 储备(`MASTRACODE_DISPATCH_MAX_IN_FLIGHT`)。

**与 core 的衔接**:`factory.prepare()` → `new Mastra(args)`(留在宿主入口文件,deployer 识别)→ `factory.finalize()`;经 `@mastra/code-sdk` 的 `prepareAgentControllerMount()` 挂载,复用 core 的 storage/memory/knowledge/events/channels/skills/goals/auth。规范示例 `mastracode/web/src/mastra/index.ts`。

**intake 路由**:集成 webhook → 项目 intake route 决定落哪个 board/phase;`POST /web/factory/projects/:id/intake/dry-run`(`src/routes/intake-check.ts`)对最近条目试跑路由**不写任何东西**(注意:与迁移 dry-run `MASTRA_FACTORY_WORK_ITEM_SPLIT=dry-run` 是两回事)。内置技能目录 `factory-skills/`:triage/plan/build/review/self-review/configure-factory-rules 等。

---

## 9. 可观测性

**包格局**(顶层 `observability/`,17 个目录):
- `@mastra/observability` — 核心:`Observability` 类、事件 bus、`MastraStorageExporter`(写 Mastra 存储,Studio 可查)/ `MastraPlatformExporter`(发 Platform)、span processors、采样、metrics/usage 提取、默认 `SensitiveDataFilter` 脱敏
- `@mastra/otel-bridge`(Mastra 操作 ↔ 真 OTel span 双向桥)、`@mastra/otel-exporter`(导出到任意 OTel 后端)
- 10 个 SaaS exporter:arize、arthur、braintrust、datadog、deepeval、laminar、langfuse、langsmith、posthog、sentry

**本月 API 特性落点**(契约层在 `packages/core/src/storage/domains/observability/`——因为各存储后端都要实现;HTTP 在 `packages/server/src/server/handlers/observability-*.ts`;SDK 在 client-js):

| 特性 | 位置 |
|---|---|
| trace 分页契约(#30512) | `trace-page.ts` + client-js `trace-page-types.ts` |
| 文本搜索谓词(#30511/#29897) | `text-search.ts`、`trace-query(-plan).ts`、`span-query.ts`;Studio 侧 playground-ui `domains/observability/lib/text-search.ts` |
| 聚合 trace root(#30054) | `trace-query-plan.ts` + server `observability-new-endpoints.ts` |
| span 按模型/provider/时长排序(#30096/#30085) | `span-query.ts`、`span-aggregate.ts` |
| span 级 originating 错误计数(#30088) | core `error-count.ts` |
| feedback/scores 读写(#30048/#30059/#29529) | 存储域 `feedback.ts`、`scores.ts`;HTTP `/observability/scores*`、`/observability/feedback*` |
| 按字节的批量 flush(#29621) | `observability/mastra/src/exporters/`(byte-batcher/byte-tracker) |

**用法**:自托管 `new Mastra({ observability: new Observability({ configs: { default: { exporters: [...] } } }) })`;查询走 `/observability/*` HTTP API(分页/搜索/排序/feedback 全在这组)。core 的 `packages/core/src/observability/` 提供底层插桩原语(context/span/error-count),`@mastra/observability` 消费事件加工后交 exporter。

---

## 10. 一页总结(给未来的自己)

| 主题 | 一句话 | 状态 |
|---|---|---|
| Factory | 外部事件→看板卡片→code-agent 按阶段自治推进的软件工厂 | v0.30,快速迭代 |
| Connect | 平台托管 OAuth 集成直接变 agent toolset,凭证不出云 | Beta,依赖 Mastra 云 |
| @mastra/platform | core 抽象的 hosted 替身(Memory/Knowledge/Browser/Sandbox) | 0.1 experimental |
| 内置 MCP endpoint | `server.mcp` 一开,整个 Mastra 实例变 MCP 服务(`/api/mcp`) | 稳定,dev 默认开 |
| MCP Apps | MCP 客户端里渲染交互 HTML(trace viewer、observability dashboard) | 0.2 |
| Storage v2 | run header + 每步 entry,长流程写入从平方增长降为增量 | experimental |
| Durable/evented v2 | evented 消息量 98MB→11MB | experimental |
| Resumable streams | 任何 agent 的 run 可 observe/重放(`durable: true`) | 稳定 |
| 崩溃恢复 | workflow 单 leader 恢复循环 + `agent.recoverResume()` | 稳定默认 auto |
| Knowledge | 跨会话共享知识图(page/note/link/scope/role) | alpha(1.82 引入) |
| 可观测性 | trace 分页/文本搜索/聚合 + feedback/scores API + 10 个 SaaS exporter | 持续增强 |
| Skills 搜索发现 | agent 主动搜 skills 而非塞 prompt | 稳定 |
| 有界调度 | `runAt` 一次性 + `cron`+`endAt` 自动 completed | 稳定 |
| Temporal suspend/resume | Temporal runtime 内持久等待信号 | 0.x |
