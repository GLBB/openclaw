# OpenClaw 会话管理模块详解

基于旧版文档与当前代码库梳理，帮助理解会话管理模块的核心架构与实现细节。

---

## 一、核心概念：会话作为记忆中枢

### 1.1 什么是 Session？

**Session（会话）** 是 OpenClaw 中承载 Agent 对话记忆的核心单元。每个会话包含：

| 组成部分       | 说明                                      |
| -------------- | ----------------------------------------- |
| **对话历史**   | 以 JSONL 格式存储的 transcript 文件       |
| **上下文状态** | token 计数、模型信息、投递上下文          |
| **会话身份**   | sessionId（UUID）、sessionKey（路由标识） |
| **运行时状态** | thinking 等级、模型覆盖、provider 覆盖等  |

**核心类型定义** 位于 [`src/config/sessions/types.ts`](src/config/sessions/types.ts)：

```typescript
type SessionEntry = {
  sessionId: string; // UUID，对应 transcript 文件名
  updatedAt: number; // 最后活动时间戳（毫秒）
  sessionFile?: string; // Transcript 文件路径
  totalTokens?: number; // 上下文使用量
  model?: string; // 运行时模型
  modelProvider?: string; // 运行时 provider
  thinkingLevel?: string; // 用户设置的思考等级
  verboseLevel?: string; // 详细模式等级
  reasoningLevel?: string; // 推理等级
  ttsAuto?: TtsAutoMode; // TTS 自动模式
  deliveryContext?: {
    // 投递上下文（用于回复路由）
    lastChannel?: string;
    lastTo?: string;
    lastAccountId?: string;
    lastThreadId?: string;
  };
  // ... 更多字段
};
```

### 1.2 会话初始化入口

[`src/auto-reply/reply/session.ts`](src/auto-reply/reply/session.ts) 中的 `initSessionState()` 是会话生命周期的核心入口：

```typescript
export async function initSessionState(params: {
  ctx: MsgContext; // 消息上下文
  cfg: OpenClawConfig; // 配置
  commandAuthorized: boolean;
}): Promise<SessionInitResult>;
```

---

## 二、Session Key：路由标识符

### 2.1 Session Key 的作用

**Session Key** 决定消息被路由到哪个会话。它是会话存储的键，而非文件名。

**标准格式：**

```
agent:<agentId>:<rest>
```

### 2.2 各种 Session Key 示例

| 场景             | Session Key 格式                         |
| ---------------- | ---------------------------------------- |
| 默认主会话       | `agent:main:main`                        |
| 按用户隔离的私聊 | `agent:main:direct:+1234567890`          |
| 按渠道+用户隔离  | `agent:main:whatsapp:direct:+1234567890` |
| 群聊会话         | `agent:main:telegram:group:12345`        |
| 话题/线程会话    | `agent:main:whatsapp:thread:abc123`      |
| 定时任务会话     | `agent:main:cron:myjob:run:timestamp`    |
| 子代理会话       | `agent:main:subagent:child-id:...`       |
| ACP 会话         | `agent:main:acp:runtime-name`            |

### 2.3 Session Key 解析工具

[`src/sessions/session-key-utils.ts`](src/sessions/session-key-utils.ts) 提供解析函数：

```typescript
// 解析 agent 作用域的 session key
function parseAgentSessionKey(sessionKey: string): {
  agentId: string;
  rest: string;
} | null;

// 判断是否为定时任务会话
function isCronSessionKey(sessionKey: string): boolean;

// 判断是否为子代理会话
function isSubagentSessionKey(sessionKey: string): boolean;

// 判断是否为 ACP 会话
function isAcpSessionKey(sessionKey: string): boolean;

// 解析线程后缀
function parseThreadSessionSuffix(sessionKey: string): {
  baseSessionKey: string | undefined;
  threadId: string | undefined;
};
```

---

## 三、dmScope：四种隔离模式

### 3.1 配置项说明

`session.dmScope` 控制私聊消息如何分组到不同会话：

```json
{
  "session": {
    "dmScope": "per-peer" // 可选值见下表
  }
}
```

### 3.2 四种隔离模式对比

| 模式                       | Session Key 格式                                        | 适用场景                                         |
| -------------------------- | ------------------------------------------------------- | ------------------------------------------------ |
| `main`                     | `agent:<agentId>:main`                                  | 单用户环境，所有私聊共享一个会话（多用户时危险） |
| `per-peer`                 | `agent:<agentId>:direct:<peerId>`                       | 每个用户独立会话，不区分渠道                     |
| `per-channel-peer`         | `agent:<agentId>:<channel>:direct:<peerId>`             | 每个渠道内的每个用户独立会话                     |
| `per-account-channel-peer` | `agent:<agentId>:<channel>:<accountId>:direct:<peerId>` | 最完整隔离，区分账号、渠道、用户                 |

### 3.3 核心实现

[`src/routing/session-key.ts`](src/routing/session-key.ts) 中的 `buildAgentPeerSessionKey()` 函数负责构建私聊 session key：

```typescript
function buildAgentPeerSessionKey(params: {
  agentId: string;
  channel: string;
  accountId?: string;
  peerId: string;
  dmScope: DmScope;
}): string;
```

---

## 四、会话生命周期

### 4.1 生命周期阶段

```
┌─────────────────────────────────────────────────────────────┐
│                     会话生命周期                              │
├─────────────────────────────────────────────────────────────┤
│  1. 创建    ──▶  首条消息触发 initSessionState()              │
│  2. 活跃    ──▶  消息更新 updatedAt 时间戳                    │
│  3. 过期    ──▶  超过 reset 阈值（daily/idle）                │
│  4. 重置    ──▶  /new、/reset 触发或定时重置                  │
│  5. 归档    ──▶  旧 transcript 移至 .archived 目录            │
│  6. 清理    ──▶  维护任务删除过期条目                         │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 重置触发条件

**用户主动触发：**

- 发送 `/new` 命令
- 发送 `/reset` 命令
- 可配置自定义触发词

**自动触发（配置）：**

```json
{
  "session": {
    "reset": {
      "daily": "04:00", // 每日 4:00 重置所有过期会话
      "idleMinutes": 60 // 空闲 60 分钟后标记为过期
    },
    "resetByType": {
      "direct": { "idleMinutes": 120 },
      "group": { "daily": "06:00" },
      "thread": { "idleMinutes": 30 }
    }
  }
}
```

### 4.3 重置时的行为

当会话被重置时（`initSessionState()` 中）：

1. **生成新 sessionId** - 新的 UUID
2. **创建新 transcript 文件** - 清空的对话历史
3. **归档旧 transcript** - 移至 `.archived` 目录
4. **保留用户设置** - thinking/verbose/reasoning 等用户偏好会被保留
5. **清除 token 计数** - 避免显示旧会话的上下文使用量
6. **触发插件钩子** - `session_end` 和 `session_start`

**核心实现：** [`src/config/sessions/reset.ts`](src/config/sessions/reset.ts)

```typescript
function evaluateSessionFreshness(params: {
  updatedAt: number;
  now: number;
  policy: ResetPolicy;
}): SessionFreshness; // { fresh: boolean, dailyResetAt?, idleExpiresAt? }
```

---

## 五、Session Store 与持久化

### 5.1 存储位置

```
~/.openclaw/agents/<agentId>/sessions.json
```

**文件结构：**

```json
{
  "agent:main:whatsapp:direct:+123456": {
    "sessionId": "uuid-xxx",
    "updatedAt": 1704067200000,
    "sessionFile": "uuid-xxx.jsonl",
    "totalTokens": 50000
    // ...
  },
  "agent:main:telegram:group:12345": {
    // ...
  }
}
```

### 5.2 Store 操作 API

[`src/config/sessions/store.ts`](src/config/sessions/store.ts) 提供核心 API：

```typescript
// 加载会话存储（支持缓存跳过）
function loadSessionStore(
  storePath: string,
  options?: { skipCache?: boolean },
): Record<string, SessionEntry>;

// 更新会话存储
async function updateSessionStore(
  storePath: string,
  mutate: (store: Record<string, SessionEntry>) => void,
  options?: { activeSessionKey?: string; onWarn?: (warning) => void },
): Promise<SessionEntry>;

// 合并会话条目（保留旧值）
function mergeSessionEntry(existing: SessionEntry | undefined, next: SessionEntry): SessionEntry;
```

### 5.3 维护机制

[`src/config/sessions/store-maintenance.ts`](src/config/sessions/store-maintenance.ts) 实现三项维护：

#### 修剪过期条目（Prune）

```json
{
  "session": {
    "maintenance": {
      "pruneAfter": "30d" // 删除超过 30 天未活跃的会话
    }
  }
}
```

#### 条目数量上限（Cap）

```json
{
  "session": {
    "maintenance": {
      "maxEntries": 500 // 最多保留 500 个会话条目
    }
  }
}
```

#### 文件轮转（Rotate）

```json
{
  "session": {
    "maintenance": {
      "rotateBytes": "10MB" // sessions.json 超过 10MB 时轮转
    }
  }
}
```

---

## 六、Transcript 文件格式

### 6.1 文件位置

```
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
```

### 6.2 JSONL 行格式

每行是一个 JSON 对象，类型包括：

**消息行：**

```json
{
  "type": "message",
  "message": {
    "role": "user",
    "content": "你好"
  },
  "parentId": null,
  "id": "msg-1"
}
```

**助手回复：**

```json
{
  "type": "message",
  "message": {
    "role": "assistant",
    "content": "你好！有什么可以帮助你的？"
  },
  "parentId": "msg-1",
  "id": "msg-2"
}
```

**压缩摘要（Compaction）：**

```json
{
  "type": "branch_summary",
  "summary": "用户询问了关于 X 的内容，助手回答了 Y...",
  "fromId": "msg-50",
  "parentId": "msg-1"
}
```

### 6.3 Transcript 操作 API

[`src/config/sessions/transcript.ts`](src/config/sessions/transcript.ts) 提供文件操作：

```typescript
// 读取 transcript 行
function readSessionTranscriptLines(sessionFile: string, agentId: string): TranscriptLine[];

// 追加消息
async function appendSessionTranscriptMessage(
  sessionFile: string,
  message: Message,
  options: { parentId?: string; id?: string },
): Promise<string>; // 返回消息 ID
```

---

## 七、路由与绑定机制

### 7.1 消息路由流程

```
┌─────────────────────────────────────────────────────────────┐
│                     消息到达                                  │
│   (Telegram/Discord/WhatsApp/Slack 等)                       │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                  resolveAgentRoute()                         │
│   - 检查 bindings 优先级                                      │
│   - 确定 agentId + sessionKey                                │
│   [src/routing/resolve-route.ts]                             │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                  initSessionState()                          │
│   - 加载 sessions.json                                       │
│   - 检查 freshness（daily/idle reset）                       │
│   - 处理 /new、/reset 触发                                   │
│   - 创建 sessionId + sessionFile                             │
│   [src/auto-reply/reply/session.ts]                          │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Agent 执行                                  │
│   - 加载 transcript 作为上下文                                │
│   - 调用 LLM                                                 │
│   - 追加回复到 transcript                                    │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Binding 优先级

[`src/routing/resolve-route.ts`](src/routing/resolve-route.ts) 定义绑定检查顺序：

```
binding.peer → binding.peer.parent → binding.peer.wildcard
→ binding.guild+roles → binding.guild → binding.team
→ binding.account → binding.channel → default
```

### 7.3 Binding 配置示例

```json
{
  "bindings": [
    {
      "peer": "+1234567890",
      "channel": "whatsapp",
      "agent": "support-bot"
    },
    {
      "guild": "12345",
      "channel": "discord",
      "agent": "community-bot",
      "sessionKey": "agent:community-bot:discord:group:12345"
    },
    {
      "channel": "telegram",
      "agent": "default-bot"
    }
  ]
}
```

---

## 八、ACP 会话（Agent Control Plane）

### 8.1 ACP 会话特点

ACP（Agent Control Plane）会话支持持久运行时会话，具有额外元数据：

```typescript
type SessionAcpMeta = {
  backend: string; // 后端类型（如 "claude-code"）
  agent: string; // Agent 名称
  runtimeSessionName: string; // 运行时会话名
  mode: "persistent" | "oneshot";
  state: "idle" | "running" | "error";
  lastActiveAt?: number;
  error?: string;
};
```

### 8.2 ACP 会话存储

[`src/acp/session.ts`](src/acp/session.ts) 管理 ACP 会话：

```typescript
// ACP 会话存储位置
// ~/.openclaw/acp-sessions.json

type AcpSessionStore = Record<string, AcpSessionEntry>;
```

### 8.3 ACP 与普通会话的关系

- **普通会话**：每次对话后 transcript 写入磁盘，下次加载恢复
- **ACP 会话**：运行时保持活跃，支持长时间后台任务、流式输出等

---

## 九、架构总览图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        消息入口层                                    │
│   Telegram │ Discord │ WhatsApp │ Slack │ Signal │ iMessage │ Web   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        路由层                                        │
│   [src/routing/resolve-route.ts]                                    │
│   - Binding 匹配（peer/guild/team/account/channel）                  │
│   - 确定 agentId + sessionKey                                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        会话初始化层                                  │
│   [src/auto-reply/reply/session.ts] - initSessionState()            │
│   - 加载 Session Store                                              │
│   - 检查 Freshness（过期判断）                                       │
│   - 处理重置触发                                                     │
│   - 创建/更新 SessionEntry                                          │
│   - 归档旧 Transcript                                                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        会话存储层                                    │
│   ~/.openclaw/agents/<agentId>/sessions.json                        │
│   [src/config/sessions/store.ts]                                    │
│   - Map<sessionKey, SessionEntry>                                   │
│   - 维护：修剪/上限/轮转                                             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Transcript 层                                 │
│   ~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl            │
│   [src/config/sessions/transcript.ts]                               │
│   - 对话历史（JSONL 格式）                                           │
│   - 支持 Compaction 压缩                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 十、关键文件索引

| 功能模块         | 关键文件                                   |
| ---------------- | ------------------------------------------ |
| 会话类型定义     | `src/config/sessions/types.ts`             |
| 会话初始化       | `src/auto-reply/reply/session.ts`          |
| Session Store    | `src/config/sessions/store.ts`             |
| Store 维护       | `src/config/sessions/store-maintenance.ts` |
| 重置策略         | `src/config/sessions/reset.ts`             |
| Transcript       | `src/config/sessions/transcript.ts`        |
| Session Key 构建 | `src/routing/session-key.ts`               |
| Session Key 解析 | `src/sessions/session-key-utils.ts`        |
| 路由解析         | `src/routing/resolve-route.ts`             |
| Binding 管理     | `src/routing/bindings.ts`                  |
| ACP 会话         | `src/acp/session.ts`                       |
| ACP 元数据       | `src/acp/runtime/session-meta.ts`          |
| 会话命令更新     | `src/agents/command/session-store.ts`      |

---

## 参考文档链接

- [Configuration](https://docs.openclaw.ai/configuration) - 会话配置详解
- [Architecture](https://docs.openclaw.ai/concepts/architecture) - 整体架构概览
- [Session Maintenance](https://docs.openclaw.ai/gateway/session-maintenance) - 维护机制文档
- [Channel Plugins](https://docs.openclaw.ai/plugins/sdk-channel-plugins) - 渠道插件集成
