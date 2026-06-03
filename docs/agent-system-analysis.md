# Claude Code Agent 系统深度分析

> 基于对 `src/tools/AgentTool/`、`src/tasks/`、`src/tools/SendMessageTool/` 等核心模块的完整源码分析。
> 代码版本：Claude Code 2.1.888

---

## 目录

1. [系统概览](#1-系统概览)
2. [AgentDefinition 接口：Agent 的"配方"](#2-agentdefinition-接口agent-的配方)
3. [Agent 类型全谱](#3-agent-类型全谱)
4. [Agent 执行全流程](#4-agent-执行全流程)
5. [同步 Agent vs 异步 Agent](#5-同步-agent-vs-异步-agent)
6. [Fork Agent：提示缓存优化路径](#6-fork-agent提示缓存优化路径)
7. [后台任务运行机制](#7-后台任务运行机制)
8. [Team Agent 协作机制](#8-team-agent-协作机制)
9. [Agent 上下文隔离机制](#9-agent-上下文隔离机制)
10. [工具并发调度](#10-工具并发调度)
11. [权限系统](#11-权限系统)
12. [Worktree 隔离](#12-worktree-隔离)
13. [Hook 系统](#13-hook-系统)
14. [自定义 Agent（用户/项目/插件）](#14-自定义-agent用户项目插件)
15. [内置 Agent 详解](#15-内置-agent-详解)
16. [完整示例](#16-完整示例)
17. [关键设计决策总结](#17-关键设计决策总结)

---

## 1. 系统概览

Claude Code 的 Agent 系统本质上是一个**递归调用框架**：主循环和每个子 Agent 跑的是完全相同的 `query()` 函数（`src/query.ts`），区别仅在于传入的参数——尤其是 `ToolUseContext`、`systemPrompt`、`tools` 三件套。

```
调用栈层次：

用户输入
  ↓
REPL.tsx → QueryEngine.submitMessage()
  ↓
query()   ← 主循环
  ↓ 输出 tool_use: { name: "Agent", input: { prompt: "..." } }
AgentTool.call()
  ↓
runAgent()
  ↓
query()   ← 子 Agent（参数被隔离/裁剪）
  ↓ 可以再调用 AgentTool
  ...（最多嵌套 N 层，受工具白名单限制）
```

### 核心文件分布

| 文件 | 职责 |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | Agent 工具入口，决定 sync/async/fork/teammate 路径 |
| `src/tools/AgentTool/runAgent.ts` | Agent 初始化（提示/工具/MCP/hooks），包装 query() |
| `src/tools/AgentTool/forkSubagent.ts` | Fork 路径的消息构造与递归防护 |
| `src/tools/AgentTool/loadAgentsDir.ts` | Agent 定义接口、加载自定义 Agent |
| `src/tools/AgentTool/builtInAgents.ts` | 内置 Agent 列表与条件开关 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 后台任务状态机、通知注入 |
| `src/tasks/InProcessTeammateTask/` | Team Agent（进程内 teammate）任务 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | Agent 间消息传递 |
| `src/tools/TaskOutputTool/TaskOutputTool.tsx` | 读取后台 Agent 输出 |
| `src/tools/TaskStopTool/TaskStopTool.ts` | 终止后台 Agent |

---

## 2. AgentDefinition 接口：Agent 的"配方"

每个 Agent 都由一个 `AgentDefinition` 对象描述（`src/tools/AgentTool/loadAgentsDir.ts:106-133`）。

```typescript
type BaseAgentDefinition = {
  // === 必填字段 ===
  agentType: string          // Agent 的唯一标识符（如 "general-purpose", "Explore"）
  whenToUse: string          // 主 Agent 用来决定是否调用该子 Agent 的描述

  // === 工具配置（三选一，优先级从高到低）===
  tools?: string[]           // 白名单：只允许列出的工具（'*' = 全部）
  disallowedTools?: string[] // 黑名单：从全部工具中排除指定工具

  // === 模型配置 ===
  model?: string             // 'inherit' | 'haiku' | 'sonnet' | 完整 model ID
  effort?: EffortValue       // 'low' | 'medium' | 'high' | 'max' | 整数

  // === 权限和隔离 ===
  permissionMode?: PermissionMode  // 'plan' | 'acceptEdits' | 'auto' | 'bypassPermissions' | 'dontAsk' | 'bubble'
  isolation?: 'worktree' | 'remote' // 是否在独立 git 工作树中运行

  // === 生命周期控制 ===
  maxTurns?: number           // 最大 API 轮数，防止无限循环
  background?: boolean        // true = 总是后台运行（spawn 立即返回）
  initialPrompt?: string      // 预置到第一轮用户消息前（支持 /slash 命令）

  // === 扩展功能 ===
  skills?: string[]           // 预加载的 skill 名称列表
  mcpServers?: AgentMcpServerSpec[] // Agent 专属的额外 MCP 服务器
  hooks?: HooksSettings       // Agent 作用域的 hooks（SubagentStart/SubagentStop）
  memory?: 'user' | 'project' | 'local' // 持久记忆作用域
  requiredMcpServers?: string[] // 必须配置的 MCP 服务器（缺失则不可用）

  // === UI 配置 ===
  color?: AgentColorName      // UI 颜色标签
  criticalSystemReminder_EXPERIMENTAL?: string // 每轮注入的强提醒

  // === 系统内部字段 ===
  omitClaudeMd?: boolean      // true = 不加载 CLAUDE.md（只读 Agent 省 token 用）
  source: SettingSource       // 'built-in' | 'userSettings' | 'projectSettings' | 'plugin' | ...
  baseDir?: string
  filename?: string           // 原始文件名（自定义 Agent 用）
}
```

### 三种具体类型

```typescript
// 内置 Agent：提示词通过函数动态生成（可接收上下文）
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext: ... }) => string
}

// 自定义 Agent：提示词在加载时通过闭包捕获
type CustomAgentDefinition = BaseAgentDefinition & {
  source: 'userSettings' | 'projectSettings' | 'policySettings' | ...
  getSystemPrompt: () => string
}

// 插件 Agent
type PluginAgentDefinition = BaseAgentDefinition & {
  source: 'plugin'
  plugin: string
  getSystemPrompt: () => string
}
```

### 优先级覆盖规则

当多个来源定义了同名 Agent 时，后注册的覆盖前面的：

```
built-in < plugin < userSettings < projectSettings < flagSettings < policySettings
```

---

## 3. Agent 类型全谱

### 3.1 内置 Agent（6 个）

| agentType | 模型 | 工具策略 | 只读 | 后台 | 条件 |
|-----------|------|---------|------|------|------|
| `general-purpose` | 默认子代理模型 | `['*']` 全部 | ❌ | ❌ | 始终 |
| `Explore` | haiku | 排除写文件5件套 | ✅ | ❌ | GrowthBook 开关 |
| `Plan` | inherit | 排除写文件5件套 | ✅ | ❌ | GrowthBook 开关 |
| `statusline-setup` | sonnet | `[Read, Edit]` | ❌ | ❌ | 始终 |
| `claude-code-guide` | haiku | `[Glob,Grep,Read,WebFetch,WebSearch]` | ❌ | ❌ | 非 SDK 入口 |
| `verification` | inherit | 排除写文件5件套 | ✅ | ✅ | feature flag（当前关闭） |

> **注意**：本逆向版代码中 `feature()` 恒返回 `false`，因此 `verification` 和 `Explore`/`Plan`（依赖 `feature('BUILTIN_EXPLORE_PLAN_AGENTS')`）在标准构建中不可用。但由于 GrowthBook 的 `tengu_amber_stoat` 默认值为 `true`，Explore/Plan 在运行时实际可用。

### 3.2 自定义 Agent（用户定义）

通过 `.claude/agents/<name>.md` 文件定义，frontmatter 格式：

```markdown
---
name: code-reviewer
description: Reviews code for bugs and style issues
model: sonnet
tools: [Read, Bash, Grep, Glob]
permissionMode: dontAsk
maxTurns: 30
background: false
---

You are a thorough code reviewer. Focus on:
1. Logic errors and edge cases
2. Security vulnerabilities
3. Performance issues
...
```

### 3.3 Fork Agent（隐式特殊类型）

不显示在工具列表里，通过省略 `subagent_type` 参数触发（需 `feature('FORK_SUBAGENT')` 开启）：

```typescript
const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],
  maxTurns: 200,
  model: 'inherit',
  permissionMode: 'bubble',  // 权限提示冒泡到父终端
  source: 'built-in',
}
```

### 3.4 Teammate Agent（Team 模式）

`AgentTool` 在 `isAgentSwarmsEnabled()` 时支持 `team_name` 参数，创建进程内 teammate。

---

## 4. Agent 执行全流程

### 4.1 从 AgentTool.call() 到 query() 的完整调用链

```
AgentTool.call(input, toolUseContext, ...)    [AgentTool.tsx:239]
  │
  ├─ [1] 选择 Agent 定义
  │       ├─ isForkPath → FORK_AGENT（省略 subagent_type 时）
  │       └─ 按 agentType 查找 activeAgents 列表
  │
  ├─ [2] 确定执行模式
  │       shouldRunAsync = run_in_background
  │                      || agentDef.background
  │                      || forceAsync
  │
  ├─ [3] 构建消息和提示
  │       ├─ Fork：buildForkedMessages(prompt, assistantMessage)
  │       │          → 克隆父消息 + 统一占位符 + 指令
  │       └─ Normal：createUserMessage(prompt)
  │
  ├─ [4] Worktree 隔离（可选）
  │       isolation === 'worktree' → createAgentWorktree(slug)
  │
  ├─ [5] 装配工具池
  │       assembleToolPool(workerPermissionContext, mcpTools)
  │       → 按 tools/disallowedTools 过滤
  │
  ├─ [6] 分支执行
  │       ├─ 异步路径 → registerAsyncAgent() + void runAsyncAgentLifecycle()
  │       │              立即返回 {status: 'async_launched', agentId, outputFile}
  │       └─ 同步路径 → runWithAgentContext() + runAgent() 阻塞等待
  │
  └─ [7] 清理（finally）
          killShellTasksForAgent()
          mcpCleanup()
          removeAgentWorktree()（如果有）
```

### 4.2 runAgent() 初始化步骤

```typescript
// src/tools/AgentTool/runAgent.ts:248
async function* runAgent({
  agentDefinition, promptMessages, toolUseContext, ...
}) {
  // 1. 创建 agentId（UUID），注册 Perfetto 追踪
  // 2. 初始化 Agent 专属 MCP 服务器（frontmatter mcpServers 字段）
  // 3. 生成系统提示
  //    - Fork：使用父级渲染好的 renderedSystemPrompt（字节一致，缓存共享）
  //    - Normal：getAgentSystemPrompt(agentDefinition, toolUseContext, model, tools)
  // 4. 合并工具池：resolvedTools + agentMcpTools
  // 5. 创建隔离的 agentToolUseContext（见第9节）
  // 6. 注册 frontmatter hooks
  // 7. 执行 SubagentStart hooks
  // 8. 预加载 skills（initialPrompt + skills 字段）
  // 9. 记录初始消息到 sidechain JSONL

  // 10. ⚡ 调用 query()
  for await (const message of query({
    messages: initialMessages,
    systemPrompt: agentSystemPrompt,
    toolUseContext: agentToolUseContext,  // 隔离后的上下文
    querySource: `agent:${agentDefinition.agentType}`,
    maxTurns: agentDefinition.maxTurns,
    ...
  })) {
    await recordSidechainTranscript([message], agentId, lastRecordedUuid)
    yield message  // 传回给父 AgentTool
  }

  // finally（9 步清理序列，runAgent.ts 末尾）：
  // 1. removeFromAgentHierarchy(agentId)          注销 Agent 层级
  // 2. clearSessionHooks(rootSetAppState, agentId) 清理 frontmatter hooks
  // 3. runSubagentStopHooks(agentId, ...)          执行 SubagentStop hooks
  // 4. await mcpCleanup()                          关闭 Agent 专属 MCP 服务器
  // 5. killShellTasksForAgent(agentId)             终止子 shell 任务
  // 6. removeAgentId(agentId)                      注销 agentId
  // 7. await removeAgentWorktree(worktreeInfo)     删除 git worktree（如有）
  // 8. stopPerfettoTrace(agentId)                  停止性能追踪
  // 9. updateAgentProgress(agentId, null)          清除进度状态
}
```

---

## 5. 同步 Agent vs 异步 Agent

### 5.1 判定逻辑

```typescript
const shouldRunAsync =
  (run_in_background === true       // 调用方显式要求后台
    || selectedAgent.background === true  // Agent 定义要求后台（如 verification）
    || forceAsync                   // 内部强制（coordinator 模式等）
    || assistantForceAsync)         // 模型层强制
  && !isBackgroundTasksDisabled
```

### 5.2 同步 Agent 执行流程

```
AgentTool.call() → runAgent() → query()
         ↑ 阻塞等待 ↓
         ←←←← 完成，返回结果 ←←←←

父 query() 拿到 tool_result，继续与模型对话
```

同步 Agent 有一个 **"升级为后台"机制**（`autoBackgroundMs = 120000ms`）：

```typescript
// AgentTool.tsx:~800
const raceResult = await Promise.race([
  agentIterator.next(),                  // Agent 正常运行
  registration.backgroundSignal          // 用户触发"后台化"信号
])

if (raceResult.type === 'background') {
  wasBackgrounded = true
  void runWithAgentContext(...)  // 后台继续执行
  return { status: 'async_launched', agentId, ... }  // 立即返回父循环
}
```

### 5.3 异步 Agent 执行流程

```
AgentTool.call()
  │
  ├─ registerAsyncAgent() → AppState.tasks[agentId] = { status: 'running', ... }
  ├─ void runAsyncAgentLifecycle(...)  ← 不 await，后台执行
  └─ 立即返回: { status: 'async_launched', agentId, outputFile }
          ↓
父 query() 拿到"回执"，继续与模型对话（模型通常说"已启动，等待通知"）

              [后台]
              runAsyncAgentLifecycle()
                ↓
              runAgent() → query() 运行
                ↓
              完成: completeAgentTask()
                ↓
              enqueueAgentNotification() → 注入 <task-notification>
                ↓
              主循环接收通知 → 触发新一轮请求 → 模型处理结果
```

### 5.4 关键差异对比

| 维度 | 同步 Agent | 异步 Agent |
|------|-----------|-----------|
| 返回时机 | Agent 完成后 | 立即返回"回执" |
| tool_result 内容 | Agent 的真实输出 | `{status:'async_launched', agentId, outputFile}` |
| 结果传递 | 直接作为 tool_result | 通过 `<task-notification>` 事件 |
| 权限弹框 | 可显示 | 自动拒绝（`shouldAvoidPermissionPrompts: true`） |
| setAppState | 共享（同步更新主 UI） | no-op（隔离） |
| 升级 | 可升级为后台（race 机制） | 始终后台 |
| 中断 | 父 abortController 关联 | 独立 AbortController |

---

## 6. Fork Agent：提示缓存优化路径

### 6.1 设计动机

当主 Agent 要并行派生多个子 Agent 处理不同子任务时，每个子 Agent 如果使用独立的系统提示，就无法共享 prompt cache，成本会线性增加。Fork Agent 的设计目标是让所有子 Agent **共享相同的 API 请求前缀**，只有最后的指令文本不同，从而最大化缓存命中率。

### 6.2 消息构造（`forkSubagent.ts:107`）

```typescript
function buildForkedMessages(directive: string, assistantMessage: AssistantMessage) {
  // 1. 克隆完整的父 Agent 最后一条消息（保留所有 tool_use 块）
  const fullAssistantMessage = { ...assistantMessage, uuid: randomUUID() }

  // 2. 为每个 tool_use 生成统一的占位符 tool_result
  //    ⚡关键：所有子 Agent 的占位符文本必须完全相同！
  const FORK_PLACEHOLDER = 'Fork started — processing in background'
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER }]
  }))

  // 3. 构造用户消息：统一占位符 + 各子 Agent 独有的指令
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,        // ← 所有子 Agent 相同
      { type: 'text', text: buildChildMessage(directive) }  // ← 每个子 Agent 不同
    ]
  })

  return [fullAssistantMessage, toolResultMessage]
  //       ↑─────────────────────────────────────────
  //       只有末尾 directive 文本不同，其余字节完全一致
  //       → 所有 fork 子 Agent 共享同一个 prompt cache 前缀
}
```

### 6.3 Fork 子 Agent 收到的指令格式

```
<fork-boilerplate>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. USE your tools directly: Bash, Read, Write, etc.
4. If you modify files, commit before reporting. Include commit hash.
5. Keep report under 500 words.
6. Your response MUST begin with "Scope:".
...

Output format:
  Scope: <assigned scope>
  Result: <findings>
  Key files: <file paths>
  Files changed: <list with commit hash>
  Issues: <if any>
</fork-boilerplate>

<fork-directive>具体任务指令</fork-directive>
```

### 6.4 递归防护

Fork Agent 检测到自身在 fork 子 Agent 中时，立即抛错：

```typescript
// forkSubagent.ts:78
function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m =>
    m.type === 'user' &&
    m.message.content.some(
      block => block.type === 'text' && block.text.includes('<fork-boilerplate>')
    )
  )
}

// AgentTool.tsx（调用时）
if (isInForkChild(toolUseContext.messages)) {
  throw new Error('Fork is not available inside a forked worker. Complete your task directly.')
}
```

---

## 7. 后台任务运行机制

### 7.1 任务状态机

后台 Agent 在 `AppState.tasks` 中维护状态（类型 `LocalAgentTaskState`）：

```
  pending
     │
     ↓
  running ──────────────────┬──────────────┬
     │                      │              │
     ↓ (正常完成)          ↓ (出错)     ↓ (被终止)
  completed              failed          killed
```

`LocalAgentTaskState` 的关键字段：

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string                    // 任务描述
  agentType: string                 // Agent 类型名
  model?: string
  abortController?: AbortController // 用于终止
  result?: AgentToolResult          // 完成后的结果
  progress?: AgentProgress          // 实时进度
  messages?: Message[]              // 可见消息列表（UI 展示用）
  pendingMessages: string[]         // SendMessage 队列（待注入）
  isBackgrounded: boolean           // false=前台运行, true=已后台化
  retain: boolean                   // UI 正在查看（阻止 GC）
  diskLoaded: boolean               // sidechain 已从磁盘加载
  evictAfter?: number               // 到期后从内存回收的时间戳
  retrieved: boolean
  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

### 7.2 任务注册（registerAsyncAgent）

```typescript
// LocalAgentTask.tsx:467
function registerAsyncAgent({
  agentId, toolUseId, prompt, agentType, model,
  parentCwd, worktreePath, abortController, ...
}) {
  // 1. 初始化输出目录
  //    ~/.claude/projects/<hash>/tasks/<agentId>/
  const outputDir = getTaskOutputDir(agentId)
  fs.mkdirSync(outputDir, { recursive: true })

  // 2. 创建 output.txt → sidechain JSONL 的符号链接
  //    使用 O_NOFOLLOW 标志防止符号链接攻击（安全防护）
  initTaskOutputAsSymlink(agentId, sidechainPath)
  //    → open(outputPath, O_WRONLY | O_CREAT | O_NOFOLLOW)

  // 3. 注册到 AppState.tasks
  setAppState(prev => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [agentId]: {
        type: 'local_agent',
        status: 'running',
        agentId,
        toolUseId,
        prompt,
        agentType,
        model,
        abortController,
        pendingMessages: [],
        isBackgrounded: true,
        retain: false,
        diskLoaded: false,
        retrieved: false,
        lastReportedToolCount: 0,
        lastReportedTokenCount: 0,
      }
    }
  }))
}
```

### 7.3 后台执行状态机（runAsyncAgentLifecycle）

`runAsyncAgentLifecycle()` 在 `agentToolUtils.ts:509-687` 中实现，是后台 Agent 的完整生命周期管理器：

```typescript
// agentToolUtils.ts:509
async function runAsyncAgentLifecycle({ agentId, agentDefinition, ... }) {
  try {
    // 1. 调用 runAgent()（内部调用 query()）
    const result = await collectResult(runAgent({ agentDefinition, ... }))

    // 2. 正常完成
    completeAgentTask(agentId, result, setAppState)
    //   → 更新状态为 'completed'
    //   → 设置 task.result = result
    //   → 调用 enqueueAgentNotification(status: 'completed', ...)

  } catch (error) {
    if (isAbortError(error)) {
      // 3a. 被终止（TaskStopTool 调用）
      killAgentTask(agentId, setAppState)
      //   → 更新状态为 'killed'
      //   → enqueueAgentNotification(status: 'killed', ...)
    } else {
      // 3b. 运行时错误
      failAgentTask(agentId, error, setAppState)
      //   → 更新状态为 'failed'
      //   → enqueueAgentNotification(status: 'failed', errorMessage, ...)
    }
  } finally {
    // 4. 自动清理（evictAfter 延迟回收）
    scheduleTaskEviction(agentId, TASK_EVICTION_DELAY_MS)
  }
}
```

### 7.4 输出持久化

后台 Agent 的输出不在内存中传递，而是写入磁盘：

```typescript
// 输出文件路径
const outputFile = getTaskOutputPath(agentId)
// → ~/.claude/projects/<hash>/tasks/<agentId>/output.txt

// AgentTool 返回 async_launched 时附带这个路径
// 模型可以用 Read 工具直接读取进度
return {
  data: {
    status: 'async_launched',
    agentId,
    outputFile,             // ← 告知父 Agent 可读取
    canReadOutputFile       // ← 父 Agent 是否有 Read/Bash 工具
  }
}
```

输出文件是一个**指向 sidechain JSONL 转录文件的符号链接**，实时追加写入。

**安全保护细节**：
- 创建符号链接时使用 `O_NOFOLLOW` 标志，防止恶意进程替换路径欺骗读写
- 单个任务输出文件读取上限：**8MB**（防止 Agent 输出过大占满内存）
- 所有任务输出合计磁盘占用上限：**5GB**（超出后最旧任务被自动清理）

### 7.5 task-notification 注入机制

这是异步 Agent 结果传回主循环的核心通道（`LocalAgentTask.tsx:198`）：

```typescript
function enqueueAgentNotification({ taskId, status, finalMessage, ... }) {
  // 防重复：原子性地设置 notified 标志
  let shouldEnqueue = false
  updateTaskState(taskId, setAppState, task => {
    if (task.notified) return task   // 已通知过，跳过
    shouldEnqueue = true
    return { ...task, notified: true }
  })
  if (!shouldEnqueue) return

  // 构造通知消息（XML 格式）
  const message = `
<task-notification>
<task-id>${taskId}</task-id>
<tool-use-id>${toolUseId}</tool-use-id>       ← 对应原始 AgentTool tool_use 的 ID
<output-file>${outputPath}</output-file>
<status>${status}</status>                    ← 'completed' | 'failed' | 'killed'
<summary>Agent "xxx" completed</summary>
<result>${finalMessage}</result>              ← Agent 最后的文本输出
<usage>
  <total_tokens>12345</total_tokens>
  <tool_uses>23</tool_uses>
  <duration_ms>45000</duration_ms>
</usage>
<worktree>...</worktree>                      ← 如果有 worktree 隔离
</task-notification>`

  // 注入到主循环的消息队列
  // priority: 'later' 确保任务通知排在队列末尾，不会抢占用户正在输入的消息
  enqueuePendingNotification({ value: message, mode: 'task-notification', priority: 'later' })
  // → 触发主 REPL 发起新一轮 API 请求，将通知作为用户消息送给模型
}
```

### 7.6 TaskOutputTool：读取后台 Agent 输出

```typescript
// TaskOutputTool.tsx:208
async call({ task_id, block = true, timeout = 30000 }) {
  if (!block) {
    // 非阻塞：立即返回当前状态
    return { retrieval_status: 'not_ready' | 'success', task: ... }
  }

  // 阻塞：轮询等待完成（每 100ms 检查一次）
  const completedTask = await waitForTaskCompletion(task_id, getAppState, timeout)

  if (task.type === 'local_agent') {
    // 优先从内存返回清洁结果（最终文本），而非磁盘上的完整 JSONL 转录
    const cleanResult = task.result
      ? extractTextContent(task.result.content, '\n')
      : undefined
    return { result: cleanResult || diskOutput }
  }
}
```

> **注意**：代码注释中明确标注 `TaskOutputTool` 为 **deprecated**，推荐直接用 `Read` 工具读取 `outputFile` 路径。

### 7.7 TaskStopTool：终止后台 Agent

```typescript
// TaskStopTool.ts:107
async call({ task_id }) {
  const result = await stopTask(task_id, { getAppState, setAppState })
  // stopTask → killAsyncAgent → task.abortController.abort()
  //          → 更新状态为 'killed'
  //          → evictTaskOutput(taskId)（磁盘输出清理）
}
```

终止链路：`TaskStopTool → stopTask() → killAsyncAgent() → abortController.abort() → 子 query() 通过 AbortError 退出`

---

## 8. Team Agent 协作机制

Team Agent（`isAgentSwarmsEnabled()` 开启时可用）支持多个 Agent 在同一会话中协作，通过 **Mailbox（邮箱）** 机制传递消息。

**启用方式**（满足其一即可）：
```bash
# 方式1：环境变量
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# 方式2：CLI 标志
claude --agent-teams
```

### 8.1 架构概览

```
team-lead（主 Agent）
    │
    ├─ SendMessage({ to: "researcher", message: "..." })
    │       ↓ 写入 mailbox 文件
    │
    ├─ researcher（teammate Agent）
    │       ↓ 从 mailbox 读取
    │       ↓ 处理任务
    │       └─ SendMessage({ to: "team-lead", message: "结果" })
    │
    └─ reviewer（teammate Agent）
            ↓ 独立运行
```

### 8.2 Teammate 创建流程

```typescript
// AgentTool.tsx 中 teammate 路径
const result = await spawnTeammate({
  name: "researcher",          // 唯一成员名
  prompt,                      // 初始任务
  team_name: "my-team",       // 团队名（全局唯一）
  use_splitpane: true,         // UI 分屏显示
  plan_mode_required: true,    // 要求 team-lead 审批计划
  model,
  agent_type: "research-agent",
})
// 返回：
// {
//   status: 'teammate_spawned',
//   teammate_id: "uuid@team-name",
//   agent_id: "uuid",
//   name: "researcher",
//   tmux_session_name: "...",
//   tmux_window_name: "researcher"
// }
```

### 8.3 agentNameRegistry：名称路由

```typescript
// AgentTool.tsx：创建 Agent 时注册名称
if (name) {
  rootSetAppState(prev => {
    const next = new Map(prev.agentNameRegistry)
    next.set(name, asAgentId(asyncAgentId))   // "researcher" → "uuid-xxx"
    return { ...prev, agentNameRegistry: next }
  })
}

// SendMessageTool.ts：通过名称路由
const registered = appState.agentNameRegistry.get(input.to)  // "researcher" → agentId
const task = appState.tasks[agentId]                         // 查找任务状态
```

### 8.4 SendMessageTool：消息传递

`SendMessageTool` 支持多种消息类型：

```typescript
// 1. 普通文本消息（写入 mailbox 文件）
SendMessage({ to: "researcher", message: "请分析 src/ 目录", summary: "分析任务" })

// 2. 广播给所有 teammates
SendMessage({ to: "*", message: "请暂停当前任务", summary: "暂停通知" })

// 3. 结构化消息：关闭请求
SendMessage({
  to: "researcher",
  message: { type: "shutdown_request", reason: "任务完成" }
})

// 4. 结构化消息：关闭响应（teammate → team-lead）
SendMessage({
  to: "team-lead",
  message: { type: "shutdown_response", request_id: "xxx", approve: true }
})

// 5. 计划审批（team-lead → teammate）
SendMessage({
  to: "researcher",
  message: { type: "plan_approval_response", request_id: "xxx", approve: true }
})
```

### 8.5 Mailbox 机制

```typescript
// teammateMailbox.ts
async function writeToMailbox(
  recipientName: string,
  message: { from, text, summary, timestamp, color },
  teamName?: string
): Promise<void> {
  // 写入 ~/.claude/teams/<teamName>/inboxes/<recipientName>.json
  // 使用 proper-lockfile 库加锁，最多重试 10 次
}
```

- **文件路径**：`~/.claude/teams/<teamName>/inboxes/<agentName>.json`
- **并发写入**：使用 `proper-lockfile` 库（文件级锁），**最多重试 10 次**，防止并发写入冲突
- **读取方式**：
  - 进程内 Teammate：`useInboxPoller.ts` 每隔 **1 秒**轮询一次 inbox 文件
  - 进程外（file-based）：同样 1 秒轮询周期
- **并发安全**：文件锁 + 重试机制，保证消息不丢失

**进程内 Teammate 的 `AsyncLocalStorage` 隔离**：

每个 in-process Teammate 在独立的 `AsyncLocalStorage` 上下文中运行，与主 Agent 共享进程内存但互不干扰：

```typescript
// InProcessTeammateTask 的隔离机制
const storage = new AsyncLocalStorage()
storage.run({ agentId, teamName, ... }, () => {
  runAgent({ ... })  // Teammate 在独立的 async 上下文中执行
})
```

这意味着 Teammate 对 `AppState` 的读写通过 `AsyncLocalStorage` 隔离，不会意外修改主 Agent 的状态。

### 8.6 进程内 Teammate vs 后台 Agent

| 特性 | 后台 Agent（LocalAgentTask） | 进程内 Teammate（InProcessTeammateTask） |
|------|----------------------------|----------------------------------------|
| 运行方式 | 独立 AsyncLocalStorage 上下文 | 同进程，共享内存 |
| 消息传递 | task-notification 事件 | Mailbox 文件（inboxes/）+ 1s 轮询 |
| 身份 | agentId（UUID） | agentName@teamName |
| 计划审批 | 不支持 | 支持（plan_mode_required） |
| 关闭协议 | TaskStopTool | shutdown_request/response 握手 |
| UI 显示 | 后台任务栏 | 分屏（splitpane）或 tmux 窗口 |

### 8.7 SendMessage 路由逻辑

```typescript
// SendMessageTool.ts:800
async call(input, context) {
  // 路由优先级：
  // 1. bridge:// → Remote Control（跨 session）
  // 2. uds://   → Unix Domain Socket（本地跨进程）
  // 3. agentNameRegistry[name] → 进程内子 Agent（按名称）
  //    ├─ 运行中 → queuePendingMessage()（下轮注入）
  //    └─ 已停止 → resumeAgentBackground()（重新激活）
  // 4. * → 广播给所有 teammates（via mailbox）
  // 5. 其他 → writeToMailbox（经典 team 模式）
}
```

---

## 9. Agent 上下文隔离机制

每次启动子 Agent，都通过 `createSubagentContext()` 创建隔离的 `ToolUseContext`：

### 9.1 三类操作

```typescript
// forkedAgent.ts:345（简化）
function createSubagentContext(parentContext, overrides) {
  // === 克隆（物理隔离）===
  readFileState = cloneFileStateCache(parent)     // 文件缓存独立
  abortController = createChildAbortController()  // 可单独取消

  // === 包装（注入约束）===
  getAppState = () => ({
    ...parentState,
    shouldAvoidPermissionPrompts: true  // 后台不弹权限框
  })

  // === 置空（no-op，防止子 Agent 改主 UI）===
  setAppState = () => {}          // 除非 shareSetAppState: true（同步 Agent）
  setToolJSX = undefined          // 不渲染工具 UI
  addNotification = undefined     // 不添加通知
  setStreamMode = undefined       // 不改流模式

  // === 新建（独立标识）===
  agentId = newUUID()
  queryTracking = {
    chainId: randomUUID(),
    depth: (parent.queryTracking?.depth ?? -1) + 1  // 深度追踪
  }
  nestedMemoryAttachmentTriggers = new Set()

  return { ...parentContext, ...overrides, 以上所有字段 }
}
```

### 9.2 同步 vs 异步的 context 差异

| 字段 | 同步 Agent | 异步 Agent |
|------|-----------|-----------|
| `setAppState` | 共享父级（UI 实时更新） | no-op（隔离） |
| `abortController` | 链接父级（父中止→子中止） | 独立（互不影响） |
| `shouldAvoidPermissionPrompts` | `false`（可弹框） | `true`（自动拒绝） |
| `querySource` | 继承 | `'agent:agentType'` |
| `agentId` | 新 UUID | 新 UUID |
| `depth` | `parent.depth + 1` | `parent.depth + 1` |

### 9.3 深度追踪与递归防护

```typescript
queryTracking = {
  chainId: randomUUID(),    // 调用链标识（用于日志和分析）
  depth: parent.depth + 1   // 嵌套深度计数
}
```

结合三层递归防护：

1. **工具白名单过滤**：异步 Agent 的工具集中移除 `Agent` 工具
2. **消息扫描**：`isInForkChild()` 检测 fork 标签，拒绝 fork 中套 fork
3. **querySource 检查**：`querySource.includes('fork')` 时拒绝再次 fork

---

## 10. 工具并发调度

### 10.1 并发判定

```typescript
// 工具是否并发安全，由工具自身声明
tool.isConcurrencySafe(input): boolean

// 并发安全的工具（可并发）：
Read, Grep, Glob, WebSearch, WebFetch  // 只读，无副作用

// 非并发安全（串行执行）：
Bash, Edit, Write                      // 可能有副作用
Agent                                  // 可能修改 AppState
```

### 10.2 并发分组（partitionToolCalls）

在实际并发执行前，`partitionToolCalls()` 先将本轮所有工具调用按 `isConcurrencySafe` 分组：

```typescript
// partitionToolCalls(toolUseBlocks)
// → 返回分组列表：[ [safe1, safe2, safe3], [unsafe1], [safe4, safe5], ... ]
//   同组内并发，跨组串行

// 例：本轮工具调用 [Read, Read, Edit, Read, Bash]
// 分组结果：
//   组1: [Read, Read]   ← 并发执行
//   组2: [Edit]         ← 串行（含写操作）
//   组3: [Read]         ← 并发（单个）
//   组4: [Bash]         ← 串行
```

### 10.3 并发执行框架

```typescript
// toolOrchestration.ts（简化）
async function* runTools(toolUseBlocks, ...) {
  // 按 isConcurrencySafe 分批（partitionToolCalls）
  // 同一批内并发执行（最多 getMaxToolUseConcurrency() 个）
  yield* all(
    toolUseBlocks.map(async function* (toolUse) {
      context.setInProgressToolUseIDs(prev => new Set(prev).add(toolUse.id))
      yield* runToolUse(toolUse, ...)
      context.setInProgressToolUseIDs(prev => {
        const next = new Set(prev)
        next.delete(toolUse.id)
        return next
      })
    }),
    MAX_CONCURRENCY
  )
}
```

**最大并发数控制**：

```bash
# 默认值：10
# 通过环境变量覆盖：
export CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=5
```
```

### 10.4 并发示例

当模型同时发出 3 个工具调用：
- `Read(file1)` + `Read(file2)` + `Grep(pattern)` → **并发执行**（都是只读）
- `Edit(file1)` + `Bash("rm -rf")` → **串行执行**（有副作用）
- `Agent(task1)` + `Agent(task2)` → **取决于 isConcurrencySafe 的实现**

---

## 11. 权限系统

### 11.1 权限检查流程

```
tool_use block
    ↓
validateInput(args, context)         → 参数验证
    ↓
checkPermissions(args, context)
    ├─ hasPermissionsToUseTool()
    │    ├─ alwaysAllowRules（CLI --allowedTools、session 规则）
    │    ├─ alwaysDenyRules
    │    └─ alwaysAskRules
    │
    ├─ 返回 { behavior: 'allow' }    → 直接执行
    ├─ 返回 { behavior: 'deny' }     → 拒绝并记录
    └─ 返回 { behavior: 'ask' }
              ↓
         shouldAvoidPermissionPrompts?
              ├─ true  → 自动拒绝（后台 Agent）
              └─ false → 显示 UI 对话框（同步 Agent / 主循环）
```

### 11.2 permissionMode 含义

| 模式 | 含义 |
|------|------|
| `plan` | 只读模式，不允许任何写操作 |
| `acceptEdits` | 自动接受文件编辑（不弹文件写权限框） |
| `auto` | 自动模式，使用分类器判断 |
| `bypassPermissions` | 跳过所有权限检查 |
| `dontAsk` | 不询问，按默认规则处理 |
| `bubble` | 将权限提示"冒泡"到父 Agent 终端 |

### 11.3 Agent 权限继承规则

子 Agent 的权限规则不是简单继承，而是有选择地覆盖：

```typescript
// runAgent.ts:465
if (allowedTools !== undefined) {
  toolPermissionContext = {
    ...toolPermissionContext,
    alwaysAllowRules: {
      cliArg: state.toolPermissionContext.alwaysAllowRules.cliArg,  // CLI 规则保留
      session: [...allowedTools],   // ← 父级 session 规则被替换为 Agent 白名单
    },
  }
}
```

---

## 12. Worktree 隔离

### 12.1 什么是 Worktree 隔离

当 Agent 定义了 `isolation: 'worktree'` 时，Agent 会在一个独立的 git 工作树中运行，确保其文件修改不影响父 Agent 的工作目录。

```
主 Agent 工作目录: /project/
  ↓
子 Agent 工作树:  /project/.claude/worktrees/<slug>/
                  └─ 同一个 git 仓库，不同的工作副本
                  └─ 子 Agent 的修改存在这里
                  └─ Agent 完成后可以合并/删除
```

### 12.2 Worktree 生命周期

```typescript
// AgentTool.tsx
if (effectiveIsolation === 'worktree') {
  worktreeInfo = await createAgentWorktree(`agent-${earlyAgentId.slice(0, 8)}`)
  // 工作树路径：<repo>/.claude/worktrees/agent-a<7hex>/
  // → git worktree add .claude/worktrees/agent-abc1234 -b worktree-agent-abc1234
}

// runAgent.ts - 完成后告知子 Agent 路径信息
const worktreeNotice = buildWorktreeNotice(parentCwd, worktreeCwd)
// → "You've inherited context from parent at {parentCwd}. 
//    You are operating in an isolated worktree at {worktreeCwd}..."

// finally: 清理
await removeAgentWorktree(worktreePath)
// → git worktree remove --force .claude/worktrees/agent-abc1234
```

**分支名称扁平化**（防止 git 引用路径冲突）：

```typescript
// 如果原始名称包含 '/'，git 会把它解释为目录层级
// user/feature → worktree-user+feature （'/' 替换为 '+'）
const safeBranchName = `worktree-${slug.replace(/\//g, '+')}`
```

**陈旧工作树自动清理**（30 天后）：

系统启动时扫描 `.claude/worktrees/` 目录，清理符合以下命名模式且超过 30 天的工作树：
- `agent-a<7hex>`：Agent 自动创建的工作树
- `wf_<runId>-<idx>`：Workflow 创建的工作树

### 12.3 worktree 完成后的结果

如果子 Agent 在 worktree 中提交了代码，`AgentToolResult` 会包含：

```typescript
{
  status: 'completed',
  worktreePath: '/project/.claude/worktrees/agent-abc1234',
  worktreeBranch: 'worktree-agent-abc1234',
  content: [{ type: 'text', text: '已完成，代码在 worktree-agent-abc1234 分支' }]
}
```

父 Agent 可以据此决定是否合并该分支。

---

## 13. Hook 系统

### 13.1 Agent 级别的 Hook

与会话级 Hook（SessionStart/Stop）不同，Agent 自己可以在 frontmatter 中定义 Hook，只在该 Agent 运行期间有效：

```markdown
---
name: my-agent
hooks:
  SubagentStart:
    - matcher: ""
      hooks:
        - type: command
          command: "echo 'Agent started' >> /tmp/agent.log"
  SubagentStop:
    - matcher: ""
      hooks:
        - type: command
          command: "echo 'Agent stopped' >> /tmp/agent.log"
---
```

### 13.2 Hook 注册和清理

```typescript
// runAgent.ts
// 注册（在 Agent 启动时）
registerFrontmatterHooks(
  rootSetAppState,
  agentId,
  agentDefinition.hooks,
  `agent '${agentDefinition.agentType}'`,
  true  // isAgent = true → Stop 事件映射到 SubagentStop
)

// 清理（在 Agent 完成时，finally 块中）
clearSessionHooks(rootSetAppState, agentId)
```

### 13.3 Hook 类型映射

| 会话 Hook | Agent 对应 Hook |
|-----------|----------------|
| `SessionStart` | `SubagentStart` |
| `SessionStop` | `SubagentStop` |
| `PreToolUse` | `PreToolUse`（在 Agent 范围内） |
| `PostToolUse` | `PostToolUse`（在 Agent 范围内） |

---

## 14. 自定义 Agent（用户/项目/插件）

### 14.1 文件位置优先级

```
~/.claude/agents/<name>.md          ← userSettings（用户全局）
.claude/agents/<name>.md            ← projectSettings（项目级，可覆盖同名用户 Agent）
<plugin-dir>/agents/<name>.md       ← plugin
```

### 14.2 Agent 文件格式（完整字段）

```markdown
---
name: code-reviewer                  # 必填：唯一标识符
description: |                       # 必填：whenToUse 描述，主 Agent 靠此判断何时调用
  Reviews code changes for correctness,
  security issues, and style.
model: sonnet                        # 可选：inherit | haiku | sonnet | 完整 model ID
tools: [Read, Bash, Grep, Glob]      # 可选：工具白名单（与 disallowedTools 互斥）
disallowedTools: [Write, Edit]       # 可选：工具黑名单
permissionMode: dontAsk              # 可选：权限模式
maxTurns: 50                         # 可选：最大轮数
background: false                    # 可选：true=总是后台运行
isolation: worktree                  # 可选：运行在独立 git 工作树
effort: high                         # 可选：low|medium|high|max 或整数
skills: [security-review, lint]      # 可选：预加载的 skill 名称
initialPrompt: "Start by reading CLAUDE.md"  # 可选：首轮前置消息
memory: project                      # 可选：user|project|local 持久记忆
mcpServers:                          # 可选：Agent 专属 MCP 服务器
  - github                           #   引用已配置的服务器名
  - my-custom-server: ...            #   或内联定义
hooks:                               # 可选：Agent 作用域 hooks
  SubagentStart:
    - hooks:
        - type: command
          command: "..."
color: blue                          # 可选：UI 颜色
---

你是一个代码审查专家...（系统提示词正文）
```

### 14.3 加载机制

```typescript
// loadAgentsDir.ts:296
const getAgentDefinitionsWithOverrides = memoize(async (cwd: string) => {
  // 1. 扫描 .claude/agents/ 和 ~/.claude/agents/ 目录
  const markdownFiles = await loadMarkdownFilesForSubdir('agents', cwd)

  // 2. 解析每个 .md 文件（frontmatter + content）
  const customAgents = markdownFiles.map(parseAgentFromMarkdown).filter(Boolean)

  // 3. 加载插件 Agent
  const pluginAgents = await loadPluginAgents()

  // 4. 合并内置 + 插件 + 自定义
  const allAgents = [...builtInAgents, ...pluginAgents, ...customAgents]

  // 5. 去重（按 agentType，后来者覆盖）
  const activeAgents = getActiveAgentsFromList(allAgents)

  // 6. 初始化颜色
  for (const agent of activeAgents) {
    if (agent.color) setAgentColor(agent.agentType, agent.color)
  }

  return { activeAgents, allAgents }
})
```

### 14.4 Memory（持久记忆）

如果 Agent 定义了 `memory` 字段，系统会在工具列表中自动注入 `Write`/`Edit`/`Read` 工具（若未包含），并在系统提示词末尾追加记忆加载指令：

```typescript
getSystemPrompt: () => {
  if (isAutoMemoryEnabled() && parsed.memory) {
    return systemPrompt + '\n\n' + loadAgentMemoryPrompt(name, parsed.memory)
  }
  return systemPrompt
}
```

---

## 15. 内置 Agent 详解

### 15.1 general-purpose

- **触发时机**：需要研究复杂问题、搜索代码、执行多步任务，搜索结果不确定时
- **模型**：默认子代理模型
- **工具**：`['*']`（全部）
- **关键提示词要素**：
  - 完成任务后返回简洁报告（调用方转述给用户）
  - 优先编辑已有文件，禁止随意创建新文件和 README

### 15.2 Explore

- **触发时机**：快速代码库探索——找文件模式、搜关键字、回答代码库问题
- **模型**：`haiku`（速度优先）
- **工具**：全部，排除 `[Agent, ExitPlanMode, Edit, Write, NotebookEdit]`
- **特殊设置**：`omitClaudeMd: true`（不加载 CLAUDE.md 省 token）
- **关键提示词要素**：
  - 严格只读模式（大段红色警告）
  - 要求并行工具调用（快速返回）
  - 调用时须指定彻底程度：`quick` / `medium` / `very thorough`

### 15.3 Plan

- **触发时机**：需要设计实现方案、识别关键文件、权衡架构取舍
- **模型**：`inherit`（与主 Agent 相同，确保理解质量）
- **工具**：同 Explore（只读）
- **特殊设置**：`omitClaudeMd: true`
- **关键提示词要素**：
  - 严格只读模式
  - 输出必须以"Critical Files for Implementation"结尾

### 15.4 statusline-setup

- **触发时机**：用户要求配置状态栏
- **模型**：`sonnet`
- **工具**：`[Read, Edit]`（白名单最小化）
- **color**：orange
- **关键提示词要素**：
  - PS1 → statusLine 转换规则（完整转义序列对照表）
  - statusLine JSON 输入格式说明（含 context_window、rate_limits 等）
  - 完成后必须告知用户可继续修改

### 15.5 claude-code-guide

- **触发时机**：用户询问 Claude Code/SDK/API 的使用方法
- **模型**：`haiku`
- **工具**：`[Glob, Grep, Read, WebFetch, WebSearch]`
- **permissionMode**：`dontAsk`
- **特殊**：系统提示动态拼接（注入用户当前的 skills、agents、MCP 服务器、settings.json）
- **调用前检查**：优先续接已有的 claude-code-guide Agent（via SendMessage），避免重复创建

### 15.6 verification（当前关闭）

- **触发时机**：非平凡任务完成后（3+ 文件编辑、后端/API/基础设施变更）
- **模型**：`inherit`
- **工具**：全部，排除写文件5件套（允许写 `/tmp`）
- **background**：`true`（总是后台运行）
- **color**：red
- **关键提示词要素**：
  - "你的职责不是确认能运行，而是努力让它崩溃"
  - 严格输出格式：每个 Check 必须包含 Command run + Output observed
  - 必须以 `VERDICT: PASS/FAIL/PARTIAL` 结尾

---

## 16. 完整示例

### 示例 1：主 Agent 派发多个并发子 Agent

```
用户输入：帮我分析这个项目的架构，同时检查安全问题

主 Agent（claude-sonnet）
  │
  ├─ [turn 1] 决策：派发两个子 Agent 并行处理
  │
  ├─ tool_use: Agent({
  │     subagent_type: "Explore",
  │     prompt: "分析项目整体架构，重点看 src/ 目录结构和主要模块关系",
  │     description: "架构分析"
  │   })
  │   → 同步阻塞，等待 Explore Agent 完成
  │   ← tool_result: "项目采用 MVC 架构，包含以下模块..."
  │
  └─ tool_use: Agent({
       subagent_type: "general-purpose",
       run_in_background: true,
       prompt: "审查代码中的安全漏洞，重点检查 SQL 注入、XSS、CSRF",
       description: "安全审查"
     })
     → 立即返回: { status: 'async_launched', agentId: 'uuid-xxx', outputFile: '...' }

[后台运行 general-purpose Agent：扫描代码，记录发现]

[15秒后]
task-notification 注入主循环：
<task-notification>
  <task-id>uuid-xxx</task-id>
  <status>completed</status>
  <result>发现 3 个安全问题：1. src/api.js:45 存在 SQL 注入风险...</result>
</task-notification>

主 Agent 收到通知，综合两个子 Agent 的结果，回复用户
```

### 示例 2：Fork Agent 并行处理多个文件

```
用户输入：重构这 5 个模块，每个加上错误处理

主 Agent
  │
  ├─ tool_use: Agent({
  │     // 省略 subagent_type → Fork 路径
  │     prompt: "重构 src/module1.ts，加入完整的错误处理",
  │   })
  │   buildForkedMessages("重构 module1..."):
  │     [完整父消息, user([PLACEHOLDER_RESULTS, "重构 module1 指令"])]
  │
  ├─ tool_use: Agent({
  │     prompt: "重构 src/module2.ts，加入完整的错误处理",
  │   })
  │   buildForkedMessages("重构 module2..."):
  │     [完整父消息, user([PLACEHOLDER_RESULTS, "重构 module2 指令"])]
  │
  ... （5 个 fork 子 Agent 并发启动）

所有子 Agent 共享前缀：[完整父消息 + PLACEHOLDER_RESULTS]
只有末尾指令文本不同
→ API 请求前缀字节完全一致 → prompt cache 命中 → 成本大幅降低
```

### 示例 3：Team Agent 协作

```
用户启动：claude --agent team-lead

team-lead
  │
  ├─ tool_use: Agent({
  │     name: "researcher",
  │     team_name: "dev-team",
  │     prompt: "你是研究员，等待任务"
  │   })
  │   → status: 'teammate_spawned', agentId: 'agent-001', tmux_window: "researcher"
  │
  ├─ tool_use: Agent({
  │     name: "implementer",
  │     team_name: "dev-team",
  │     prompt: "你是实现者，等待任务"
  │   })
  │   → status: 'teammate_spawned', agentId: 'agent-002', tmux_window: "implementer"
  │
  ├─ tool_use: SendMessage({
  │     to: "researcher",
  │     message: "请调研 React Query 与 SWR 的对比",
  │     summary: "调研任务"
  │   })
  │   → 写入 mailbox: ~/.claude/teams/dev-team/inboxes/researcher.json
  │
  │   [researcher Agent 在下轮工具调用时读取 mailbox，开始工作]
  │   [researcher 完成后：SendMessage({ to: "team-lead", message: "调研结果..." })]
  │
  └─ team-lead 收到 researcher 的消息
     → tool_use: SendMessage({ to: "implementer", message: "基于调研结果实现..." })
```

### 示例 4：Worktree 隔离 Agent

```markdown
---
name: safe-refactor
description: 在独立分支中安全重构代码
isolation: worktree
background: true
---
在独立工作树中重构代码，完成后提交到新分支，不影响主工作目录
```

```
AgentTool.call()
  ↓
createAgentWorktree("agent-abc1234")
  → git worktree add /project/.claude/worktrees/agent-abc1234 -b worktree-agent-abc1234
  → 子 Agent 在新目录中工作

子 Agent 完成：
  → 在 worktree-agent-abc1234 分支提交所有更改
  → 返回结果附带 worktreePath + worktreeBranch

父 Agent 收到结果：
  {
    status: 'completed',
    worktreePath: '/project/.claude/worktrees/agent-abc1234',
    worktreeBranch: 'worktree-agent-abc1234',
    content: "已完成重构，代码在 worktree-agent-abc1234 分支"
  }
  → 父 Agent 决定是否 git merge worktree-agent-abc1234

removeAgentWorktree()（自动清理工作树文件，分支保留供合并）
```

### 示例 5：自定义 Agent 定义（完整）

```markdown
---
name: pr-reviewer
description: |
  Reviews pull requests for code quality, security, and maintainability.
  Use when the user asks to review a PR, check code changes, or audit a diff.
model: sonnet
tools: [Read, Bash, Grep, Glob, WebFetch]
permissionMode: dontAsk
maxTurns: 40
background: true
color: purple
mcpServers:
  - github
hooks:
  SubagentStart:
    - hooks:
        - type: command
          command: "echo 'PR review started at $(date)' >> ~/.claude/review.log"
---

你是一名资深代码审查员。审查重点：

1. **安全性**：SQL 注入、XSS、权限绕过、敏感数据泄露
2. **正确性**：边界条件、并发问题、错误处理
3. **可维护性**：命名规范、代码复杂度、测试覆盖

审查流程：
1. 读取 PR 的 diff（通过 git diff 或 GitHub MCP）
2. 逐文件分析变更
3. 用 Bash 运行相关测试
4. 输出结构化审查报告

报告格式：
## 总体评价：APPROVE/REQUEST_CHANGES/COMMENT
### 发现的问题（按严重程度）
...
```

---

## 17. 关键设计决策总结

### 17.1 为什么主循环和子 Agent 用同一个 `query()`

**好处**：
- 代码复用：工具执行、流式处理、错误处理逻辑只写一份
- 递归自然：子 Agent 可以再派发子子 Agent，无需特殊代码
- 统一的 token 计数和 compact 机制

**关键差异完全通过参数实现**：`ToolUseContext` 控制隔离程度，`systemPrompt` 控制角色，`tools` 控制能力范围。

### 17.2 为什么后台 Agent 用"回执 + 通知"而不是阻塞

**阻塞方式的问题**：子 Agent 可能运行几分钟，阻塞期间主 Agent 无法响应用户，token 计数也一直累积。

**回执 + 通知的优势**：
- 主 Agent 立即解锁，可继续响应用户
- 多个后台 Agent 真正并发运行
- 用户可随时查看进度（`Read(outputFile)`）
- 通知以"用户消息"形式进入对话，自然触发新一轮思考

### 17.3 为什么 Fork Agent 要用统一占位符

**直接原因**：`prompt cache` 要求 API 请求的前缀字节完全一致。

**效果**：假设同时 Fork 10 个子 Agent，每个 Agent 的系统提示约 10K tokens，如果没有缓存，一次并发就要处理 100K tokens；有了缓存，9 个 Agent 命中缓存，只处理 10K + 9×指令文本 ≈ 11K tokens，**成本降低约 90%**。

### 17.4 权限设计：为什么后台 Agent 不弹框

后台 Agent 在没有 UI 上下文的情况下运行（甚至可能在用户离开屏幕时运行）。自动弹框会：
1. 无人应答，Agent 永久挂起
2. 在用户不知情时修改文件

因此 `shouldAvoidPermissionPrompts: true` 是后台 Agent 的强制约束，它会自动拒绝需要用户确认的操作，保证后台行为的可预期性。

### 17.5 设计哲学：隔离但协调

**隔离**：每个子 Agent 有独立的沙箱——文件缓存、中止信号、权限决策、转录日志。子 Agent 不能随意改变主 UI 状态。

**协调**：子 Agent 可以通过以下渠道与父/兄弟 Agent 协作：
- `task-notification`：异步结果通知
- `SendMessage`：主动消息传递
- `Mailbox` 文件：持久化消息队列
- Worktree 分支：代码变更隔离后合并

这种设计让每个子 Agent 可以放心地在自己的范围内工作，同时又有清晰的边界和协调机制——正是分布式系统中"隔离但有序通信"思想在 AI Agent 领域的体现。

---

### 17.6 安全设计：O_NOFOLLOW 和磁盘上限

后台 Agent 的输出路径涉及符号链接，存在被恶意进程替换的风险。系统通过以下手段防御：
- **O_NOFOLLOW**：创建输出符号链接时使用此标志，防止路径被替换后意外写入错误位置
- **8MB 单文件读取上限**：防止恶意 Agent 生成超大输出耗尽内存
- **5GB 总磁盘上限**：系统级配额，超出后自动清理最旧任务

### 17.7 Team Agent 的 1 秒轮询权衡

Mailbox 读取采用 1 秒轮询而非基于文件事件（inotify/FSEvents），原因是：
- 跨平台一致性（inotify 在 Linux/macOS 行为有差异）
- 轮询间隔 1 秒对协作 Agent 来说延迟可接受（人类沟通粒度）
- 实现简单，无需依赖 OS 事件机制

代价是 Teammate Agent 最多有 **1 秒** 的消息接收延迟。

---

*文档生成自对 `/home/user/claude-code` 代码库的完整静态分析，并整合了多轮深度代码探索的补充发现。*
*核心分析文件：`AgentTool.tsx`（1263行）、`runAgent.ts`（878行）、`LocalAgentTask.tsx`（516行+）、`agentToolUtils.ts`（687行+）、`loadAgentsDir.ts`（756行）、`forkSubagent.ts`（211行）、`SendMessageTool.ts`（917行）、`AppStateStore.ts`、`useInboxPoller.ts`。*
