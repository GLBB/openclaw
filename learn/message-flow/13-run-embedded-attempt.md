# (13) runEmbeddedAttempt - 核心执行

> 本文档分析 `runEmbeddedAttempt` 函数，这是 OpenClaw Agent 执行的核心。
>
> 函数位置: `src/agents/pi-embedded-runner/run/attempt.ts:704`
> 函数行数: ~3700 行

---

## 函数签名

```typescript
export async function runEmbeddedAttempt(
  params: EmbeddedRunAttemptParams,
): Promise<EmbeddedRunAttemptResult>;
```

---

## 执行阶段概览

`runEmbeddedAttempt` 是 PI Embedded Runner 的核心执行函数，包含 6 个主要阶段：

```
┌─────────────────────────────────────────────────────────────────┐
│                    runEmbeddedAttempt 执行流程                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [阶段 A] 初始化 (704-850)                                      │
│  ├── workspace 解析                                            │
│  ├── sandbox 配置                                              │
│  ├── skills 加载                                               │
│  └── diagnostic 事件                                           │
│                                                                 │
│  [阶段 B] ★ Tools 构建 (850-1150)                               │
│  ├── createOpenClawCodingTools → 核心工具集                    │
│  │   └── read/write/edit/grep/exec/web_search/message...     │
│  ├── getOrCreateSessionMcpRuntime → MCP 工具                   │
│  ├── createBundleLspToolRuntime → LSP 工具                     │
│  └── applyEmbeddedAttemptToolsAllow → 工具过滤                 │
│                                                                 │
│  [阶段 C] ★ PI Agent Session 创建 (1150-1600)                   │
│  ├── SessionManager.fromFile ← @mariozechner/pi-coding-agent    │
│  ├── createAgentSession ← @mariozechner/pi-coding-agent         │
│  │   └── 返回 AgentSession，管理 conversation loop             │
│  ├── history 处理                                              │
│  └── bootstrap context                                         │
│                                                                 │
│  [阶段 D] Prompt 构建 (1600-2000)                               │
│  ├── systemPrompt 组装                                         │
│  ├── Provider contribution                                     │
│  ├── cache boundary                                            │
│  └── bootstrap files 注入                                      │
│                                                                 │
│  [阶段 E] StreamFn 注册 (1870-1920)                             │
│  ├── registerProviderStreamForModel                            │
│  │   └── providerStreamFn = Provider.streamCompletion          │
│  │   └── (函数引用，尚未调用)                                   │
│  │                                                              │
│  └── resolveEmbeddedAgentStreamFn                              │
│      └── session.agent.streamFn = 包装后的 streamFn            │
│                                                                 │
│  [阶段 E-2] ★ PI Agent 执行 (1920-3200)                          │
│  ├── subscribeEmbeddedPiSession                                │
│  │   └── session.subscribe(handler) ← 注册事件处理器           │
│  │                                                              │
│  ├── activeSession.prompt() ← 启动对话循环                     │
│  │   └── [Conversation Loop] ← @mariozechner/pi-coding-agent   │
│  │       └── 循环: streamFn() → Provider.streamCompletion      │
│  │                                                              │
│  ├── 工具执行循环 (PI Agent 管理)                              │
│  └── 流式响应处理                                              │
│                                                                 │
│  [阶段 F] 结果与清理 (3200-3699)                                │
│  ├── 结果分类                                                  │
│  ├── usage 统计                                                │
│  ├── 诊断事件                                                  │
│  └── 资源清理                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**重要说明**：

`runEmbeddedAttempt` **不直接调用** `Provider.streamCompletion`。

调用链（含 PI Agent 角色）：

```
runEmbeddedAttempt
    │
    ├── registerProviderStreamForModel → providerStreamFn (函数引用)
    │   └── providerStreamFn = Provider.streamCompletion
    │
    ├── resolveEmbeddedAgentStreamFn → 包装成 streamFn
    │   └── session.agent.streamFn = streamFn
    │
    ├── subscribeEmbeddedPiSession
    │       │
    │       └── session.subscribe(handler) ← 注册事件处理器
    │
    └── activeSession.prompt() ← ★ 启动 PI Agent 对话循环
            │
            └── [PI Agent Conversation Loop] ← @mariozechner/pi-coding-agent
                    │
                    └── 循环执行直到完成:
                        │
                        ├── session.agent.streamFn(model, context, options)
                        │   │
                        │   └── Provider.streamCompletion ← 此时才真正调用
                        │       │
                        │       └── 流式响应 → toolCall?
                        │               │
                        │               ├── 有 toolCall → executeToolCall
                        │               │       │
                        │               │       └── 继续循环 ↺
                        │               │
                        │               └── 无 toolCall → 结束循环 ✓
                        │
                        └── 收集结果
```

---

## 阶段 A: 初始化

### 代码位置

`attempt.ts:704-850`

### 核心逻辑

```typescript
export async function runEmbeddedAttempt(params) {
  // 1. workspace 解析
  const resolvedWorkspace = resolveUserPath(params.workspaceDir);

  // 2. AbortController
  const runAbortController = new AbortController();

  // 3. HTTP runtime 配置
  configureEmbeddedAttemptHttpRuntime({ timeoutMs: params.timeoutMs });

  // 4. Stage tracker (性能追踪)
  const prepStages = createEmbeddedRunStageTracker();

  // 5. workspace 目录创建
  await fs.mkdir(resolvedWorkspace, { recursive: true });

  // 6. Sandbox 解析
  const sandbox = await resolveSandboxContext({
    config: params.config,
    sessionKey: sandboxSessionKey,
    workspaceDir: resolvedWorkspace,
  });

  // 7. Agent ID 解析
  const { sessionAgentId } = resolveSessionAgentIds({
    sessionKey: params.sessionKey,
    config: params.config,
    agentId: params.agentId,
  });
}
```

### 关键概念

| 概念                   | 说明                                                   |
| ---------------------- | ------------------------------------------------------ |
| **resolvedWorkspace**  | 用户工作目录（经过路径解析）                           |
| **effectiveWorkspace** | 实际工作目录（可能被 sandbox 修改）                    |
| **sandbox**            | Sandbox 配置（enabled、workspaceAccess、workspaceDir） |
| **sessionAgentId**     | 当前 session 使用的 agent ID                           |
| **prepStages**         | 性能追踪器，记录各阶段耗时                             |

### Sandbox 模式

```typescript
const effectiveWorkspace = sandbox?.enabled
  ? sandbox.workspaceAccess === "rw"
    ? resolvedWorkspace //读写模式：使用原目录
    : sandbox.workspaceDir //只读/无访问：使用 sandbox 目录
  : resolvedWorkspace; //无 sandbox：使用原目录
```

---

## 阶段 B: Tools 构建

### 代码位置

`attempt.ts:850-1150`

### 核心逻辑

```typescript
// 1. Skills 加载
const { shouldLoadSkillEntries, skillEntries } = resolveEmbeddedRunSkillEntries({
  workspaceDir: effectiveWorkspace,
  config: params.config,
  agentId: sessionAgentId,
  skillsSnapshot: params.skillsSnapshot,
});

// 2. Skills ENV 应用
restoreSkillEnv = applySkillEnvOverrides({
  skills: skillEntries ?? [],
  config: params.config,
});

// 3. Skills Prompt
const skillsPrompt = resolveSkillsPromptForRun({
  skillsSnapshot: params.skillsSnapshot,
  entries: shouldLoadSkillEntries ? skillEntries : undefined,
  config: params.config,
  workspaceDir: effectiveWorkspace,
  agentId: sessionAgentId,
});

// 4. 核心工具创建
const toolsRaw = params.disableTools || isRawModelRun
  ? []
  : createOpenClawCodingTools({
      agentId: sessionAgentId,
      workspaceDir: effectiveWorkspace,
      sandbox,
      config: params.config,
      abortSignal: runAbortController.signal,
      modelProvider: params.provider,
      modelId: params.modelId,
      // ... 更多参数
    });

// 5. Bundle MCP Tools
const bundleMcpRuntime = await getOrCreateSessionMcpRuntime({
  sessionId: params.sessionId,
  sessionKey: params.sessionKey,
  workspaceDir: effectiveWorkspace,
  cfg: params.config,
});

// 6. Bundle LSP Tools
const bundleLspRuntime = await createBundleLspToolRuntime({
  workspaceDir: effectiveWorkspace,
  cfg: params.config,
  reservedToolNames: [...],
});

// 7. Tool Policy 应用
const filteredBundledTools = applyFinalEffectiveToolPolicy({
  bundledTools: [...(bundleMcpRuntime?.tools ?? []), ...(bundleLspRuntime?.tools ?? [])],
  config: params.config,
  sandboxToolPolicy: sandbox?.tools,
  // ...
});

// 8. 合成最终工具列表
const effectiveTools = [...tools, ...filteredBundledTools];
```

### 工具来源

| 来源                 | 说明                                                                  |
| -------------------- | --------------------------------------------------------------------- |
| **Core Tools**       | `createOpenClawCodingTools` 创建的核心工具（read/write/exec/grep 等） |
| **Bundle MCP Tools** | 通过 MCP 协议连接的外部工具                                           |
| **Bundle LSP Tools** | 语言服务器协议工具                                                    |
| **Client Tools**     | 客户端提供的工具（如 Claude Code 的工具）                             |

### 工具过滤流程

```
原始工具列表
    ↓ applyEmbeddedAttemptToolsAllow (toolsAllow 配置)
核心工具
    ↓ applyFinalEffectiveToolPolicy (sandbox/config policy)
有效工具
```

---

## 阶段 C: Session 管理

### 代码位置

`attempt.ts:1150-1600`

### 核心逻辑

```typescript
// 1. Bootstrap routing 解析
const bootstrapRouting = await resolveAttemptWorkspaceBootstrapRouting({
  isWorkspaceBootstrapPending,
  bootstrapContextRunKind: params.bootstrapContextRunKind,
  trigger: params.trigger,
  sessionKey: params.sessionKey,
  // ...
});

// 2. Bootstrap context 解析
const {
  bootstrapFiles: hookAdjustedBootstrapFiles,
  contextFiles: resolvedContextFiles,
  shouldRecordCompletedBootstrapTurn,
} = await resolveAttemptBootstrapContext({
  contextInjectionMode,
  bootstrapContextMode: params.bootstrapContextMode,
  bootstrapMode,
  sessionFile: params.sessionFile,
  // ...
});

// 3. Bootstrap预算分析
const bootstrapAnalysis = analyzeBootstrapBudget({
  files: buildBootstrapInjectionStats({
    bootstrapFiles: bootstrapFilesForInjectionStats,
    injectedFiles: contextFiles,
  }),
  bootstrapMaxChars,
  bootstrapTotalMaxChars,
});

// 4. SessionManager 加载
const sessionManager = await SessionManager.fromFile({
  file: params.sessionFile,
  autoload: false,
  subscriptions: { onStore: onSessionStore },
});

// 5. PI Session 创建
const session = await createAgentSession({
  model: modelForSession,
  tools: effectiveTools,
  systemPrompt: systemPromptContent,
  clientTools,
  resourceLoader: new DefaultResourceLoader(),
  subscriptions: {
    assistantMessage: onAssistantMessage,
    toolCall: onToolCall,
    toolResult: onToolResult,
  },
});

// 6. History 处理
const history = await loadSessionHistory();
const limitedHistory = limitHistoryTurns({
  history,
  maxTurns: getDmHistoryLimitFromSessionKey(params.sessionKey),
});
```

### 关键概念

| 概念                  | 说明                                                                           |
| --------------------- | ------------------------------------------------------------------------------ |
| **SessionManager**    | PI Agent Session 管理器，负责持久化 (`@mariozechner/pi-coding-agent`)          |
| **AgentSession**      | PI Agent Session，管理整个 conversation loop (`@mariozechner/pi-coding-agent`) |
| **bootstrapMode**     | Bootstrap 状态（`none` / `limited` / `full`）                                  |
| **contextFiles**      | 注入到 Prompt 的上下文文件（AGENTS.md 等）                                     |
| **bootstrapAnalysis** | Bootstrap 文件预算分析（字符限制）                                             |
| **conversation loop** | PI Agent 核心机制：LLM 响应 → 工具执行 → 继续调用 LLM                          |

### Bootstrap 文件注入

```typescript
const contextFiles = shouldStripBootstrapFromContext
  ? remappedContextFiles.filter((file) => !/(^|[\\/])BOOTSTRAP\.md$/iu.test(file.path.trim()))
  : remappedContextFiles;
```

---

## 阶段 D: Prompt 构建

### 代码位置

`attempt.ts:1600-2000`

### 核心逻辑

```typescript
// 1. Provider Prompt Contribution
const promptContribution = resolveProviderSystemPromptContribution({
  config: params.config,
  providerId: params.provider,
  modelId: params.modelId,
  // ...
});

// 2. System Prompt 构建
const systemPromptReport = buildSystemPromptReport({
  workspaceDir: effectiveWorkspace,
  agentId: sessionAgentId,
  modelDisplay: modelForDisplay,
  tools: effectiveTools,
  contextFiles,
  skillsPrompt,
  heartbeatPrompt,
  promptContribution,
  // ...
});

// 3. Provider Text Transform
const transformPromptText = resolveProviderTextTransforms({
  config: params.config,
  providerId: params.provider,
});

// 4. 最终 System Prompt
const systemPromptContent = transformProviderSystemPrompt({
  prompt: systemPromptReport.prompt,
  transformPromptText,
});

// 5. User Prompt 构建
const userPrompt = buildUserPrompt({
  commandBody: params.commandBody,
  bootstrapMode,
  bootstrapPromptWarning,
  // ...
});
```

### Prompt 组成

```
System Prompt
├── Tooling section
├── Execution Bias section
├── Safety section
├── Skills section (if available)
├── Workspace section
├── Project Context section (bootstrap files)
│   ├── AGENTS.md
│   ├── SOUL.md
│   ├── IDENTITY.md
│   ├── USER.md
│   ├── TOOLS.md
│   └── MEMORY.md
├── ─────────── Cache Boundary ───────────
├── Current Date & Time section
├── Messaging section
├── Runtime section
└── Heartbeats section (if enabled)

User Prompt
├── Bootstrap warning (if needed)
├── User message content
└── Image blocks (if provided)
```

### Cache Boundary

```typescript
// 稳定内容在 cache boundary 之上
const stablePrefix = [
  toolingSection,
  executionBiasSection,
  safetySection,
  skillsSection,
  workspaceSection,
  projectContextSection,
];

// 动态内容在 cache boundary 之下
const dynamicSuffix = [timeSection, messagingSection, runtimeSection, heartbeatSection];
```

---

## 阶段 E: 执行

### 代码位置

`attempt.ts:2000-3200`

### 核心逻辑

```typescript
// 1. 注册 Provider Stream Function
const providerStreamFn = registerProviderStreamForModel({
  provider: params.provider,
  model: params.model,
  config: params.config,
  // ...
});
// providerStreamFn = Provider.streamCompletion (函数引用，尚未调用)

// 2. 解析并包装 Stream Function
activeSession.agent.streamFn = resolveEmbeddedAgentStreamFn({
  currentStreamFn: defaultSessionStreamFn,
  providerStreamFn,
  shouldUseWebSocketTransport,
  wsApiKey,
  sessionId: params.sessionId,
  signal: runAbortController.signal,
  model: params.model,
  resolvedApiKey: params.resolvedApiKey,
  authStorage: params.authStorage,
});
// streamFn 被赋值给 session.agent.streamFn

// 3. 启动 Session 执行
const subscribeResult = await subscribeEmbeddedPiSession({
  session,
  runId: params.runId,
  hookRunner,
  onBlockReply,
  onToolResult,
  // ...
});

// 4. session.run() 在 subscribeEmbeddedPiSession 内部被调用
//    → 调用 session.agent.streamFn → Provider.streamCompletion
```

### 关键说明

**重要**：`runEmbeddedAttempt` **不直接调用** `Provider.streamCompletion`！

PI Agent (`@mariozechner/pi-coding-agent`) 的 Session 管理整个对话循环：

```
┌─────────────────────────────────────────────────────────────────────┐
│                 PI Agent Conversation Loop                           │
│                 (@mariozechner/pi-coding-agent)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Session.subscribe() 启动后：                                       │
│                                                                     │
│  while (!finished) {                                                │
│      // 1. 调用 streamFn 获取 LLM 响应                              │
│      const response = await session.agent.streamFn(model, context); │
│                                                                     │
│      // 2. 处理流式响应                                             │
│      for (const chunk of response) {                                │
│          if (chunk.type === "text") {                               │
│              onAssistantMessage(chunk.text);                        │
│          }                                                          │
│          if (chunk.type === "toolCall") {                           │
│              // 3. 执行工具                                         │
│              const result = await executeToolCall(chunk.toolCall);  │
│              // 4. 将结果加入 context，继续循环                     │
│              context.push(toolResult);                              │
│          }                                                          │
│      }                                                              │
│                                                                     │
│      // 5. 检查是否完成                                             │
│      if (finishReason === "stop" || noToolCalls) {                  │
│          finished = true;                                           │
│      }                                                              │
│  }                                                                  │
│                                                                     │
│  return { assistantTexts, toolMetas, usage };                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

完整调用链：

```
runEmbeddedAttempt
    │
    ├── registerProviderStreamForModel → providerStreamFn (函数引用)
    │   └── providerStreamFn = Provider.streamCompletion
    │
    ├── resolveEmbeddedAgentStreamFn → 包装成 streamFn
    │   └── session.agent.streamFn = streamFn
    │
    ├── subscribeEmbeddedPiSession({ session })
    │       │
    │       └── session.subscribe(handler) ← 注册事件处理器
    │       └── 返回 subscription
    │
    └── activeSession.prompt() ← ★ 启动对话循环
            │
            └── [PI Agent Conversation Loop] ← @mariozechner/pi-coding-agent
                    │
                    └── 循环: session.agent.streamFn()
                            │
                            └── Provider.streamCompletion ← 此时才真正调用
```

### 执行流程

```
subscribeEmbeddedPiSession → session.subscribe(handler)
        │
        └── 注册事件处理器（message_update, tool_execution 等）
        └── 返回 subscription

activeSession.prompt() ← ★ 启动 PI Agent 对话循环
        │
        └── [PI Agent Conversation Loop] ← @mariozechner/pi-coding-agent
                │
                ├── 调用 session.agent.streamFn(model, context, options)
                │   │
                │   └── Provider.streamCompletion
                │       │
                │       ├── HTTP POST → LLM Provider API
                │       │
                │       └── AsyncIterable<StreamChunk>
                │           ├── text chunk → message_update 事件
                │           ├── toolCall → executeToolCall
                │           └── finishReason → 检查是否继续
                    │
                    ├── 工具执行循环 (如有 toolCall) ← PI Agent 管理
                    │   │
                    │   └── executeToolCall
                    │       │
                    │       └── tool.execute(input)
                    │       │
                    │       └── toolResult 加入 context
                    │       │
                    │       └── 继续调用 streamFn → 循环 ↺
                    │
                    └── 收集结果
                        ├── assistantTexts
                        ├── toolMetas
                        └── usage
```

### PI Agent Session 的核心贡献

PI Agent (`@mariozechner/pi-coding-agent`) 提供的 `AgentSession` 负责：

| 功能                     | 说明                                                 |
| ------------------------ | ---------------------------------------------------- |
| **Conversation Loop**    | 管理整个对话循环：LLM 响应 → 工具执行 → 继续调用 LLM |
| **Tool Execution**       | 自动执行 toolCall，将结果加入 context                |
| **Stream Management**    | 处理流式响应，调用 onAssistantMessage 回调           |
| **History Management**   | 维护对话历史，支持 compaction                        |
| **Abort Handling**       | 支持 abortSignal，正确中断循环                       |
| **Subscription Pattern** | 通过 subscribe() 启动执行，支持事件订阅              |

### 工具执行

```typescript
// 工具执行在 subscribeEmbeddedPiSession 内部
const executeToolCall = async (toolCall) => {
  // 1. 获取工具定义
  const tool = effectiveTools.find((t) => t.name === toolCall.name);

  // 2. 验证输入
  const validatedInput = validateToolInput(tool, toolCall.input);

  // 3. 执行工具
  const result = await tool.execute(validatedInput, {
    abortSignal: runAbortController.signal,
    // ...
  });

  // 4. 返回结果
  return {
    toolCallId: toolCall.id,
    result,
  };
};
```

---

## 阶段 F: 结果与清理

### 代码位置

`attempt.ts:3200-3699`

### 核心逻辑

```typescript
// 1. 结果收集
const assistantTexts = subscribeResult.assistantTexts;
const toolMetas = subscribeResult.toolMetas;
const usage = subscribeResult.usage;

// 2. 结果分类
const classification = classifyRunResult({
  assistantTexts,
  toolMetas,
  finishReason: subscribeResult.finishReason,
});

// 3. 诊断事件
emitDiagnosticRunCompleted(
  classification === "ok" ? "completed" : "error",
  classification === "error" ? subscribeResult.error : undefined,
);

// 4. Session 持久化
await sessionManager.store();

// 5. 资源清理
finally {
  // 6. Skills ENV 恢复
  restoreSkillEnv?.();

  // 7. Bundle MCP/LSP清理
  await bundleMcpRuntime?.dispose();
  await bundleLspRuntime?.dispose();

  // 8. Session释放
  releaseWsSession(session);
}
```

### 返回值

```typescript
type EmbeddedRunAttemptResult = {
  // 文本响应
  assistantTexts: string[];

  // 工具元数据
  toolMetas: ToolMeta[];

  // Token 使用
  usage: {
    inputTokens: number;
    outputTokens: number;
    cacheWriteTokens?: number;
    cacheReadTokens?: number;
  };

  // 结果分类
  agentHarnessResultClassification?: "ok" | "error" | "aborted" | "timeout";

  // 其他字段
  finishReason?: string;
  model?: string;
  provider?: string;
  // ...
};
```

---

## 关键辅助函数

### createOpenClawCodingTools

创建 OpenClaw 核心工具集：

```typescript
const tools = createOpenClawCodingTools({
  workspaceDir,
  sandbox,
  config,
  abortSignal,
  modelProvider,
  modelId,
  // ...
});

// 返回工具列表
[
  "read",
  "write",
  "edit",
  "apply_patch",
  "grep",
  "find",
  "ls",
  "exec",
  "process",
  "web_search",
  "web_fetch",
  "browser",
  "canvas",
  "nodes",
  "cron",
  "message",
  "gateway",
  "agents_list",
  "sessions_list",
  "sessions_spawn",
  "subagents",
  "session_status",
  "image",
  "image_generate",
];
```

### subscribeEmbeddedPiSession

订阅 PI Session 执行：

```typescript
const result = await subscribeEmbeddedPiSession({
  session,
  streamFn,
  abortSignal,
  onAssistantMessage,
  onToolCall,
  onToolResult,
});
```

### resolveSandboxContext

解析 Sandbox 配置：

```typescript
const sandbox = await resolveSandboxContext({
  config,
  sessionKey,
  workspaceDir,
});

// 返回
{
  enabled: boolean;
  workspaceAccess: "rw" | "ro" | "none";
  workspaceDir: string;
  tools?: SandboxToolPolicy;
}
```

---

## 异常处理

### Abort 处理

```typescript
// 外部 abort
params.abortSignal?.addEventListener("abort", () => {
  externalAbort = true;
  runAbortController.abort(params.abortSignal.reason);
});

// Yield abort (sessions_yield 工具触发)
onYield: (message) => {
  yieldDetected = true;
  yieldMessage = message;
  runAbortController.abort("sessions_yield");
  abortSessionForYield?.();
},
```

### Timeout 处理

```typescript
// LLM idle timeout
const idleTimeout = resolveLlmIdleTimeoutMs(params.config);
const idleTimer = setTimeout(() => {
  idleTimedOut = true;
  runAbortController.abort("idle_timeout");
}, idleTimeout);

//总 timeout
const totalTimeout = params.timeoutMs;
const totalTimer = setTimeout(() => {
  timedOut = true;
  runAbortController.abort("timeout");
}, totalTimeout);
```

### 错误分类

```typescript
type RunErrorClassification =
  | "ok" // 正常完成
  | "aborted" // 用户中断
  | "timeout" //超时
  | "idle_timeout" // LLM 空闲超时
  | "error" // 其他错误
  | "yield"; // sessions_yield 触发
```

---

## 性能追踪

### Stage Tracker

```typescript
const prepStages = createEmbeddedRunStageTracker();

//标记各阶段
prepStages.mark("workspace-sandbox");
prepStages.mark("skills");
prepStages.mark("core-plugin-tools");
prepStages.mark("bundle-tools");
prepStages.mark("bootstrap-context");
prepStages.mark("session-manager");
prepStages.mark("system-prompt");
prepStages.mark("session-creation");

// 输出耗时摘要
const summary = prepStages.snapshot();
// [{ stage: "workspace-sandbox", durationMs: 50 }, ...]
```

---

## 调用关系

### 上游调用

```
runAgentHarnessV2LifecycleAttempt (12)
    │
    └── harness.send(session)
            │
            └── harness.runAttempt(params)  ← V1接口
                    │
                    └── runEmbeddedAttempt(params) ← 本函数
```

### 下游调用

```
runEmbeddedAttempt (13)
    │
    ├── resolveSandboxContext
    ├── resolveEmbeddedRunSkillEntries
    ├── createOpenClawCodingTools
    ├── getOrCreateSessionMcpRuntime
    ├── createBundleLspToolRuntime
    │
    ├── SessionManager.fromFile ← PI Agent (@mariozechner/pi-coding-agent)
    │
    ├── createAgentSession ← PI Agent (@mariozechner/pi-coding-agent)
    │       │
    │       └── 返回 AgentSession，管理 conversation loop
    │
    ├── buildSystemPromptReport
    │
    ├── registerProviderStreamForModel
    │       └── 返回 Provider.streamCompletion 函数引用
    │
    ├── resolveEmbeddedAgentStreamFn
    │       └── 包装 streamFn，赋值给 session.agent.streamFn
    │
    ├── subscribeEmbeddedPiSession
    │       │
    │       └── session.subscribe(handler) ← 注册事件处理器
    │
    ├── activeSession.prompt() ← 启动对话循环
    │       │
    │       └── [PI Agent Conversation Loop]
    │               │
    │               └── streamFn → Provider.streamCompletion (14)
    │
    └── sessionManager.store()
```

---

## 数据流

```
输入: EmbeddedRunAttemptParams
├── workspaceDir
├── sessionKey / sessionId
├── sessionFile
├── provider / modelId / model
├── commandBody
├── config
├── toolsAllow
├── abortSignal
└── ...

    ↓ [阶段 A-D 准备]

执行上下文
├── effectiveWorkspace
├── sandbox
├── effectiveTools
├── session
├── systemPromptContent
├── userPrompt
└── streamFn

    ↓ [阶段 E 执行]

流式输出
├── assistantTexts (累积)
├── toolMetas (累积)
├── usage (累积)
└── finishReason

    ↓ [阶段 F 结果]

输出: EmbeddedRunAttemptResult
├── assistantTexts
├── toolMetas
├── usage
├── finishReason
├── model / provider
└── classification
```

---

## 关键文件依赖

| 文件                                   | 作用                         |
| -------------------------------------- | ---------------------------- |
| `attempt.ts`                           | 本函数，核心执行             |
| `subscribe.ts`                         | `subscribeEmbeddedPiSession` |
| `tools.ts`                             | `createOpenClawCodingTools`  |
| `session-tool-result-guard-wrapper.ts` | Session 工具结果保护         |
| `bootstrap-files.ts`                   | Bootstrap 文件处理           |
| `system-prompt.ts`                     | System Prompt 构建           |
| `sandbox.ts`                           | Sandbox 配置解析             |
| `../compact.ts`                        | Compaction 触发              |
| `../model.ts`                          | 模型处理                     |

---

## 延伸阅读

- [12-run-harness-v2-lifecycle.md](12-run-harness-v2-lifecycle.md) - V2 生命周期（上游调用）
- [14-provider-stream-completion.md](14-provider-stream-completion.md) - Provider API 调用（下游调用）
- [../session-management.md](../session-management.md) - Session 管理机制
- [../architecture.md](../architecture.md) - 整体架构

---

> **文档版本**: 2026-05-04
> **分析基于**: OpenClaw v2026.5.3
> **函数行数**: ~3700 行
