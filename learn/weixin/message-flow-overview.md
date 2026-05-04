# 一条消息的奇幻旅程：从飞书到 AI 回复的 16 步之旅

> 当你在飞书发送一条消息给 OpenClaw Bot，这条消息经历了什么？
>
> 让我们跟随一条消息，探索它从发送到收到回复的完整旅程。

---

## 序章：3秒背后的故事

你在飞书群里 @ 了 OpenClaw Bot，发送了 "帮我分析一下这段代码"。

短短3秒后，一条流畅的 AI 回复出现在你的屏幕上。

这3秒里，你的消息经历了一场跨越16个步骤的奇幻旅程。

---

## 第一章：穿越防火墙（步骤1-4）

### 第1站：交通枢纽

飞书服务器推送消息 → WebSocket/Webhook → OpenClaw Gateway

```
飞书服务器 ──WebSocket长连接──> OpenClaw
                │
                │ 或
                │
飞书服务器 ──HTTP POST──> Webhook接口 ──> OpenClaw
```

两种"交通工具"：

- **WebSocket**：专线直达，实时推送（内网首选）
- **Webhook**：邮递员模式，HTTP回调（公网简单）

### 第2站：安检关卡

消息进入 OpenClaw 后，遇到三道安检：

**第一道：防抖门**

```
用户快速输入: "帮我" → "帮我分析" → "帮我分析代码"
                    │
                    └── 3秒窗口内合并成一条
                    │
                    结果: "帮我分析代码"
```

为什么？防止用户输入过程中发送多条半成品消息。

**第二道：去重门**

```
飞书网络波动，同一条消息推送了2次
                │
                └── 检查 message_id
                │
                结果: 只处理第一条，第二条丢弃
```

**第三道：排队门**

```
群聊中多个用户同时发消息
                │
                └── 同一个群的 messages 按顺序排队
                │
                结果: A的消息处理完，再处理B的消息
```

### 第3站：身份验证

```
消息进入业务层
        │
        ├── 是群聊？
        │   ├── 群是否在白名单？
        │   ├── 发送者是否在白名单？
        │   └── 是否 @ 了 Bot？
        │
        ├── 是私聊？
        │   ├── 用户是否已配对？
        │   ├── 是否在白名单？
        │
        └── 权限不足？→ 发送提示消息，旅程结束
```

### 第4站：任务编排

```
runChannelTurn
        │
        ├── 解析 Agent 路由（哪个AI模型？）
        ├── 解析 Session 绑定（哪个对话？）
        ├── 构建消息上下文
        │
        └── 启动回复生成流程
```

---

## 第二章：四层接力（步骤5-8）

消息进入 OpenClaw 核心区，开始一场四层接力赛。

### 第一棒：分发协调员

```
dispatchReplyFromConfig
        │
        ├── 【紧急出口】检查是否可以直接回复
        │   │
        │   └── 用户发送了 /stop、/help 等命令？
        │   └── 直接返回预设回复，不用调用 AI
        │   └── 省时间、省成本！
        │
        ├── 【插件插队】让插件有机会介入
        │   │
        │   ├── message_received：插件"偷看"消息（日志、统计）
        │   ├── before_dispatch：插件"拦截"消息（敏感词过滤）
        │   └── reply_dispatch：插件修改分发方式（多渠道发送）
        │
        ├── 【工具授权】决定 AI 能用什么工具
        │   │
        │   └── 不同用户、不同场景，工具箱不同
        │   └── 安全考虑：限制危险操作（如 exec 命令）
        │
        └── 传给第二棒
```

### 第二棒：回复准备员

```
getReplyFromConfig
        │
        ├── 【定位角色】解析 Agent ID
        │   │
        │   └── sessionKey → agentId
        │   └── 比如："agent:main:feishu:group:oc_xxx" → "main"
        │
        ├── 【选择模型】决定用哪个 AI
        │   │
        │   ├── 默认模型配置
        │   ├── 渠道覆盖（飞书专用模型？）
        │   ├── 存储覆盖（用户上次用的模型）
        │   │
        │   └── 结果："claude-opus-4-7" 或 "gpt-5.5"
        │
        ├── 【准备战场】创建工作目录
        │   │
        │   ├── 解析 workspaceDir（在哪工作？）
        │   ├── 创建目录（如果不存在）
        │   ├── 写入 bootstrap 文件（注入项目知识）
        │   │
        │   └── 结果：/home/user/workspace/
        │
        ├── 【理解内容】媒体和链接处理
        │   │
        │   ├── 用户发了图片？→ 图片理解
        │   ├── 用户发了链接？→ 链接理解
        │   │
        │   └── 提取关键信息，加入对话
        │
        ├── 【解析命令】用户说了什么？
        │   │
        │   ├── 用户说 "/think high" → 启用深度思考
        │   ├── 用户说 "/model gpt" → 切换模型
        │   ├── 用户说 "/reset" → 重置会话
        │   │
        │   └── 解析成指令，传给下游
        │
        ├── 【初始化会话】加载历史对话
        │   │
        │   ├── 读取 session 文件
        │   ├── 加载历史消息
        │   ├── 设置会话状态
        │   │
        │   └── 结果：SessionEntry（对话历史）
        │
        └── 传给第三棒
```

**关键决策**：这一棒决定"用什么 AI"、"在哪工作"、"用什么模式"。

### 第三棒：执行准备员

```
runPreparedReply
        │
        ├── 【场合设定】告诉 AI 在哪说话、怎么表现
        │   │
        │   ├── 群聊场合：
        │   │   ├── 行为规范："低调参与，被 @ 才回复"
        │   │   ├── 激活模式：always-on（收到所有）/ trigger-only（只有 @）
        │   │   └── 群介绍（首次进入时注入）
        │   │
        │   ├── 私聊场合：
        │   │   └── 行为规范："可以更详细，不用担心群噪音"
        │   │
        │   ├── 静默规则：
        │   │   └── AI 可以不发消息吗？
        │   │   ├── allow：返回特殊标记，不发送
        │   │   ├── rewrite：发简短回复代替
        │   │   └── disallow：必须回复
        │   │
        │   └── 组装成 extraSystemPrompt → 和核心 System Prompt 合并
        │
        ├── 【消息正文】用户到底说了啥
        │   │
        │   ├── 提取核心内容
        │   ├── 处理引用消息（用户回复了谁？）
        │   ├── 添加消息元信息（ID、时间）
        │   │
        │   └── 结果：prefixedCommandBody
        │
        ├── 【队列状态】要不要排队？
        │   │
        │   ├── 有其他消息正在处理？
        │   │   ├── 是 → 排队等待
        │   │   └── 否 → 立即执行
        │   │
        │   ├── 队列模式：
        │   │   ├── "run" → 直接执行
        │   │   ├── "steer" → 注入到现有对话
        │   │   ├── "followup" → 完成后继续
        │   │   └── "collect" → 收集模式
        │   │
        │   └── 结果：QueueAction
        │
        ├── 【Thinking 级别】思考深度
        │   │
        │   ├── 用户说 "/think high" → 深度思考
        │   ├── 用户说 "/think medium" → 中等思考
        │   ├── 用户没说 → 用默认级别
        │   │
        │   └── 模型不支持？自动降级
        │
        ├── 【技能快照】加载技能
        │   │
        │   ├── 用户配置了 skills？
        │   │   └── 加载技能列表
        │   │
        │   └── 传给 AI 作为可用能力
        │
        ├── 【媒体附件】图片、音频
        │   │
        │   ├── 暂存到 sandbox
        │   ├── 生成路径引用
        │   │
        │   └── AI 可以"看到"这些文件
        │
        └── 传给第四棒
```

**关键决策**：这一棒决定"排队策略"、"思考深度"、"辅助信息"。

### 第四棒：Agent 编排员

```
runReplyAgent
        │
        ├── 【Steering 检查】能不能插队？
        │   │
        │   ├── AI 正在回复？
        │   │   ├── 是 + 流式输出中 → 注入新消息
        │   │   │   └── 用户追加提问，AI 继续回复
        │   │   │
        │   │   └── 否 → 正常队列处理
        │   │
        │   └── Steering = 消息"搭便车"
        │
        ├── 【队列决策】怎么处理？
        │   │
        │   ├── "drop" → 丢弃（已有太多排队）
        │   ├── "enqueue-followup" → 排队等待
        │   ├── "run-now" → 立即执行
        │   ├── "wait" → 等待当前完成
        │   │
        │   └── 决策依据：队列设置、当前状态
        │
        ├── 【预压缩】对话太长了？
        │   │
        │   ├── 检查 context window 使用率
        │   │   ├── >80%？需要压缩
        │   │   │   │
        │   │   │   └── runPreflightCompaction
        │   │   │       ├── 保留关键信息
        │   │   │       ├── 删除冗余内容
        │   │   │       └── 写入压缩后的历史
        │   │   │
        │   │   └── <80%？跳过压缩
        │   │
        │   └── 避免"爆内存"
        │
        ├── 【内存刷新】长期记忆
        │   │
        │   ├── 有长期记忆需要保存？
        │   │   │
        │   │   └── runMemoryFlushIfNeeded
        │   │       └── 写入 MEMORY.md 文件
        │   │
        │   └── AI 记住了关键信息
        │
        ├── 【注册操作】生命周期管理
        │   │
        │   ├── 创建 ReplyOperation
        │   │   ├── 状态：queued → running → completed
        │   │   │
        │   │   └── 其他组件可以监控状态
        │
        ├── 【核心执行】调用 AI！
        │   │
        │   └── runAgentTurnWithFallback
        │       │
        │       ├── 主模型失败？
        │       │   └── 尝试备用模型
        │       │
        │       └── 成功？返回 AI 回复
        │       │
        │       └── 进入执行层（第9站）
        │
        ├── 【结果处理】组装回复
        │   │
        │   ├── buildReplyPayloads
        │   │   ├── 合并文本片段
        │   │   ├── 添加 usage 统计
        │   │   ├── 添加 model 信息
        │   │   │
        │   │   └── 结果：ReplyPayload
        │
        └── 进入执行层
```

**关键决策**：这一棒决定"压缩策略"、"模型 fallback"、"生命周期管理"。

---

## 第三章：核心执行（步骤9-14）

### 第9站：模型降级站

```
runAgentTurnWithFallback
        │
        ├── 【候选模型列表】解析可用模型
        │   │
        │   ├── 主模型（用户请求的）
        │   │   └── provider/model: "claude/claude-opus-4-7"
        │   │
        │   ├── Fallback 模型（配置的备用）
        │   │   ├── agents.defaults.model.fallbacks
        │   │   ├── 示例配置：
        │   │   │   ├── "openai/gpt-5.5" ← 第1备用
        │   │   │   ├── "anthropic/claude-sonnet-4-6" ← 第2备用
        │   │   │   └── "openai/gpt-4o-mini" ← 第3备用（本地）
        │   │   │
        │   │   └── 为什么要有备用？
        │   │   ├── 主模型可能 rate_limit（限流）
        │   │   ├── 主模型可能 overload（过载）
        │   │   ├── 主模型可能 billing 问题（账单）
        │   │   └── 主模型可能 timeout（超时）
        │   │
        │   ├── Auth Profile 检查
        │   │   │
        │   │   ├── 有认证配置？
        │   │   ├── 所有 profile 都在 cooldown？
        │   │   │   ├── 是 → 可能跳过这个候选
        │   │   │   └── 否 → 继续尝试
        │   │   │
        │   │   └── cooldown = 刚失败过，需要等待
        │   │
        │   └── 结果：candidates[] ← 可尝试的模型列表
        │
        ├── 【逐个尝试】循环执行直到成功
        │   │
        │   │  for (candidate of candidates) {
        │   │      │
        │   │      ├── 调用 run(candidate.provider, candidate.model)
        │   │      │   │
        │   │      │   └── 执行 LLM 调用
        │   │      │
        │   │      ├── 成功？
        │   │      │   ├── 是 → 返回结果，结束循环 ✓
        │   │      │   │
        │   │      │   └── 否 → 记录失败原因
        │   │      │       ├── rate_limit：API 限流
        │   │      │       ├── overloaded：服务过载
        │   │      │       ├── billing：账单问题
        │   │      │       ├── timeout：请求超时
        │   │      │       ├── context_overflow：对话太长
        │   │      │       ├── auth_error：认证失败
        │   │      │       └── unknown：其他错误
        │   │      │
        │   │      ├── 错误处理：
        │   │      │   ├── AbortError？→ 直接抛出（用户取消）
        │   │      │   ├── FailoverError？→ 继下一个候选
        │   │      │   ├── 其他错误？→ 继续下一个
        │   │      │
        │   │      └── 尝试下一个候选 ↺
        │   │  }
        │   │
        │   └── 所有候选都失败？
        │   │   │
        │   │   └── 抛出 FallbackSummaryError
        │   │   ├── 包含：所有尝试的详情
        │   │   ├── 包含：最短的 cooldown 过期时间
        │   │   │
        │   │   └── 用户看到：
        │   │   ├── "Claude rate-limited, retry in 30s"
        │   │   ├── "GPT overloaded, retry in 2m"
        │   │   ├── "所有模型暂时不可用，请稍后重试"
        │   │
        │   └── 成功？返回 { result, provider, model, attempts }
        │
        └── 模型可用？→ 继续执行
```

**关键点**：Fallback 不是"随便换模型"，而是按配置顺序逐个尝试，每个失败都有明确原因。

### 第10站：Harness 选择站

```
runEmbeddedPiAgent
        │
        ├── 【选择执行引擎】谁来跑这个任务？
        │   │
        │   ├── PI Harness（默认）
        │   │   │
        │   │   ├── id: "pi"
        │   │   ├── 内置引擎，支持所有 provider
        │   │   └── runAttempt → runEmbeddedAttempt（第13站）
        │   │
        │   ├── Plugin Harness（自定义引擎）
        │   │   │
        │   │   ├── 示例：Codex App Server Harness
        │   │   │   ├── 只处理特定 provider（如 "codex"）
        │   │   │   ├── 优先级更高（priority: 100）
        │   │   │   ├── 有自己的 runAttempt 实现
        │   │   │   └── 可能有自己的 compact/reset 逻辑
        │   │   │
        │   │   └── 为什么？某些模型有特殊执行方式
        │   │
        │   ├── 选择逻辑：
        │   │   │
        │   │   ├── pinned：用户指定了 harness
        │   │   ├── forced_pi：配置强制用 PI
        │   │   ├── forced_plugin：配置强制用插件
        │   │   ├── auto_plugin：自动选择匹配的插件
        │   │   └── auto_pi：没有匹配插件，用 PI
        │   │
        │   └── 结果：选定的 Harness + Policy
        │
        ├── 【准备执行环境】搭好舞台
        │   │
        │   ├── Session Key 补全
        │   │   └── 确保下游都能拿到 sessionKey
        │   │
        │   ├── Lane 队列设置
        │   │   ├── globalLane：全局任务队列
        │   │   ├── sessionLane：会话任务队列
        │   │   └── enqueueGlobal / enqueueSession
        │   │
        │   ├── Timeout 配置
        │   │   └── laneTaskTimeoutMs（超时限制）
        │   │
        │   ├── Workspace 解析
        │   │   ├── resolveRunWorkspaceDir
        │   │   └── 如果用户指定的目录不存在？用 fallback
        │   │
        │   ├── Runtime Plugins 加载
        │   │   └── ensureRuntimePluginsLoaded
        │   │   └── 加载 agent 可用的运行时插件
        │   │
        │   ├── Provider/Model 解析
        │   │   ├── provider: "claude" / "openai" / ...
        │   │   └── modelId: "claude-opus-4-7" / "gpt-5.5" / ...
        │   │
        │   ├── Hook Runner 获取
        │   │   └── getGlobalHookRunner()
        │   │   └── 插件 hook 可能在执行中触发
        │   │
        │   └── 结果：执行环境就绪
        │
        └── 运行 Harness → runAgentHarnessAttempt
```

### 第11站：Harness 选择与适配

```
runAgentHarnessAttempt
        │
        ├── 【选择 Harness】哪个引擎执行？
        │   │
        │   ├── selectAgentHarnessDecision()
        │   │   │
        │   │   ├── 输入：provider, model, agentHarnessId
        │   │   │
        │   │   ├── 决策逻辑：
        │   │   │   ├── pinned：用户显式指定 harness
        │   │   │   ├── forced_pi：配置强制用 PI
        │   │   │   ├── forced_plugin：配置强制用插件
        │   │   │   ├── auto_plugin：自动匹配插件 harness
        │   │   │   └── auto_pi：无匹配，用 PI
        │   │   │
        │   │   ├── 候选列表：
        │   │   │   ├── PI Harness（priority: 0）
        │   │   │   ├── Plugin Harness（priority: 100+）
        │   │   │   └── 按优先级排序
        │   │   │
        │   │   └── 输出：
        │   │   ├── harness: AgentHarness
        │   │   ├── selectedHarnessId: "pi" | "codex" | ...
        │   │   ├── selectedReason: 为什么选这个？
        │   │   └── candidates: 所有候选及其优先级
        │   │
        │   ├── 为什么有多个 Harness？
        │   │   │
        │   │   ├── PI Harness：通用，支持所有模型
        │   │   ├── Plugin Harness：特定模型优化
        │   │   │   ├── Codex Harness：专门处理 codex provider
        │   │   │   ├── 可能有自己的 runAttempt 实现
        │   │   │   ├── 可能有自己的 compact/reset 逻辑
        │   │   │   └── 更高优先级（priority: 100）
        │   │   │
        │   │   └── 不同模型可能需要不同的执行方式
        │   │
        │   └── 【V2 适配】统一接口
        │   │   │
        │   │   ├── adaptAgentHarnessToV2(harness)
        │   │   │   │
        │   │   ├── 为什么需要适配？
        │   │   │   ├── V1 Harness（旧接口）：
        │   │   │   │   └── 只有 runAttempt() 方法
        │   │   │   │   └── 一次性执行
        │   │   │   │
        │   │   ├── V2 Harness（新接口）：
        │   │   │   │   ├── prepare() → 准备
        │   │   │   │   ├── start() → 启动
        │   │   │   │   ├── send() → 执行（核心）
        │   │   │   │   ├── resolveOutcome() → 结果
        │   │   │   │   └── cleanup() → 清理
        │   │   │   │   └── 四阶段生命周期
        │   │   │   │
        │   │   └── 适配 = 包装 V1 成 V2
        │   │   └── V2.send() 内部调用 V1.runAttempt()
        │   │
        │   └── 输出：AgentHarnessV2（统一的四阶段接口）
        │
        └── 传给第12站
```

**一句话**：选择合适的执行引擎，并适配成统一的四阶段接口。

### 第12站：V2 生命周期执行

```
runHarnessV2LifecycleAttempt
        │
        │  执行 Harness 的四阶段生命周期
        │  V2 是统一的执行框架
        │
        ├── 【阶段1: prepare】准备资源
        │   │
        │   ├── harness.prepare(params)
        │   │   │
        │   ├── 对于 PI Harness：
        │   │   │   ├── 仅标记 lifecycleState = "prepared"
        │   │   │   └── 不做实际操作（V1 适配）
        │   │   │
        │   ├── 对于原生 V2 Harness：
        │   │   │   ├── 可能构建 Prompt
        │   │   │   ├── 可能初始化 Tools
        │   │   │   └── 依赖具体实现
        │   │   │
        │   └── 输出：{ harnessId, params, lifecycleState: "prepared" }
        │
        ├── 【阶段2: start】启动生命周期
        │   │
        │   ├── harness.start(prepared)
        │   │   │
        │   ├── 对于 PI Harness：
        │   │   │   └── 仅标记 lifecycleState = "started"
        │   │   │
        │   ├── 对于原生 V2 Harness：
        │   │   │   ├── 可能初始化 Session
        │   │   │   ├── 可能加载数据
        │   │   │   └── 依赖具体实现
        │   │   │
        │   └── 输出：{ harnessId, params, lifecycleState: "started" }
        │
        ├── 【阶段3: send】★ 核心执行
        │   │
        │   ├── harness.send(started)
        │   │   │
        │   ├── 对于 PI Harness（V1 适配）：
        │   │   │   │
        │   │   │   └── send() 内部调用 runAttempt()
        │   │   │   └── runAttempt → runEmbeddedAttempt（第13站）
        │   │   │   └── 这是真正的执行！
        │   │   │   │
        │   │   └── 对于原生 V2 Harness：
        │   │   │   └── 自定义的执行逻辑
        │   │   │   └── 调用特定 API
        │   │   │
        │   ├── async：是（等待执行完成）
        │   │
        │   └── 输出：EmbeddedRunAttemptResult
        │   └── { assistantTexts, toolMetas, usage, classification }
        │
        ├── 【阶段4: resolveOutcome】结果分类
        │   │
        │   ├── harness.resolveOutcome(result)
        │   │   │
        │   ├── classifyRunResult()
        │   │   │   ├── "ok" → 成功
        │   │   │   ├── "error" → 失败
        │   │   │   ├── "aborted" → 用户中断
        │   │   │   ├── "timeout" → 超时
        │   │   │   └── "yielded" → yield 检测
        │   │   │
        │   ├── applyClassification()
        │   │   └── 标记最终的 outcome
        │   │
        │   └── 输出：最终状态
        │
        ├── 【阶段5: cleanup】清理资源
        │   │
        │   ├── harness.cleanup()
        │   │   │
        │   ├── 清理临时文件
        │   ├── 释放锁
        │   ├── 关闭连接
        │   │
        │   └── 无论成功失败，都执行 cleanup
        │
        └── 返回结果 → 第13站的输出
```

**关键理解**：

| 阶段           | V1 适配（PI Harness） | 原生 V2 Harness       |
| -------------- | --------------------- | --------------------- |
| prepare        | 仅标记状态            | 可能构建 Prompt/Tools |
| start          | 仅标记状态            | 可能初始化 Session    |
| send           | ★ 调用 runAttempt     | 自定义执行            |
| resolveOutcome | classifyRunResult     | 自定义分类            |
| cleanup        | 清理资源              | 自定义清理            |

**一句话**：执行四阶段生命周期，send() 是真正的核心执行。

### 第13站：核心执行站（PI Agent 入口）

> **重要**：从这一站开始，进入 @mariozechner/pi-coding-agent 的世界。
> runEmbeddedAttempt 是 OpenClaw 对 PI Agent 的封装，核心对话循环由 PI Agent 管理。

```
runEmbeddedAttempt（~3700行核心代码）
        │
        │  ★ PI Agent Session 是对话循环的核心对象
        │
        ├── [阶段A] 初始化
        │   ├── workspace 设置
        │   ├── sandbox 配置
        │   └── skills 加载
        │
        ├── [阶段B] 工具准备
        │   │
        │   ├── createOpenClawCodingTools ← ★ OpenClaw 工具集
        │   │   ├── read/write/edit/grep/exec/web_search...
        │   │   └── 这些工具会传给 PI Agent
        │   │
        │   ├── MCP 工具（外部服务）
        │   ├── LSP 工具（代码补全）
        │   │
        │   └── applyEmbeddedAttemptToolsAllow ← 工具过滤
        │   └── 根据授权决定最终可用的工具列表
        │
        ├── [阶段C] ★ PI Agent Session 创建
        │   │
        │   ├── SessionManager.fromFile ← @mariozechner/pi-coding-agent
        │   │   └── 加载 session 文件（对话历史）
        │   │
        │   ├── createAgentSession ← @mariozechner/pi-coding-agent
        │   │   └── 创建 AgentSession 对象
        │   │   │
        │   │   └── 这个对象管理整个对话循环：
        │   │       ├── agent.streamFn ← LLM 调用函数
        │   │       ├── conversation history ← 对话历史
        │   │       ├── tool registry ← 工具注册表
        │   │       └── event handlers ← 事件处理器
        │   │
        │   └── OpenClaw 包装：activeSession = wrapSession()
        │
        ├── [阶段D] ★ System Prompt 构建
        │   │
        │   ├── buildSystemPromptParams()
        │   │   └── 收集运行时信息：OS、Node版本、模型、Shell、时间
        │   │
        │   ├── buildEmbeddedSystemPrompt() ← ★ 核心组装
        │   │   ├── 工具描述列表
        │   │   ├── workspace 信息
        │   │   ├── skills 提示
        │   │   ├── runtime 信息（时间、操作系统）
        │   │   ├── Provider 特定内容
        │   │   │
        │   │   └── 输出：完整 System Prompt（几千字符）
        │   │   └── 这个 Prompt 会传给 PI Agent
        │   │
        │   ├── Bootstrap Files（知识注入）
        │   ├── Cache Boundary（缓存优化）
        │   │
        │   └── transformProviderSystemPrompt() ← Provider 转换
        │
        ├── [阶段E] ★ 注册 PI Agent 事件处理器
        │   │
        │   ├── subscribeEmbeddedPiSession ← OpenClaw 包装
        │   │   │
        │   │   └── session.subscribe(handler)
        │   │       │
        │   │       └── 注册 PI Agent 事件处理器：
        │   │           ├── message_start → 开始生成
        │   │           ├── message_update → 流式文本块
        │   │           ├── message_end → 消息完成
        │   │           ├── tool_execution_start → 工具开始
        │   │           ├── tool_execution_update → 工具进度
        │   │           ├── tool_execution_end → 工具完成
        │   │           ├── agent_start → Agent 启动
        │   │           ├── agent_end → Agent 结束
        │   │           ├── compaction_start → 压缩开始
        │   │           ├── compaction_end → 压缩完成
        │   │
        │   └── 事件驱动：PI Agent 发事件，OpenClaw 处理
        │
        ├── [阶段E-2] ★ 注册 streamFn（LLM 调用函数）
        │   │
        │   ├── registerProviderStreamForModel
        │   │   └── providerStreamFn = Provider.streamCompletion
        │   │   └── （函数引用，尚未调用）
        │   │
        │   ├── resolveEmbeddedAgentStreamFn
        │   │   └── 包装 providerStreamFn
        │   │   └── 添加 Provider 特定的转换
        │   │
        │   └── session.agent.streamFn = streamFn
        │   └── PI Agent 内部会调用这个函数
        │
        └── [阶段E-3] ★ 启动 PI Agent 对话循环
            │
            └── activeSession.prompt(userPrompt)
                │
                │  ★ 此时才真正开始！PI Agent 接管控制
                │
                └── 进入 PI Agent Conversation Loop（第14站）
```

**关键区分**：

| 职责          | OpenClaw                    | PI Agent          |
| ------------- | --------------------------- | ----------------- |
| 工具创建      | createOpenClawCodingTools   | 接收工具列表      |
| Session 创建  | 调用 createAgentSession     | 管理 Session 对象 |
| System Prompt | buildEmbeddedSystemPrompt   | 接收 Prompt 文本  |
| 事件处理      | subscribeEmbeddedPiSession  | 发送事件          |
| streamFn      | Provider.streamCompletion   | 内部调用          |
| 对话循环      | activeSession.prompt() 启动 | while 循环管理    |

### 第14站：PI Agent 对话循环

```
PI Agent Conversation Loop（来自 @mariozechner/pi-coding-agent）

while (!finished) {
        │
        ├── 调用 streamFn()
        │       │
        │       └── Provider.streamCompletion
        │       │
        │       └── 调用 Claude/GPT API
        │       │
        │       └── 返回：AsyncIterable<StreamChunk>
        │               │
        │               ├── { text: "部分文本..." }
        │               ├── { toolCall: { name, args } }
        │               └── { finishReason: "stop" }
        │
        ├── 收到 text？
        │       │
        │       └── 立即发送 Block Reply
        │       │
        │       └── 用户看到"正在打字..."
        │
        ├── 收到 toolCall？
        │       │
        │       └── executeToolCall()（执行工具）
        │       │       │
        │       │       └── 例如：web_search → 搜索网络
        │       │       │
        │       │       └── 工具结果加入对话
        │       │       │
        │       │       └── 再次调用 streamFn() ↺
        │       │
        │       └── 用户看到"暂停"（工具执行中）
        │
        └── 收到 finishReason？
                │
                └── 结束循环 ✓
}
```

**流式 vs 阻塞对比**：

| 环节        | 类型   | 用户感知     |
| ----------- | ------ | ------------ |
| 文本输出    | 流式   | 看到"打字中" |
| 工具执行    | 阻塞   | 看到"暂停"   |
| 再次调用LLM | 流式   | 看到"继续"   |
| 最终回复    | 一次性 | 收到完整卡片 |

---

## 第四章：回复归途（步骤15-16）

### 第15站：回复分发器

```
ReplyDispatcher
        │
        ├── sendBlockReply(chunk) → 流式更新飞书卡片
        │       │
        │       └── 用户看到实时生成
        │
        ├── sendToolResult(result) → 工具结果
        │       │
        │       └── 飞书不显示（返回false）
        │       └── 其他渠道可能显示
        │
        └── sendFinalReply(payload) → 最终回复
                │
                ├── 判断：卡片 or 文本？
                │       │
                │       ├── 有代码块/表格 → 交互式卡片
                │       └── 普通文本 → 富文本消息
                │
                └── 发送给飞书
```

### 第16站：飞书送达

```
sendMessageFeishu
        │
        ├── 解析目标（chat_id / open_id）
        ├── 构建飞书消息格式
        ├── 调用飞书 API
        │       │
        │       └── POST /im/v1/messages
        │
        └── 返回 { message_id }
                │
                └── 飞书服务器推送给用户 ✓
```

---

## 终章：完整旅程图

```
飞书用户发送消息
        │
        ▼ ──────────────────── 第一章
┌────────────────────────────────────────────┐
│ 1. WebSocket/Webhook 接收                  │
│ 2. 防抖 → 去重 → 排队                      │
│ 3. 权限检查                                │
│ 4. 任务编排                                │
└────────────────────────────────────────────┘
        │
        ▼ ──────────────────── 第二章
┌────────────────────────────────────────────┐
│ 5. 分发协调                                │
│ 6. 回复准备                                │
│ 7. 执行准备                                │
│ 8. Agent 编排                              │
└────────────────────────────────────────────┘
        │
        ▼ ──────────────────── 第三章
┌────────────────────────────────────────────┐
│ 9. 模型 Fallback                           │
│ 10. Harness 选择                           │
│ 11-12. V2 Lifecycle                        │
│ 13. ★ runEmbeddedAttempt                   │
│     ├── Tools 构建                         │
│     ├── Session 创建                       │
│     └── Prompt 构建                        │
│ 14. ★ PI Agent Conversation Loop           │
│     └── streamFn() → Provider API          │
└────────────────────────────────────────────┘
        │
        ▼ ──────────────────── 第四章
┌────────────────────────────────────────────┐
│ 15. ReplyDispatcher                        │
│     ├── Block Reply（流式）                 │
│     └── Final Reply（完整）                 │
│ 16. sendMessageFeishu                      │
│     └── 飞书 API                            │
└────────────────────────────────────────────┘
        │
        ▼
飞书用户收到回复
```

---

## 后记：可选的旁支旅程

主干流程之外，还有8条可选的旁支：

| 旁支                | 触发条件     | 作用                   |
| ------------------- | ------------ | ---------------------- |
| **Compaction**      | 对话太长     | 压缩历史，保留关键信息 |
| **Memory Flush**    | 内存超限     | 清理旧对话，释放空间   |
| **Bootstrap**       | 需要知识注入 | 加载项目关键文件       |
| **Sandbox**         | 安全隔离     | 限制文件访问范围       |
| **Skills**          | 技能加载     | 加载预定义的技能脚本   |
| **Queue**           | 多消息并发   | 排队处理，避免冲突     |
| **Fallback**        | 模型失败     | 降级到备用模型         |
| **Block Streaming** | 实时显示     | 流式输出，用户体验     |

---

## 结语

一条消息的旅程，看似简单的"发送-回复"，背后是：

- **16个核心步骤**
- **~3700行核心代码**（runEmbeddedAttempt）
- **PI Agent Conversation Loop**（对话循环）
- **流式输出 + 工具执行**的混合模式

3秒钟，16站，4章旅程。

这就是 OpenClaw 一条消息的奇幻之旅。

---

> 本文基于 OpenClaw 源码分析整理
>
> 详细技术文档见：[learn/message-flow/](../message-flow/)

---

**互动话题**：你认为这16步中，哪一步最关键？欢迎评论区讨论！
