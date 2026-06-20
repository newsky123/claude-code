# Hook 机制与遥测实现分析

本文档分析 Claude Code 查询执行引擎（engine）中的全部 Hook 机制，以及遥测（telemetry / analytics）功能如何基于这些 Hook 点实现。

> 说明：本仓库是反编译/重构版本。`feature()` 在本构建中恒为 `false`，因此所有 feature-gated 的 Hook 事件（如 beta tracing OTEL、KAIROS 等）是死代码，但其设计逻辑仍有参考价值。本文会标注哪些路径在当前构建中实际生效。

---

## 一、总览：两类"Hook"

项目里"hook"一词承载两种完全不同的机制，必须先区分清楚：

| 类别 | 含义 | 配置/定义位置 | 用途 |
|------|------|--------------|------|
| **用户可配置 Hooks** | `settings.json` 里声明的、在生命周期事件触发时执行的外部命令/Prompt/Agent/HTTP | `settings.json` 的 `hooks` 字段 | 用户扩展引擎行为（lint、格式化、阻断危险操作等）|
| **内部回调 Hooks** | 代码内部注册的 `type: 'callback'` 且 `internal: true` 的函数钩子 | `registerHookCallbacks()` | 引擎自身用来在生命周期点埋遥测、做归因（attribution）|

两者**走同一套执行引擎**（`executeHooks`），但内部 Hook 走快速路径且不计入 `tengu_run_hook` 指标。这一点是"遥测基于 Hook 实现"的核心——遥测本身就被实现成了一种内部 Hook。

---

## 二、Hook 事件全集

完整事件列表定义在 `src/entrypoints/sdk/coreTypes.ts:25`（和 `coreSchemas.ts:355` 的 zod 版本）：

```
PreToolUse          工具执行前（可阻断/改写输入/决定权限）
PostToolUse         工具执行后（可注入上下文/改写 MCP 输出）
PostToolUseFailure  工具执行失败后
Notification        发送通知时
UserPromptSubmit    用户提交 prompt 时（可注入 additionalContext）
SessionStart        会话开始（source: startup|resume|clear|compact）
SessionEnd          会话结束（reason）
Stop                Claude 结束响应前（可阻止收尾、强制继续）
StopFailure         API 错误导致 turn 结束
SubagentStart       子 agent 启动
SubagentStop        子 agent 结束响应前
PreCompact          上下文压缩前（trigger: manual|auto）
PostCompact         压缩后
PermissionRequest   权限对话弹出时
PermissionDenied    自动模式分类器拒绝工具调用后
Setup               仓库 setup（trigger: init|maintenance）
TeammateIdle        队友即将空闲
TaskCreated         任务创建
TaskCompleted       任务完成
Elicitation         MCP 服务器请求用户输入
ElicitationResult   用户响应 MCP elicitation 后
ConfigChange        配置文件 session 中改变
WorktreeCreate      创建隔离 worktree
WorktreeRemove      删除 worktree
InstructionsLoaded  指令文件（CLAUDE.md / rule）加载
CwdChanged          工作目录改变
FileChanged         被监视文件改变
```

其中 `ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup']`（`hookEvents.ts:18`）总会向 SDK 事件流广播；其余事件仅在 SDK 设置 `includeHookEvents` 或 `CLAUDE_CODE_REMOTE` 模式下广播。

---

## 三、关键文件地图

### 类型与协议
| 文件 | 作用 |
|------|------|
| `src/types/hooks.ts` | Hook 输入/输出 zod schema、`HookCallback`、`HookResult`、`AggregatedHookResult` 类型 |
| `src/entrypoints/sdk/coreTypes.ts` | `HOOK_EVENTS` 常量、`HookInput`/`HookJSONOutput` SDK 类型 |

### 执行引擎
| 文件 | 作用 |
|------|------|
| `src/utils/hooks.ts` | **核心**：`executeHooks()`、`getMatchingHooks()`、各 `execute*Hooks()` 生命周期封装、`execCommandHook()` |
| `src/utils/hooks/execPromptHook.ts` | prompt 型 hook（跑独立 LLM 查询）|
| `src/utils/hooks/execAgentHook.ts` | agent 型 hook（调起子 agent）|
| `src/utils/hooks/execHttpHook.ts` | http 型 hook（POST）|
| `src/utils/hooks/ssrfGuard.ts` | http hook 的 SSRF 防护 |
| `src/utils/hooks/sessionHooks.ts` | session 级动态 hook（function hook）存储 |
| `src/utils/hooks/AsyncHookRegistry.ts` | 异步 hook 注册表与 rewake |

### 配置加载与匹配
| 文件 | 作用 |
|------|------|
| `src/utils/hooks/hooksConfigManager.ts` | hook 事件元数据、`groupHooksByEventAndMatcher()`、matcher 排序 |
| `src/utils/hooks/hooksConfigSnapshot.ts` | trust 对话前抓取 hook 配置（安全）、`shouldAllowManagedHooksOnly()`、`shouldDisableAllHooksIncludingManaged()` |
| `src/utils/hooks/hookEvents.ts` | SDK 事件广播（started/progress/response）|

### 工具调用集成层
| 文件 | 作用 |
|------|------|
| `src/services/tools/toolHooks.ts` | `runPreToolUseHooks` / `runPostToolUseHooks` / `runPostToolUseFailureHooks`，把 hook 结果翻译成消息与权限决策 |
| `src/services/tools/toolExecution.ts` | 在工具执行流程中调用上述三个 hook 包装器 |

### 遥测/Analytics
| 文件 | 作用 |
|------|------|
| `src/services/analytics/index.ts` | `logEvent()` 公共 API、事件队列、`stripProtoFields()` |
| `src/services/analytics/sink.ts` | 实际路由：采样 → Datadog + 1P |
| `src/services/analytics/firstPartyEventLogger.ts` | 1P 事件日志 + `shouldSampleEvent()` |
| `src/services/analytics/metadata.ts` | `sanitizeToolNameForAnalytics()` 等元数据净化 |
| `src/utils/sessionFileAccessHooks.ts` | **内部 Hook 埋点**：把文件访问遥测实现为 PostToolUse callback hook |
| `src/utils/attributionHooks.ts` | 内部 Hook：commit 归因 |

---

## 四、执行引擎工作流：`executeHooks()`

`src/utils/hooks.ts:2034` 是所有 hook 的统一入口（async generator）。流程：

```
executeHooks({ hookInput, toolUseID, matchQuery, ... })
│
├─ 1. 短路检查
│    ├─ shouldDisableAllHooksIncludingManaged() → 直接 return
│    └─ CLAUDE_CODE_SIMPLE 模式 → return
│
├─ 2. 安全门禁（关键）
│    └─ shouldSkipHookDueToTrust()
│         交互模式必须通过 trust dialog；非交互(SDK)隐式信任。
│         "ALL hooks require workspace trust" —— 防 RCE 的集中检查点
│
├─ 3. getMatchingHooks(appState, sessionId, hookEvent, hookInput, tools)
│         按事件 + matcher 匹配出本次要跑的 hook 列表
│         匹配为空 → return
│
├─ 4. 分流：用户 hook vs 全内部 hook
│    │
│    ├─ userHooks.length > 0：
│    │    logEvent('tengu_run_hook', { hookName, numCommands,
│    │                                 hookTypeCounts, pluginHookCounts })
│    │    → 进入完整路径（progress 消息、span、超时、JSON 解析、结果聚合）
│    │
│    └─ 全部是 internal callback（快速路径）：
│         直接顺序 await hook.callback(...)，跳过 span/progress/abort
│         (实测 6.01µs → ~1.8µs，-70%)
│         logEvent('tengu_repl_hook_finished', { numSuccess, ... })
│
└─ 5. 完整路径执行各 hook（command/prompt/agent/http/callback）
     ├─ 逐个 yield hook_progress 消息（UI 展示）
     ├─ execCommandHook / execPromptHook / execAgentHook / execHttpHook
     ├─ processHookJSONOutput 解析 stdout JSON（decision/permissionDecision/...）
     └─ 聚合为 AggregatedHookResult yield 出去
```

**Hook 的 5 种类型**（由 `getHookDisplayText` 等区分）：
- `command` — shell 命令（bash / PowerShell），通过 stdin 传 JSON、stdout 回 JSON、exit code 2 表示阻断
- `prompt` — 跑一次独立 LLM 查询
- `agent` — 调起一个子 agent
- `http` — HTTP POST（带 SSRF 防护）
- `callback` / `function` — 进程内函数（内部遥测/归因走这条）

---

## 五、Hook 在查询生命周期中的挂载点（全量）

每个 `execute*Hooks()` 都是对 `executeHooks()`（REPL 路径）或 `executeHooksOutsideREPL()`（非 REPL 路径，如 shutdown、通知）的一层封装。

> **为什么第二节列了 27 个事件，但之前只列了 10 行？**
>
> 前一版只列了 REPL 内的"主要"调用点。实际上所有 27 个事件**都有**对应的 `execute*Hooks` 封装函数，只是部分事件走 `executeHooksOutsideREPL`（非 async generator，直接 Promise）或专用的 `executeEnvHooks` 包装。以下是完整列表。

### 5.1 AsyncGenerator 路径（REPL 内，可 yield 消息/阻断）

| Hook 事件 | 封装函数 | matcher 含义 | 调用点 |
|-----------|---------|-------------|--------|
| PreToolUse | `executePreToolHooks` | tool_name | `toolHooks.ts:466` ← `toolExecution.ts:800` |
| PostToolUse | `executePostToolHooks` | tool_name | `toolHooks.ts:56` ← `toolExecution.ts:1483` |
| PostToolUseFailure | `executePostToolUseFailureHooks` | tool_name | `toolHooks.ts:212` ← `toolExecution.ts:1700` |
| PermissionDenied | `executePermissionDeniedHooks` | tool_name | `toolExecution.ts`（自动模式分类器拒绝后）|
| UserPromptSubmit | `executeUserPromptSubmitHooks` | — | `processUserInput.ts:182` |
| SessionStart | `executeSessionStartHooks` | source | `sessionStart.ts:132` |
| Setup | `executeSetupHooks` | trigger | `setup.ts`（仓库初始化/维护）|
| Stop / SubagentStop | `executeStopHooks` | — | `query/stopHooks.ts:180`、`runAgent.ts` |
| SubagentStart | `executeSubagentStartHooks` | agent_type | `tools/AgentTool/runAgent.ts` |
| TeammateIdle | `executeTeammateIdleHooks` | — | 队友空闲前触发 |
| TaskCreated | `executeTaskCreatedHooks` | — | 任务创建工具内 |
| TaskCompleted | `executeTaskCompletedHooks` | — | 任务完成工具内 |
| PermissionRequest | `executePermissionRequestHooks` | tool_name | 权限弹窗前 |

### 5.2 Promise 路径（REPL 外 / fire-and-forget）

| Hook 事件 | 封装函数 | matcher 含义 | 调用点 |
|-----------|---------|-------------|--------|
| Notification | `executeNotificationHooks` | notification_type | 通知发送时 |
| StopFailure | `executeStopFailureHooks` | error | API 错误导致 turn 结束 |
| PreCompact | `executePreCompactHooks` | trigger | `services/compact/compact.ts` |
| PostCompact | `executePostCompactHooks` | trigger | compact 完成后 |
| SessionEnd | `executeSessionEndHooks` | reason | `gracefulShutdown.ts` |
| ConfigChange | `executeConfigChangeHooks` | source | 配置文件 watcher |
| InstructionsLoaded | `executeInstructionsLoadedHooks` | load_reason | `claudemd.ts`、`attachments.ts` |
| Elicitation | `executeElicitationHooks` | mcp_server_name | `services/mcp/elicitationHandler.ts:214` |
| ElicitationResult | `executeElicitationResultHooks` | mcp_server_name | `elicitationHandler.ts:264` |

### 5.3 Env 路径（返回 watchPaths + systemMessages）

| Hook 事件 | 封装函数 | matcher 含义 | 调用点 |
|-----------|---------|-------------|--------|
| CwdChanged | `executeCwdChangedHooks` | — | CWD 切换后 |
| FileChanged | `executeFileChangedHooks` | — | `fileChangedWatcher.ts` 检测到文件变化 |

### 5.4 专用路径（独立返回值格式）

| Hook 事件 | 封装函数 | 调用点 | 返回值特殊性 |
|-----------|---------|--------|------------|
| WorktreeCreate | `executeWorktreeCreateHook` | `worktree.ts:715/912` | 返回 `{ worktreePath }` |
| WorktreeRemove | `executeWorktreeRemoveHook` | `worktree.ts:826/968` | 返回 `boolean`（hook 是否执行了）|

---

## 5.5 PreToolUse 的特殊地位：权限钩子

`runPreToolUseHooks`（`toolHooks.ts:435`）不仅能注入上下文，还能**决定权限**。其 yield 出的 `hookPermissionResult` 经 `resolveHookPermissionDecision()`（`toolHooks.ts:332`）解析。关键不变量：

> Hook 的 `allow` **不能**绕过 `settings.json` 的 deny/ask 规则——`checkRuleBasedPermissions` 仍会执行。

决策优先级：
- hook `deny` → 直接拒绝
- hook `allow` → 再过规则检查；命中 deny 规则则 deny，命中 ask 规则仍弹窗
- hook `ask` → 走正常弹窗，但用 hook 的 message 作为提示
- hook 无决策 → 正常权限流程（可附带 `updatedInput` 改写工具入参）

这是引擎内部"权限钩子"与"用户 Hook"交汇的地方：`canUseTool`（来自 `QueryEngine.wrappedCanUseTool`）是底层权限回调，PreToolUse hook 是叠加在其之上的可编程层。

---

## 六、遥测如何基于 Hook 实现

### 6.1 logEvent 的基础设施

`src/services/analytics/index.ts`：
- `logEvent(eventName, metadata)` 是无依赖的纯入口，sink 未挂载前事件进 `eventQueue` 排队，`attachAnalyticsSink()` 后用 `queueMicrotask` 异步排空（不阻塞启动）。
- metadata 类型刻意**禁止裸字符串**：要传字符串必须断言 `as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`，从类型层面防止误把代码/文件路径上报。
- `_PROTO_*` 前缀的 key 是 PII-tagged，仅 1P 特权列可见；`stripProtoFields()` 在送 Datadog 前剥离。

`src/services/analytics/sink.ts` 的 `logEventImpl`：采样（`shouldSampleEvent`）→ Datadog（剥离 PROTO）+ 1P（保留 PROTO）。

### 6.2 两种埋点方式

遥测与 Hook 的关系体现在**两个层面**：

**(A) Hook 执行本身被埋点** —— 引擎在 hook 执行的关键节点直接 `logEvent`：

| 事件 | 位置 | 含义 |
|------|------|------|
| `tengu_run_hook` | `hooks.ts:2105` | 用户 hook 批次开始（含 hookName、数量、类型分布、插件分布）|
| `tengu_repl_hook_finished` | `hooks.ts:2138/3018` | 批次完成（numSuccess/numBlocking/numNonBlockingError/numCancelled/totalDurationMs）|
| `tengu_pre_tool_hooks_cancelled` | `toolHooks.ts:583` | PreToolUse 被中止 |
| `tengu_pre_tool_hook_error` | `toolHooks.ts:607` | PreToolUse 执行报错 |
| `tengu_post_tool_hooks_cancelled` | `toolHooks.ts:72` | PostToolUse 被中止 |
| `tengu_post_tool_hook_error` | `toolHooks.ts:154` | PostToolUse 执行报错 |
| `tengu_post_tool_failure_hook_error` | `toolHooks.ts:283` | PostToolUseFailure 执行报错 |
| `tengu_agent_stop_hook_*` | `execAgentHook.ts` | agent hook 成功/超 turns/报错 |

这些事件都带 `queryChainId` 和 `queryDepth`（来自 `toolUseContext.queryTracking`），可在多 agent / fork 场景下还原调用链。

**(B) 遥测本身被实现为内部 Hook** —— 这是更精巧的部分。

`src/utils/sessionFileAccessHooks.ts:233` `registerSessionFileAccessHooks()`：

```ts
const hook: HookCallback = {
  type: 'callback',
  callback: handleSessionFileAccess,
  timeout: 1,           // 极短超时，只是打日志
  internal: true,       // 不计入 tengu_run_hook
}
registerHookCallbacks({
  PostToolUse: [
    { matcher: FILE_READ_TOOL_NAME, hooks: [hook] },
    { matcher: GREP_TOOL_NAME,      hooks: [hook] },
    { matcher: GLOB_TOOL_NAME,      hooks: [hook] },
    { matcher: FILE_EDIT_TOOL_NAME, hooks: [hook] },
    { matcher: FILE_WRITE_TOOL_NAME,hooks: [hook] },
  ],
})
```

引擎不需要在每个文件工具里手写埋点代码，而是把"记录文件访问"注册成一个监听 PostToolUse 的内部 callback hook。每当 Read/Grep/Glob/Edit/Write 执行完，引擎走 PostToolUse hook 流程，自动触发 `handleSessionFileAccess`，内部再 `logEvent('tengu_team_mem_file_edit', ...)` 等。`attributionHooks`（commit 归因）同理。

### 6.3 为什么内部 Hook 要走快速路径

`executeHooks` 第 4 步用 `isInternalHook`（`hooks.ts:1522`）过滤：若本次匹配的全是 internal callback，则跳过 span/progress/abort/JSON 解析，直接顺序 await。原因：
- 内部 hook 返回 `{}`、不使用 abort 信号、不需要 UI progress；
- 文件工具调用极频繁，快速路径把单次开销从 ~6µs 降到 ~1.8µs；
- 内部 hook 不发 `tengu_run_hook`（那是用户可见 hook 的指标），只发 `tengu_repl_hook_finished`，避免污染用户 hook 统计。

这就是"遥测寄生在 Hook 系统上但对用户透明"的设计：**复用 Hook 的生命周期分发能力做埋点，同时通过 `internal: true` 把自己从用户 Hook 的可观测面里摘出去。**

---

## 七、安全与信任模型

Hook 可执行任意命令，是 RCE 的高危面，因此有多层防护：

1. **集中信任门禁**：`shouldSkipHookDueToTrust()` 在 `executeHooks` 入口，所有 hook（含未来新增的）统一检查 workspace trust。交互模式需通过 trust dialog，非交互 SDK 隐式信任。
2. **配置快照**：`hooksConfigSnapshot.ts` 在 trust dialog 显示**之前**抓取 hook 配置，防止 dialog 显示后被篡改绕过。
3. **托管隔离**：`shouldAllowManagedHooksOnly()` 可限制只跑 `policySettings`（企业托管）来源的 hook。`shouldDisableAllHooksIncludingManaged()` 可全关。
4. **SSRF 防护**：http hook 经 `ssrfGuard.ts` 校验目标地址。
5. **PII 防护**：遥测层用 `never` 标记类型 + `stripProtoFields` 双重保证不泄露代码/路径。

---

## 八、端到端示例：一次 Edit 工具调用触发了哪些 Hook 与遥测

```
用户让 Claude 改一个文件
  │
  ├─ [UserPromptSubmit hook] processUserInput.ts:182
  │     用户可注册 hook 注入 additionalContext
  │
  ├─ 模型返回 tool_use(Edit)
  │
  ├─ [PreToolUse hook] toolExecution.ts:800 → runPreToolUseHooks
  │     ├─ 用户 hook 可 deny / 改写 input / 注入上下文
  │     ├─ logEvent('tengu_run_hook', { hookName:'PreToolUse:Edit', ... })  (若有用户 hook)
  │     └─ resolveHookPermissionDecision → 叠加 settings 规则 → 最终权限
  │
  ├─ 工具实际执行（写文件）
  │
  ├─ [PostToolUse hook] toolExecution.ts:1483 → runPostToolUseHooks
  │     ├─ 内部 hook：handleSessionFileAccess（internal:true，快速路径）
  │     │     → logEvent('tengu_..._file_edit', { queryChainId, queryDepth })
  │     ├─ logEvent('tengu_repl_hook_finished', { numSuccess:1, ... })
  │     └─ 用户 hook（如有）：可注入 additionalContext / 改写 MCP 输出
  │
  └─ turn 结束 → [Stop hook] query/stopHooks.ts:180
        用户 hook 可阻止收尾、强制模型继续
```

---

## 九、如何添加新的 Hook 事件

添加一个全新的 Hook 事件需要改 4 层。以假设新增 `PreFileWrite` 事件为例：

### 步骤 1：注册事件名

在 `src/entrypoints/sdk/coreTypes.ts`（和 `coreSchemas.ts`）的 `HOOK_EVENTS` 数组中添加：

```ts
export const HOOK_EVENTS = [
  // ...existing
  'PreFileWrite',     // ← 新增
] as const
```

### 步骤 2：定义 HookInput schema

在 `src/entrypoints/sdk/coreSchemas.ts` 中添加对应的输入 schema：

```ts
export const PreFileWriteHookInputSchema = lazySchema(() =>
  BaseHookInputSchema().extend({
    hook_event_name: z.literal('PreFileWrite'),
    file_path: z.string(),
    content: z.string(),
  })
)
```

### 步骤 3：添加事件元数据

在 `src/utils/hooks/hooksConfigManager.ts` 的 `getHookEventMetadata()` 中添加：

```ts
PreFileWrite: {
  description: 'Runs before a file write operation',
  matcherDescription: 'File path pattern',
  matcherKey: 'file_path',
  supportsMatcher: true,
}
```

### 步骤 4：编写 execute 封装函数

在 `src/utils/hooks.ts` 中添加：

```ts
export async function* executePreFileWriteHooks(
  filePath: string,
  content: string,
  toolUseContext: ToolUseContext,
  signal?: AbortSignal,
): AsyncGenerator<AggregatedHookResult> {
  const hookInput: PreFileWriteHookInput = {
    ...createBaseHookInput(undefined, undefined, toolUseContext),
    hook_event_name: 'PreFileWrite',
    file_path: filePath,
    content,
  }
  yield* executeHooks({
    hookInput,
    toolUseID: randomUUID(),
    matchQuery: filePath,
    signal,
    toolUseContext,
  })
}
```

### 步骤 5：在业务代码中调用

在工具执行路径（如 `toolExecution.ts`）的适当位置调用。

### 注册内部 Callback Hook（遥测用）

不需要新事件，只需要在已有事件上挂新回调：

```ts
registerHookCallbacks({
  PostToolUse: [
    { matcher: 'MyNewTool', hooks: [{
      type: 'callback',
      callback: myTelemetryHandler,
      timeout: 1,
      internal: true,  // 走快速路径，不污染用户 hook 指标
    }] },
  ],
})
```

调用时机：`setup.ts` 初始化阶段（`registerSessionFileAccessHooks()` 就在这里被调用）。

### 注册 Session 级动态 Hook

通过 `sessionHooks.ts` 的 `addSessionHook()` 或 `addFunctionHook()`，运行时动态添加 hook，session 结束自动清除：

```ts
addFunctionHook(setAppState, sessionId, 'Stop', '*', myCallback, 'error msg')
```

---

## 十、每个 Hook 的函数原型

所有封装函数定义在 `src/utils/hooks.ts`，以下按返回值类型分组列出完整签名。

### 10.1 AsyncGenerator 路径（可 yield 消息、阻断、权限决策）

```ts
// ─── 工具相关 ───

async function* executePreToolHooks<ToolInput>(
  toolName: string,
  toolUseID: string,
  toolInput: ToolInput,
  toolUseContext: ToolUseContext,
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,    // default: TOOL_HOOK_EXECUTION_TIMEOUT_MS
  requestPrompt?: (sourceName: string, toolInputSummary?: string | null)
    => (request: PromptRequest) => Promise<PromptResponse>,
  toolInputSummary?: string | null,
): AsyncGenerator<AggregatedHookResult>

async function* executePostToolHooks<ToolInput, ToolResponse>(
  toolName: string,
  toolUseID: string,
  toolInput: ToolInput,
  toolResponse: ToolResponse,
  toolUseContext: ToolUseContext,
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
): AsyncGenerator<AggregatedHookResult>

async function* executePostToolUseFailureHooks<ToolInput>(
  toolName: string,
  toolUseID: string,
  toolInput: ToolInput,
  error: string,
  toolUseContext: ToolUseContext,
  isInterrupt?: boolean,
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
): AsyncGenerator<AggregatedHookResult>

async function* executePermissionDeniedHooks<ToolInput>(
  toolName: string,
  toolUseID: string,
  toolInput: ToolInput,
  reason: string,
  toolUseContext: ToolUseContext,
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
): AsyncGenerator<AggregatedHookResult>

async function* executePermissionRequestHooks<ToolInput>(
  toolName: string,
  toolUseID: string,
  toolInput: ToolInput,
  toolUseContext: ToolUseContext,
  permissionMode?: string,
  permissionSuggestions?: PermissionUpdate[],
  signal?: AbortSignal,
  timeoutMs?: number,
  requestPrompt?: (...) => ...,
  toolInputSummary?: string | null,
): AsyncGenerator<AggregatedHookResult>

// ─── 用户输入与会话 ───

async function* executeUserPromptSubmitHooks(
  prompt: string,
  permissionMode: string,
  toolUseContext: ToolUseContext,
  requestPrompt?: (...) => ...,
): AsyncGenerator<AggregatedHookResult>

async function* executeSessionStartHooks(
  source: 'startup' | 'resume' | 'clear' | 'compact',
  sessionId?: string,
  agentType?: string,
  model?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
  forceSyncExecution?: boolean,
): AsyncGenerator<AggregatedHookResult>

async function* executeSetupHooks(
  trigger: 'init' | 'maintenance',
  signal?: AbortSignal,
  timeoutMs?: number,
  forceSyncExecution?: boolean,
): AsyncGenerator<AggregatedHookResult>

// ─── 停止与 Agent ───

async function* executeStopHooks(
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
  stopHookActive?: boolean,  // default: false
  subagentId?: AgentId,      // 有值时 hookEvent='SubagentStop'，否则='Stop'
  toolUseContext?: ToolUseContext,
  messages?: Message[],
  agentType?: string,
  requestPrompt?: (...) => ...,
): AsyncGenerator<AggregatedHookResult>

async function* executeSubagentStartHooks(
  agentId: string,
  agentType: string,
  signal?: AbortSignal,
  timeoutMs?: number,
): AsyncGenerator<AggregatedHookResult>

// ─── 团队与任务 ───

async function* executeTeammateIdleHooks(
  teammateName: string,
  teamName: string,
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
): AsyncGenerator<AggregatedHookResult>

async function* executeTaskCreatedHooks(
  taskId: string,
  taskSubject: string,
  taskDescription?: string,
  teammateName?: string,
  teamName?: string,
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
  toolUseContext?: ToolUseContext,
): AsyncGenerator<AggregatedHookResult>

async function* executeTaskCompletedHooks(
  taskId: string,
  taskSubject: string,
  taskDescription?: string,
  teammateName?: string,
  teamName?: string,
  permissionMode?: string,
  signal?: AbortSignal,
  timeoutMs?: number,
  toolUseContext?: ToolUseContext,
): AsyncGenerator<AggregatedHookResult>
```

### 10.2 Promise 路径（fire-and-forget 或同步等待结果）

```ts
async function executeNotificationHooks(
  notificationData: {
    message: string
    title?: string
    notificationType: string
  },
  timeoutMs?: number,
): Promise<void>

async function executeStopFailureHooks(
  lastMessage: AssistantMessage,
  toolUseContext?: ToolUseContext,
  timeoutMs?: number,
): Promise<void>

async function executePreCompactHooks(
  compactData: {
    trigger: 'manual' | 'auto'
    customInstructions: string | null
  },
  signal?: AbortSignal,
  timeoutMs?: number,
): Promise<{
  newCustomInstructions?: string
  userDisplayMessage?: string
}>

async function executePostCompactHooks(
  compactData: {
    trigger: 'manual' | 'auto'
    compactSummary: string
  },
  signal?: AbortSignal,
  timeoutMs?: number,
): Promise<{ userDisplayMessage?: string }>

async function executeSessionEndHooks(
  reason: ExitReason,
  options?: {
    getAppState?: () => AppState
    setAppState?: (updater: (prev: AppState) => AppState) => void
    signal?: AbortSignal
    timeoutMs?: number
  },
): Promise<void>

async function executeConfigChangeHooks(
  source: ConfigChangeSource,  // 'user_settings'|'project_settings'|'local_settings'|'policy_settings'|'skills'
  filePath?: string,
  timeoutMs?: number,
): Promise<HookOutsideReplResult[]>

async function executeInstructionsLoadedHooks(
  filePath: string,
  memoryType: InstructionsMemoryType,  // 'User'|'Project'|'Local'|'Managed'
  loadReason: InstructionsLoadReason,   // 'session_start'|'nested_traversal'|'path_glob_match'|'include'|'compact'
  options?: {
    globs?: string[]
    triggerFilePath?: string
    parentFilePath?: string
    timeoutMs?: number
  },
): Promise<void>

async function executeElicitationHooks(params: {
  serverName: string
  message: string
  requestedSchema?: Record<string, unknown>
  permissionMode?: string
  signal?: AbortSignal
  timeoutMs?: number
  mode?: 'form' | 'url'
  url?: string
  elicitationId?: string
}): Promise<ElicitationHookResult>

async function executeElicitationResultHooks(params: {
  serverName: string
  action: 'accept' | 'decline' | 'cancel'
  content?: Record<string, unknown>
  permissionMode?: string
  signal?: AbortSignal
  timeoutMs?: number
  mode?: 'form' | 'url'
  elicitationId?: string
}): Promise<ElicitationResultHookResult>
```

### 10.3 Env 路径（CWD/文件变化）

```ts
function executeCwdChangedHooks(
  oldCwd: string,
  newCwd: string,
  timeoutMs?: number,
): Promise<{
  results: HookOutsideReplResult[]
  watchPaths: string[]
  systemMessages: string[]
}>

function executeFileChangedHooks(
  filePath: string,
  event: 'change' | 'add' | 'unlink',
  timeoutMs?: number,
): Promise<{
  results: HookOutsideReplResult[]
  watchPaths: string[]
  systemMessages: string[]
}>
```

### 10.4 Worktree 专用路径

```ts
async function executeWorktreeCreateHook(
  slug: string,
): Promise<{ worktreePath: string }>
// 返回 hook 创建的 worktree 路径

async function executeWorktreeRemoveHook(
  worktreePath: string,
): Promise<boolean>
// 返回 hook 是否存在并被执行
```

### 10.5 PostSampling（采样后处理）

```ts
// src/utils/hooks/postSamplingHooks.ts:45
async function executePostSamplingHooks(
  toolUseContext: ToolUseContext,
  messages: Message[],
  signal?: AbortSignal,
): Promise<void>
```

---

## 十一、要点总结

1. **统一引擎**：所有 hook（用户的 + 内部遥测的）共用 `executeHooks()` 的事件匹配与分发。
2. **遥测即 Hook**：文件访问、commit 归因等遥测被实现为 `internal: true` 的 PostToolUse callback hook，复用生命周期分发，不污染用户 hook 指标，且走快速路径。
3. **双重埋点**：既给"hook 执行本身"埋点（`tengu_run_hook` 等），又把"业务遥测"寄生为 hook。
4. **权限是可编程层**：PreToolUse hook 叠加在底层 `canUseTool` 之上，但 hook 的 allow 不能越过 settings 的 deny/ask。
5. **安全前置**：trust 门禁集中在引擎入口，配置在 trust dialog 前快照，杜绝绕过。
6. **类型即护栏**：`AnalyticsMetadata_I_VERIFIED_*` 用 `never` 强制开发者显式确认上报内容无 PII。
