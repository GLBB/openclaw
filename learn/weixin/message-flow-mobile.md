# 一条消息的奇幻旅程：从飞书到 AI 回复的 16 步之旅

> 基于 OpenClaw v2026.5.3 源码分析（2026-05-05）

你在飞书群里 @ 了 OpenClaw Bot，发送了"帮我分析一下这段代码"。

短短3秒后，一条流畅的 AI 回复出现在你的屏幕上。

这3秒里，你的消息经历了一场跨越16个步骤的奇幻旅程。让我们跟随它，探索从发送到回复的完整路径。

---

## 第一章：穿越防火墙（步骤 1-4）

### 第1站：交通枢纽

飞书服务器推送消息到 OpenClaw，有两种方式：

| 方式      | 说明                 | 适用场景 |
| --------- | -------------------- | -------- |
| WebSocket | 专线直达，实时推送   | 内网首选 |
| Webhook   | HTTP回调，邮递员模式 | 公网简单 |

### 第2站：安检关卡

消息进入后遇到三道安检：

(1) **防抖门** - 3秒窗口内合并快速输入

> "帮我" → "帮我分析" → "帮我分析代码" → 合并成一条

(2) **去重门** - 检查 message_id，丢弃重复推送

(3) **排队门** - 同一群的消息按顺序处理

### 第3站：身份验证

| 消息类型 | 检查项                           | 不满足时               |
| -------- | -------------------------------- | ---------------------- |
| 群聊     | 群白名单、发送者白名单、是否@Bot | 发送提示消息，旅程结束 |
| 私聊     | 用户已配对、用户白名单           | 发送提示消息，旅程结束 |

权限不足 → 发送提示消息，旅程结束

### 第4站：任务编排

```
runChannelTurn
  ├── 解析 Agent 路由（哪个 Agent 实例？）
  ├── 解析 Session 绑定（哪个对话？）
  ├── 构建消息上下文
  └── 启动回复生成流程
```

---

## 第二章：四层接力（步骤 5-8）

消息进入核心区，开始四层接力赛。

### 第一棒：分发协调员

**dispatchReplyFromConfig** 做三件事：

(1) **紧急出口** - `/stop`、`/help` 等命令直接返回预设回复

(2) **插件插队** - 让插件介入消息流程

> `message_received` - 偷看消息（日志、统计）
> `before_dispatch` - 拦截消息（敏感词过滤）
> `reply_dispatch` - 修改分发（多渠道发送）

(3) **工具授权** - 决定 AI 能用什么工具

### 第二棒：回复准备员

**getReplyFromConfig** 准备执行环境：

| 任务       | 说明                                                                       |
| ---------- | -------------------------------------------------------------------------- |
| 定位角色   | 从 sessionKey 解析出 agentId，如 "agent:main:feishu:group:oc_xxx" → "main" |
| 模型路由   | 三层解析：Agent默认配置 → 渠道覆盖 → 存储覆盖，最终确定具体模型            |
| 准备战场   | 创建 workspaceDir 目录，写入 bootstrap 文件注入项目知识                    |
| 理解内容   | 图片→图片理解，链接→链接理解，提取关键信息加入对话                         |
| 解析命令   | `/think` 启用深度思考，`/model` 切换模型，`/reset` 重置会话                |
| 初始化会话 | 读取 session 文件，加载历史消息，构建 SessionEntry                         |

### 第三棒：执行准备员

**runPreparedReply** 设置执行参数：

| 任务         | 说明                                                                                     |
| ------------ | ---------------------------------------------------------------------------------------- |
| 场合设定     | 群聊："低调参与，被@才回复"；私聊："可以更详细"。组装成 extraSystemPrompt                |
| 消息正文     | 提取用户说的核心内容，处理引用消息（用户回复了谁），生成 prefixedCommandBody             |
| 队列状态     | 检查是否有其他消息正在处理，决定排队策略：run直接执行、steer注入对话、followup完成后继续 |
| Thinking级别 | 用户 `/think high` 深度思考，模型不支持时自动降级                                        |
| 静默规则     | AI能否不发消息？（见下表）                                                               |
| 技能快照     | 用户配置了 skills 时加载技能列表，传给 AI 作为可用能力                                   |
| 媒体附件     | 图片、音频暂存到 sandbox，生成路径引用，AI 可以"看到"这些文件                            |

**静默规则说明：**

通过 `silentToken`（如 "NO_REPLY"）实现：AI回复这个标记表示不需要发送消息。

| 场景 | policy   | rewrite | AI行为                      | 用户看到                              |
| ---- | -------- | ------- | --------------------------- | ------------------------------------- |
| 群聊 | allow    | false   | 可回复 `NO_REPLY` 不发消息  | 无消息，避免刷屏                      |
| 私聊 | disallow | true    | 不能用 `NO_REPLY`，必须回复 | 简短文本："Nothing to add right now." |
| 内部 | allow    | false   | 可回复 `NO_REPLY`           | 无消息                                |

**典型使用场景：**

- 群聊中AI只是监听消息，不需要回复 → 回复 `NO_REPLY` → 不发送
- AI执行工具后没有需要告诉用户的内容 → 回复 `NO_REPLY` → 不发送
- 私聊中AI没有实质内容要回复 → 发简短文本替代

### 第四棒：Agent 编排员

**runReplyAgent** 核心编排：

**(1) Steering检查** - 能否"搭便车"？

> AI正在回复+流式输出中 → 新消息注入现有对话
> 用户追加提问，AI继续回复（无需新建对话）

**(2) 队列决策** - 怎么处理这条消息？

| 决策             | 含义         | 场景             |
| ---------------- | ------------ | ---------------- |
| drop             | 丢弃         | 已有太多排队消息 |
| enqueue-followup | 排队等待     | 完成当前后再处理 |
| run-now          | 立即执行     | 当前无其他任务   |
| wait             | 等待当前完成 | 正在执行其他任务 |

**(3) 预压缩** - context window >80%？需要压缩

| 操作                   | 说明                   |
| ---------------------- | ---------------------- |
| runPreflightCompaction | 执行预压缩             |
| 保留                   | 关键决策和结果         |
| 删除                   | 冗余对话细节           |
| 目的                   | 确保对话不超出模型限制 |

**(4) 内存刷新** - 长期记忆需要保存？

> runMemoryFlushIfNeeded → 写入 MEMORY.md

**(5) 注册操作** - 回复任务生命周期管理

> 创建 ReplyOperation，跟踪一次回复生成任务的状态
> 状态流转：queued → preflight_compacting → memory_flushing → running → completed/failed/aborted
> 提供 abortSignal 支持取消，其他组件可监控执行进度

**(6) 核心执行** - runAgentTurnWithFallback → 进入执行层

**(7) 结果处理** - 组装回复

| 操作               | 说明                 |
| ------------------ | -------------------- |
| buildReplyPayloads | 构建回复payload      |
| 合并文本片段       | 拼接AI生成的各段文本 |
| 添加usage统计      | token使用量等        |
| 添加model信息      | 实际使用的模型       |
| 输出               | ReplyPayload         |

**关键决策：** 这一棒决定"压缩策略"、"模型fallback"、"生命周期管理"

---

## 第三章：核心执行（步骤 9-14）

### 第9站：模型降级站

**runAgentTurnWithFallback** - 模型 Fallback 机制

**候选模型来源：**
| 来源 | 说明 |
|------|------|
| 主模型 | 用户请求的模型，如 claude/claude-opus-4-7 |
| 第1备用 | openai/gpt-5.5，主模型失败时优先尝试 |
| 第2备用 | anthropic/claude-sonnet-4-6，第1备用也失败时尝试 |
| 第3备用 | openai/gpt-4o-mini，本地小模型，最后兜底 |

**为什么要有备用？**

- rate_limit（限流）
- overload（过载）
- billing（账单问题）
- timeout（超时）

**Auth Profile 检查：**

- 有认证配置？
- 所有 profile 都在 cooldown？
- cooldown = 刚失败过，需要等待

**错误类型：**
| 错误 | 说明 |
|------|------|
| rate_limit | API 限流，短时间内调用次数超限 |
| overloaded | 服务过载，模型服务高峰期 |
| billing | 账单问题，账户欠费或额度用尽 |
| timeout | 请求超时，网络问题或模型处理慢 |
| context_overflow | 对话太长，context window 超限 |
| auth_error | 认证失败，API Key 过期或配置错误 |
| unknown | 其他错误，未预期的异常情况 |

**特殊处理：**

- AbortError → 直接抛出（用户取消）
- FailoverError → 继续下一个候选

**FallbackSummaryError：** 包含所有尝试详情 + 最短 cooldown 过期时间

**用户看到的错误示例：**

> "Claude rate-limited, retry in 30s"
> "GPT overloaded, retry in 2m"
> "所有模型暂时不可用，请稍后重试"

### 第10站：Harness 选择站

**runEmbeddedPiAgent** - 选择执行引擎

**两种 Harness：**
| 类型 | priority | 说明 |
|------|----------|------|
| PI Harness | 0 | OpenClaw 内置引擎，支持所有 provider，runAttempt → runEmbeddedAttempt |
| Plugin Harness | 100+ | 自定义引擎，如 Codex App Server，只处理特定 provider，有独立实现 |

**执行环境准备：**
| 任务 | 说明 |
|------|------|
| Session Key | 确保下游组件都能拿到 sessionKey，用于识别对话 |
| Lane队列 | globalLane 全局任务队列 + sessionLane 会话任务队列，避免并发冲突 |
| Timeout | laneTaskTimeoutMs 超时限制，防止任务无限等待 |
| Workspace | resolveRunWorkspaceDir 解析工作目录，不存在时用 fallback |
| Runtime Plugins | ensureRuntimePluginsLoaded 确保插件在 Gateway 启动时已加载，未加载则补加载 |
| Hook Runner | getGlobalHookRunner() 获取全局 Hook 执行器，插件可在特定时机介入 |

**Hook 列表（插件可在特定时机介入）：**

| 类别            | Hook                 | 说明                       |
| --------------- | -------------------- | -------------------------- |
| 消息流程        | message_received     | 收到消息时（日志、统计）   |
| 消息流程        | before_dispatch      | 分发前（敏感词过滤）       |
| 消息流程        | reply_dispatch       | 回复分发（多渠道发送）     |
| 消息流程        | message_sending      | 发送前（格式转换）         |
| 消息流程        | message_sent         | 发送后（状态记录）         |
| Agent执行       | before_agent_start   | 启动前（注入额外 prompt）  |
| Agent执行       | before_model_resolve | 模型选择前（强制指定模型） |
| Agent执行       | before_prompt_build  | Prompt构建前（修改上下文） |
| Agent执行       | before_tool_call     | 工具调用前（安全检查）     |
| Agent执行       | after_tool_call      | 工具调用后（结果处理）     |
| Agent执行       | before_compaction    | 压缩前（保留关键信息）     |
| Agent执行       | agent_end            | Agent结束（清理资源）      |
| 会话生命周期    | session_start        | 会话开始（初始化）         |
| 会话生命周期    | session_end          | 会话结束（归档）           |
| Gateway生命周期 | gateway_start        | Gateway启动                |
| Gateway生命周期 | gateway_stop         | Gateway停止                |

### 第11站：Harness 选择与适配

**runAgentHarnessAttempt** - V2 适配

**选择逻辑：**

- pinned：用户指定
- forced_pi：配置强制 PI
- forced_plugin：配置强制插件
- auto_plugin：自动匹配
- auto_pi：无匹配用 PI

**V1 vs V2 接口对比：**
| 阶段 | V1 | V2 |
|------|----|----|
| prepare | 不支持 | 准备资源 |
| start | 不支持 | 初始化 Session |
| send | runAttempt() | 核心执行 |
| resolveOutcome | 不支持 | 结果分类 |
| cleanup | 不支持 | 清理资源 |
| resume | 不支持 | 中断恢复（可选） |

**设计动机**（来自 upstream commit #71722）：

| 目的                           | 说明                         |
| ------------------------------ | ---------------------------- |
| 统一 RuntimePlan               | PI + Plugin Harness 共享策略 |
| tools.normalize/logDiagnostics | 工具策略共享                 |
| transcript.resolvePolicy       | transcript 处理共享          |
| outcome.classifyRunResult      | fallback 分类统一            |
| resume                         | 支持中断恢复                 |
| handleToolCall                 | 自定义工具处理               |

核心目的：让 Codex 等 Plugin Harness 和 PI 行为一致

### 第12站：V2 生命周期执行

**runHarnessV2LifecycleAttempt** - 五阶段执行

| 阶段           | PI Harness        | 原生 V2           |
| -------------- | ----------------- | ----------------- |
| prepare        | 仅标记状态        | 构建 Prompt/Tools |
| start          | 仅标记状态        | 初始化 Session    |
| send           | ★调用 runAttempt  | 自定义执行        |
| resolveOutcome | classifyRunResult | 自定义分类        |
| cleanup        | 清理资源          | 自定义清理        |

### 第13站：核心执行站

**runEmbeddedAttempt** (~3700行核心代码)

> 从这里开始，进入 @mariozechner/pi-coding-agent 的世界

**执行阶段：**

| 步骤                                                     | 说明                                                                  |
| -------------------------------------------------------- | --------------------------------------------------------------------- |
| 初始化 workspace、sandbox、skills                        | 设置工作目录、安全沙箱、加载技能配置                                  |
| 工具准备 createOpenClawCodingTools + MCP + LSP           | 构建 OpenClaw 工具集、MCP 外部服务工具、LSP 代码补全工具              |
| Session创建 SessionManager.fromFile + createAgentSession | 加载 session 文件历史对话，创建 AgentSession 对象管理对话循环         |
| Prompt构建 buildEmbeddedSystemPrompt + Bootstrap         | 收集运行时信息，组装完整 System Prompt（几千字符）                    |
| 事件注册 subscribeEmbeddedPiSession                      | 注册 PI Agent 事件处理器：message_start/update/end、tool_execution 等 |
| streamFn Provider.streamCompletion                       | 注册 LLM 调用函数，PI Agent 内部会调用                                |
| 启动循环 activeSession.prompt()                          | 启动 PI Agent 对话循环，真正开始执行                                  |

**关键概念：**
| 概念 | 说明 |
|------|------|
| Bootstrap Files | 项目上下文注入：BOOTSTRAP.md、MEMORY.md、SOUL.md、USER.md 等，有三种模式（none/limited/full），受字符限制，注入到 systemPrompt |
| Cache Boundary | 缓存优化，Prompt分段以利用模型缓存 |
| transformProviderSystemPrompt | Provider特定的Prompt转换：让插件自定义systemPrompt格式，如GPT-5需要特殊的prompt overlay |

**OpenClaw vs PI Agent 职责：**
| 职责 | OpenClaw | PI Agent |
|------|----------|-----------|
| 工具创建 | createOpenClawCodingTools | 接收工具列表 |
| Session创建 | 调用 createAgentSession | 管理 Session |
| System Prompt | buildEmbeddedSystemPrompt | 接收 Prompt |
| 事件处理 | subscribeEmbeddedPiSession | 发送事件 |
| streamFn | Provider.streamCompletion | 内部调用 |
| 对话循环 | prompt() 启动 | while 循环管理 |

### 第14站：PI Agent 对话循环

```
while (!finished) {
  调用 streamFn() → Provider API

  收到 text？ → 立即发送 Block Reply
  收到 toolCall？ → executeToolCall() → 再次调用 streamFn()
  收到 finishReason？ → 结束循环
}
```

**流式 vs 阻塞：**
| 环节 | 类型 | 用户感知 |
|------|------|----------|
| 文本输出 | 流式 | "打字中" |
| 工具执行 | 阻塞 | "暂停" |
| 再次调用LLM | 流式 | "继续" |
| 最终回复 | 一次性 | 完整卡片 |

---

## 第四章：回复归途（步骤 15-16）

### 第15站：回复分发器

**ReplyDispatcher** 分发结果：

| 类型        | 说明                                 | 飞书显示                                      |
| ----------- | ------------------------------------ | --------------------------------------------- |
| Block Reply | 流式更新飞书卡片，实时显示生成过程   | 用户看到"正在打字..."                         |
| Tool Result | 工具执行结果，如 web_search 返回内容 | 飞书不显示，其他渠道可能显示                  |
| Final Reply | 最终回复，判断格式后发送             | 有代码块/表格→交互式卡片，普通文本→富文本消息 |

### 第16站：飞书送达

**sendMessageFeishu** 发送消息：

- 解析目标（chat_id / open_id）
- 构建飞书消息格式
- POST /im/v1/messages
- 返回 message_id → 推送给用户

---

## 终章：完整旅程图

```
飞书用户发送消息
        │
        ▼ 第一章
┌────────────────────┐
│ 1. 接收            │
│ 2. 防抖→去重→排队  │
│ 3. 权限检查        │
│ 4. 任务编排        │
└────────────────────┘
        │
        ▼ 第二章
┌────────────────────┐
│ 5. 分发协调        │
│ 6. 回复准备        │
│ 7. 执行准备        │
│ 8. Agent编排       │
└────────────────────┘
        │
        ▼ 第三章
┌────────────────────┐
│ 9. 模型Fallback    │
│ 10. Harness选择    │
│ 11-12. V2 Lifecycle│
│ 13. runEmbedded    │
│ 14. PI Agent Loop  │
└────────────────────┘
        │
        ▼ 第四章
┌────────────────────┐
│ 15. ReplyDispatcher│
│ 16. 飞书送达       │
└────────────────────┘
        │
        ▼
飞书用户收到回复
```

---

## 后记：可选的旁支

| 旁支            | 触发条件                             | 作用                                           |
| --------------- | ------------------------------------ | ---------------------------------------------- |
| Compaction      | 对话太长，context window 使用率 >80% | 压缩历史对话，保留关键决策和结果，删除冗余细节 |
| Memory Flush    | 长期记忆需要保存                     | 写入 MEMORY.md 文件，AI 记住关键信息           |
| Bootstrap       | 需要知识注入                         | 加载项目关键文件（CLAUDE.md 等），注入项目知识 |
| Sandbox         | 安全隔离                             | 限制文件访问范围，防止 AI 操作敏感路径         |
| Skills          | 技能加载                             | 加载预定义的技能脚本，扩展 AI 能力             |
| Queue           | 多消息并发                           | 同一群的消息按顺序排队处理，避免冲突           |
| Fallback        | 模型失败                             | 按配置顺序逐个尝试备用模型                     |
| Block Streaming | 实时显示                             | 流式输出文本，用户实时看到生成过程             |

---

## 结语

一条消息的旅程，看似简单的"发送-回复"，背后是：

- **16个核心步骤**
- **~3700行核心代码**（runEmbeddedAttempt）
- **PI Agent Conversation Loop**（对话循环）
- **流式输出 + 工具执行**的混合模式

3秒钟，16站，4章旅程。

这就是 OpenClaw 一条消息的奇幻之旅。
