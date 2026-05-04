# `getReplyFromConfig` 函数技术文档

## 函数签名

```typescript
export async function getReplyFromConfig(
  ctx: MsgContext,
  opts?: GetReplyOptions,
  configOverride?: OpenClawConfig,
): Promise<ReplyPayload | ReplyPayload[] | undefined>;
```

## 位置

- **文件**: `src/auto-reply/reply/get-reply.ts`
- **起始行**: 173
- **结束行**: 680

---

## 一句话理解

> **这是 Agent 执行的"总入口"——负责准备一切，然后调用模型生成回复。**

---

## 在消息处理流程中的位置

```
消息处理流程三层架构:

┌─────────────────────────────────────────────────────┐
│         协调层: dispatchReplyFromConfig              │  ← 决策、协调、监控
│         (去重、路由、Hook、策略)                      │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│         执行层: getReplyFromConfig                   │  ← 本函数
│         (配置、模型、会话、指令)                      │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│         运行层: runPreparedReply                     │  ← 实际调用模型
│         (Prompt 构建、工具执行、流式输出)            │
└─────────────────────────────────────────────────────┘
```

**通俗理解**:

- `dispatchReplyFromConfig` 是"指挥官"，决定要不要执行
- `getReplyFromConfig` 是"参谋长"，准备执行所需的一切
- `runPreparedReply` 是"士兵"，真正冲锋陷阵

---

## 输入输出

### 输入：需要什么？

| 参数             | 是什么     | 通俗理解                       |
| ---------------- | ---------- | ------------------------------ |
| `ctx`            | 消息上下文 | "谁发了什么消息"               |
| `opts`           | 执行选项   | "回调函数、中断信号、各种控制" |
| `configOverride` | 配置覆盖   | "临时改配置"                   |

### `opts` 的主要回调（控制回复发送）

| 回调              | 触发时机       | 用途             |
| ----------------- | -------------- | ---------------- |
| `onBlockReply`    | 流式块到达     | 发送流式回复     |
| `onToolResult`    | 工具执行结果   | 发送工具结果     |
| `onPlanUpdate`    | Agent 计划更新 | 显示进度         |
| `onReplyStart`    | Agent 开始回复 | 发送"正在输入"   |
| `onModelSelected` | 模型选定后     | 知道用了什么模型 |

### 输出：返回什么？

| 类型             | 含义                     |
| ---------------- | ------------------------ |
| `ReplyPayload`   | 单条回复                 |
| `ReplyPayload[]` | 多条回复                 |
| `undefined`      | 无回复（如被 Hook 拦截） |

---

## 整体流程图

```
getReplyFromConfig
    │
    ├─► [配置解析] ──────────────────────┐
    │   ├─ 获取运行时配置                 │
    │   ├─ 确定 Agent ID                  │
    │   └─ 确定 Workspace                 │
    │                                      │
    ├─► [模型选择] ──────────────────────┤
    │   ├─ 默认模型                       │
    │   ├─ Heartbeat 模型覆盖             │  ← 多层 override
    │   ├─ Session 模型覆盖               │
    │   ├─ Channel 模型覆盖               │
    │   └─ 最终确定 provider + model      │
    │                                      │
    ├─► [预处理] ────────────────────────┤
    │   ├─ 媒体理解（图片/文件分析）      │
    │   ├─ 链接理解（URL 内容提取）       │
    │   └─ Hook 触发                      │
    │                                      │
    ├─► [会话状态初始化] ────────────────┤
    │   ├─ 加载 session-store             │
    │   ├─ 判断是否新会话                 │
    │   ├─ 判断是否触发 reset             │
    │   └─ 提取会话历史                   │
    │                                      │
    ├─► [指令解析] ──────────────────────┤
    │   ├─ /think, /verbose, /reasoning   │
    │   ├─ /elevated, /block              │
    │   ├─ 技能命令                       │
    │   └─ 模型临时切换                   │
    │                                      │
    ├─► [Inline Actions] ────────────────┤
    │   ├─ 处理内联操作                   │
    │   └─ 可能直接返回                   │  ← 退出点
    │                                      │
    ├─► [before_agent_reply Hook] ───────┤
    │   ├─ Plugin 拦截机会                │
    │   └─ 可能直接返回                   │  ← 退出点
    │                                      │
    ├─► [媒体暂存] ──────────────────────┤
    │   └─ 将媒体文件暂存到 workspace     │
    │                                      │
    └─► [runPreparedReply] ───────────────┘
        │
        └─► 实际调用模型，生成回复
```

---

## 详细处理步骤

### Phase 1: 配置解析

**目的**: 确定"用什么配置"、"哪个 Agent"、"在哪工作"。

```
┌─────────────────────────────────────────────────────┐
│                    配置解析                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. 获取运行时配置                                   │
│     cfg = resolveGetReplyConfig(...)               │
│     → 可能用 configOverride 覆盖                   │
│                                                     │
│  2. 确定 Agent ID                                   │
│     agentId = resolveSessionAgentId(sessionKey)    │
│     → 从 sessionKey 解析出 agentId                 │
│                                                     │
│  3. 确定 Workspace                                  │
│     workspaceDir = ensureAgentWorkspace(...)       │
│     → 确保 Agent 工作目录存在                       │
│     → 可能创建 bootstrap 文件                      │
│                                                     │
│  4. 确定超时                                         │
│     timeoutMs = resolveAgentTimeoutMs(...)         │
│                                                     │
│  5. 创建 Typing 控制器                              │
│     typing = createTypingController(...)           │
│     → 控制"正在输入"提示                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 2: 模型选择（多层 Override）

**目的**: 决定"用什么模型"。这是最复杂的部分，有多层优先级。

#### Override 优先级（从高到低）

```
┌─────────────────────────────────────────────────────┐
│              模型 Override 优先级                    │
│              （高 → 低）                             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ① Heartbeat 模型覆盖                               │
│     ├─ opts.heartbeatModelOverride                 │
│     ├─ agentCfg.heartbeat.model                    │
│     └─ 仅在 isHeartbeat=true 时生效                │
│     └─────────────────────────────────────────────│
│                     ↓                              │
│  ② Session 存储覆盖                                 │
│     ├─ sessionEntry.modelOverride                  │
│     ├─ sessionEntry.providerOverride               │
│     └─ 用户通过 /model 命令设置的                   │
│     └─────────────────────────────────────────────│
│                     ↓                              │
│ ③ Channel 模型覆盖                                  │
│     ├─ cfg.channels.modelByChannel                 │
│     ├─ 按渠道/群组配置不同模型                      │
│     └─ 例如：Slack 用 claude，Telegram 用 gpt      │
│     └─────────────────────────────────────────────│
│                     ↓                              │
│  ④ 默认模型                                         │
│     ├─ agentCfg.model                              │
│     ├─ agentCfg.provider                           │
│     └─ 全局默认配置                                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### 选择流程图

```
开始选择模型
    │
    ▼
┌─────────────────┐
│ isHeartbeat?    │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
   Yes       No
    │         │
    ▼         │
┌─────────────────┐
│ Heartbeat覆盖?  │
└────────┬────────┘
         │ Yes → 用 Heartbeat 模型
         │
         ▼
┌─────────────────┐
│ Session覆盖?    │
└────────┬────────┘
         │ Yes → 用 Session 存储的模型
         │
         ▼
┌─────────────────┐
│ Channel覆盖?    │
└────────┬────────┘
         │ Yes → 用 Channel 配置的模型
         │
         ▼
┌─────────────────┐
│ 用默认模型      │
└─────────────────┘
```

**通俗理解**: 就像"选拔队员"——先看有没有特殊指定（Heartbeat），再看有没有临时调整（Session），再看场地要求（Channel），最后用默认配置。

---

### Phase 3: 预处理

**目的**: 在 Agent 执行前，对消息内容进行增强理解。

```
┌─────────────────────────────────────────────────────┐
│                    预处理                            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. 媒体理解 (Media Understanding)                  │
│     ├─ 检查: 消息是否包含图片/文件                  │
│     ├─ 处理: 分析图片内容，生成文字描述             │
│     └─ 结果: Agent 能"看懂"图片                    │
│                                                     │
│  2. 链接理解 (Link Understanding)                   │
│     ├─ 检查: 消息是否包含 URL                      │
│     ├─ 处理: 抓取 URL 内容，生成摘要               │
│     └─ 结果: Agent 能"读懂"链接                    │
│                                                     │
│  3. Hook 触发                                       │
│     ├─ emitPreAgentMessageHooks                    │
│     └─ 触发消息预处理事件                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**通俗理解**: 就像"情报预处理"——把图片、链接变成 Agent 能直接理解的内容。

---

### Phase 4: 会话状态初始化

**目的**: 加载"对话历史"和"会话状态"。

```
┌─────────────────────────────────────────────────────┐
│              会话状态初始化                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  输出变量:                                          │
│                                                     │
│  sessionCtx      → 会话上下文                       │
│  sessionEntry    → session-store 中的条目          │
│  sessionKey      → 会话唯一键                       │
│  sessionId       → 会话 ID                          │
│  isNewSession    → 是否新会话                       │
│  resetTriggered  → 是否触发 reset (/new /reset)    │
│  systemSent      → 是否已发送 system prompt        │
│  abortedLastRun  → 上次运行是否被中断              │
│  storePath       → session-store.json 路径         │
│  groupResolution → 群组解析结果                     │
│  isGroup         → 是否群聊                         │
│  bodyStripped    → 去除指令后的消息体               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### reset 处理

```
┌─────────────────────────────────────────────────────┐
│              reset 触发检测                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  检测条件:                                          │
│  ├─ 用户发送 /new 或 /reset 命令                   │
│  └─ 或者会话配置触发重置                            │
│                                                     │
│  处理:                                              │
│  ├─ 调用 applyResetModelOverride                   │
│  ├─ 可能临时切换模型                               │
│  └─ 清空对话历史                                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 5: 指令解析

**目的**: 解析用户消息中的"特殊指令"（如 `/think`, `/verbose`）。

#### 支持的指令

| 指令         | 作用           | 示例                   |
| ------------ | -------------- | ---------------------- |
| `/think`     | 控制思考深度   | `/think high`          |
| `/verbose`   | 控制输出详细度 | `/verbose on`          |
| `/reasoning` | 控制推理显示   | `/reasoning show`      |
| `/elevated`  | 启用提升权限   | `/elevated`            |
| `/block`     | 启用块流式输出 | `/block`               |
| `/model`     | 临时切换模型   | `/model claude-opus-4` |

#### 解析流程

```
┌─────────────────────────────────────────────────────┐
│              resolveReplyDirectives                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  输入:                                              │
│  ├─ 消息内容                                        │
│  ├─ 会话状态                                        │
│  └─ 配置                                            │
│                                                     │
│  输出:                                              │
│  ├─ directives      → 解析出的指令集合              │
│  ├─ cleanedBody     → 去除指令后的纯文本            │
│  ├─ skillCommands   → 技能命令                      │
│  ├─ resolvedThinkLevel    → 思考级别               │
│  ├─ resolvedVerboseLevel  → 详细级别               │
│  ├─ resolvedReasoningLevel → 推理级别              │
│  ├─ blockStreamingEnabled  → 是否块流式            │
│  ├─ modelState      → 模型选择状态                  │
│  └─ ...                                             │
│                                                     │
│  特殊情况:                                          │
│  ├─ directiveResult.kind === "reply"               │
│  │  → 指令本身直接产生回复                         │
│  │  → 直接返回 [退出点 1]                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 6: Fast Path 检查

**目的**: 某些简单场景可以快速处理，不需要完整流程。

```
┌─────────────────────────────────────────────────────┐
│              Fast Path 检查                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  检测条件:                                          │
│  ├─ useFastTestRuntime = true                      │
│  ├─ 简单指令执行                                    │
│  └─ 或测试环境                                      │
│                                                     │
│  如果满足:                                          │
│  ├─ 直接调用 runPreparedReply                      │
│  ├─ 跳过后续复杂处理                               │
│  └─ 快速返回 [可能退出点 2]                         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 7: Inline Actions 处理

**目的**: 处理"内联操作"——可能在 Agent 执行前就完成的任务。

```
┌─────────────────────────────────────────────────────┐
│              handleInlineActions                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  处理内容:                                          │
│  ├─ 技能命令处理                                    │
│  ├─ 指令执行                                        │
│  ├─ 状态更新                                        │
│                                                     │
│  可能结果:                                          │
│  ├─ inlineActionResult.kind === "reply"            │
│  │  → 直接产生回复                                 │
│  │  → 返回 [退出点 3]                              │
│  │                                                 │
│  └─ inlineActionResult.kind === "continue"         │
│     → 继续 Agent 执行                              │
│     → 更新 directives                              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 8: before_agent_reply Hook

**目的**: 给 Plugin 最后一次机会拦截 Agent 执行。

```
┌─────────────────────────────────────────────────────┐
│         before_agent_reply Hook                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  检测:                                              │
│  ├─ !useFastTestBootstrap                          │
│  └─ hookRunner.hasHooks("before_agent_reply")      │
│                                                     │
│  调用:                                              │
│  hookRunner.runBeforeAgentReply({                  │
│    cleanedBody,                                     │
│    agentId,                                         │
│    sessionKey,                                      │
│    workspaceDir,                                    │
│    trigger: "user" | "heartbeat",                  │
│    ...                                              │
│  })                                                 │
│                                                     │
│  结果:                                              │
│  ├─ hookResult.handled = true                      │
│  │  → Plugin 拦截了                                │
│  │  → 返回 hookResult.reply [退出点 4]             │
│  │                                                 │
│  └─ hookResult.handled = false                     │
│     → Plugin 不拦截                                │
│     → 继续 Agent 执行                              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**通俗理解**: 就像"最后一道安检"——Agent 执行前，Plugin 可以决定是否放行。

---

### Phase 9: 媒体暂存

**目的**: 将消息中的媒体文件暂存到 workspace。

```
┌─────────────────────────────────────────────────────┐
│              媒体暂存                                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  条件:                                              │
│  ├─ !useFastTestBootstrap                          │
│  ├─ sessionKey 存在                                 │
│  ├─ !ctx.MediaStaged                               │
│  └─ hasInboundMedia(ctx)                           │
│                                                     │
│  处理:                                              │
│  stageSandboxMedia({                               │
│    ctx,                                             │
│    sessionCtx,                                      │
│    cfg,                                             │
│    sessionKey,                                      │
│    workspaceDir,                                    │
│  })                                                 │
│                                                     │
│  结果:                                              │
│  ├─ 媒体文件保存到 workspace                       │
│  └─ Agent 工具可以访问这些文件                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 10: 执行 Agent (runPreparedReply)

**目的**: 真正调用模型，生成回复。

```
┌─────────────────────────────────────────────────────┐
│              runPreparedReply                        │
│              (Agent 实际执行)                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  准备好的参数:                                      │
│  ├─ ctx          → 消息上下文                       │
│  ├─ sessionCtx   → 会话上下文                       │
│  ├─ cfg          → 配置                             │
│  ├─ agentId      → Agent ID                         │
│  ├─ provider     → 模型 Provider                    │
│  ├─ model        → 模型 ID                          │
│  ├─ directives   → 解析出的指令                     │
│  ├─ typing       → Typing 控制器                    │
│  ├─ opts         → 回调函数                         │
│  └─ ...                                            │
│                                                     │
│  执行过程:                                          │
│  ├─ 构建 System Prompt                             │
│  ├─ 加载对话历史                                   │
│  ├─ 准备工具定义                                   │
│  ├─ 调用模型 API                                   │
│  ├─ 处理工具调用                                   │
│  ├─ 流式输出回调                                   │
│  └─ 返回最终回复                                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 退出点速览

| 退出点 | 在哪一步 | 什么情况                | 返回什么                 |
| ------ | -------- | ----------------------- | ------------------------ |
| 1      | Phase 5  | 指令解析直接产生回复    | directiveResult.reply    |
| 2      | Phase 6  | Fast Path 快速执行      | 快速回复                 |
| 3      | Phase 7  | Inline Actions 产生回复 | inlineActionResult.reply |
| 4      | Phase 8  | Plugin 拦截 Agent 执行  | hookResult.reply         |

---

## 关键依赖函数

### 配置解析

| 函数                    | 用途                       |
| ----------------------- | -------------------------- |
| `resolveGetReplyConfig` | 获取运行时配置             |
| `resolveSessionAgentId` | 从 sessionKey 解析 agentId |
| `ensureAgentWorkspace`  | 确保 workspace 存在        |
| `resolveAgentTimeoutMs` | 解析超时配置               |

### 模型选择

| 函数                          | 用途                        |
| ----------------------------- | --------------------------- |
| `resolveDefaultModel`         | 解析默认模型                |
| `resolveModelRefFromString`   | 解析模型引用字符串          |
| `resolveChannelModelOverride` | 解析渠道模型覆盖            |
| `resolveStoredModelOverride`  | 解析 session 存储的模型覆盖 |

### 会话管理

| 函数                     | 用途             |
| ------------------------ | ---------------- |
| `initSessionState`       | 初始化会话状态   |
| `finalizeInboundContext` | 最终化入站上下文 |

### 指令处理

| 函数                     | 用途             |
| ------------------------ | ---------------- |
| `resolveReplyDirectives` | 解析消息中的指令 |
| `handleInlineActions`    | 处理内联操作     |
| `clearInlineDirectives`  | 清除内联指令     |

### Agent 执行

| 函数               | 用途                   |
| ------------------ | ---------------------- |
| `runPreparedReply` | 实际执行 Agent（核心） |

### 预处理

| 函数                              | 用途        |
| --------------------------------- | ----------- |
| `applyMediaUnderstandingIfNeeded` | 图片理解    |
| `applyLinkUnderstandingIfNeeded`  | 链接理解    |
| `emitPreAgentMessageHooks`        | 预处理 Hook |

---

## 主要副作用

| 副作用类型        | 具体操作                  | 影响                         |
| ----------------- | ------------------------- | ---------------------------- |
| **Workspace**     | `ensureAgentWorkspace`    | 创建/更新工作目录            |
| **Session Store** | `initSessionState`        | 读取/更新 session-store.json |
| **媒体暂存**      | `stageSandboxMedia`       | 将媒体保存到 workspace       |
| **Hook**          | `runBeforeAgentReply`     | Plugin 可能执行操作          |
| **Reset**         | `applyResetModelOverride` | 可能清空对话历史             |

---

## 与 dispatchReplyFromConfig 的关系

### 职责分工

```
┌─────────────────────────────────────────────────────┐
│          dispatchReplyFromConfig                    │
│          (协调层)                                    │
├─────────────────────────────────────────────────────┤
│  负责:                                              │
│  ├─ 去重管理                                        │
│  ├─ 路由决策                                        │
│  ├─ Plugin 绑定处理                                 │
│  ├─ 发送策略                                        │
│  ├─ 回复发送控制                                    │
│  └─ TTS 处理                                        │
│                                                     │
│  不负责:                                            │
│  ├─ 模型选择                                        │
│  ├─ 指令解析                                        │
│  ├─ 会话状态                                        │
│  └─ Agent 执行                                      │
└─────────────────────┬───────────────────────────────┘
                      │ 调用
                      ▼
┌─────────────────────────────────────────────────────┐
│          getReplyFromConfig                         │
│          (执行层)                                    │
├─────────────────────────────────────────────────────┤
│  负责:                                              │
│  ├─ 配置解析                                        │
│  ├─ 模型选择（多层 override）                       │
│  ├─ 会话状态初始化                                  │
│  ├─ 指令解析                                        │
│  ├─ 预处理（媒体/链接理解）                         │
│  └─ 调用 runPreparedReply                           │
│                                                     │
│  不负责:                                            │
│  ├─ 去重                                            │
│  ├─ 发送回复                                        │
│  ├─ TTS                                             │
│  └─ 路由                                            │
└─────────────────────────────────────────────────────┘
```

### 回调注入

`dispatchReplyFromConfig` 通过 `opts` 注入回调，控制回复发送：

```typescript
// dispatchReplyFromConfig 调用 getReplyFromConfig 时注入回调
const replyResult = await getReplyFromConfig(ctx, {
  ...opts,

  // 这些回调让 dispatchReplyFromConfig 控制发送
  onBlockReply: (payload) => {
    // TTS + 规范化 + 发送
    if (shouldRouteToOriginating) {
      routeReplyToOriginating(payload);
    } else {
      dispatcher.sendBlockReply(payload);
    }
  },
  onToolResult: (payload) => {
    // 类似处理
  },
  ...
});
```

**设计优势**:

- `getReplyFromConfig` 只关心"生成回复"
- `dispatchReplyFromConfig` 控制"怎么发送"
- 职责清晰，便于测试

---

## 配置驱动行为

### 模型选择配置

```yaml
# 配置示例
agents:
  defaults:
    provider: openai
    model: gpt-4o

    # Heartbeat 模型覆盖
    heartbeat:
      model: openai/gpt-4o-mini

    # 思考级别默认
    thinkLevel: medium
    verboseDefault: off

# 渠道模型覆盖
channels:
  modelByChannel:
    slack:
      default: anthropic/claude-opus-4
    telegram:
      groups:
        default: openai/gpt-4o-mini
```

### Workspace 配置

```yaml
# Workspace 配置
agents:
  defaults:
    workspace: ~/.openclaw/agents/default

    # 是否跳过 bootstrap
    skipBootstrap: false
    skipOptionalBootstrapFiles: false

    # 超时
    timeoutSeconds: 300
```

### Typing 配置

```yaml
# Typing 配置
agents:
  defaults:
    typingIntervalSeconds: 6

session:
  typingIntervalSeconds: 6
```

---

## 设计要点

### 1. 分层职责

- **getReplyFromConfig**: 准备执行环境
- **runPreparedReply**: 执行 Agent

### 2. 多层 Override 设计

优先级从高到低，确保灵活性：

- Heartbeat → 特定场景
- Session → 用户临时调整
- Channel → 渠道特性
- Default → 兜底配置

### 3. Hook 拦截点

`before_agent_reply` 提供最后拦截机会：

- Plugin 可以完全接管
- 或返回自定义回复

### 4. 预处理增强

- 媒体理解让 Agent 能"看图"
- 链接理解让 Agent 能"读懂 URL"

---

## 相关参考

- [dispatch-reply-from-config.md](./dispatch-reply-from-config.md) - 协调层文档
- [session-management.md](./session-management.md) - 会话管理
- [architecture.md](./architecture.md) - 整体架构
