---
summary: "Gateway 模块开发者入门指南，采用索引型学习方式"
read_when:
  - 需要快速理解 Gateway 模块架构
  - 需要知道代码入口在哪里
  - 需要追踪数据流
title: "Gateway 模块入门"
---

# Gateway 模块入门

本文档帮助开发者快速理解 Gateway 模块，采用"索引型学习"而非"逐行阅读"。

---

## 学习流程概览

```
索引建立（记住入口）
  │
  ▼
边界理解（契约 + 不变量）
  │
  ▼
数据流追踪（核心流程）
  │
  ▼
实践验证（启动 + 测试）
  │
  ▼
按需深入（问题驱动）
```

---

## Step 1: 建立索引（记住"入口在哪"）

### 核心关心路径索引

| 你想知道...              | 入口文件                                                     | 关键函数/导出                                                   |
| ------------------------ | ------------------------------------------------------------ | --------------------------------------------------------------- |
| Gateway 如何启动？       | [server.impl.ts](src/gateway/server.impl.ts)                 | `startGatewayServer()`                                          |
| WebSocket 如何处理连接？ | [ws-connection.ts](src/gateway/server/ws-connection.ts)      | `attachGatewayWsConnectionHandler()`                            |
| 有哪些 RPC 方法？        | [server-methods-list.ts](src/gateway/server-methods-list.ts) | `BASE_METHODS`, `GATEWAY_EVENTS`                                |
| 方法如何分发？           | [server-methods.ts](src/gateway/server-methods.ts)           | `coreGatewayHandlers`, `handleGatewayRequest()`                 |
| 协议帧格式？             | [frames.ts](src/gateway/protocol/schema/frames.ts)           | `RequestFrameSchema`, `ResponseFrameSchema`, `EventFrameSchema` |
| 认证如何工作？           | [auth.ts](src/gateway/auth.ts)                               | `resolveGatewayAuth()`                                          |
| 设备签名验证？           | [device-auth.ts](src/gateway/device-auth.ts)                 | `verifyDeviceSignature()`                                       |
| Agent 如何执行？         | [agent.ts](src/gateway/server-methods/agent.ts)              | `agentHandlers`                                                 |
| Session 如何管理？       | [session-utils.ts](src/gateway/session-utils.ts)             | `loadSessionEntry()`, `loadGatewaySessionRow()`                 |
| Node 设备注册？          | [node-registry.ts](src/gateway/node-registry.ts)             | `NodeRegistry` 类                                               |
| 事件如何广播？           | [server-broadcast.ts](src/gateway/server-broadcast.ts)       | `broadcast()`                                                   |
| 配置如何热加载？         | [config-reload.ts](src/gateway/config-reload.ts)             | `startGatewayConfigReloader()`                                  |

### 子模块结构索引

```
src/gateway/
├── protocol/              # 协议定义（契约层）
│   ├── schema/            # TypeBox Schema 定义
│   │   ├── frames.ts      # 帧类型：req/res/event
│   │   ├── agent.ts       # Agent 请求/响应
│   │   ├── sessions.ts    # Session 操作
│   │   ├── nodes.ts       # Node invoke
│   │   └── error-codes.ts # 错误码定义
│   └── index.ts           # barrel 导出
│
├── server/                # WebSocket 服务器核心
│   ├── ws-connection.ts   # 连接入口处理
│   ├── health-state.ts    # 健康/状态缓存
│   └── preauth-*.ts       # 认证前预算管理
│
├── server-methods/        # RPC 方法实现（业务逻辑）
│   ├── agent.ts           # Agent 执行（最核心）
│   ├── connect.ts         # 连接处理
│   ├── sessions.ts        # Session 管理
│   ├── chat.ts            # 聊天历史/发送
│   ├── nodes.ts           # Node invoke
│   ├── config.ts          # 配置操作
│   ├── health.ts          # 健康检查
│   └── ...                 # 其他 ~30 个方法
│
├── auth.ts                # Gateway 认证解析
├── device-auth.ts         # 设备签名验证
├── node-registry.ts       # Node 设备注册表
├── server-broadcast.ts    # 事件广播
├── config-reload.ts       # 配置热加载
├── server-channels.ts     # 通道管理
└── session-utils.ts       # Session 工具函数
```

---

## Step 2: 理解边界（契约和不变量）

### 协议边界（最重要的契约）

来源：[protocol/CLAUDE.md](src/gateway/protocol/CLAUDE.md)

```markdown
边界规则：

1. Schema 变更 = Protocol 变更（视为契约修改，非本地重构）
2. 优先 additive evolution（渐进式演进）
3. 不兼容变更需要显式版本化（PROTOCOL_VERSION）
4. 新方法必须通过 protocol/schema/\*.ts 定义
5. 保持 schema/runtime/docs/tests/client-artifacts 同步
```

**协议版本**：`PROTOCOL_VERSION = 3`

### 认证边界

```markdown
认证层级：

1. Gateway Auth（gateway.auth.\*）— 所有连接必须通过
2. Device Pairing — 新设备需审批，发放 scoped device token
3. Challenge-Response 签名 — 所有连接必须签名 connect.challenge nonce

不变量：

- 非本地连接必须显式审批
- Token 签发受 role + scope 限制
- 旋转 token 不能扩展权限范围
```

### Session 边界

来源：[server-methods/CLAUDE.md](src/gateway/server-methods/CLAUDE.md)

```markdown
Session 不变量：

- Pi session transcripts 是 parentId 链/DAG
- 不能通过 raw JSONL 写入（会破坏 parentId 链）
- 必须通过 SessionManager.appendMessage() 写入
```

### Gateway 核心不变量

来源：[Gateway Protocol](/gateway/protocol)

```markdown
核心不变量：

1. 一台主机 = 一个 Gateway = 单一 Baileys session
2. WebSocket 第一帧必须是 connect
3. 事件不重放（client 需在 gap 时刷新状态）
4. Idempotency key 要求：side-effect 方法（send, agent）
```

---

## Step 3: 追踪数据流（核心流程）

### 流程 1: WebSocket 连接生命周期

```
Client                    Gateway                     关键文件
  │                          │
  │──── WebSocket connect ──▶│  server/ws-connection.ts:120
  │                          │  wss.on("connection", socket)
  │                          │
  │◀── connect.challenge ───│  device-auth.ts
  │    { nonce, ts }         │  生成挑战 nonce
  │                          │
  │──── connect (签名) ─────▶│  server/ws-connection/message-handler.ts
  │  { device.signature }    │  验证签名
  │                          │
  │◀── hello-ok ─────────────│  server/ws-connection.ts
  │  { protocol, snapshot,   │  返回握手成功 + 状态快照
  │    auth.deviceToken }    │
  │                          │
  │◀── event:tick ───────────│  server-maintenance.ts
  │    (周期心跳)             │  定期推送
  │                          │
  │──── req:agent ──────────▶│  server-methods/agent.ts
  │                          │
  │◀── res:agent (ack) ──────│  { status: "accepted", runId }
  │                          │
  │◀── event:agent ──────────│  (streaming events)
  │                          │
  │◀── res:agent (final) ────│  { status: "ok", summary }
```

**关键节点**：

- `server/ws-connection.ts:120` — WebSocket 连接入口
- `device-auth.ts` — 挑战生成和签名验证
- `server/ws-connection/message-handler.ts` — 消息帧处理

### 流程 2: Agent 执行流程

```
Client                    Gateway                     关键文件
  │                          │
  │──── req:agent ──────────▶│  server-methods/agent.ts
  │  { prompt, sessionKey }  │
  │                          │
  │                          │──▶ session-utils.ts
  │                          │    loadSessionEntry(sessionKey)
  │                          │
  │                          │──▶ assistant-identity.ts
  │                          │    resolveAssistantIdentity()
  │                          │
  │                          │──▶ agent-job.ts
  │                          │    waitForAgentJob()
  │                          │
  │◀── res:agent (ack) ──────│  { runId, status: "accepted" }
  │                          │
  │◀── event:agent ──────────│  server-chat.ts
  │  { type, content }       │  createAgentEventHandler()
  │    (streaming)           │
  │                          │
  │                          │──▶ agents/pi-runner.ts
  │                          │    实际执行 Agent Loop
  │                          │
  │◀── res:agent (final) ────│  { runId, status: "ok" }
```

**关键节点**：

- `server-methods/agent.ts` — Agent RPC 入口
- `session-utils.ts` — Session 加载
- `agent-job.ts` — Job 等待和去重
- `server-chat.ts` — Agent 事件处理和广播

### 流程 3: Node Invoke 流程

```
Node                     Gateway                     Operator Client
  │                          │                          │
  │──── WS connect ─────────▶│                          │
  │  { role: "node",         │                          │
  │    caps, commands }      │                          │
  │                          │                          │
  │◀── hello-ok ─────────────│                          │
  │                          │                          │
  │                          │◀─── req:node.invoke ─────│
  │                          │  { command, args }       │
  │                          │                          │
  │◀── node.invoke ──────────│  node-registry.ts        │
  │  { command, args }       │  转发给 Node             │
  │                          │                          │
  │──── node.invoke.result ─▶│                          │
  │  { result }              │                          │
  │                          │                          │
  │                          │──▶ node-invoke-sanitize.ts
  │                          │    命令净化/验证          │
  │                          │                          │
  │                          │◀─── res:node.invoke ─────│
  │                          │  { ok, payload }         │
```

**关键节点**：

- `node-registry.ts` — Node 注册和查找
- `server-methods/nodes.ts` — Node invoke RPC
- `node-invoke-sanitize.ts` — 命令净化

---

## Step 4: 实践验证

### 验证 1: 启动 Gateway 并检查健康

```bash
# 启动 Gateway
openclaw gateway --port 18789 --verbose

# 健康检查
openclaw gateway status
openclaw channels status --probe
```

**观察点**：

- `server.impl.ts` 中的启动日志
- `server-maintenance.ts` 中的健康刷新

### 验证 2: WebSocket 连接测试

```bash
# 使用 CLI 测试（CLI 自动连接 WS）
openclaw status
```

**观察点**：

- `connect.challenge → connect → hello-ok` 流程
- `auth.ts` 中的认证日志

### 验证 3: Agent 执行测试

```bash
# 发送 Agent 请求
openclaw agent --message "hello"
```

**观察点**：

- `server-methods/agent.ts` 处理流程
- `agent-job.ts` 去重逻辑
- `server-chat.ts` 事件广播

### 验证 4: 测试作为验证

```bash
# 运行关键测试
pnpm test src/gateway/server-methods/agent.test.ts
pnpm test src/gateway/auth.test.ts
pnpm test src/gateway/device-auth.test.ts
```

**测试告诉你**：

- Agent 方法的各种参数组合和边界情况
- 认证的成功/失败场景
- 设备签名的验证逻辑

---

## Step 5: 按需深入路径（问题驱动）

### 常见问题 → 深入路径

| 问题                    | 深入路径                                                                                                                   |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 如何添加新的 RPC 方法？ | 1. `protocol/schema/*.ts` 定义参数/响应<br>2. `server-methods/*.ts` 实现 handler<br>3. `server-methods-list.ts` 添加方法名 |
| 如何理解认证失败？      | `auth.ts` → `device-auth.ts` → `protocol/connect-error-details.ts`                                                         |
| 如何调试 Agent 执行？   | `server-methods/agent.ts` → `agent-job.ts` → `agents/pi-runner.ts`                                                         |
| 如何理解 Node 配对？    | `node-registry.ts` → `server-methods/nodes.ts` → `protocol/schema/nodes.ts`                                                |
| 如何修改协议帧格式？    | `protocol/schema/frames.ts` → 生成 JSON Schema → Swift models                                                              |
| 如何添加新事件类型？    | `protocol/schema/*.ts` → `server-methods-list.ts` (GATEWAY_EVENTS)                                                         |

---

## 学习检查清单

完成以下项目后，你就可以说"我理解 Gateway 了"：

```
□ 知道 Gateway 启动入口在哪里（server.impl.ts:startGatewayServer）
□ 知道 WebSocket 连接的 3 步流程（challenge → connect → hello-ok）
□ 知道有哪些主要 RPC 方法（~100 个，见 server-methods-list.ts）
□ 知道 Agent 执行的完整流程
□ 知道认证的 3 层结构（Gateway Auth → Device Pairing → Challenge-Response）
□ 知道协议边界规则（Schema = 契约）
□ 知道 Session 的 parentId 链不变量
□ 知道 Node 和 Operator 的 role 区别
□ 能追踪一条完整的数据流（req → res）
□ 知道关键测试文件在哪里
```

---

## 相关文档

- [Gateway Runbook](/gateway) — 运维手册
- [Gateway Protocol](/gateway/protocol) — WebSocket 协议规范
- [Architecture Overview](/concepts/architecture) — 系统架构
- [Protocol Boundary](src/gateway/protocol/CLAUDE.md) — 协议边界规则
- [Server Methods Notes](src/gateway/server-methods/CLAUDE.md) — 方法实现注意事项
