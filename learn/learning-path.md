# OpenClaw 学习路径

> 本文档整理了 OpenClaw 项目的完整学习主题和推荐学习顺序。
>
> 更新日期：2026-05-05 | 版本：2026.5.4

---

## 学习地图总览

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              OpenClaw 学习地图                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │
│  │  阶段 1     │ →  │  阶段 2     │ →  │  阶段 3     │ →  │  阶段 4     │   │
│  │  基础架构   │    │  核心运行时 │    │  Agent核心  │    │  会话记忆   │   │
│  │  (3 主题)   │    │  (4 主题)   │    │  (11 主题)  │    │  (3 主题)   │   │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘   │
│         ↓                  ↓                  ↓                  ↓          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────────────┐  │
│  │  阶段 5     │    │  阶段 6     │    │  ★ 横向主题: Message Flow       │  │
│  │  高级能力   │    │  能力模块   │    │  (15 步完整执行链)               │  │
│  │  (4 主题)   │    │  (6 主题)   │    │  learn/message-flow/            │  │
│  └─────────────┘    └─────────────┘    └─────────────────────────────────┘  │
│                                                                              │
│  总计: 29 个学习主题                                                          │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 学习路径图

```
阶段1: 基础架构
    Architecture → Entry → Config
           ↓
阶段2: 核心运行时
    Gateway → Plugin → Channel → Hook
           ↓
阶段3: Agent 核心
    Runtime概念 → Harness → PI Runner → PI Hooks → ACP
           ↓
    Agent → Agent Loop → Tools → Context → Prompt → Sandbox
           ↓
阶段4: 会话与记忆
    Session → Compaction → Memory
           ↓
阶段5: 高级能力
    Model → Multi-Agent → MCP → Codex Harness
           ↓
阶段6: 能力模块（按需选择）
    Web → Media → Voice → Security → Cron → UI
           ↓
★ 横向主题: Message Flow（贯穿所有阶段）
```

---

## 主题清单

| 分类                  | 主题数 | 主题列表                                                                                 |
| --------------------- | ------ | ---------------------------------------------------------------------------------------- |
| **阶段1: 基础架构**   | 3      | Architecture, Entry, Config                                                              |
| **阶段2: 核心运行时** | 4      | Gateway, Plugin, Channel, Hook                                                           |
| **阶段3: Agent核心**  | 10     | Runtime, Harness, PI Runner, PI Hooks, ACP, Agent, Loop, Tools, Context, Prompt, Sandbox |
| **阶段4: 会话记忆**   | 3      | Session, Compaction, Memory                                                              |
| **阶段5: 高级能力**   | 4      | Model, Multi-Agent, MCP, Codex Harness                                                   |
| **阶段6: 能力模块**   | 6      | Web, Media, Voice, Security, Secrets, Cron, UI                                           |
| **★横向主题**         | 1      | Message Flow                                                                             |

---

## 阶段 1: 基础架构

理解 OpenClaw 的整体结构和启动机制。

### 1.1 Architecture 整体架构

| 属性         | 值                                   |
| ------------ | ------------------------------------ |
| **核心目录** | `src/`                               |
| **核心文档** | `docs/concepts/architecture.md`      |
| **学习文档** | `learn/architecture.md`              |
| **学习目标** | 理解模块划分、依赖关系、核心组件职责 |

**关键模块**：

```
src/
├── agents/          # Agent 核心（PI Runner、Harness、Tools）
├── channels/        # 渠道实现（Telegram、Discord、飞书等）
├── gateway/         # WebSocket 网关、协议层
├── plugins/         # 插件加载、注册
├── config/          # 配置系统
├── sessions/        # 会话管理
├── memory/          # 记忆系统
├── auto-reply/      # 自动回复编排
├── model-catalog/   # 模型目录
├── acp/             # ACP 协议（外部 Harness）
└── ...
```

### 1.2 Entry 启动流程

| 属性         | 值                                     |
| ------------ | -------------------------------------- |
| **核心文件** | `src/entry.ts`, `src/bootstrap/`       |
| **学习目标** | 理解进程启动、配置加载、runtime 初始化 |

**启动链路**：

```
entry.ts
    │
    ├── 解析命令行参数
    ├── 加载环境变量
    ├── 初始化 config
    ├── 启动 gateway (WebSocket)
    ├── 加载 plugins
    ├── 注册 channels
    └── 进入运行状态
```

### 1.3 Config 配置系统

| 属性         | 值                                                                         |
| ------------ | -------------------------------------------------------------------------- |
| **核心目录** | `src/config/`                                                              |
| **核心文档** | `docs/gateway/configuration.md`, `docs/gateway/configuration-reference.md` |
| **学习目标** | 理解配置层级、schema、validation、合并策略                                 |

**配置层级**：

```
默认配置 → profile 配置 → agent 配置 → 环境变量 → 命令行参数
    │          │            │           │            │
    ↓          ↓            ↓           ↓            ↓
  合并优先级：低 ←─────────────────────────────→ 高
```

---

## 阶段 2: 核心运行时

理解消息如何进出系统、插件如何扩展、渠道如何对接。

### 2.1 Gateway 网关

| 属性         | 值                                    |
| ------------ | ------------------------------------- |
| **核心目录** | `src/gateway/`                        |
| **核心文档** | `src/gateway/AGENTS.md`               |
| **学习文档** | `learn/gateway.md`                    |
| **学习目标** | 理解 WebSocket 服务、协议层、消息路由 |

**核心组件**：

| 文件              | 职责               |
| ----------------- | ------------------ |
| `server.ts`       | WebSocket 服务器   |
| `protocol/`       | 协议定义、消息格式 |
| `server-methods/` | RPC 方法实现       |

### 2.2 Plugin 插件系统

| 属性         | 值                                                  |
| ------------ | --------------------------------------------------- |
| **核心目录** | `src/plugins/`, `src/plugin-sdk/`, `extensions/`    |
| **核心文档** | `src/plugins/AGENTS.md`, `src/plugin-sdk/AGENTS.md` |
| **学习目标** | 理解插件 manifest、registry、生命周期、SDK 接口     |

**插件类型**：

| 类型                | 示例                    | 目录                                          |
| ------------------- | ----------------------- | --------------------------------------------- |
| **Channel Plugin**  | Telegram, Discord, 飞书 | `extensions/telegram/`, `extensions/discord/` |
| **Provider Plugin** | OpenAI, Bailian         | `extensions/openai/`, `extensions/bailian/`   |
| **Harness Plugin**  | Codex                   | `extensions/codex/`                           |
| **Tool Plugin**     | Brave Search            | `extensions/brave/`                           |

### 2.3 Channel 渠道

| 属性         | 值                                                           |
| ------------ | ------------------------------------------------------------ |
| **核心目录** | `src/channels/`, `extensions/*/src/channel.ts`               |
| **核心文档** | `src/channels/AGENTS.md`, `docs/concepts/channel-docking.md` |
| **学习文档** | `learn/channel-routing.md`                                   |
| **学习目标** | 理解消息接收、解析、路由、回复发送                           |

**渠道生命周期**：

```
monitor (WebSocket/Webhook)
    │
    ├── 接收消息
    ├── 解析事件
    ├── 去重/debounce
    ├── 权限检查
    ├── 路由解析
    ├── runChannelTurn
    └── ReplyDispatcher 发送回复
```

### 2.4 Hook 钩子系统

| 属性         | 值                                      |
| ------------ | --------------------------------------- |
| **核心目录** | `src/hooks/`                            |
| **学习目标** | 理解事件触发机制、Hook 与 Plugin 的区别 |

**Hook vs Plugin**：

| 概念       | 定义       | 用途                                    |
| ---------- | ---------- | --------------------------------------- |
| **Hook**   | 事件触发器 | 在特定时机执行脚本，注入自定义逻辑      |
| **Plugin** | 扩展模块   | 提供完整的 Channel/Provider/Tool 等能力 |

**配置示例**：

```json5
{
  hooks: {
    before_tool_call: "scripts/check-tool.sh",
    after_agent_reply: "scripts/log-reply.sh",
  },
}
```

---

## 阶段 3: Agent 核心

理解 Agent 如何运行、如何执行 LLM 调用、如何处理工具。

### 3.1 Agent Runtime 运行时概念

| 属性         | 值                                                                         |
| ------------ | -------------------------------------------------------------------------- |
| **核心文档** | `docs/concepts/agent-runtimes.md`                                          |
| **学习目标** | 理解 Provider/Model/Runtime/Channel 四层分离、Runtime 类型、ownership 边界 |

**四层分离**：

| Layer             | Examples                              | What it means                  |
| ----------------- | ------------------------------------- | ------------------------------ |
| **Provider**      | `openai`, `anthropic`, `bailian`      | 认证、模型发现、model ref 命名 |
| **Model**         | `gpt-5.5`, `claude-opus-4-7`, `glm-5` | 选中的模型                     |
| **Agent Runtime** | `pi`, `codex`, `claude-cli`           | 执行 prepared turn 的底层循环  |
| **Channel**       | Telegram, Discord, 飞书               | 消息进出通道                   |

**Runtime 类型**：

| Runtime   | Owner            | Thread History        | Dynamic Tools | Compaction   |
| --------- | ---------------- | --------------------- | ------------- | ------------ |
| **PI**    | OpenClaw         | OpenClaw transcript   | Native        | OpenClaw     |
| **Codex** | Codex app-server | Codex thread + mirror | Bridged       | Codex-native |
| **ACP**   | External harness | External              | Varies        | External     |

### 3.2 Harness 实现

| 属性         | 值                                                 |
| ------------ | -------------------------------------------------- |
| **核心目录** | `src/agents/harness/`                              |
| **核心文件** | `types.ts`, `selection.ts`, `v2.ts`, `registry.ts` |
| **学习目标** | 理解 Harness 接口、选择机制、V1/V2 生命周期适配    |

**核心文件**：

| 文件                   | 职责                  | 行数  |
| ---------------------- | --------------------- | ----- |
| `types.ts`             | AgentHarness 接口定义 | ~58   |
| `registry.ts`          | Harness 注册表        | ~101  |
| `selection.ts`         | Harness 选择决策      | ~300  |
| `v2.ts`                | V2 生命周期适配       | ~240  |
| `builtin-pi.ts`        | PI Harness 创建       | ~20   |
| `native-hook-relay.ts` | Native Hook 中继      | ~1500 |

**AgentHarness 接口 (V1)**：

```typescript
type AgentHarness = {
  id: string;
  label: string;
  pluginId?: string;
  supports(ctx): AgentHarnessSupport; // 支持度检查
  runAttempt(params): Promise<Result>; // ★ 执行入口
  classify?(result): Classification; // 结果分类
  compact?(params): Promise<CompactResult>; // 压缩
  reset?(params): Promise<void>; // 重置
  dispose?(): Promise<void>; // 销毁
};
```

**V2 生命周期**：

```
prepare → start → send → resolveOutcome → cleanup
   │        │       │          │              │
   ↓        ↓       ↓          ↓              ↓
 准备状态  启动状态  执行     结果处理       清理资源
```

**选择流程**：

```
selectAgentHarnessDecision()
    │
    ├── 1. pinnedPolicy (显式指定) → "pinned"
    ├── 2. runtime: "pi" → "forced_pi"
    ├── 3. runtime: "plugin" → "forced_plugin"
    ├── 4. auto + plugin supports → "auto_plugin"
    └── 5. auto + no plugin → "auto_pi" (fallback)
```

### 3.3 PI Embedded Runner

| 属性         | 值                                                    |
| ------------ | ----------------------------------------------------- |
| **核心目录** | `src/agents/pi-embedded-runner/`                      |
| **核心文件** | `run.ts`, `run/attempt.ts`, `model.ts`, `compact.ts`  |
| **学习目标** | 理解核心执行循环、Session 管理、Prompt 构建、API 调用 |

**核心文件**：

| 文件                     | 职责               | 行数   |
| ------------------------ | ------------------ | ------ |
| `run.ts`                 | 入口、Harness 选择 | ~700   |
| `run/attempt.ts`         | ★ 核心执行循环     | ~3700  |
| `model.ts`               | 模型处理           | ~37155 |
| `compact.ts`             | 压缩触发           | ~55359 |
| `run/auth-controller.ts` | 认证控制           | ~20810 |
| `run/payloads.ts`        | 请求/响应载荷      | ~17078 |
| `run/incomplete-turn.ts` | 不完整回合         | ~27503 |

**attempt.ts 执行阶段**：

```
runEmbeddedAttempt()
    │
    ├── [阶段 A] 初始化
    │   ├── workspace 解析
    │   ├── sandbox 配置 ← 详见 3.11 Sandbox
    │   ├── skills 加载
    │   └── tools 注册
    │
    ├── [阶段 B] Session
    │   ├── SessionManager 加载
    │   ├── PI Session 创建
    │   └── history 处理
    │
    ├── [阶段 C] Prompt
    │   ├── systemPrompt 构建
    │   ├── bootstrap files 注入
    │   └── cache boundary 处理
    │
    ├── [阶段 D] API
    │   ├── streamFn 配置
    │   ├── provider 设置
    │   └── 调用参数
    │
    ├── [阶段 E] 执行
    │   ├── subscribeEmbeddedPiSession
    │   ├── streamFn → Provider API
    │   ├── 工具调用循环
    │   └── 结果收集
    │
    └── [阶段 F] 结果
        ├── 构造返回值
        ├── usage 统计
        └── 清理资源
```

### 3.4 PI Hooks

| 属性         | 值                                              |
| ------------ | ----------------------------------------------- |
| **核心目录** | `src/agents/pi-hooks/`                          |
| **核心文件** | `compaction-safeguard.ts`, `context-pruning.ts` |
| **学习目标** | 理解压缩保护机制、上下文修剪策略                |

**核心文件**：

| 文件                         | 职责                       |
| ---------------------------- | -------------------------- |
| `compaction-safeguard.ts`    | 压缩质量保护、防止过度压缩 |
| `context-pruning.ts`         | 上下文修剪、移除冗余信息   |
| `compaction-instructions.ts` | 压缩指令生成               |

### 3.5 ACP 协议

| 属性         | 值                                                   |
| ------------ | ---------------------------------------------------- |
| **核心目录** | `src/acp/`                                           |
| **学习目标** | 理解外部 Harness 协议、Claude Code/Gemini CLI 等适配 |

**ACP 组件**：

```
acp/
├── runtime/
│   ├── registry.ts        # ACP 适配器注册
│   ├── session-identity.ts # 会话身份
│   └── availability.ts    # 可用性检查
└── adapter-contract.testkit.ts # 适配器契约
```

### 3.6 Agent 定义

| 属性         | 值                                         |
| ------------ | ------------------------------------------ |
| **核心目录** | `src/agents/`                              |
| **核心文档** | `docs/concepts/agent.md`                   |
| **学习目标** | 理解 Agent 配置、类型、defaults、workspace |

### 3.7 Agent Loop 循环

| 属性         | 值                                       |
| ------------ | ---------------------------------------- |
| **核心目录** | `src/agents/loop/`                       |
| **核心文档** | `docs/concepts/agent-loop.md`            |
| **学习目标** | 理解消息处理 → 工具调用 → 响应生成的循环 |

### 3.8 Tools 工具系统

| 属性         | 值                                   |
| ------------ | ------------------------------------ |
| **核心目录** | `src/agents/tools/`                  |
| **核心文档** | `src/agents/tools/AGENTS.md`         |
| **学习目标** | 理解工具定义、注册、执行、策略、审批 |

**核心工具**：

| 工具             | 职责          |
| ---------------- | ------------- |
| `read`           | 读取文件      |
| `write`          | 写入文件      |
| `edit`           | 编辑文件      |
| `exec`           | 执行命令      |
| `grep`           | 搜索内容      |
| `message`        | 发送消息      |
| `gateway`        | 配置/更新操作 |
| `sessions_spawn` | 创建子代理    |
| `subagents`      | 管理子代理    |
| `cron`           | 定时任务      |

### 3.9 Context 上下文

| 属性         | 值                                                            |
| ------------ | ------------------------------------------------------------- |
| **核心目录** | `src/context-engine/`                                         |
| **核心文档** | `docs/concepts/context.md`, `docs/concepts/context-engine.md` |
| **学习目标** | 理解 Token 计算、窗口管理、上下文引擎                         |

### 3.10 Prompt 系统提示词

| 属性         | 值                                                                    |
| ------------ | --------------------------------------------------------------------- |
| **核心文件** | `src/agents/system-prompt.ts`                                         |
| **核心文档** | `docs/concepts/system-prompt.md`                                      |
| **学习目标** | 理解 Sections、Cache Boundary、Bootstrap Files、Provider Contribution |

**Prompt Sections**：

```
┌─────────────────────────────────────────────────┐
│  Tooling          → 工具使用指南                 │
│  Execution Bias   → 执行偏好（行动导向）          │
│  Safety           → 安全护栏                     │
│  Skills           → 可用技能列表                 │
│  OpenClaw Self-Update → 配置/更新操作指南        │
│  Workspace        → 工作目录                     │
│  Project Context  → AGENTS.md/SOUL.md 等注入    │
│  ─────────── Cache Boundary ───────────         │
│  Current Date & Time → 时区信息                 │
│  Messaging        → 消息发送指南                 │
│  Runtime          → 运行时信息                   │
│  Heartbeats       → 心跳行为指南                 │
└─────────────────────────────────────────────────┘
```

**Bootstrap Files**：

| 文件           | 作用                     |
| -------------- | ------------------------ |
| `AGENTS.md`    | 项目规则、命令、架构指南 |
| `SOUL.md`      | 人格定义（语气、风格）   |
| `IDENTITY.md`  | 身份说明                 |
| `USER.md`      | 用户信息                 |
| `TOOLS.md`     | 工具使用指南             |
| `HEARTBEAT.md` | 心跳任务定义             |
| `BOOTSTRAP.md` | 新工作空间引导（仅首次） |
| `MEMORY.md`    | 记忆索引                 |

### 3.11 Sandbox 沙箱

| 属性         | 值                                                                                 |
| ------------ | ---------------------------------------------------------------------------------- |
| **核心目录** | `src/agents/sandbox/`                                                              |
| **核心文档** | `docs/gateway/sandboxing.md`, `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md` |
| **CLI 文档** | `docs/cli/sandbox.md`                                                              |
| **学习目标** | 理解沙箱隔离机制、后端类型、工具执行环境                                           |

**核心概念**：

| 概念                | 选项                           | 说明                       |
| ------------------- | ------------------------------ | -------------------------- |
| **Mode**            | `off` / `non-main` / `all`     | 控制何时启用沙箱           |
| **Scope**           | `agent` / `session` / `shared` | 控制容器数量和共享范围     |
| **Backend**         | `docker` / `ssh` / `openshell` | 控制沙箱运行位置           |
| **WorkspaceAccess** | `none` / `ro` / `rw`           | 控制沙箱对工作区的访问权限 |

**核心文件**：

| 文件             | 职责                                   |
| ---------------- | -------------------------------------- |
| `context.ts`     | resolveSandboxContext - 解析沙箱上下文 |
| `config.ts`      | 配置解析和合并                         |
| `backend.ts`     | 后端注册和管理                         |
| `docker.ts`      | Docker 后端实现                        |
| `ssh.ts`         | SSH 后端实现                           |
| `fs-bridge.ts`   | 文件系统桥接（读写操作代理）           |
| `tool-policy.ts` | 沙箱工具策略                           |
| `types.ts`       | 类型定义                               |

**后端对比**：

| 后端          | 运行位置                 | 适用场景                   |
| ------------- | ------------------------ | -------------------------- |
| **Docker**    | 本地容器                 | 本地开发、完全隔离         |
| **SSH**       | SSH 可访问的远程主机     | 负载转移到远程机器         |
| **OpenShell** | OpenShell 管理的远程环境 | 托管远程沙箱、可选双向同步 |

**沙箱执行流程**：

```
runEmbeddedAttempt()
    │
    ├── resolveSandboxContext() → 解析沙箱配置
    │   ├── mode: off → 无沙箱
    │   ├── mode: non-main + isMain → 无沙箱
    │   └── mode: non-main + !isMain → 启用沙箱
    │   └── mode: all → 启用沙箱
    │
    ├── effectiveWorkspace 计算
    │   ├── workspaceAccess: none → ~/.openclaw/sandboxes
    │   ├── workspaceAccess: ro → /agent (只读)
    │   └── workspaceAccess: rw → /workspace (读写)
    │
    ├── 工具执行
    │   ├── 通过 FS Bridge → 沙箱内执行
    │   └── elevated: true → 绕过沙箱（需授权）
    │
    └── 结果返回
```

**与 Tool Policy / Elevated 的关系**：

```
Tool Policy (which tools exist)
    │
    ├── deny → 工具不可用（沙箱无法恢复）
    └── allow → 工具可用
              │
              ↓
Sandbox (where tools run)
    │
    ├── mode: off → 主机执行
    └── mode: on → 沙箱执行
              │
              ↓
Elevated (exec escape hatch)
    │
    ├── enabled + authorized → 绕过沙箱执行
    └── disabled → 沙箱内执行
```

---

## 阶段 4: 会话与记忆

理解会话生命周期和记忆系统。

### 4.1 Session 会话

| 属性         | 值                                                |
| ------------ | ------------------------------------------------- |
| **核心目录** | `src/sessions/`                                   |
| **核心文档** | `docs/reference/session-management-compaction.md` |
| **学习文档** | `learn/session-management.md`                     |
| **学习目标** | 理解会话生命周期、状态管理、transcript            |

**Session Key 格式**：

```
agent:<agentId>:<channel>:<peerKind>:<peerId>

示例: agent:main:feishu:direct:ou_xxx
```

### 4.2 Compaction 压缩

| 属性         | 值                                         |
| ------------ | ------------------------------------------ |
| **核心文件** | `src/agents/pi-embedded-runner/compact.ts` |
| **核心文档** | `docs/concepts/compaction.md`              |
| **学习目标** | 理解压缩触发条件、策略、保留关键信息       |

**压缩流程**：

```
旧消息 → LLM Summary → 替换为摘要 → 空出空间
```

**触发条件**：

- Context window 超出阈值
- Tool result 过长
- 显式触发 `/compact`

### 4.3 Memory 记忆

| 属性         | 值                                                          |
| ------------ | ----------------------------------------------------------- |
| **核心目录** | `src/memory/`                                               |
| **核心文档** | `docs/concepts/memory.md`, `docs/concepts/active-memory.md` |
| **学习目标** | 理解长期记忆存储、检索、daily files                         |

---

## 阶段 5: 高级能力

理解模型系统、多代理、外部协议。

### 5.1 Model 模型系统

| 属性         | 值                                                                                               |
| ------------ | ------------------------------------------------------------------------------------------------ |
| **核心目录** | `src/model-catalog/`                                                                             |
| **核心文档** | `docs/concepts/models.md`, `docs/concepts/model-providers.md`, `docs/concepts/model-failover.md` |
| **学习目标** | 理解模型选择、Failover、Provider Catalog                                                         |

**Failover 策略**：

```
glm-5 失败 → gpt-5.5 → sonnet-4.6 → ...
```

### 5.2 Multi-Agent 多代理

| 属性         | 值                                                                       |
| ------------ | ------------------------------------------------------------------------ |
| **核心目录** | `src/agents/subagent/`                                                   |
| **核心文档** | `docs/concepts/multi-agent.md`, `docs/concepts/delegate-architecture.md` |
| **学习目标** | 理解 sessions_spawn、subagents、delegation                               |

**子代理类型**：

| Runtime               | 用途             |
| --------------------- | ---------------- |
| `runtime: "subagent"` | OpenClaw 子代理  |
| `runtime: "acp"`      | ACP 外部 harness |

### 5.3 MCP 协议

| 属性         | 值                                        |
| ------------ | ----------------------------------------- |
| **核心目录** | `src/mcp/`                                |
| **学习目标** | 理解 Model Context Protocol、外部工具连接 |

### 5.4 Codex Harness

| 属性         | 值                                         |
| ------------ | ------------------------------------------ |
| **核心目录** | `extensions/codex/`                        |
| **核心文档** | `docs/plugins/codex-harness.md`            |
| **学习目标** | 理解 Plugin Runtime 实现、与 PI 的边界差异 |

---

## 阶段 6: 能力模块（可选）

按需选择学习的功能模块。

### 6.1 Web Search/Fetch

| 属性         | 值                                  |
| ------------ | ----------------------------------- |
| **核心目录** | `src/web-search/`, `src/web-fetch/` |
| **学习目标** | 理解网络搜索与内容抓取              |

### 6.2 Media Generation

| 属性         | 值                                                                                                 |
| ------------ | -------------------------------------------------------------------------------------------------- |
| **核心目录** | `src/media-generation/`, `src/image-generation/`, `src/video-generation/`, `src/music-generation/` |
| **学习目标** | 理解图片/视频/音乐生成                                                                             |

### 6.3 Realtime Voice

| 属性         | 值                                                               |
| ------------ | ---------------------------------------------------------------- |
| **核心目录** | `src/realtime-voice/`, `src/realtime-transcription/`, `src/tts/` |
| **学习目标** | 理解实时语音、语音识别、TTS                                      |

### 6.4 Security/Secrets

| 属性         | 值                                               |
| ------------ | ------------------------------------------------ |
| **核心目录** | `src/security/`, `src/secrets/`                  |
| **核心文档** | `docs/reference/secretref-credential-surface.md` |
| **学习目标** | 理解安全机制、密钥管理、权限控制                 |

### 6.5 Cron 定时任务

| 属性         | 值                     |
| ------------ | ---------------------- |
| **核心目录** | `src/cron/`            |
| **学习目标** | 理解定时任务、提醒机制 |

### 6.6 UI/TUI 界面

| 属性         | 值                    |
| ------------ | --------------------- |
| **核心目录** | `ui/`, `src/tui/`     |
| **核心文档** | `ui/AGENTS.md`        |
| **学习目标** | 理解 Web UI、终端界面 |

---

## ★ 横向主题: Message Flow

**Message Flow** 是贯穿所有阶段的完整执行链学习主题。

| 属性         | 值                                 |
| ------------ | ---------------------------------- |
| **学习目录** | `learn/message-flow/`              |
| **文档数量** | 15 个步骤文档 + 1 个 README        |
| **学习目标** | 理解一条消息从接收到回复的完整流程 |

### Message Flow 目录

| 文件                                                                              | 函数                                | 层级        | 作用                       |
| --------------------------------------------------------------------------------- | ----------------------------------- | ----------- | -------------------------- |
| [01-monitor-transport.md](message-flow/01-monitor-transport.md)                   | `monitorWebSocket/Webhook`          | 接入层      | WebSocket/Webhook 消息接收 |
| [02-monitor-message-handler.md](message-flow/02-monitor-message-handler.md)       | `createFeishuMessageReceiveHandler` | 解析层      | 事件解析、去重、debounce   |
| [03-handle-feishu-message.md](message-flow/03-handle-feishu-message.md)           | `handleFeishuMessage`               | 业务层      | 权限检查、路由解析         |
| [04-run-channel-turn.md](message-flow/04-run-channel-turn.md)                     | `runChannelTurn`                    | 编排层      | Turn 生命周期管理          |
| [05-dispatch-reply-from-config.md](message-flow/05-dispatch-reply-from-config.md) | `dispatchReplyFromConfig`           | **Layer 1** | 分发协调器                 |
| [06-get-reply-from-config.md](message-flow/06-get-reply-from-config.md)           | `getReplyFromConfig`                | **Layer 2** | 回复准备器                 |
| [07-run-prepared-reply.md](message-flow/07-run-prepared-reply.md)                 | `runPreparedReply`                  | **Layer 3** | 执行准备器                 |
| [08-run-reply-agent.md](message-flow/08-run-reply-agent.md)                       | `runReplyAgent`                     | **Layer 4** | Agent 编排器               |
| [09-run-agent-turn.md](message-flow/09-run-agent-turn.md)                         | `runAgentTurnWithFallback`          | 执行层      | Agent Turn 执行            |
| [10-run-embedded-pi-agent.md](message-flow/10-run-embedded-pi-agent.md)           | `runEmbeddedPiAgent`                | Pi层        | 嵌入式 Pi Agent            |
| [11-run-agent-harness-attempt.md](message-flow/11-run-agent-harness-attempt.md)   | `runAgentHarnessAttempt`            | Harness层   | Harness 尝试               |
| [12-run-harness-v2-lifecycle.md](message-flow/12-run-harness-v2-lifecycle.md)     | `runHarnessV2LifecycleAttempt`      | V2层        | Harness V2 生命周期        |
| [13-provider-stream-completion.md](message-flow/13-provider-stream-completion.md) | `Provider.streamCompletion`         | Provider层  | LLM API 调用               |
| [14-reply-dispatcher.md](message-flow/14-reply-dispatcher.md)                     | `ReplyDispatcher`                   | 分发层      | 回复分发器                 |
| [15-send-message-feishu.md](message-flow/15-send-message-feishu.md)               | `sendMessageFeishu`                 | 发送层      | 飞书消息发送               |

### Message Flow 架构

```
前置层（渠道接入，步骤 1-4）
    ↓
核心四层（回复生成，步骤 5-8）
    Layer 1: dispatchReplyFromConfig → 路由决策
    Layer 2: getReplyFromConfig       → 模型选择
    Layer 3: runPreparedReply         → Prompt 构建
    Layer 4: runReplyAgent            → LLM 调用
    ↓
执行层（LLM 调用，步骤 9-13）
    runAgentTurn → runEmbeddedPi → Harness → Provider API
    ↓
后置层（回复发送，步骤 14-15）
    ReplyDispatcher → 飞书 API
```

---

## 学习策略建议

### 策略 A: 纵向深入（模块优先）

适合：想系统掌握每个模块内部机制的学习者

```
阶段1 → 阶段2 → 阶段3 → 阶段4 → 阶段5 → 阶段6
                                              ↓
                                      最后读 Message Flow
                                      （串联所有模块）
```

### 策略 B: 横向切入（流程优先）

适合：想快速理解整体运作，再针对性深入的学习者

```
先读 Message Flow → 难点模块深入 → 继续下一阶段
        ↓
   理解整体执行链
        ↓
   针对性深入 PI Runner / Compaction 等
```

### 策略 C: 交替进行（推荐）

适合：想在理解模块后立即看它在实际流程中的位置

```
阶段 1-2（基础运行时）
        ↓
Message Flow 步骤 1-8（接入→编排）
        ↓
阶段 3（Agent 核心 + PI Runner）
        ↓
Message Flow 步骤 9-13（执行层）
        ↓
阶段 4-5（会话记忆 + 高级能力）
        ↓
Message Flow 步骤 14-15（发送层）
```

---

## 调试排查索引

根据问题现象定位对应学习主题：

| 问题现象       | 定位主题                 | 相关 Message Flow |
| -------------- | ------------------------ | ----------------- |
| 消息未收到     | Gateway, Channel         | 步骤 1-2          |
| 消息处理异常   | Channel, Hook            | 步骤 3-4          |
| Agent 执行失败 | PI Runner, Harness       | 步骤 9-12         |
| LLM 调用失败   | Model, Provider          | 步骤 13           |
| 回复发送失败   | Channel, ReplyDispatcher | 步骤 14-15        |
| 上下文过大     | Context, Compaction      | Session 管理      |
| 工具执行异常   | Tools, Tool Policy       | 工具调用分支      |
| 多代理协作问题 | Multi-Agent, ACP         | sessions_spawn    |

---

## 延伸阅读

| 资源             | 位置                                                   |
| ---------------- | ------------------------------------------------------ |
| **官方文档**     | `docs/` 目录，Mintlify 发布于 https://docs.openclaw.ai |
| **项目规则**     | `AGENTS.md`（根目录及各子目录）                        |
| **测试用例**     | `*.test.ts` 文件，理解实际行为                         |
| **现有学习文档** | `learn/*.md`                                           |

---

> **文档版本**: 2026-05-05
> **分析基于**: OpenClaw v2026.5.3
> **主题总数**: 29 个
