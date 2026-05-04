# OpenClaw 通道路由与消息分发模块详解

基于旧版文档与当前代码库梳理，帮助理解通道路由与消息分发的核心架构与实现细节。

---

## 一、三层架构概览

OpenClaw 采用三层架构处理消息的接收、路由与分发：

```
┌─────────────────────────────────────────────────────────────────┐
│                    LLM/Agent 层（大脑与执行）                      │
│   Agent Core · Memory · Skills · Tools                          │
│   执行推理、调用工具、生成回复                                      │
└───────────────────────────────────────────┬─────────────────────┘
                                            │ 调用/执行
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Gateway 层（路由与安全中枢）                    │
│   安全鉴权 Auth · 消息路由 Router · 会话管理 Session              │
│   统一入口、权限验证、路由决策、会话持久化                           │
└───────────────────────────────────────────┬─────────────────────┘
                                            │ 标准化 Event
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Channels 层（外部输入输出）                     │
│   Telegram · WhatsApp · Discord · Slack · Signal · iMessage     │
│   消息接收、格式转换、消息发送                                       │
└─────────────────────────────────────────────────────────────────┘
```

**核心设计理念：**

- **Channels 层**负责将不同平台的原始事件标准化为 OpenClaw 内部统一格式
- **Gateway 层**作为唯一入口，执行路由决策和安全检查
- **Agent 层**只处理标准化消息，不感知具体渠道差异

---

## 二、Channel Plugin 接口

### 2.1 接口设计目的

每个消息平台（WhatsApp、Telegram、Discord 等）实现统一的 `ChannelPlugin` 接口，使核心路由代码无需陷入平台特定的条件分支。

**核心类型定义** 位于 [`src/channels/plugins/types.plugin.ts`](src/channels/plugins/types.plugin.ts):

```typescript
type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  id: ChannelId; // 平台标识: "whatsapp"、"telegram" 等
  meta: ChannelMeta; // 展示信息: 名称、文档路径、图标
  capabilities: ChannelCapabilities; // 支持哪些能力: 群组、投票、媒体、编辑消息等
  config: ChannelConfigAdapter; // 【必须】配置读取/解析
  security?: ChannelSecurityAdapter; // 【可选】dmPolicy、allowFrom 等安全策略
  outbound?: ChannelOutboundAdapter; // 【可选】发送消息的实现
  pairing?: ChannelPairingAdapter; // 【可选】二维码配对流程
  groups?: ChannelGroupAdapter; // 【可选】群组管理
  mentions?: ChannelMentionAdapter; // 【可选】提及解析
  status?: ChannelStatusAdapter; // 【可选】状态探测
  gateway?: ChannelGatewayAdapter; // 【可选】Gateway 接入方法
  commands?: ChannelCommandAdapter; // 【可选】命令处理
  approvals?: ChannelApprovalAdapter; // 【可选】审批流程
  // ... 更多适配器
};
```

### 2.2 适配器分类

| 适配器类型          | 用途                            | 实现要求         |
| ------------------- | ------------------------------- | ---------------- |
| **ConfigAdapter**   | 读取渠道配置、解析账号          | 必须             |
| **SecurityAdapter** | 安全策略（dmPolicy、allowFrom） | 推荐             |
| **OutboundAdapter** | 发送消息到外部平台              | 消息渠道必须     |
| **PairingAdapter**  | 配对流程（扫码、验证码）        | 用户交互渠道推荐 |
| **StatusAdapter**   | 探测连接状态、账号信息          | 监控场景推荐     |
| **GroupAdapter**    | 群组列表、成员管理              | 群聊渠道可选     |

### 2.3 核心适配器详解

**ConfigAdapter** ([`src/channels/plugins/types.adapters.ts`](src/channels/plugins/types.adapters.ts)):

```typescript
type ChannelConfigAdapter<ResolvedAccount = any> = {
  // 解析配置中的账号信息
  resolveAccounts?: (cfg: OpenClawConfig) => ResolvedAccount[];
  // 查找默认账号
  resolveDefaultAccount?: (
    cfg: OpenClawConfig,
    options?: { channelId?: string },
  ) => ResolvedAccount | undefined;
  // 读取账号特定配置
  resolveAccountConfig?: (cfg: OpenClawConfig, accountId: string) => Record<string, unknown>;
};
```

**SecurityAdapter**:

```typescript
type ChannelSecurityAdapter<ResolvedAccount = any> = {
  // 私聊消息的安全策略
  dmPolicy?: (params: { cfg: OpenClawConfig; account: ResolvedAccount }) => DmPolicy;
  // 允许的消息来源
  allowFrom?: (params: { cfg: OpenClawConfig; account: ResolvedAccount }) => AllowFromPolicy;
};
```

---

## 三、路由决策流程

### 3.1 路由决策入口

**核心文件**：[`src/routing/resolve-route.ts`](src/routing/resolve-route.ts)

`resolveAgentRoute()` 函数是路由决策的核心入口：

```typescript
function resolveAgentRoute(input: ResolveAgentRouteInput): ResolvedAgentRoute;

type ResolveAgentRouteInput = {
  cfg: OpenClawConfig;
  channel: string; // 渠道标识
  accountId?: string; // 账号 ID
  peer?: RoutePeer; // 消息来源（私聊用户、群组、频道）
  parentPeer?: RoutePeer; // 父级 peer（线程继承场景）
  guildId?: string; // Discord 服务器 ID
  teamId?: string; // Slack 团队 ID
  memberRoleIds?: string[]; // Discord 成员角色 ID
};
```

### 3.2 Binding 匹配优先级

路由按以下优先级顺序匹配 Binding：

| 优先级 | 匹配类型           | matchedBy 字段          | 说明                                |
| ------ | ------------------ | ----------------------- | ----------------------------------- |
| 1      | **精确 peer 匹配** | `binding.peer`          | peer.kind + peer.id 精确匹配        |
| 2      | **父级 peer 匹配** | `binding.peer.parent`   | 线程/话题继承父级路由               |
| 3      | **peer 通配匹配**  | `binding.peer.wildcard` | 如 `group:*` 匹配所有群组           |
| 4      | **Guild + 角色**   | `binding.guild+roles`   | Discord: guildId + roles 组合匹配   |
| 5      | **Guild 匹配**     | `binding.guild`         | Discord: 仅 guildId 匹配            |
| 6      | **Team 匹配**      | `binding.team`          | Slack: teamId 匹配                  |
| 7      | **Account 匹配**   | `binding.account`       | accountId 匷配                      |
| 8      | **Channel 匹配**   | `binding.channel`       | accountId: "\*" 匹配任意账号        |
| 9      | **默认 Agent**     | `default`               | 兜底：agents.default 或第一个 agent |

**匹配规则：**

- 当 Binding 包含多个匹配字段（peer、guildId、teamId、roles）时，**所有字段都必须匹配**
- 同一优先级内按配置顺序匹配
- 优先级越高的 Binding 越先被检查

### 3.3 Binding 配置示例

```json5
{
  bindings: [
    // 精确 peer 匹配：特定 Telegram 群组路由到 support agent
    {
      match: { channel: "telegram", peer: { kind: "group", id: "-100123456" } },
      agentId: "support",
    },

    // Guild + 角色匹配：Discord 管理员消息路由到 ops agent
    {
      match: { channel: "discord", guildId: "G789", roles: ["admin", "moderator"] },
      agentId: "ops",
    },

    // Team 匹配：特定 Slack 团队路由到 work agent
    { match: { channel: "slack", teamId: "T123" }, agentId: "work" },

    // Account 匹配：特定 WhatsApp 商务账号路由到 business agent
    { match: { channel: "whatsapp", accountId: "business-account" }, agentId: "business" },

    // Channel 匹配：所有 Telegram 消息（任意账号）路由到 telegram-bot
    { match: { channel: "telegram", accountId: "*" }, agentId: "telegram-bot" },
  ],
}
```

### 3.4 路由结果结构

```typescript
type ResolvedAgentRoute = {
  agentId: string; // 目标 Agent ID
  channel: string; // 渠道标识
  accountId: string; // 账号 ID（规范化后）
  sessionKey: string; // 内部会话键（用于持久化）
  mainSessionKey: string; // DM collapse 别名（用于私聊合并场景）
  lastRoutePolicy: "main" | "session"; // lastRoute 更新策略
  matchedBy: string; // 匹配方式（用于调试日志）
};
```

---

## 四、Session Key 构造规则

### 4.1 Session Key 的作用

**Session Key** 是会话持久化和并发控制的关键标识：

- 作为 `sessions.json` 中的键，存储会话状态
- 决定消息被路由到哪个会话桶（bucket）
- 控制会话隔离级别

**标准格式：**

```
agent:<agentId>:<rest>
```

### 4.2 各种场景的 Session Key

| 场景                                 | Session Key 格式                                        | 示例                                          |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------- |
| **默认主会话**                       | `agent:<agentId>:<mainKey>`                             | `agent:main:main`                             |
| **私聊（per-peer）**                 | `agent:<agentId>:direct:<peerId>`                       | `agent:main:direct:+123456`                   |
| **私聊（per-channel-peer）**         | `agent:<agentId>:<channel>:direct:<peerId>`             | `agent:main:whatsapp:direct:+123456`          |
| **私聊（per-account-channel-peer）** | `agent:<agentId>:<channel>:<accountId>:direct:<peerId>` | `agent:main:whatsapp:business:direct:+123456` |
| **群组**                             | `agent:<agentId>:<channel>:group:<groupId>`             | `agent:main:telegram:group:-100123`           |
| **频道**                             | `agent:<agentId>:<channel>:channel:<channelId>`         | `agent:main:discord:channel:456789`           |
| **线程/话题**                        | 基础key + `:thread:<threadId>`                          | `agent:main:discord:channel:456:thread:789`   |
| **Telegram 话题**                    | 基础key + `:topic:<topicId>`                            | `agent:main:telegram:group:-100:topic:42`     |

### 4.3 dmScope：四种隔离模式

`session.dmScope` 配置控制私聊消息如何分组到不同会话：

| 模式                       | Session Key 格式                                        | 适用场景                                                 |
| -------------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| `main`                     | `agent:<agentId>:main`                                  | 单用户环境，所有私聊共享一个会话（**多用户环境不推荐**） |
| `per-peer`                 | `agent:<agentId>:direct:<peerId>`                       | 每个用户独立会话，不区分渠道                             |
| `per-channel-peer`         | `agent:<agentId>:<channel>:direct:<peerId>`             | 同一用户在不同渠道使用不同会话                           |
| `per-account-channel-peer` | `agent:<agentId>:<channel>:<accountId>:direct:<peerId>` | 最完整隔离，区分账号、渠道、用户                         |

**配置示例：**

```json5
{
  session: {
    dmScope: "per-channel-peer", // 推荐：按渠道+用户隔离
  },
}
```

### 4.4 Session Key 构建函数

**核心文件**：[`src/routing/session-key.ts`](src/routing/session-key.ts)

```typescript
// 构建 Agent 主会话 Key
function buildAgentMainSessionKey(params: { agentId: string; mainKey?: string }): string;

// 构建 Agent 私聊会话 Key
function buildAgentPeerSessionKey(params: {
  agentId: string;
  channel: string;
  accountId?: string;
  peerKind?: ChatType;
  peerId?: string;
  dmScope?: DmScope;
  identityLinks?: Record<string, string[]>;
}): string;

// 构建线程 Session Key
function resolveThreadSessionKeys(params: {
  baseSessionKey: string;
  threadId?: string;
  parentSessionKey?: string;
}): { sessionKey: string; parentSessionKey?: string };

// 从 Session Key 解析 Agent ID
function resolveAgentIdFromSessionKey(sessionKey: string): string;
```

---

## 五、消息处理核心流程

### 5.1 会话初始化入口

**核心文件**：[`src/auto-reply/reply/session.ts`](src/auto-reply/reply/session.ts)

`initSessionState()` 函数处理消息到达后的会话初始化：

```typescript
async function initSessionState(params: {
  ctx: MsgContext; // 消息上下文（渠道、发送者、内容等）
  cfg: OpenClawConfig; // 配置
  commandAuthorized: boolean;
}): Promise<SessionInitResult>;
```

### 5.2 初始化流程步骤

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 解析会话绑定上下文                                            │
│     resolveConversationBindingContext(cfg, ctx)                  │
│     - 检查 ACP/Conversation 绑定                                  │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 规范化 Session Key                                           │
│     resolveSessionKey() + canonicalizeMainSessionAlias()         │
│     - 根据 dmScope 构建正确的 Key                                 │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 加载 Session Store                                           │
│     loadSessionStore(storePath, { skipCache: true })             │
│     - 读取 sessions.json（跳过缓存确保数据新鲜）                    │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 检测重置触发                                                  │
│     - 检查 /new、/reset 等命令                                    │
│     - 检查是否匹配 resetTriggers                                  │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 评估会话新鲜度                                                │
│     evaluateSessionFreshness()                                   │
│     - 检查 daily reset 时间点                                     │
│     - 检查 idle 过期时间                                          │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 创建/更新 SessionEntry                                       │
│     - isNewSession: 生成新 sessionId                             │
│     - freshEntry: 继承现有 entry                                  │
│     - 保留用户设置（thinking/verbose/reasoning）                   │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. 持久化 Session Store                                         │
│     updateSessionStore(storePath, mutate)                        │
│     - 写入 sessions.json                                         │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  8. 归档旧 Transcript                                             │
│     archiveSessionTranscriptsDetailed()                          │
│     - 将旧 transcript 移至 .archived 目录                         │
└───────────────────────────────────────────┬─────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  9. 触发插件钩子                                                  │
│     - session_end（旧会话结束）                                   │
│     - session_start（新会话开始）                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 返回结果结构

```typescript
type SessionInitResult = {
  sessionCtx: TemplateContext; // 模板上下文（用于 Agent prompt）
  sessionEntry: SessionEntry; // 当前会话条目
  previousSessionEntry?: SessionEntry; // 旧会话条目（用于归档）
  sessionStore: Record<string, SessionEntry>; // 完整 store
  sessionKey: string; // 规范化的 Session Key
  sessionId: string; // UUID
  isNewSession: boolean; // 是否新会话
  resetTriggered: boolean; // 是否被重置触发
  systemSent: boolean; // 系统消息已发送
  abortedLastRun: boolean; // 上次运行是否中断
  storePath: string; // Store 文件路径
  sessionScope: SessionScope; // 会话作用域
  groupResolution?: GroupKeyResolution; // 群组解析结果
  isGroup: boolean; // 是否群组消息
  bodyStripped?: string; // 去除命令前缀后的消息体
  triggerBodyNormalized: string; // 规范化的触发消息
};
```

---

## 六、回复路由机制

### 6.1 回复路由原理

OpenClaw 的回复路由遵循 **"回复到哪里来"** 原则：

- 消息从哪个渠道来，回复就发到哪个渠道
- Agent 不选择渠道，路由由配置和会话状态决定

### 6.2 DeliveryContext

每个会话条目维护 `deliveryContext`，记录最后消息来源：

```typescript
type DeliveryContext = {
  lastChannel?: string; // 最后消息来源渠道
  lastTo?: string; // 最后发送目标（用户/群组 ID）
  lastAccountId?: string; // 最后使用的账号
  lastThreadId?: string; // 最后线程/话题 ID
};
```

### 6.3 Main DM Route Pinning

当 `session.dmScope` 为 `main` 时，私聊共享一个主会话。为防止非所有者消息覆盖 `lastRoute`，系统会从 `allowFrom` 推断一个 **pinned owner**：

**推断条件（需全部满足）：**

1. `allowFrom` 只有唯一一个非通配符条目
2. 该条目可规范化为具体的发送者 ID
3. 当前私聊发送者不匹配该 pinned owner

**结果：**

- 匹配时：正常更新 lastRoute
- 不匹配时：仅记录会话元数据，不更新 lastRoute

---

## 七、广播组

### 7.1 广播组用途

当需要让 **多个 Agent** 同时响应同一消息时，使用广播组：

```json5
{
  broadcast: {
    strategy: "parallel", // 并行执行所有 Agent
    "120363403215116621@g.us": ["alfred", "baerbel"], // WhatsApp 群组
    "+15555550123": ["support", "logger"], // 电话号码
  },
}
```

### 7.2 工作流程

1. 消息到达 → 正常路由匹配确定一个主 Agent
2. 检查 broadcast 配置 → 找到匹配的 peer/group
3. 并行启动所有配置的 Agent
4. 各 Agent 独立处理，各自回复

---

## 八、关键数据结构

### 8.1 MsgContext

消息上下文，包含所有消息相关信息：

```typescript
type MsgContext = {
  Surface: string; // 渠道标识
  Provider: string; // 提供者标识
  SessionKey: string; // Session Key
  From: string; // 发送者 ID
  To: string; // 接收者 ID
  AccountId: string; // 账号 ID
  Body: string; // 消息内容
  RawBody: string; // 原始消息内容（可能含结构化前缀）
  CommandBody?: string; // 命令内容
  ChatType: ChatType; // 聊天类型：direct/group/channel/thread
  MessageThreadId?: string; // 线程/话题 ID
  ReplyToId?: string; // 回复的消息 ID
  ReplyToBody?: string; // 回复的消息内容
  ReplyToSender?: string; // 回复的发送者
  OriginatingChannel?: string; // 原始渠道（子代理 announce-back）
  // ... 更多字段
};
```

### 8.2 RoutePeer

路由匹配中的 peer 结构：

```typescript
type RoutePeer = {
  kind: ChatType; // direct/group/channel/thread
  id: string; // peer ID
};
```

### 8.3 EvaluatedBinding

评估后的 Binding 结构（用于路由匹配）：

```typescript
type EvaluatedBinding = {
  binding: AgentRouteBinding; // 原始 Binding 配置
  match: NormalizedBindingMatch; // 规范化的匹配条件
  order: number; // 配置顺序
};
```

---

## 九、缓存与性能优化

### 9.1 路由缓存

[`src/routing/resolve-route.ts`](src/routing/resolve-route.ts) 实现多层缓存：

```typescript
// Binding 评估缓存
const evaluatedBindingsCacheByCfg = new WeakMap<OpenClawConfig, EvaluatedBindingsCache>();

// 路由结果缓存
const resolvedRouteCacheByCfg = new WeakMap<OpenClawConfig, RouteCache>();

// 缓存上限
const MAX_EVALUATED_BINDINGS_CACHE_KEYS = 2000;
const MAX_RESOLVED_ROUTE_CACHE_KEYS = 4000;
```

### 9.2 缓存失效策略

- 配置引用变化时自动失效（WeakMap + ref 比较）
- 超过上限时清空重建
- identityLinks 存在时跳过缓存（动态链接）

---

## 十、调试技巧

### 10.1 开启调试日志

```bash
# 开启 verbose 日志（包含路由匹配详情）
OPENCLAW_VERBOSE=1

# 开启会话加载计时
OPENCLAW_DEBUG_INGRESS_TIMING=1
```

**日志示例：**

```
[routing] resolveAgentRoute: channel=telegram accountId=personal peer=group:-100123 guildId=none teamId=none bindings=3
[routing] binding: agentId=support accountPattern=default peer=group:-100123 guildId=none teamId=none roles=0
[routing] match: matchedBy=binding.peer agentId=support

session-init store-load agent=main session=agent:main:telegram:group:-100123 elapsedMs=12 path=/path/to/sessions.json
```

### 10.2 检查会话状态

```bash
# 查看 sessions.json
cat ~/.openclaw/agents/<agentId>/sessions.json | jq .

# 查看特定会话的 transcript
cat ~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl | head -20
```

### 10.3 理解路由决策

检查 `ResolvedAgentRoute.matchedBy` 字段：

- `binding.peer` → 精确 peer 匹配成功
- `binding.peer.parent` → 父级 peer 继承
- `binding.guild+roles` → Discord 角色匹配
- `default` → 兜底路由

---

## 十一、关键文件索引

| 功能模块                | 关键文件                                                                           |
| ----------------------- | ---------------------------------------------------------------------------------- |
| **路由决策**            | [`src/routing/resolve-route.ts`](src/routing/resolve-route.ts)                     |
| **Session Key 构建**    | [`src/routing/session-key.ts`](src/routing/session-key.ts)                         |
| **Binding 管理**        | [`src/routing/bindings.ts`](src/routing/bindings.ts)                               |
| **Account ID 规范化**   | [`src/routing/account-id.ts`](src/routing/account-id.ts)                           |
| **Channel Plugin 接口** | [`src/channels/plugins/types.plugin.ts`](src/channels/plugins/types.plugin.ts)     |
| **Channel 适配器类型**  | [`src/channels/plugins/types.adapters.ts`](src/channels/plugins/types.adapters.ts) |
| **会话初始化**          | [`src/auto-reply/reply/session.ts`](src/auto-reply/reply/session.ts)               |
| **Session Store**       | [`src/config/sessions/store.ts`](src/config/sessions/store.ts)                     |
| **重置策略**            | [`src/config/sessions/reset.ts`](src/config/sessions/reset.ts)                     |
| **群组 Session Key**    | [`src/config/sessions/group.ts`](src/config/sessions/group.ts)                     |
| **会话元数据**          | [`src/config/sessions/metadata.ts`](src/config/sessions/metadata.ts)               |

---

## 参考文档链接

- [Channel Routing](https://docs.openclaw.ai/channels/channel-routing) - 官方路由文档
- [Gateway Architecture](https://docs.openclaw.ai/concepts/architecture) - 整体架构概览
- [Channel Plugins SDK](https://docs.openclaw.ai/plugins/sdk-channel-plugins) - 渠道插件开发指南
- [Session Management](https://docs.openclaw.ai/concepts/session) - 会话管理概念
- [Broadcast Groups](https://docs.openclaw.ai/channels/broadcast-groups) - 广播组配置
