# `runPreparedReply` 函数技术文档

## 函数签名

```typescript
export async function runPreparedReply(
  params: RunPreparedReplyParams,
): Promise<ReplyPayload | ReplyPayload[] | undefined>;
```

## 位置

- **文件**: `src/auto-reply/reply/get-reply-run.ts`
- **起始行**: 343
- **结束行**: 1059
- **总行数**: **717 行**

---

## 一句话理解

> **这是 Agent 执行的"总引擎"——负责构建 Prompt、管理会话状态、处理队列，最终调用模型。**

---

## 在消息处理流程中的位置

```
消息处理流程四层架构:

┌─────────────────────────────────────────────────────┐
│    协调层: dispatchReplyFromConfig                   │  ← 决策、协调、监控
│    (去重、路由、Hook、策略)                           │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│    准备层: getReplyFromConfig                        │  ← 配置、模型选择
│    (模型选择、会话初始化、指令解析)                   │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│    执行层: runPreparedReply                          │  ← 本函数
│    (Prompt构建、队列管理、状态管理)                   │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│    运行层: runReplyAgent                             │  ← 实际模型调用
│    (流式输出、工具执行、回复生成)                     │
└─────────────────────────────────────────────────────┘
```

**通俗理解**:

- `dispatchReplyFromConfig` = 指挥官（决定要不要执行）
- `getReplyFromConfig` = 参谋长（准备执行环境）
- `runPreparedReply` = 工兵（准备 Prompt、管理状态）
- `runReplyAgent` = 前锋（真正调用模型）

---

## 输入参数（RunPreparedReplyParams）

### 参数分类

| 分类         | 主要参数                                        | 用途                     |
| ------------ | ----------------------------------------------- | ------------------------ |
| **上下文**   | `ctx`, `sessionCtx`                             | 消息和会话上下文         |
| **配置**     | `cfg`, `agentCfg`, `sessionCfg`                 | 各种配置                 |
| **模型**     | `provider`, `model`, `modelState`               | 模型信息                 |
| **指令**     | `directives`, `resolvedThinkLevel`, ...         | 解析后的指令             |
| **会话**     | `sessionKey`, `sessionEntry`, `sessionStore`    | 会话状态                 |
| **工作空间** | `workspaceDir`, `agentDir`                      | 文件路径                 |
| **队列**     | `perMessageQueueMode`, `perMessageQueueOptions` | 队列控制                 |
| **流式**     | `blockStreamingEnabled`, `blockReplyChunking`   | 流式控制                 |
| **回调**     | `opts`, `typing`                                | 回调函数和 Typing 控制器 |

### 核心参数详解

```
┌─────────────────────────────────────────────────────┐
│              核心参数说明                            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ctx: MsgContext                                    │
│  ├─ 消息内容、发送者信息                            │
│  ├─ Body, CommandBody, RawBody                     │
│  └─ Provider, Surface, ChatType                   │
│                                                     │
│  sessionCtx: TemplateContext                        │
│  ├─ 会话模板上下文                                  │
│  └─ 用于构建 Prompt                                │
│                                                     │
│  provider + model                                   │
│  ├─ 决定用什么模型                                  │
│  └─ 影响支持的特性（thinking/reasoning 等）        │
│                                                     │
│  directives                                         │
│  ├─ 解析后的用户指令                                │
│  └─ /think, /verbose, /elevated 等                │
│                                                     │
│  sessionKey + sessionEntry                          │
│  ├─ 会话唯一标识                                    │
│  ├─ 会话持久化状态                                  │
│  └─ 存储在 session-store.json                     │
│                                                     │
│  workspaceDir                                       │
│  ├─ Agent 工作目录                                  │
│  └─ 存放 transcript、文件等                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 输出结果

| 类型             | 含义     | 场景                   |
| ---------------- | -------- | ---------------------- |
| `ReplyPayload`   | 单条回复 | 正常回复               |
| `ReplyPayload[]` | 多条回复 | 特殊场景               |
| `undefined`      | 无回复   | 被拦截/空消息/队列丢弃 |

---

## 整体流程图

```
runPreparedReply
    │
    ├─► [上下文解析] ──────────────────────┐
    │   ├─ 会话上下文 (promptSessionCtx)   │
    │   ├─ 群聊/私聊判断                   │
    │   ├─ Silent Reply 设置               │
    │   └─ Typing 模式                     │
    │                                      │
    ├─► [Prompt 构建] ─────────────────────┤
    │   ├─ System Prompt 部分:            │
    │   │  ├─ inboundMetaPrompt           │
    │   │  ├─ directChatContext           │
    │   │  ├─ groupChatContext            │
    │   │  ├─ groupIntro                  │
    │   │  ├─ groupSystemPrompt           │
    │   │  └─ execOverrideHint            │
    │   │                                  │
    │   ├─ User Prompt 部分:              │
    │   │  ├─ baseBody (消息体)           │
    │   │  ├─ inboundUserContext          │
    │   │  ├─ threadContextNote           │
    │   │  └─ systemEventBlocks           │
    │   │                                  │
    │   └─ Reset 特殊处理:                │
    │      ├─ bareResetPromptState        │
    │      └─ startupContextPrelude       │
    │                                      │
    ├─► [空消息检查] ──────────────────────┤
    │   └─ 无内容 → 返回提示 [退出点 1]    │
    │                                      │
    ├─► [Think Level 处理] ───────────────┤
    │   ├─ 检查模型是否支持               │
    │   ├─ 不支持 → fallback 或报错       │
    │   └─ [退出点 2] 报错返回             │
    │                                      │
    ├─► [会话状态准备] ────────────────────┤
    │   ├─ sessionId                       │
    │   ├─ sessionFile                     │
    │   └─ skillsSnapshot                  │
    │                                      │
    ├─► [队列处理] ────────────────────────┤
    │   ├─ 队列模式解析                    │
    │   ├─ Interrupt 模式 → 中断现有运行  │
    │   ├─ Steer 模式 → Steering           │
    │   ├─ Followup 模式 → 排队等待       │
    │   └─ [多个退出点]                    │
    │                                      │
    ├─► [构建 FollowupRun] ────────────────┤
    │   └─ 包含所有运行参数的结构体        │
    │                                      │
    └─► [runReplyAgent] ───────────────────┘
        │
        └─► 实际调用模型
```

---

## 详细处理步骤

### Phase 1: 上下文解析

**目的**: 理解"这是什么场景"——群聊/私聊/Heartbeat/Reset。

```
┌─────────────────────────────────────────────────────┐
│              上下文解析                              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. promptSessionCtx                                │
│     ├─ 从 sessionCtx 派生                          │
│     ├─ 处理 System Event 场景                      │
│     └─ 补充持久化的会话信息                        │
│                                                     │
│  2. 聊天类型判断                                    │
│     ├─ isGroupChat = "group" | "channel"           │
│     ├─ isDirectChat = "direct" | "dm"              │
│     └─ 影响后续 Prompt 构建                        │
│                                                     │
│  3. Silent Reply 设置                               │
│     ├─ resolveSilentReplySettings                  │
│     ├─ 控制 Agent 是否可以"静默回复"               │
│     └─ 即不发送任何可见内容                        │
│                                                     │
│  4. Typing 模式                                     │
│     ├─ resolveTypingMode                           │
│     ├─ "none" | "single" | "continuous"           │
│     └─ 控制"正在输入"提示行为                      │
│                                                     │
│  5. 特殊场景标记                                    │
│     ├─ isFirstTurnInSession                        │
│     ├─ isHeartbeat                                 │
│     ├─ resetTriggered                              │
│     └─ softResetTriggered                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 2: Prompt 构建（最核心）

**目的**: 构建 System Prompt 和 User Prompt。

#### System Prompt 组成

```
┌─────────────────────────────────────────────────────┐
│              System Prompt 组成                      │
│              (extraSystemPromptParts)                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  inboundMetaPrompt                           │   │
│  │  ├─ 入站消息元信息                           │   │
│  │  ├─ 时间戳、消息ID 等                        │   │
│  │  └─ 格式化提示                              │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  directChatContext                           │   │
│  │  ├─ 私聊场景专用                             │   │
│  │  ├─ Silent Reply 指导                       │   │
│  │  └─ 回复行为说明                            │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  groupChatContext                            │   │
│  │  ├─ 群聊场景专用                             │   │
│  │  ├─ 提及/未提及行为                         │   │
│  │  └─ 群组回复指导                            │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  groupIntro                                  │   │
│  │  ├─ 群组行为介绍                             │   │
│  │  ├─ 只在首次/激活时注入                     │   │
│  │  └─ 激活模式说明                            │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  groupSystemPrompt                           │   │
│  │  ├─ 群组自定义 System Prompt                │   │
│  │  └─ 来自 sessionCtx.GroupSystemPrompt      │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  buildExecOverridePromptHint                 │   │
│  │  ├─ Exec 会话状态                            │   │
│  │  ├─ host/security/ask/node 设置            │   │
│  │  └─ Elevated 级别说明                       │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  最终:                                              │
│  extraSystemPrompt = parts.join("\n\n")            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### User Prompt 组成

```
┌─────────────────────────────────────────────────────┐
│              User Prompt 组成                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  inboundUserContext                          │   │
│  │  ├─ 入站用户上下文前缀                       │   │
│  │  ├─ 发送者信息                               │   │
│  │  └─ 时间、渠道等                            │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  baseBodyFinal                               │   │
│  │  ├─ 用户消息内容                             │   │
│  │  ├─ 去除指令后的纯文本                       │   │
│  │  └─ 或 Reset 特殊内容                       │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  threadContextNote                           │   │
│  │  ├─ Thread 历史/起始消息                     │   │
│  │  └─ 仅在 Thread 场景                         │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  systemEventBlocks                           │   │
│  │  ├─ 系统事件                                 │   │
│  │  ├─ 从队列中取出                             │   │
│  │  └─ 格式化为 System: 前缀                   │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  最终:                                              │
│  prefixedCommandBody = buildReplyPromptBodies(...) │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### Reset 场景特殊处理

```
┌─────────────────────────────────────────────────────┐
│              Reset 场景处理                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  检测:                                              │
│  ├─ /new 或 /reset 命令                            │
│  ├─ 新会话 + 空消息体                              │
│  └─ softResetTriggered                             │
│                                                     │
│  处理:                                              │
│  ├─ resolveBareSessionResetPromptState            │
│  │  ├─ 检查 workspace 是否有 startup 文件         │
│  │  └─ 决定是否使用 bootstrap prompt             │
│  │                                                 │
│  ├─ buildSessionStartupContextPrelude             │
│  │  ├─ 构建启动上下文                             │
│  │  └─ 从 workspace 读取                         │
│  │                                                 │
│  └─ baseBodyFinal                                  │
│     ├─ 可能是空（纯 Reset）                       │
│     ├─ 可能是 bootstrap prompt                    │
│     └─ 可能带 softResetTail                       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 3: 空消息检查

**目的**: 如果用户没发送有效内容，提前返回提示。

```
┌─────────────────────────────────────────────────────┐
│              空消息检查                              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  检查条件:                                          │
│  ├─ hasUserBody = false                            │
│  │  ├─ baseBodyFinal.trim().length === 0          │
│  │  ├─ softResetTail.length === 0                 │
│  │  └─ !hasInboundHistoryBody                     │
│  │                                                 │
│  └─ hasMediaAttachment = false                    │
│     ├─ !hasInboundMedia                            │
│     └─ (opts.images?.length ?? 0) === 0           │
│                                                     │
│  处理:                                              │
│  ├─ await typing.onReplyStart()                   │
│  ├─ typing.cleanup()                               │
│  └─ 返回提示消息 [退出点 1]                         │
│     "I didn't receive any text..."                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 4: Think Level 处理

**目的**: 确保模型支持请求的 Think Level，否则 fallback 或报错。

```
┌─────────────────────────────────────────────────────┐
│              Think Level 处理                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  流程:                                              │
│                                                     │
│  1. 从消息体提取首词 Think Hint                     │
│     ├─ 用户消息开头可能是 "low" "medium" "high"   │
│     └─ 解析为 resolvedThinkLevel                   │
│                                                     │
│  2. 检查模型是否支持                                │
│     isThinkingLevelSupported({                     │
│       provider, model, level, catalog              │
│     })                                              │
│                                                     │
│  3. 处理结果:                                       │
│     ├─ 支持 → 继续                                 │
│     │                                               │
│     ├─ 不支持 + 用户显式指定                        │
│     │  ├─ 返回错误消息 [退出点 2]                  │
│     │  └─ "Thinking level X not supported..."     │
│     │                                               │
│     └─ 不支持 + 非显式指定                          │
│     │  ├─ fallback 到支持的级别                   │
│     │  └─ 更新 sessionEntry                       │
│     │  └─ 写入 session-store                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 5: 会话状态准备

**目的**: 准备会话运行所需的状态信息。

```
┌─────────────────────────────────────────────────────┐
│              会话状态准备                            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. sessionId                                       │
│     ├─ sessionId ?? crypto.randomUUID()            │
│     └─ 用于 transcript 文件命名                    │
│                                                     │
│  2. sessionFile                                     │
│     ├─ resolveSessionFilePath                     │
│     └─ transcript JSON 文件路径                    │
│                                                     │
│  3. skillsSnapshot                                  │
│     ├─ ensureSkillSnapshot                         │
│     ├─ 当前启用的技能列表                          │
│     └─ 写入 sessionEntry                          │
│                                                     │
│  4. currentSystemSent                               │
│     ├─ 是否已发送 System Prompt                   │
│     └─ 影响是否注入 groupIntro                    │
│                                                     │
│  5. preparedSessionState                            │
│     ├─ sessionEntry                                │
│     ├─ sessionId                                   │
│     └─ sessionFile                                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 6: 队列处理（复杂）

**目的**: 处理消息队列模式——中断、Steering、Followup。

#### 队列模式说明

```
┌─────────────────────────────────────────────────────┐
│              队列模式说明                            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  模式:                                              │
│                                                     │
│  ┌─────────────────┐                              │
│  │  interrupt      │  中断现有运行                 │
│  │                 │  ├─ /new /reset 触发         │
│  │                 │  ├─ abortEmbeddedPiRun       │
│  │                 │  └─ clearCommandLane         │
│  └─────────────────┘                              │
│                                                     │
│  ┌─────────────────┐                              │
│  │  collect        │  收集消息，稍后执行           │
│  │                 │  ├─ 多条消息合并             │
│  │                 │  └─ debounce 等待           │
│  └─────────────────┘                              │
│                                                     │
│  ┌─────────────────┐                              │
│  │  followup       │  当前运行结束后执行           │
│  │                 │  ├─ 排队等待                 │
│  │                 │  └─ waitForEmbeddedPiRunEnd │
│  └─────────────────┘                              │
│                                                     │
│  ┌─────────────────┐                              │
│  │  steer          │  Steering 模式               │
│  │                 │  ├─ 实时注入消息             │
│  │                 │  ├─ 不中断运行               │
│  │                 │  └─ queueEmbeddedPiMessage  │
│  └─────────────────┘                              │
│                                                     │
│  ┌─────────────────┐                              │
│  │  steer-backlog  │  Steering + backlog          │
│  │                 │  ├─ Steering 优先            │
│  │                 │  └─ backlog 作为 followup   │
│  └─────────────────┘                              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### 队列处理流程

```
┌─────────────────────────────────────────────────────┐
│              队列处理流程                            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. 解析队列设置                                    │
│     resolveQueueSettings({                         │
│       cfg, channel, sessionEntry,                  │
│       inlineMode, inlineOptions                    │
│     })                                              │
│                                                     │
│  2. 检查当前运行状态                                │
│     ├─ piRuntime.resolveActiveEmbeddedRunSessionId │
│     ├─ isEmbeddedPiRunActive                      │
│     └─ isEmbeddedPiRunStreaming                   │
│                                                     │
│  3. Interrupt 模式处理                              │
│     if (activeRunQueueMode === "interrupt") {      │
│       ├─ clearCommandLane(sessionLaneKey)         │
│       ├─ abortEmbeddedPiRun(activeSessionId)      │
│       └─ 终止现有运行                              │
│     }                                               │
│                                                     │
│  4. 队列状态处理                                    │
│     resolvePreparedReplyQueueState({               │
│       activeRunQueueAction,                        │
│       activeSessionId,                             │
│       abortActiveRun,                              │
│       waitForActiveRunEnd,                         │
│       refreshPreparedState,                        │
│     })                                              │
│                                                     │
│     结果:                                           │
│     ├─ kind === "reply" → 返回 [退出点 3]         │
│     └─ kind === "continue" → 继续准备             │
│                                                     │
│  5. Steering 模式处理                               │
│     if (shouldSteer && isStreaming) {              │
│       ├─ queueEmbeddedPiMessage                   │
│       ├─ 实时注入消息                              │
│       └─ [退出点 4] 返回 undefined                 │
│     }                                               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 7: 构建 FollowupRun

**目的**: 把所有运行参数打包成一个结构体，传给 `runReplyAgent`。

```
┌─────────────────────────────────────────────────────┐
│              FollowupRun 结构                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  followupRun = {                                    │
│                                                     │
│    // Prompt 相关                                   │
│    prompt: queuedBody,                              │
│    transcriptPrompt: transcriptCommandBody,        │
│                                                     │
│    // 消息标识                                      │
│    messageId: ctx.MessageSidFull,                  │
│    summaryLine: baseBodyTrimmedRaw,                │
│    enqueuedAt: Date.now(),                          │
│                                                     │
│    // 媒体                                          │
│    images: opts.images,                             │
│    imageOrder: opts.imageOrder,                     │
│                                                     │
│    // 原始渠道信息（用于回复路由）                  │
│    originatingChannel,                              │
│    originatingTo,                                   │
│    originatingAccountId,                            │
│    originatingThreadId,                             │
│                                                     │
│    // 运行参数                                      │
│    run: {                                            │
│      agentId,                                        │
│      sessionId,                                      │
│      sessionKey,                                     │
│      sessionFile,                                    │
│      workspaceDir,                                   │
│                                                     │
│      provider,                                       │
│      model,                                          │
│      thinkLevel,                                     │
│      verboseLevel,                                   │
│      reasoningLevel,                                 │
│      elevatedLevel,                                  │
│                                                     │
│      timeoutMs,                                      │
│      blockReplyBreak,                               │
│                                                     │
│      authProfileId,                                  │
│      skillsSnapshot,                                 │
│      extraSystemPrompt,                             │
│                                                     │
│      bashElevated: {                                 │
│        enabled, allowed, defaultLevel,             │
│        fullAccessAvailable                          │
│      },                                              │
│                                                     │
│      // ... 更多参数                                │
│    }                                                 │
│  }                                                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Phase 8: 调用 runReplyAgent

**目的**: 真正执行 Agent，调用模型 API。

```
┌─────────────────────────────────────────────────────┐
│              runReplyAgent 调用                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  return runReplyAgent({                             │
│    commandBody: prefixedCommandBody,               │
│    transcriptCommandBody,                           │
│    followupRun,                                      │
│                                                     │
│    queueKey,                                         │
│    resolvedQueue,                                    │
│    shouldSteer,                                      │
│    shouldFollowup,                                   │
│                                                     │
│    isActive,                                         │
│    isStreaming,                                      │
│    isRunActive: () => boolean,                      │
│                                                     │
│    opts,                                             │
│    typing,                                           │
│                                                     │
│    sessionEntry,                                     │
│    sessionStore,                                     │
│    sessionKey,                                       │
│                                                     │
│    blockStreamingEnabled,                            │
│    blockReplyChunking,                               │
│                                                     │
│    typingMode,                                       │
│    resetTriggered,                                   │
│  })                                                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 退出点速览

| 退出点 | 在哪一步 | 什么情况                       | 返回什么     |
| ------ | -------- | ------------------------------ | ------------ |
| 1      | Phase 3  | 空消息（无内容无媒体）         | 提示消息     |
| 2      | Phase 4  | Think Level 不支持（显式指定） | 错误消息     |
| 3      | Phase 6  | 队列处理返回 reply             | 队列状态消息 |
| 4      | Phase 6  | Steering 成功（非 followup）   | undefined    |

---

## 调用链简图

```
runPreparedReply
    │
    ├─► resolvePromptSessionContextForSystemEvent
    ├─► resolveSilentReplySettings
    ├─► resolveTypingMode
    │
    ├─► [构建 System Prompt]
    │   ├─► buildInboundMetaSystemPrompt
    │   ├─► buildDirectChatContext / buildGroupChatContext
    │   ├─► buildGroupIntro
    │   ├─► buildExecOverridePromptHint
    │
    ├─► [构建 User Prompt]
    │   ├─► resolveBareSessionResetPromptState (Reset场景)
    │   ├─► buildSessionStartupContextPrelude (Reset场景)
    │   ├─► applySessionHints
    │   ├─► drainFormattedSystemEvents
    │   ├─► buildReplyPromptBodies
    │
    ├─► [空消息检查] → [退出点 1]
    │
    ├─► [Think Level 处理]
    │   ├─► modelState.resolveThinkingCatalog
    │   ├─► isThinkingLevelSupported
    │   ├─► [不支持显式指定] → [退出点 2]
    │   └─► resolveSupportedThinkingLevel (fallback)
    │
    ├─► [会话状态准备]
    │   ├─► ensureSkillSnapshot
    │   ├─► resolveSessionFilePath
    │
    ├─► [队列处理]
    │   ├─► resolveQueueSettings
    │   ├─► resolveActiveRunQueueAction
    │   ├─► clearCommandLane (interrupt)
    │   ├─► abortEmbeddedPiRun (interrupt)
    │   ├─► resolvePreparedReplyQueueState → [退出点 3]
    │   ├─► queueEmbeddedPiMessage (steer) → [退出点 4]
    │
    ├─► [构建 FollowupRun]
    │   ├─► resolveOriginMessageProvider
    │   ├─► resolveFastModeState
    │   ├─► isReasoningTagProvider
    │
    └─► runReplyAgent
        │
        └─► 实际模型调用
```

---

## 关键依赖函数

### Prompt 构建

| 函数                           | 用途                         |
| ------------------------------ | ---------------------------- |
| `buildInboundMetaSystemPrompt` | 构建入站元信息 System Prompt |
| `buildDirectChatContext`       | 构建私聊上下文               |
| `buildGroupChatContext`        | 构建群聊上下文               |
| `buildGroupIntro`              | 构建群组介绍                 |
| `buildExecOverridePromptHint`  | 构建 Exec 状态提示           |
| `buildReplyPromptBodies`       | 构建回复 Prompt 主体         |
| `applySessionHints`            | 应用会话提示（abort 等）     |

### 会话管理

| 函数                                        | 用途                         |
| ------------------------------------------- | ---------------------------- |
| `resolvePromptSessionContextForSystemEvent` | 解析 System Event 会话上下文 |
| `ensureSkillSnapshot`                       | 确保技能快照                 |
| `resolveSessionFilePath`                    | 解析 transcript 文件路径     |

### 队列管理

| 函数                             | 用途              |
| -------------------------------- | ----------------- |
| `resolveQueueSettings`           | 解析队列设置      |
| `resolveActiveRunQueueAction`    | 解析队列动作      |
| `resolvePreparedReplyQueueState` | 解析队列状态      |
| `clearCommandLane`               | 清空命令通道      |
| `queueEmbeddedPiMessage`         | Steering 消息注入 |

### Think Level

| 函数                                | 用途                       |
| ----------------------------------- | -------------------------- |
| `modelState.resolveThinkingCatalog` | 解析 thinking 配置         |
| `isThinkingLevelSupported`          | 检查是否支持               |
| `resolveSupportedThinkingLevel`     | 解析支持的级别（fallback） |

### Reset 处理

| 函数                                 | 用途                   |
| ------------------------------------ | ---------------------- |
| `resolveBareSessionResetPromptState` | 解析 Reset Prompt 状态 |
| `buildSessionStartupContextPrelude`  | 构建启动上下文         |

### 最终执行

| 函数            | 用途                   |
| --------------- | ---------------------- |
| `runReplyAgent` | 实际执行 Agent（核心） |

---

## 主要副作用

| 副作用类型               | 具体操作                          | 影响                      |
| ------------------------ | --------------------------------- | ------------------------- |
| **Session Store**        | `updateSessionStore`              | 更新 session-store.json   |
| **Think Level Fallback** | 修改 `sessionEntry.thinkingLevel` | 持久化 fallback 后的级别  |
| **队列中断**             | `abortEmbeddedPiRun`              | 终止正在运行的 Agent      |
| **Steering**             | `queueEmbeddedPiMessage`          | 向运行中的 Agent 注入消息 |
| **Typing**               | `typing.onReplyStart`             | 发送"正在输入"提示        |

---

## 设计要点

### 1. Prompt 构建分层

```
System Prompt 分层:
├─ 基础层: inboundMetaPrompt (消息元信息)
├─ 场景层: directChatContext / groupChatContext
├─ 激活层: groupIntro (首次激活)
├─ 自定义层: groupSystemPrompt
├─ 状态层: execOverrideHint
└─ 动态层: systemEventBlocks (系统事件)
```

### 2. Reset 场景特殊处理

Reset 有三种情况：

- `/new` - 新会话，清空历史
- `/reset` - 重置会话，清空历史
- `softReset` - 软重置，保留部分状态

每种都有不同的 Prompt 处理逻辑。

### 3. 队列模式灵活控制

不同队列模式适应不同场景：

- `interrupt`: 用户主动中断（命令）
- `collect`: 快速连续消息合并
- `followup`: 等待当前完成
- `steer`: 实时注入不中断

### 4. Think Level 自动 Fallback

用户请求的级别可能不被支持：

- 显式指定 → 报错提示
- 非显式指定 → 自动 fallback 到支持的级别

### 5. Steering vs Interrupt

- Steering: 实时注入，不中断运行（适合对话中补充信息）
- Interrupt: 完全中断，重新开始（适合命令场景）

---

## 与其他函数的关系

### 上游：getReplyFromConfig

```
getReplyFromConfig 准备:
├─ 配置解析
├─ 模型选择
├─ 会话初始化
├─ 指令解析
└─ 预处理

然后调用:
runPreparedReply(准备好的参数)
```

### 下游：runReplyAgent

```
runPreparedReply 准备:
├─ Prompt 构建
├─ 会话状态
├─ 队列处理
└─ FollowupRun

然后调用:
runReplyAgent(执行参数)
```

---

## 相关参考

- [dispatch-reply-from-config.md](./dispatch-reply-from-config.md) - 协调层
- [get-reply-from-config.md](./get-reply-from-config.md) - 准备层
- [session-management.md](./session-management.md) - 会话管理
- [architecture.md](./architecture.md) - 整体架构
