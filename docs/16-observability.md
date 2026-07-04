# 第 16 章：可观测性——一次 Prompt 的 EXPLAIN

> 分布式数据库能对一条 SQL 做 `EXPLAIN`：它经过哪些算子、每个算子读了多少行、花了多久、有没有数据倾斜。Claude Code 对**它自己**做的是同一件事——一次 prompt 跨多个 turn，每个 turn 打了哪些 API、调了哪些工具、消耗多少 token 与钱、在哪儿等用户批权限、有没有报错重试，全部被结构化地记录并串联起来。这一章讲它怎么做到。

## 16.1 一次 prompt 的可观测链路总览

先把问题说清楚。你在终端敲下一句 "帮我修好登录的 500 错误"，接下来发生的不是"一次 LLM 调用"，而是一条**执行链**：

```
用户 prompt
  └─ turn 1: LLM 请求（读文件的意图）→ Read 工具 → tool_result 回注
  └─ turn 2: LLM 请求（定位 bug）→ Grep 工具 → tool_result 回注
  └─ turn 3: LLM 请求（提出改法）→ Edit 工具 → 等你批权限 → 执行 → tool_result
  └─ turn 4: LLM 请求（跑测试）→ Bash 工具 → tool_result
  └─ turn 5: LLM 请求（总结）→ 结束
```

读者想问的正是：**这条链，Claude Code 内部是怎么记录、又怎么串起来的？** 这不是"harness 帮 Claude 写可观测代码"的问题，而是 Claude Code 对自身运行时的自省能力（self-observability）。

答案是：CC 没有用一个大日志文件糊弄这件事，而是搭了**三个正交的观测平面 + 一个持久化底座**，各管一段职责：

| 平面 | 记什么 | 形态 | 默认开? | 粒度 | 落在哪 |
|---|---|---|---|---|---|
| **Metrics** | 聚合量：会话数、token、成本、代码行数、决策计数、活跃时长 | OTel Counter | 需 `CLAUDE_CODE_ENABLE_TELEMETRY`（用户侧）/ 组织门控（一方 BigQuery） | 低基数、可累加 | 用户自己的 OTLP 后端 / Anthropic BigQuery |
| **Events（Logs）** | 离散事件：每次 API 请求、每个工具结果、每次权限决策、每次 hook 执行 | OTel Log Record | 需 `CLAUDE_CODE_ENABLE_TELEMETRY` + `OTEL_LOGS_EXPORTER`（普通事件非 beta；仅 `system_prompt`/`tool` 详细追踪属 beta） | 一事件一记录 | OTLP logs endpoint |
| **Trace（Spans）** | 因果树：一次交互 → 每个 LLM 请求 / 工具调用 / 权限等待的父子与耗时 | OTel Span | BETA，需额外开 enhanced/beta tracing | 因果嵌套 | OTLP traces endpoint |
| **Transcript JSONL** | 完整可重放的会话记录：每条消息、usage、parentUuid 链 | 本地 append-only JSONL | **默认开、day-0 就有** | 逐消息 | `~/.claude/projects/…/<sessionId>.jsonl` |

**为什么要分四层，而不是一个大日志？** 因为四种问题需要四种数据结构：

- 想画"本周这个组织烧了多少 token" → 要**低基数、可聚合**的 metric，绝不能把每条 prompt 的 ID 塞进维度里（会把时序库的基数打爆）。
- 想查"上周二那次 `git push` 到底是谁批的" → 要**可按字段过滤的离散事件**。
- 想看"这一次 prompt 的 5 个 turn 里，哪个 turn 的哪个工具最慢" → 要**带因果父子的 span 树**——这正是 `EXPLAIN` 的算子树。
- 想**完整复现**一次会话（resume、审计、社区分析工具） → 要**逐条、可回放**的持久记录。

### 把一次 prompt 串起来的锚点：`prompt.id`

要把一次 prompt 的观测数据拼回"一条链"，靠的是一个关联锚点。当用户输入进入系统时，`processTextPrompt` 先生成一个 UUID 存进全局状态，并在**同一处**既开一个 root span、又发一条 `user_prompt` 事件——这一步就把追踪与事件挂到了同一次 prompt 上：

```typescript
// utils/processUserInput/processTextPrompt.ts:31
const promptId = randomUUID()
setPromptId(promptId)
// …
startInteractionSpan(userPromptText)          // ← 开一个 root span
void logOTelEvent('user_prompt', {            // ← 发一条 user_prompt 事件
  prompt_length: String(otelPromptText.length),
  prompt: redactIfDisabled(otelPromptText),   // 默认 <REDACTED>
  'prompt.id': promptId,
})
```

此后，**每一条 OTel 事件都会自动带上这个 `prompt.id`**：

```typescript
// utils/telemetry/events.ts:49
// Add prompt ID to events (but not metrics, where it would cause unbounded cardinality)
const promptId = getPromptId()
if (promptId) {
  attributes['prompt.id'] = promptId
}
```

同一个 ID 还会被写进本地 transcript 的 **user 行**（`sessionStorage.ts:1045` 只在 `message.type === 'user'` 时盖 `promptId`；`types/logs.ts:230`：`promptId?: string // Correlates with OTel prompt.id for user prompt messages`）。官方文档一句话点破它的用途：

> "The `prompt.id` attribute lets you tie all of those events back to the single prompt that triggered them."

但要精确——`prompt.id` **不是**平均铺在四层的同一个字段，各层的关联方式不同：

- **Events**：每条 OTel 事件都带 `prompt.id`（`events.ts:49`）——这是它真正的"关联键"，同一次 prompt 的所有事件靠它归拢。
- **Transcript**：只盖在 **user 行**上（含用户输入行与 tool_result 回注行，二者都是 `user` 类型消息）——它是这次 prompt 在本地记录里的**锚点**，assistant 行并不带它。
- **Trace**：span **不带** `prompt.id`（`sessionTracing.ts` 的 span 属性里没有它），一次交互的父子关系靠 **span parentage**（全部挂在 `claude_code.interaction` 这个 root 下，见 §16.4）来表达；它和 events/transcript 指的是同一次 prompt，只因为是"同一处代码同时开的"（上面那段 `startInteractionSpan` 与 `logOTelEvent` 并列）。
- **Metrics**：**故意不带**任何 prompt 级 id（`events.ts:49` 注释：否则造成 unbounded cardinality）。

这条分界——**离散层用 id 关联、聚合层拒绝 id、因果层用 parentage**——是整套遥测设计的第一性原则，后面会反复出现。

接下来四节分别拆这四层。

---

## 16.2 Metrics 面：OpenTelemetry 导出

### 一个开关，两条并行管线

Metrics 面有一个容易被误解的地方：**它其实喂两条独立的管线**，由两套完全不同的开关控制。

```typescript
// utils/telemetry/instrumentation.ts:458
const telemetryEnabled = isTelemetryEnabled()   // CLAUDE_CODE_ENABLE_TELEMETRY
if (telemetryEnabled) {
  readers.push(...(await getOtlpReaders()))      // ① 用户自己的 OTLP 后端
}
// Add BigQuery exporter (for API customers, C4E users, and internal users)
if (isBigQueryMetricsEnabled()) {
  readers.push(getBigQueryExportingReader())     // ② Anthropic 一方 BigQuery
}
```

- **① 用户侧 OTLP 导出**：由 `CLAUDE_CODE_ENABLE_TELEMETRY` 开关（`instrumentation.ts:324`），把 metric 送到**你自己**的 Prometheus / OTLP collector。这是企业给自己团队做用量看板用的。
- **② 一方 BigQuery 导出**：面向 API 客户 / 企业版 / 团队版（`isBigQueryMetricsEnabled`，`instrumentation.ts:336`），把 metric 送到 **Anthropic 自己的** BigQuery，供官方做产品分析。

两者相互独立：即使你没开 `CLAUDE_CODE_ENABLE_TELEMETRY`，一方 BigQuery 导出仍可能在跑——但它受**组织级 opt-out** 门控（下面讲）。理解这一点，才不会把"我没开遥测"误当成"什么都没上报"。

### 组织级 opt-out：一个被精心设计成"几乎不打网络"的开关

一方 BigQuery 导出会在每次导出前问一句"这个组织允许上报吗"。天真的实现是每次 export 都打一次 API，但那会给启动路径和网络添巨大负担——尤其 `claude -p` 这种一次性调用可能一天跑几百次。`metricsOptOut.ts` 用**两级缓存**把它压成"约一天一次 API 调用"：

```typescript
// services/api/metricsOptOut.ts:22
const CACHE_TTL_MS = 60 * 60 * 1000          // 内存 TTL：进程内去重
const DISK_CACHE_TTL_MS = 24 * 60 * 60 * 1000 // 磁盘 TTL：跨进程存活
// 注释原文：This is what collapses N `claude -p` invocations into ~1 API call/day.
```

导出前 `bigqueryExporter` 调 `checkMetricsEnabled()`（`bigqueryExporter.ts:105`）：磁盘缓存新鲜就直接返回、零网络；过期才后台异步刷新。还有一道**事故熔断**——`isEssentialTrafficOnly()` 为真（非必要流量被全局关闭）时直接返回 `enabled:false`，在消费端就把导出停掉（`metricsOptOut.ts:57`）。这是一个很典型的"可观测性本身不能拖垮主流程"的工程判断：观测组件必须能被廉价地关掉、能容忍陈旧读、且默认不阻塞。

### 八个 metric：稳定的自观测底座

CC 内部注册的 metric 只有八个，在 `setMeter` 里集中定义：

```typescript
// bootstrap/state.ts:955
createCounter('claude_code.session.count',        …) // 启动的会话数
createCounter('claude_code.lines_of_code.count',  …) // 改动的代码行（type=added/removed）
createCounter('claude_code.pull_request.count',   …) // 创建的 PR 数
createCounter('claude_code.commit.count',         …) // 创建的 commit 数
createCounter('claude_code.cost.usage',           …) // 会话花费（USD）
createCounter('claude_code.token.usage',          …) // token 数（type=input/output/cacheRead/cacheCreation）
createCounter('claude_code.code_edit_tool.decision', …) // Edit/Write/NotebookEdit 的批/拒计数
createCounter('claude_code.active_time.total',    …) // 活跃秒数
```

这八个和官方 [Monitoring 文档](https://code.claude.com/docs/en/monitoring-usage)列出的 metric 清单**逐条对齐**——这是一个难得的"快照即当前"的稳定点（对比下一节会看到 events 清单在快照之后大幅扩张）。设计上它们全是 **Counter（单调累加）**：可观测领域里，能用 counter 就不用 gauge，因为 counter 对采样丢失、重启、乱序都更鲁棒，聚合时只需做 delta。CC 甚至默认把时序偏好设成 `delta`（`instrumentation.ts:113`，注释 "the more sane default"）。

**注意 metric 不带内容、只带"形状"**：token 计数带 `type=input/output/cacheRead/cacheCreation` 和 `model`，但绝不带 prompt 文本。代码行计数带 `type=added/removed`，但绝不带文件路径或代码。这不是疏忽，是刻意——见 §16.8。

### 属性装配与基数控制

每个 metric/event 的公共属性由 `getTelemetryAttributes()` 统一装配（`telemetryAttributes.ts:29`），而**哪些属性进得去，本身也是可配的基数旋钮**：

```typescript
// utils/telemetryAttributes.ts:10
const METRICS_CARDINALITY_DEFAULTS = {
  OTEL_METRICS_INCLUDE_SESSION_ID: true,   // 默认带 session.id
  OTEL_METRICS_INCLUDE_VERSION: false,     // 默认不带 app.version（否则每次升级都新增一维）
  OTEL_METRICS_INCLUDE_ACCOUNT_UUID: true,
}
```

`app.version` 默认关，理由同 `prompt.id`：CC 周更，如果每个版本都成为一个新的 metric 维度，时序库会被版本号切得粉碎。把这些做成 env 开关，等于把"我要多细的归因 vs 我能承受多大的基数"的选择权交给运维。

### 导出器的惰性加载

一个不起眼但值得学的细节：OTLP 支持 grpc / http-json / http-protobuf 三种协议，各自的 exporter 包加起来约 1.2MB。CC 没有在启动时全部静态导入，而是在协议 switch 里**按需 `await import()`**：

```typescript
// utils/telemetry/instrumentation.ts:3
// OTLP/Prometheus exporters are dynamically imported inside the protocol
// switch statements below. … static imports would load all 6 (~1.2MB) on every startup.
```

一个 CLI 工具对冷启动延迟极其敏感，把"只有开了遥测、且用某协议才需要"的重依赖挪出启动关键路径，是 harness 级性能纪律的体现。

---

## 16.3 Events 面：离散事件管线

### 一个 call site，两条事件流

这是理解 CC 遥测的第二个关键分层：**内部分析**和**客户可观测**是两条并行的事件流，常常从同一个 call site 同时发出。看 API 成功时的处理（`services/api/logging.ts`）：

```typescript
// 第一条：内部分析事件（Statsig / Datadog / 一方 BigQuery）
logEvent('tengu_api_success', { model, inputTokens, outputTokens, costUSD, durationMs, ttftMs, … })
// 第二条：客户 OTel 事件（用户自己的 OTLP）
void logOTelEvent('api_request', {
  model, input_tokens, output_tokens, cache_read_tokens,
  cache_creation_tokens, cost_usd, duration_ms, speed,
})
// 第三条：结束这个请求对应的 trace span（见 §16.4）
endLLMRequestSpan(llmSpan, { success: true, inputTokens, … })
```

一次 API 完成，三个观测平面被同一处代码同时喂到。两条事件流的分工是清晰的：

| | 内部分析（`logEvent('tengu_*')`） | 客户可观测（`logOTelEvent`） |
|---|---|---|
| 去向 | Anthropic 的 Statsig/Datadog/BigQuery | 用户自己的 OTLP 后端 |
| 目的 | 官方做产品分析、A/B、行为回归 | 用户给自己团队做用量/健康看板 |
| 规模 | 数百个 `tengu_` 事件名（快照里 grep 到的 `logEvent('tengu_*')` 名有数百个，随去重口径而异） | 官方文档化的一二十类事件 |
| 门控 | GrowthBook feature flag + 采样 | `CLAUDE_CODE_ENABLE_TELEMETRY` |
| 隐私 | `AnalyticsMetadata_…_NOT_CODE_OR_FILEPATHS` 类型强制不带代码/路径 | `redactIfDisabled` 默认脱敏内容 |

`tengu_` 这条内部流有个有意思的类型级护栏：`logEvent` 的 metadata **不允许裸字符串**，任何要传的字符串必须显式断言成 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`（`services/analytics/index.ts:19`）——用编译期类型把"别把代码/文件名误传进分析后台"变成一条过不了 CI 的硬约束。还有一个 `_PROTO_*` 前缀约定，把 PII 字段路由到有访问控制的 BigQuery 专列，并在 fan-out 到 Datadog 前统一 strip（`stripProtoFields`，`services/analytics/index.ts:45`）。`tengu_skill_loaded` 事件就用 `_PROTO_skill_name` 把技能名送进受控列而不落进通用元数据（`skillLoadedEvent.ts:23`）。

### `logOTelEvent` 管线：每个事件的骨架

客户侧的 OTel 事件全部走 `logOTelEvent`（`utils/telemetry/events.ts:21`）。它给每条事件盖三个"信封"字段，再合上公共属性和 `prompt.id`：

```typescript
// utils/telemetry/events.ts:42
const attributes: Attributes = {
  ...getTelemetryAttributes(),                 // session.id / user.id / org.id / terminal.type …
  'event.name': eventName,
  'event.timestamp': new Date().toISOString(),
  'event.sequence': eventSequence++,           // 会话内单调递增，用于排序
}
```

`event.sequence` 这个会话内单调计数很关键：它是**发射计数器**（`events.ts:8` 的 `let eventSequence = 0`，每发一条 `++`），批处理/网络会打乱到达顺序，有了它后端能把一次 prompt 内的事件**按发射先后重新排回**。注意它不是墙钟时刻——并发/重叠的 API 与工具活动谁真正先发生、各持续多久，要看 `event.timestamp` 与 §16.4 的 span 耗时，`event.sequence` 只保证"离散事件的发射次序可恢复"。

### 快照有哪些事件，当前又扩了多少

在 v2.1.88 快照里，实际通过 `logOTelEvent` 发出的客户事件是这一组（grep 可逐一确证）：`user_prompt`、`api_request`、`api_error`、`tool_result`、`tool_decision`，加上 BETA 详细追踪路径下的 `system_prompt`、`tool`，以及 hook 的 `hook_execution_start` / `hook_execution_complete`、`feedback_survey`。

而**当前官方文档**的事件清单已经大幅扩张，新增了 `assistant_response`、`api_refusal`、`api_request_body` / `api_response_body`、`api_retries_exhausted`、`permission_mode_changed`、`auth`、`mcp_server_connection`、`internal_error`、`plugin_installed` / `plugin_loaded`、`skill_activated`、`at_mention`、`hook_registered`、`compaction` 等。

> **快照 vs 当前**：本章的 metric 清单（8 个）在快照与官方文档间稳定一致；但 events 清单在 v2.1.88 之后明显增长。凡本节以外提到 `assistant_response` / `compaction` / `skill_activated` / `permission_mode_changed` / `OTEL_LOG_RAW_API_BODIES` 等**未在快照 grep 命中**的标识符，一律以官方 [Monitoring 文档](https://code.claude.com/docs/en/monitoring-usage)为准，标注"据官方文档（超出 v2.1.88 快照）"，不作源码级实锤。

### 走读一条事件：`api_request`

以最能回答读者的 `api_request` 为例（`logging.ts:718`）。它在**每次 API 调用成功后**发一条，带 `model` / `input_tokens` / `output_tokens` / `cache_read_tokens` / `cache_creation_tokens` / `cost_usd` / `duration_ms` / `speed`。把一次 prompt 里所有带同一 `prompt.id` 的 `api_request` 事件拉出来按 `event.sequence` 排序，你就得到了这条 prompt 的**一份按发射次序排列的模型调用审计流**：这次 prompt 一共打了几次 API、各花了多少 token 和钱、命中了多少缓存、耗时多久。

`tool_decision` 事件（`permissionLogging.ts:230`）和 `tool_result` 事件（`toolExecution.ts:1381`）分别补上"工具被怎么放行/拦截"和"工具跑了多久、成功没、结果多大"。三类事件拼起来，一次 prompt 的**外部动作总账**就可查了。

**但要诚实标一条边界**：在 v2.1.88 快照里，这些事件是 **prompt 粒度关联、不是 turn 粒度关联**——`api_request` 不带 turn 序号，`tool_decision` / `tool_result` 也**不带 `tool_use_id`**（官方文档的当前版本已给它们加了 `tool_use_id`，属快照之后新增）。所以"哪个 tool_result 对应哪次 api_request/哪个 turn"这种**精确因果归因**，光靠 events 字段在快照里拼不出来——那正是 §16.4 的 span 树与 transcript 的 parentUuid 连边负责的事。Events 面给的是"这次 prompt 的有序动作审计"，因果树留给 trace 层。

---

## 16.4 Trace 面：一次交互的算子树 + 会话 transcript

这是全章的重心，也是最贴近读者"EXPLAIN of a prompt"诉求的一层。它其实由**两套东西**共同承担：一套是 OpenTelemetry 的 span 树（BETA、精确因果、默认关），一套是本地 transcript JSONL（day-0、默认开、人人可用）。

### (A) OTel Span 树：把一次交互建成因果嵌套

`sessionTracing.ts` 定义了六种 span，它们的父子关系恰好就是 `EXPLAIN` 的算子树：

```
claude_code.interaction              ← root，一次用户 prompt → 回复的完整周期
  ├─ claude_code.llm_request         ← 每个 turn 的一次模型调用
  ├─ claude_code.tool                ← 一次工具调用
  │    ├─ claude_code.tool.blocked_on_user   ← 卡在"等用户批权限"的墙钟时间
  │    └─ claude_code.tool.execution         ← 工具真正执行的时间
  └─ claude_code.hook                ← 一次 hook 执行（仅 beta 详细追踪）
```

（span 类型定义见 `sessionTracing.ts:49`。）root 由 §16.1 那个 `startInteractionSpan(userPrompt)` 开启（`sessionTracing.ts:176`），并把用户 prompt 长度、`interaction.sequence` 记进去；结束时补上 `interaction.duration_ms`（`sessionTracing.ts:263`）。

父子关系怎么建立？靠 `AsyncLocalStorage`。interaction span 存进一个 ALS，之后开 llm_request / tool span 时从 ALS 取出父 span 作为 OTel context 的 parent（`sessionTracing.ts:308`、`495`）。这样即使中间隔着若干层异步 await，子 span 也能正确挂到当前交互下。

几个值得学的设计细节：

- **`tool.blocked_on_user` 单独成一个 span**（`sessionTracing.ts:526`）。为什么把"等用户点批准"单独计时？因为一次 Edit 慢，可能是模型慢、可能是磁盘慢、也可能纯粹是**人去喝咖啡了**。把"等人"从"执行"里剥出来，才能在 trace 里一眼分清墙钟时间花在哪儿——这直接决定了你优化的方向是模型、是 IO、还是工作流。它结束时会记下 `decision` 和 `source`（批还是拒、谁批的），数据来自权限层（见 §16.6）。

- **并行请求必须传具体 span**。一次交互里常有多个 LLM 请求并发（主线程、warmup 预热、topic 分类器、文件路径抽取器）。`endLLMRequestSpan` 的注释专门警告：不显式传入 `startLLMRequestSpan` 返回的那个 span，回填响应时就可能挂到"恰好最后一个"的 span 上、造成响应错配（`sessionTracing.ts:342`）。这是并发追踪里最经典的坑，CC 用"显式传 span 句柄"而非"隐式取最近"来根治。

- **span 泄漏兜底**。正常路径靠 `endInteractionSpan` / `endToolSpan` 立即删除 span；但流被 abort、turn 中途抛异常时 span 可能永远不 end。CC 用 `WeakRef` 存活跃 span、配一个 30 分钟的后台清理 interval，把没人再引用或超时的 span 强制 flush 并回收（`sessionTracing.ts:79`、`100`）——观测设施自己不能变成内存泄漏源。

每个 span 结束时都带上 `duration_ms` 和（LLM span 的）token 明细（`sessionTracing.ts:429`）。于是这棵树天然就是一份带耗时和资源消耗的 `EXPLAIN`：哪个 turn、哪次工具、哪段等待最贵，一目了然。

**为什么 trace 默认关、还是 BETA？** 因为它贵且敏感。开启条件是 `telemetryEnabled && isEnhancedTelemetryEnabled()`（`instrumentation.ts:628`），后者要 `feature('ENHANCED_TELEMETRY_BETA')` 编译期开 + 运行时 env/GrowthBook 允许（`sessionTracing.ts:126`）。更详细的"beta 追踪"路径（连 system prompt、模型输出都进 span）门槛更高，还带内容脱敏（见 §16.7 / §16.8）。span 数量随工具调用线性增长、又携带较多属性，让它默认开会同时压上性能和隐私两笔账。

### (B) Transcript JSONL：默认就在的那份 trace

如果说 OTel span 树是"要专门开、给可观测后端看"的 trace，那么 **transcript JSONL 是 day-0 就在、每个用户本地都有、且是 `/resume` 与几乎所有社区分析工具真正读的那份 trace**。它才是绝大多数人实际能用上的"一次 prompt 的执行记录"。

**路径与布局**。每个会话一个 append-only 的 JSONL 文件：

```typescript
// utils/sessionStorage.ts:199
function getProjectsDir() { return join(getClaudeConfigHomeDir(), 'projects') }
// utils/sessionStorage.ts:202
export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

即 `~/.claude/projects/<把 cwd 转义成目录名>/<sessionId>.jsonl`。子 agent（sidechain）另写 `agent-<agentId>.jsonl`（`sessionStorage.ts:247`）——多 agent 的每条委派链各有自己的 trace 文件，与主线程分开。

**它是一个 parentUuid 链表**。每写一条消息，都会盖上一套字段，其中 `parentUuid` 指向上一条：

```typescript
// utils/sessionStorage.ts:1039
const transcriptMessage: TranscriptMessage = {
  parentUuid: isCompactBoundary ? null : effectiveParentUuid,
  logicalParentUuid: isCompactBoundary ? parentUuid : undefined,
  isSidechain,
  teamName, agentName, agentId,
  promptId: message.type === 'user' ? (getPromptId() ?? undefined) : undefined, // ← 同 OTel prompt.id
  ...message,           // usage / content / tool_use / …
  sessionId, version, gitBranch, cwd, userType, entrypoint,
}
```

（字段定义见 `types/logs.ts:221` 的 `TranscriptMessage`。）几个精妙处：

- **tool_result 的父指针指向发起它的那条 assistant 消息**，而不是简单的"上一条"。代码专门处理：若消息带 `sourceToolAssistantUUID`，就用它当 `parentUuid`（`sessionStorage.ts:1031`）。于是"某个工具结果"能精确回溯到"哪个 assistant turn 里的哪次 tool_use 请求了它"——因果不是靠时间顺序猜的，是显式连边。
- **compaction 边界断链但留逻辑链**。上下文压缩会把一段历史换成摘要；此时 `parentUuid` 置 `null`（物理断开，让 prompt cache 稳定），但 `logicalParentUuid` 保留原父指针，读取时再桥接（`sessionStorage.ts:1040`）。压缩不破坏 trace 的逻辑连续性。
- **`promptId` 只盖在用户消息上**，正是 §16.1 那个 join key 在持久层的落点。

**怎么串成"一次 prompt 的多 turn"？** 主力是 **`parentUuid` 连边**：从任意一条消息沿 `parentUuid` 回溯到根，或从某条消息向下顺着子边展开，就能拿到完整因果链——assistant turn、它发起的 tool_use、以及回来的 tool_result（tool_result 的父指针精确指向发起它的 assistant 行）层层相连。`promptId` 在这里是**定位锚点**而非分组键：它只盖在 user 行上（assistant 行没有），所以你用它**找到某次 prompt 的起点 user 行**，再沿 `parentUuid`/子边把这次 prompt 的 assistant 与 tool 因果走出来——**单靠"按 promptId 分组"收不齐所有消息**（会漏掉不带 promptId 的 assistant 行）。这就是 transcript 版的 `EXPLAIN`——而且它默认就在你硬盘上，不需要开任何遥测。

**配套的只读入口**。用户不必手动解析 JSONL，CC 内置了几个观测面板：`/cost`（本次会话的 token/成本/时长汇总，见 §16.5）、`/status`、`/context`，以及 `--debug` 打开的调试日志。社区工具（如按 transcript 统计用量的 ccusage、抓 wire 的 claude-tap）也都建立在这份 JSONL 或其等价数据上。

### 活例子：`/goal` 与 `/loop` 的跨 turn trace

抽象讲完，用两个真实功能落地"一次任务跨多 turn 怎么被追踪"。以下来自对 2.1.201 客户端二进制的静态字符串抽取 + 明文反代抓包（**post-snapshot 逆向实锤**，非 v2.1.88 源码，独立标注）：

- **`/goal`（会话级自治循环）** 自带 trace 字段。设一个目标后，transcript 里会挂一条 `goal_status` attachment，字段 `{condition, iterations, durationMs, tokens, met}`——**每个目标迭代了几轮、耗了多久、烧了多少 token、达成没**，全落进会话记录。配套遥测事件 `tengu_goal_achieved` / `tengu_goal_failed` / `tengu_goal_restored_on_resume`。这正是"一次目标跨多 turn 的执行链"的具体化：一个 prompt-based Stop hook 每轮结束后派一个独立评估器 LLM 判 yes/no，每一轮的 iteration/duration/token 都被累计进 `goal_status`。
- **`/loop`（定时/自定节奏循环）** 靠一组 `tengu_` 事件追踪自排程的每次 tick：`tengu_loop_dynamic_wakeup_scheduled` / `_ends_turn` / `_aged_out`、`tengu_loop_keepalive_fired`、`tengu_loop_ended`，以及后台 KAIROS daemon 的 `tengu_kairos_*`。每次 loop tick 重投同一个 prompt，都在这条事件流里留下痕迹。

这两个功能共同说明：**CC 的"跨多 turn 的一次任务"不是黑盒**——它要么落成 transcript 里的结构化 attachment（goal），要么落成一串带语义的 `tengu_` 事件（loop），两条路都能重建"这次自治跑了几轮、每轮什么状态"。

---

## 16.5 成本与 token 核算：按 turn 累计

`/cost` 面板背后是 `cost-tracker.ts` 的按 turn 累计。每次 API 成功，`addToTotalSessionCost` 把这次的 usage 累进 per-model 汇总，并**同时喂给两个 metric counter**：

```typescript
// cost-tracker.ts:291
getCostCounter()?.add(cost, attrs)                                    // claude_code.cost.usage
getTokenCounter()?.add(usage.input_tokens,  { ...attrs, type: 'input' })
getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0,     { ...attrs, type: 'cacheRead' })
getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0, { ...attrs, type: 'cacheCreation' })
```

`attrs` 带 `model`，fast 模式下再带 `speed:'fast'`。token 被拆成 input/output/cacheRead/cacheCreation 四类——因为四类**单价不同**，缓存读命中远比新输入便宜，分开计才能算准钱。CC 甚至递归给"advisor"子调用单独计费并发 `tengu_advisor_tool_token_usage` 事件（`cost-tracker.ts:304`），确保派生调用的成本不漏计。

**跨会话接续**。成本状态会存进 project config（`saveCurrentSessionCosts`，`cost-tracker.ts:143`），`/resume` 同一会话时 `restoreCostStateForSession` 按 sessionId 匹配后恢复（`cost-tracker.ts:130`）——续跑一个会话，`/cost` 不会从零开始。进程退出时 `costHook.ts` 挂的 `useCostSummary` 打印总账并存盘（`costHook.ts:6`），`formatTotalCost` 的输出（`cost-tracker.ts:228`：Total cost / duration API+wall / code changes / usage by model）就是你在终端看到的那块摘要。

这一节回答了读者 `EXPLAIN` 里的"每环节消耗哪些资源"：token 与钱不是一次性总数，而是**每次 API 调用即时累加、按模型和 token 类型分维、且能跨 resume 接续**的运行量。

---

## 16.6 权限决策日志：可观测的"为什么放行/拦截"

`EXPLAIN` 里还有一环是"每个动作是否正常、是否被拦"。CC 把每一次工具权限决策都记成可观测数据。核心是 `permissionLogging.ts` 的 `logPermissionDecision`——它是**所有 approve/reject 的单一出口**，一次决策 fan-out 到四个去处：

```typescript
// hooks/toolPermission/permissionLogging.ts:181（简化）
function logPermissionDecision(ctx, args, permissionPromptStartTimeMs?) {
  // ① 内部分析事件：按来源分不同事件名，做审批漏斗
  //    tengu_tool_use_granted_in_config / _in_prompt_permanent / _by_classifier / …
  //    tengu_tool_use_rejected_in_prompt / _denied_in_config
  // ② 代码编辑工具计数器（Edit/Write/NotebookEdit）
  getCodeEditToolDecisionCounter()?.add(1, { decision, source, tool_name, language })
  // ③ 把决策存进 toolUseContext，供 trace 的 tool.blocked_on_user span 读取
  toolUseContext.toolDecisions.set(toolUseID, { source, decision, timestamp })
  // ④ 客户 OTel 事件
  void logOTelEvent('tool_decision', { decision, source, tool_name })
}
```

几个观测价值点：

- **`source` 把"谁批的"结构化**：`config`（allowlist 自动放行）、`hook`、`user_permanent` / `user_temporary`（用户点了"总是/这次"）、`classifier`（分类器）、`user_reject` / `user_abort`。审计时你能区分"这次危险操作是用户手动放的、还是被规则自动放的"——这对安全复盘是决定性的。
- **`waiting_for_user_permission_ms` 把"等人"量化**（`permissionLogging.ts:189`）：只在真正弹了 prompt 时才记，自动放行不计。它和 §16.4 的 `tool.blocked_on_user` span 是一回事的两种呈现。
- **决策被回写进 `toolUseContext.toolDecisions`**，于是 trace 层的 `tool.blocked_on_user` span 结束时能标上准确的 `decision`/`source`（`toolExecution.ts:1171`）——三个平面在权限这一点上是打通的。

关于权限模型本身（`ask`/`allow`/`deny`、分类器、hook 决策）详见[第 12 章：权限与安全](11-permission-security.md)；本章只关心"决策如何被观测"。

---

## 16.7 一方 / 内部专用的观测设施（诚实边界）

不是所有观测设施都是给外部用户的，讲清楚哪些你能用、哪些只在 Anthropic 内部或编译期就被裁掉，才不会误判。

- **`ant-trace`：外部构建里是空壳**。这个命令在快照里的实现就是一行 stub：`export default { isEnabled: () => false, isHidden: true, name: 'stub' }`（`commands/ant-trace/index.js`）。它是 `feature()` 编译期 dead-code-elimination 的活标本——**外部包里物理上没有真实现，只留一个永远 disabled 的占位**。这也提醒：在公开产物里搜不到某功能的完整逻辑，不等于该功能不存在，只是被编译期裁掉了（详见方法论：抽"文字"可实锤，抽"被裁掉的逻辑"不可得）。

- **Perfetto 本地追踪：Ant-only、编译期门控**。这是一份能在 Perfetto UI 里当火焰图看的本地 trace（interaction / llm_request / tool 的 span + swarm 的 agent 层级），但**不是外部用户随手可开的**：源码头注明 "Perfetto Tracing for Claude Code (Ant-only)" 且 "ant-only and eliminated from external builds"（`perfettoTracing.ts:2`、`:7`），初始化整体裹在 `feature('PERFETTO_TRACING')` 编译门控里（`perfettoTracing.ts:260`）。也就是说，`CLAUDE_CODE_PERFETTO_TRACE=1` 只在 Perfetto 特性被编译进包时才生效——外部构建里这段被 DCE 掉了，env 变量置了也没用。它和 §16.4 的 OTel span 复用同一套 start/end 埋点，但落地在本地文件、走独立的一方观测路径。

- **一方 BigQuery 导出**：§16.2 讲过，受组织级 `metricsOptOut` 门控，去 Anthropic 的 BigQuery，外部用户不经手。

- **FPS 自观测**：`fpsTracker.ts` / `context/fpsMetrics.tsx` 追踪终端渲染帧率（average / low-1%），会话结束时随成本一起存盘（`cost-tracker.ts:158`）。一个 TUI 把自己的渲染流畅度也纳入自观测，这是 UX 工程的细致处（渲染器细节见[第 14 章：用户体验设计](12-user-experience.md)）。

- **beta 追踪的可见性矩阵**：详细追踪路径（`betaSessionTracing.ts`）能把 system prompt、模型输出、工具输入输出也写进 span，但有一张严格的可见性表——`thinking_output` 是 **ant-only**（`betaSessionTracing.ts:12` 的矩阵 + `logging.ts:746` 的 `USER_TYPE === 'ant'` 门），外部用户即使开满详细追踪也拿不到模型的思考原文。system prompt 全文只在这条 beta 路径、且每个唯一 hash 只发一次（`betaSessionTracing.ts:266`），避免重复上报大文本。

> 关于 CC 反过来对**逆向者**埋的观测/防御仪器（anti-distillation 假工具注入、隐写指纹等），属于另一个话题，本项目走 clean-room 解读、不展开；参见仓库免责声明与逆向方法论笔记。

---

## 16.8 隐私与安全边界：能观测什么 / 故意不导出什么

可观测性和隐私天然拉扯：记得越多越好查，但也越危险。CC 的总原则是——**默认只导出"形状"，内容一律 opt-in**。

"形状"指长度、token 数、时长、模型名、工具名、决策类型这类元数据；"内容"指 prompt 原文、模型回复、工具的输入输出、代码与文件内容。默认导出的是前者，后者要逐项显式开启：

| 内容 | 开关（env） | 默认行为 | 源码/依据 |
|---|---|---|---|
| 用户 prompt 原文 | `OTEL_LOG_USER_PROMPTS` | `<REDACTED>`，只导出 `prompt_length` | `events.ts:17` `redactIfDisabled` |
| 助手回复原文 | `OTEL_LOG_ASSISTANT_RESPONSES` | 脱敏，只导出长度 | 据官方文档（超快照） |
| 工具参数/命令 | `OTEL_LOG_TOOL_DETAILS` | 省略/脱敏 | `toolExecution.ts:1135`、`metadata.ts:87` |
| 工具输入输出内容 | `OTEL_LOG_TOOL_CONTENT` | 省略 | `sessionTracing.ts:738` |
| 原始 API 请求/响应体 | `OTEL_LOG_RAW_API_BODIES` | 不导出 | 据官方文档（v2.1.88 快照中查无此变量） |
| system prompt 全文 | 仅 beta 详细追踪路径 | 不进普通导出 | `betaSessionTracing.ts:266` |
| 模型 thinking 原文 | 仅 `USER_TYPE=ant` | 外部永不导出 | `logging.ts:746` |

几条设计意图值得点明：

1. **代码与文件内容永不进 metrics/events**。官方明言 "Raw file contents and code snippets are not included in metrics or events."，源码侧则用 `AnalyticsMetadata_…_NOT_CODE_OR_FILEPATHS` 类型把它变成编译期约束（§16.3）。默认可观测面里，你能看到"改了 N 行 TypeScript"，但看不到改了什么。
2. **隐私门控和基数控制是同一套旋钮的两面**。`prompt.id` 只进 events、不进 metrics；`app.version` 则由 `OTEL_METRICS_INCLUDE_VERSION` 控制、**默认在共享属性集里就关掉**（`telemetryAttributes.ts:40`，该函数同时供 metrics/events/spans 用，所以关掉是全局的，只是基数爆炸的顾虑对 metrics 最尖锐）。这些既是防基数爆炸，也是"少往聚合后台塞可归因信息"。可观测性设计里，隐私和成本经常指向同一个决定。
3. **`OTEL_LOG_RAW_API_BODIES` 的诚实说明**：当前官方文档确有此开关（配 `api_request_body` / `api_response_body` 事件，60KB 截断、支持写文件），但它**不在 v2.1.88 快照里**——属于快照之后新增。本章据官方文档陈述，不作源码级实锤。
4. **OTel 导出配置不向子进程传播，但 trace 上下文会**。官方明确 "Claude Code doesn't pass `OTEL_*` environment variables to the subprocesses it spawns"——Bash 工具、hook、MCP server、language server 拿不到你的 OTLP endpoint/headers/凭证，防止观测配置外泄给第三方进程。**但这不等于"什么都不传"**：据当前官方文档，追踪激活时 CC 会把 W3C trace context（`TRACEPARENT`，由 `CLAUDE_CODE_PROPAGATE_TRACEPARENT` 控制）注入 Bash/PowerShell 子进程，好让子进程的 span 挂到同一条 trace 上。传的是"这是哪条 trace"的关联信息，不是"往哪导、用什么凭证"的配置——两者要分清。（`TRACEPARENT` / `CLAUDE_CODE_PROPAGATE_TRACEPARENT` 在 v2.1.88 快照中 grep 不到，属快照之后新增，据官方文档陈述。）

---

## 小结

Claude Code 的可观测性不是"打一堆 log"，而是一套**分平面、按用途裁剪、隐私优先**的自省系统。它能正面回答读者的 `EXPLAIN of a prompt`：一次 prompt 的多个 turn、每 turn 的 API 与工具动作、各自的耗时/token/成本、在哪等人、决策来源、是否报错——其中 events 由 `prompt.id` 归拢，trace 由 span parentage 串起，transcript 由 `parentUuid` 连边重建因果链。核心设计决策：

| 设计决策 | 原因 |
|---|---|
| 四层分离（Metrics / Events / Trace / Transcript） | 聚合、离散查询、因果分析、完整重放是四种问题，各需一种数据结构 |
| `prompt.id` 关联 events、锚定 transcript user 行；trace 靠 span parentage | 离散层用 id 关联、因果层用 parentage、聚合层拒绝 id——各层用最合适的关联方式 |
| `prompt.id` 仅 event 级；`app.version` 默认从共享属性集关掉 | 高基数维度会打爆时序库——基数纪律 |
| metric 全用单调 Counter + delta 时序 | 对丢失/重启/乱序鲁棒，聚合只需 delta |
| 内部 `tengu_` 与客户 OTel 双管线 | 官方产品分析与用户自有可观测互不干扰，从同一 call site 并发 |
| `tool.blocked_on_user` 独立成 span | 把"等人"从"执行"里剥离，墙钟时间归因才准确 |
| 并发 LLM 请求显式传 span 句柄 | 根治并行追踪的响应错配 |
| WeakRef + 30min 兜底清理 span | 观测设施自己不能变成内存泄漏 |
| transcript = parentUuid 链表 + tool_result 指向发起它的 assistant | 因果靠显式连边，不靠时间顺序猜 |
| metricsOptOut 两级缓存 + 熔断 | 观测不能拖垮主流程：可廉价关闭、容忍陈旧读 |
| 默认只导出"形状"、内容逐项 opt-in | 可观测性与隐私的张力，用"默认脱敏 + 显式开启"化解 |
| 导出器惰性 `import()` | 重依赖挪出冷启动关键路径 |

> **版本说明**：本章基于 v2.1.88 泄露源码快照做源码级走读，并与官方 [Monitoring 文档](https://code.claude.com/docs/en/monitoring-usage)交叉核对。8 个 metric 在快照与当前文档间稳定一致；events 清单在快照之后大幅扩张，凡超出快照的标识符均已标注"据官方文档"。`/goal`、`/loop` 的 trace 细节来自对 2.1.201 客户端的 post-snapshot 逆向（静态字符串 + 反代抓包），已单独标注，不与快照实锤混同。
