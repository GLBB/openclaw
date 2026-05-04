# 消息执行流程总览

> 一条消息从飞书服务器到用户回复的完整旅程。

---

## 整体概览

当用户在飞书发送消息给 OpenClaw Bot 时，消息经历以下旅程：

```
飞书服务器 → WebSocket/Webhook → OpenClaw Gateway → LLM → 飞书用户
     │              │                 │              │         │
   事件推送      接收解析          业务处理       AI生成    消息回复
```

**全程耗时**：通常 3-15 秒（取决于 LLM 响应速度）

---

## 核心概念

### (1) 接入：两种传输模式

| 模式          | 原理             | 适用场景           |
| ------------- | ---------------- | ------------------ |
| **WebSocket** | 长连接，实时推送 | 内网部署、低延迟   |
| **Webhook**   | HTTP POST 回调   | 公网部署、简单配置 |

### (2) 处理：三道防线

1. **Debounce（防抖）**：合并短时间内的连续消息（3 秒窗口）
2. **Deduplication（去重）**：基于 `message_id` 防止重复处理
3. **Sequential Queue（顺序队列）**：保证同一会话消息按顺序处理

### (3) 执行：四层架构

```
Layer 1: dispatchReplyFromConfig  → 路由决策、快速路径
Layer 2: getReplyFromConfig       → 模型选择、会话初始化
Layer 3: runPreparedReply         → Prompt 构建、执行准备
Layer 4: runReplyAgent            → LLM 调用、结果构造
```

### (4) 发送：三种回复类型

| 类型            | 方法             | 说明               |
| --------------- | ---------------- | ------------------ |
| **Block Reply** | `sendBlockReply` | 流式更新，实时显示 |
| **Tool Result** | `sendToolResult` | 工具执行结果       |
| **Final Reply** | `sendFinalReply` | 最终完整回复       |

---

## 关键数据流

```
飞书事件 JSON
    │
    ▼ 解析
MessageContext { chatId, senderId, content }
    │
    ▼ 路由
SessionKey = "agent:main:feishu:direct:ou_xxx"
    │
    ▼ 模型选择
Model = { provider: "bailian", model: "glm-5" }
    │
    ▼ Prompt 构建
API Payload = { messages, tools, stream: true }
    │
    ▼ LLM 调用
StreamChunk { text, finishReason }
    │
    ▼ 结果构造
ReplyPayload { text, usage }
    │
    ▼ 发送
飞书消息 → 用户
```

---

## 15 步调用链

从 `architecture.md` 主调用链提取：

| 步  | 函数                                | 层级        | 作用                     |
| --- | ----------------------------------- | ----------- | ------------------------ |
| 1   | `monitorWebSocket/Webhook`          | 接入层      | 监听飞书事件             |
| 2   | `createFeishuMessageReceiveHandler` | 解析层      | 事件解析、去重、debounce |
| 3   | `handleFeishuMessage`               | 业务层      | 权限检查、路由解析       |
| 4   | `runChannelTurn`                    | 编排层      | Turn 生命周期            |
| 5   | `dispatchReplyFromConfig`           | 第 1 层     | 分发协调                 |
| 6   | `getReplyFromConfig`                | 第 2 层     | 回复准备                 |
| 7   | `runPreparedReply`                  | 第 3 层     | 执行准备                 |
| 8   | `runReplyAgent`                     | 第 4 层     | Agent 编排               |
| 9   | `runAgentTurnWithFallback`          | 执行层      | 模型 fallback            |
| 10  | `runEmbeddedPiAgent`                | Pi 层       | 嵌入式运行               |
| 11  | `runAgentHarnessAttempt`            | Harness 层  | Harness 选择             |
| 12  | `runHarnessV2LifecycleAttempt`      | V2 层       | 四阶段生命周期           |
| 13  | `Provider.streamCompletion`         | Provider 层 | LLM API                  |
| 14  | `ReplyDispatcher`                   | 分发层      | 回复队列                 |
| 15  | `sendMessageFeishu`                 | 发送层      | 飞书 API                 |

---

## 关键机制

### Fallback（降级）

当主模型失败时，自动尝试备用模型：

```
glm-5 失败 → gpt-5.5 → sonnet-4.6 → ...
```

触发条件：

- API 错误（401/429/500）
- 超时
- Rate Limit

### Compaction（压缩）

当会话历史超出 context window 时：

```
旧消息 → LLM Summary → 替换为摘要 → 空出空间
```

### Block Streaming（流式回复）

实时更新飞书卡片，用户看到"打字"效果：

```
LLM chunk → onBlockReply → 飞书卡片更新 → 用户可见
```

---

## 阅读指引

**快速入门**：先读本文总览，理解整体流程。

**深入理解**：按 README.md 阅读顺序，逐层阅读详细分析。

**调试排查**：根据问题现象定位对应层级：

- 消息未收到 → 检查 01-02（接入层）
- 消息处理异常 → 检查 03-04（业务层）
- LLM 调用失败 → 检查 09-13（执行层）
- 回复发送失败 → 检查 14-15（发送层）

---

## 延伸阅读

- [README.md](README.md) - 详细目录和调用链
- [../architecture.md](../architecture.md) - 项目整体架构
- [../session-management.md](../session-management.md) - 会话管理
- [../channel-routing.md](../channel-routing.md) - 渠道路由

---

> **文档版本**: 2026-05-04
> **分析基于**: OpenClaw v2026.5.3
