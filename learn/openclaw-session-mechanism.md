# OpenClaw Session 实现机制深度解析

> 本文档基于 OpenClaw 源码深度分析，系统梳理 Session 的完整实现机制。

## 目录

- [1. 架构概览](#1-架构概览)
- [2. 存储机制](#2-存储机制)
- [3. 数据结构](#3-数据结构)
- [4. 加载流程](#4-加载流程)
- [5. 压缩机制](#5-压缩机制)
- [6. Checkpoint 机制](#6-checkpoint-机制)
- [7. Session 转记忆](#7-session-转记忆)
- [8. Session 工具家族](#8-session-工具家族)
- [9. 核心类梳理](#9-核心类梳理)

---

## 1. 架构概览

### 双层存储架构

```
┌─────────────────────────────────────────────────────────┐
│                    Session Layer                         │
├──────────────────────┬──────────────────────────────────┤
│   Index Layer        │      Transcript Layer            │
│   (sessions.json)    │      (*.jsonl files)             │
├──────────────────────┼──────────────────────────────────┤
│ - SessionEntry 元数据 │ - SessionManager 树结构         │
│ - sessionId 映射      │ - Entry 数组（物理存储）         │
│ - token 统计          │ - parentId 链（逻辑树）         │
│ - 状态标记            │ - append-only 特性              │
└──────────────────────┴──────────────────────────────────┘
```

**设计哲学**：

- **Index Layer**：快速查找、元数据索引、状态管理
- **Transcript Layer**：完整历史、append-only、树结构
- **分离关注点**：查找 vs 内容，状态 vs 历史

### 核心设计原则

| 原则                    | 说明                                    |
| ----------------------- | --------------------------------------- |
| **Append-only**         | Entries 不可修改或删除，只能追加        |
| **Tree structure**      | 通过 parentId 构建逻辑树，支持多分支    |
| **Logical compression** | 压缩不删除消息，只改变 context 构建方式 |
| **Safety first**        | Checkpoint 保护每次压缩，可回退         |
| **Explicit cleanup**    | 文件轮转是可选的清理机制                |

---

## 2. 存储机制

### 2.1 Session Store（sessions.json）

**位置**：`~/.openclaw/agents/<agentId>/sessions.json`

**结构**：

```json
{
  "telegram:123:456": {
    "sessionId": "abc-123-def",
    "sessionFile": "~/.pi/agent/sessions/.../abc-123-def.jsonl",
    "createdAt": 1704067200000,
    "updatedAt": 1704153600000,
    "totalTokens": 45000,
    "totalTokensFresh": true,
    "contextTokens": 128000,
    "chatType": "direct",
    "channel": "telegram",
    "providerOverride": "openai",
    "modelOverride": "gpt-4",
    "thinkingLevel": "medium",
    "verboseLevel": "off",
    "reasoningLevel": "on",
    "elevatedLevel": "ask",
    "compactionCount": 2,
    "memoryFlushAt": 1704100000000,
    "compactionCheckpoints": [...],
    "skillsSnapshot": {...}
  }
}
```

**核心字段说明**：

| 字段                    | 类型    | 用途                |
| ----------------------- | ------- | ------------------- |
| `sessionId`             | string  | 唯一标识符（UUID）  |
| `sessionFile`           | string  | Transcript 文件路径 |
| `totalTokens`           | number  | 当前 token 数       |
| `totalTokensFresh`      | boolean | token 数是否最新    |
| `contextTokens`         | number  | Context window 大小 |
| `providerOverride`      | string  | 模型提供商覆盖      |
| `modelOverride`         | string  | 模型 ID 覆盖        |
| `compactionCount`       | number  | 压缩次数            |
| `memoryFlushAt`         | number  | 上次记忆刷新时间    |
| `compactionCheckpoints` | array   | 压缩快照列表        |

### 2.2 Transcript Files（.jsonl）

**位置**：`~/.pi/agent/sessions/<encoded-cwd>/<sessionId>.jsonl`

**格式**：

```jsonl
{"type":"session","id":"abc-123","timestamp":"...","cwd":"/project"}
{"type":"message","id":"e1","parentId":null,"timestamp":"...","message":{...}}
{"type":"message","id":"e2","parentId":"e1","timestamp":"...","message":{...}}
{"type":"compaction","id":"c1","parentId":"e2","summary":"...","firstKeptEntryId":"e2"}
{"type":"message","id":"e3","parentId":"c1","timestamp":"...","message":{...}}
```

**Entry 类型**：

| Type                    | 说明       | 关键字段                  |
| ----------------------- | ---------- | ------------------------- |
| `session`               | Header     | id, cwd, timestamp        |
| `message`               | 对话消息   | role, content, model      |
| `compaction`            | 压缩节点   | summary, firstKeptEntryId |
| `model_change`          | 模型切换   | provider, modelId         |
| `thinking_level_change` | 思考级别   | thinkingLevel             |
| `branch_summary`        | 分支摘要   | fromId, summary           |
| `custom`                | 自定义数据 | customType, data          |
| `label`                 | 用户标签   | targetId, label           |

---

## 3. 数据结构

### 3.1 双重表示：数组 vs 树

**物理存储（数组）**：

```jsonl
Entry1, Entry2, Entry3, Entry4, Compaction1, Entry5, Entry6
```

**逻辑结构（树）**：

```
Entry1 (root, parentId: null)
  └── Entry2 (parentId: Entry1)
      └── Entry3 (parentId: Entry2)
          └── Entry4 (parentId: Entry3)
              └── Compaction1 (parentId: Entry4)
                  └── Entry5 (parentId: Compaction1)
                      └── Entry6 (parentId: Entry5) ← leaf
```

**内存索引**：

```typescript
class SessionManager {
  private fileEntries: FileEntry[]; // 数组（物理）
  private byId: Map<string, SessionEntry>; // Map（索引）
  private leafId: string | null; // 当前 leaf pointer

  // 从数组构建索引
  _buildIndex() {
    for (const entry of this.fileEntries) {
      this.byId.set(entry.id, entry);
      this.leafId = entry.id; // 最后一个成为 leaf
    }
  }
}
```

### 3.2 SessionEntry 类型定义

```typescript
interface SessionEntryBase {
  type: string;
  id: string;
  parentId: string | null;
  timestamp: string;
}

interface SessionMessageEntry extends SessionEntryBase {
  type: "message";
  message: AgentMessage;
}

interface CompactionEntry extends SessionEntryBase {
  type: "compaction";
  summary: string;
  firstKeptEntryId: string; // 边界标记
  tokensBefore: number;
  details?: T;
  fromHook?: boolean;
}
```

### 3.3 Session Manager 核心方法

| 方法                    | 返回类型          | 说明                   |
| ----------------------- | ----------------- | ---------------------- |
| `getEntries()`          | SessionEntry[]    | 获取数组（扁平）       |
| `getTree()`             | SessionTreeNode[] | 获取树（层级）         |
| `getBranch(leafId)`     | SessionEntry[]    | 从 leaf 回溯到 root    |
| `buildSessionContext()` | SessionContext    | 构建 LLM context       |
| `appendMessage(msg)`    | string            | 追加消息，返回 entryId |
| `appendCompaction(...)` | string            | 追加压缩节点           |
| `branch(entryId)`       | void              | 改变 leaf pointer      |
| `getLeafId()`           | string            | 当前 leaf ID           |

---

## 4. 加载流程

### 4.1 Session Store 加载

```typescript
// src/config/sessions/store-load.ts

export function loadSessionStore(storePath: string): SessionStore {
  // 1. 尝试从缓存读取
  const cached = SESSION_STORE_CACHE.get(storePath);
  if (cached && !isStale(cached)) {
    return cached.store;
  }

  // 2. 读取 JSON 文件
  const raw = fs.readFileSync(storePath, "utf-8");
  const parsed = JSON.parse(raw);

  // 3. 迁移和标准化
  const migrated = migrateSessionStore(parsed);
  const normalized = normalizeSessionStore(migrated);

  // 4. 更新缓存
  SESSION_STORE_CACHE.set(storePath, {
    store: normalized,
    loadedAt: Date.now(),
  });

  return normalized;
}
```

### 4.2 Transcript 加载

```typescript
// SessionManager.open()

static open(path: string): SessionManager {
  // 1. 读取 .jsonl 文件
  const raw = fs.readFileSync(path, "utf-8");
  const lines = raw.split("\n");

  // 2. 解析每一行 JSON
  const fileEntries: FileEntry[] = [];
  for (const line of lines) {
    if (!line.trim()) continue;
    const entry = JSON.parse(line);
    fileEntries.push(entry);
  }

  // 3. 构建索引
  this._buildIndex();

  // 4. 标记为已加载
  this.flushed = true;

  return sessionManager;
}
```

### 4.3 Context 构建（buildSessionContext）

```typescript
function buildSessionContext(entries, leafId, byId) {
  // 1. 从 leaf 回溯到 root，收集路径
  const path = [];
  let current = leaf;
  while (current) {
    path.unshift(current);
    current = byId.get(current.parentId);
  }

  // 2. 查找路径上的 compaction entry
  let compaction = null;
  for (const entry of path) {
    if (entry.type === "compaction") {
      compaction = entry;
    }
  }

  // 3. 构建消息列表
  if (compaction) {
    // 先添加 summary
    messages.push(createCompactionSummaryMessage(compaction.summary));

    // 从 firstKeptEntryId 开始添加完整消息
    let foundFirstKept = false;
    for (let i = 0; i < compactionIdx; i++) {
      if (path[i].id === compaction.firstKeptEntryId) {
        foundFirstKept = true;
      }
      if (foundFirstKept) {
        messages.push(path[i].message);
      }
    }

    // 添加 compaction 之后的消息
    for (let i = compactionIdx + 1; i < path.length; i++) {
      messages.push(path[i].message);
    }
  } else {
    // 没有 compaction，添加所有消息
    for (const entry of path) {
      messages.push(entry.message);
    }
  }

  return { messages, thinkingLevel, model };
}
```

---

## 5. 压缩机制

### 5.1 压缩触发条件

| Trigger              | 说明                 | Threshold                                      |
| -------------------- | -------------------- | ---------------------------------------------- |
| **manual**           | 用户 `/compact` 命令 | 无阈值                                         |
| **budget**           | Token 达到预算阈值   | `contextTokens - reserveFloor - softThreshold` |
| **overflow**         | 超过 context window  | `contextTokens * 0.95`                         |
| **transcript_bytes** | 文件大小过大         | 配置的 `maxActiveTranscriptBytes`              |

**计算公式**：

```
threshold = 128k - 20k - 4k = 104k
当 totalTokens > 104k 时触发预算压缩
```

### 5.2 压缩流程

```typescript
async function compactEmbeddedPiSession(params) {
  // 1. 创建 pre-compaction checkpoint
  const snapshot = captureCompactionCheckpointSnapshot({
    sessionManager,
    sessionFile,
  });

  // 2. 构建 context
  const context = sessionManager.buildSessionContext();

  // 3. 生成 summary（使用 LLM）
  const summary = await generateCompactionSummary({
    messages: context.messages,
    model: params.model,
    instructions: params.customInstructions,
  });

  // 4. 确定 firstKeptEntryId（边界）
  const firstKeptEntryId = resolveCompactionBoundary({
    entries,
    tokenLimit: params.tokenLimit,
  });

  // 5. 追加 CompactionEntry
  const compactionId = sessionManager.appendCompaction(summary, firstKeptEntryId, tokensBefore);

  // 6. 保存 checkpoint 元数据
  await persistSessionCompactionCheckpoint({
    cfg,
    sessionKey,
    snapshot,
    summary,
    tokensBefore,
    tokensAfter,
  });

  // 7. 可选：轮转 transcript 文件
  if (shouldRotate) {
    await rotateTranscriptAfterCompaction({
      sessionManager,
      sessionFile,
    });
  }

  return { ok: true, compacted: true };
}
```

### 5.3 CompactionEntry 的作用

**关键理解**：

- CompactionEntry **追加**到 session，不是删除旧消息
- `firstKeptEntryId` 是**边界标记**，不是删除标记
- buildSessionContext() 通过树遍历**选择**使用 summary 还是完整消息
- 旧消息作为树节点**永久保留**在文件中

**Context 构建示例**：

```
路径：[e1, e2, e3, Compaction1, e4, e5]

生成 LLM context：
1. CompactionSummaryMessage(Compaction1.summary)  ← 摘要代替 e1-e3
2. e4.message  ← firstKeptEntryId 开始的完整消息
3. e5.message

原始 e1-e3 仍在文件中，只是 context 不使用
```

---

## 6. Checkpoint 机制

### 6.1 Checkpoint 创建

```typescript
function captureCompactionCheckpointSnapshot(params) {
  // 1. 检查文件大小（限制 64MB）
  const stat = fs.statSync(sessionFile);
  if (stat.size > maxBytes) return null;

  // 2. 复制整个 session 文件
  const snapshotFile = `${sessionFile}.checkpoint.${randomUUID()}.jsonl`;
  fs.copyFileSync(sessionFile, snapshotFile);

  // 3. 返回快照信息
  return {
    sessionId,
    sessionFile: snapshotFile,
    leafId: getLeafId(), // 压缩前的 leaf 位置
  };
}
```

### 6.2 Checkpoint 元数据

```typescript
interface SessionCompactionCheckpoint {
  checkpointId: string;
  createdAt: number;
  reason: "auto-threshold" | "manual" | "timeout-retry";

  preCompaction: {
    sessionId: string;
    sessionFile: string; // 快照文件路径
    leafId: string; // 压缩前的 leaf
  };

  postCompaction: {
    sessionId: string;
    sessionFile?: string;
    leafId?: string;
    entryId?: string; // CompactionEntry ID
  };

  summary?: string;
  tokensBefore?: number;
  tokensAfter?: number;
}
```

### 6.3 Checkpoint 用途

| 用途         | 说明                                                |
| ------------ | --------------------------------------------------- |
| **Rollback** | 压缩失败时，打开快照文件，回到 preCompaction.leafId |
| **Audit**    | 记录每次压缩的原因、效果、时间                      |
| **Debug**    | 对比压缩前后的 context 差异                         |
| **Analysis** | 分析压缩策略效果                                    |

**清理机制**：

```typescript
const MAX_CHECKPOINTS = 25;

// 保留最近 25 个，删除旧的快照文件
function trimSessionCheckpoints(checkpoints) {
  const kept = checkpoints.slice(-25);
  const removed = checkpoints.slice(0, -25);

  for (const checkpoint of removed) {
    fs.unlink(checkpoint.preCompaction.sessionFile);
  }

  return kept;
}
```

---

## 7. Session 转记忆

### 7.1 转换架构

```
Session Transcript          Memory Sources          Embedding Index
┌────────────────┐         ┌────────────────┐      ┌────────────────┐
│ .jsonl 文件     │ ──────▶ │ MEMORY.md      │ ───▶ │ sqlite-vec     │
│ User: ...       │         │ daily/*.md     │      │ 向量存储       │
│ Assistant: ...  │         │ sessions/*.jsonl│      │ 语义搜索       │
└────────────────┘         └────────────────┘      └────────────────┘
                                    │                       │
                                    ▼                       ▼
                              Memory Flush           memory_search
                              Dreaming               memory_get
```

### 7.2 Memory Flush（实时）

**触发条件**：

```typescript
threshold = contextWindow - reserveFloor - softThreshold;
// 例如：128k - 20k - 4k = 104k

if (totalTokens > threshold) {
  await runMemoryFlush();
}
```

**执行流程**：

```typescript
async function runMemoryFlush(params) {
  // 1. 获取 flush plan
  const plan = resolveMemoryFlushPlan({ cfg });

  // 2. 运行 memory flush agent
  await runEmbeddedPiAgent({
    trigger: "memory",
    prompt: plan.prompt, // "总结当前对话关键信息..."
    memoryFlushWritePath: plan.relativePath, // "MEMORY.md"
    model: plan.model, // 可用 cheaper model
  });

  // 3. Agent 分析 transcript，提取：
  // - 用户意图和需求
  // - 重要决策和结论
  // - 关键代码片段
  // - 未解决的问题

  // 4. 写入 MEMORY.md
  // 5. 更新 session 元数据
  await updateSessionStoreEntry({
    sessionKey,
    update: {
      memoryFlushAt: Date.now(),
      memoryFlushCompactionCount: compactionCount,
    },
  });
}
```

### 7.3 Memory Dreaming（批量）

**三阶段睡眠式整合**：

| Phase     | 频率      | Sources                       | 作用         |
| --------- | --------- | ----------------------------- | ------------ |
| **Light** | 每 6 小时 | daily, sessions, recall       | 短期记忆去重 |
| **Deep**  | 每天 3 点 | daily, memory, sessions, logs | 长期记忆沉淀 |
| **REM**   | 每周日    | memory, daily, deep           | 模式发现     |

**Deep Dreaming 示例**：

```typescript
async function runDeepDreaming(cfg) {
  // 1. 从多个来源收集内容
  const sources = ["daily", "memory", "sessions", "logs", "recall"];
  const content = await collectFromSources(sources, {
    lookbackDays: 30,
    limit: 10,
  });

  // 2. 分析用户偏好、常用模式、重要决策
  const insights = await analyzeWithLLM(content, {
    minScore: 0.8,
    minRecallCount: 3,
    minUniqueQueries: 3,
  });

  // 3. 写入 MEMORY.md
  await writeMemory(inights);

  // 4. Recovery 机制（健康度 < 0.35 时）
  if (memoryHealth < 0.35) {
    await recoverLostMemory({
      lookbackDays: 30,
      minConfidence: 0.9,
    });
  }
}
```

### 7.4 Session Files 处理

```typescript
// packages/memory-host-sdk/src/host/session-files.ts

async function buildSessionEntry(absPath) {
  // 1. 读取 .jsonl
  const raw = await fs.readFile(absPath, "utf-8");
  const lines = raw.split("\n");

  // 2. 解析每一行
  for (const line of lines) {
    const record = JSON.parse(line);
    if (record.type !== "message") continue;

    // 3. 提取和清理文本
    const rawText = collectRawSessionText(message.content);
    const text = sanitizeSessionText(rawText, message.role);

    // 清理步骤：
    // - stripInboundMetadata: 去除信道元数据
    // - stripInternalRuntimeContext: 去除运行时上下文
    // - normalizeSessionText: 合并空格
    // - 去除生成的系统消息
    // - redactSensitiveText: 去除敏感信息

    // 4. 格式化
    const label = message.role === "user" ? "User" : "Assistant";
    const rendered = `User: 请帮我分析代码`;

    // 5. 记录元数据
    const timestampMs = parseSessionTimestampMs(record, message);
    collected.push(rendered);
    lineMap.push(jsonlIdx + 1);
    messageTimestampsMs.push(timestampMs);
  }

  return {
    path: "sessions/abc-123.jsonl",
    content: "User: ...\nAssistant: ...",
    lineMap: [1, 2, 3],
    messageTimestampsMs: [1234567890, ...],
  };
}
```

### 7.5 记忆检索工具

**memory_search**：

```typescript
{
  name: "memory_search",
  parameters: { query: string, maxResults: number }
}

// 流程：
1. Embedding query
2. 在 sqlite-vec 搜索相似向量
3. 返回匹配片段（包含 corpus, path, snippet, score）
```

**memory_get**：

```typescript
{
  name: "memory_get",
  parameters: { lookup: string, fromLine: number, lineCount: number }
}

// 流程：
1. 定位文件（MEMORY.md 或 daily/*.md）
2. 读取指定行范围
3. 返回完整内容（受 maxChars 限制）
```

---

## 8. Session 工具家族

### 8.1 核心工具列表

| 工具               | 说明            | 核心参数                        |
| ------------------ | --------------- | ------------------------------- |
| `session_status`   | 状态卡片        | sessionKey, model               |
| `sessions_list`    | Session 列表    | kinds, limit, activeMinutes     |
| `sessions_history` | 对话历史        | sessionKey, limit, includeTools |
| `sessions_spawn`   | 创建 subagent   | prompt, mode, sessionKey        |
| `sessions_send`    | 跨 session 消息 | sessionKey, message             |
| `subagents`        | Subagent 管理   | action, target                  |
| `sessions_yield`   | 暂停等待        | message                         |

### 8.2 session_status 详细说明

**功能**：

- 显示 token 使用量：`45k/128k (35%)`
- 显示时间信息：当前时间、时区
- 显示运行模式：Reasoning/Verbose/Elevated/Fast
- 显示模型信息：当前模型、是否默认
- **切换模型**：`session_status(model="claude-3")`

**返回示例**：

```
📊 Status
───────────────────────────────
Session: telegram:123:456
Channel: telegram
Model: openai/gpt-4 (default)
Context: 45k/128k tokens (35%)

Time: 2024-01-15 10:30 PST
Mode: Reasoning=on · Verbose=off · Elevated=ask

📌 Tasks: 2 active · node · Processing files
───────────────────────────────
```

### 8.3 sessions_spawn 使用场景

```typescript
// 并行分析三个文件
const task1 = await sessions_spawn({
  prompt: "分析 src/auth.ts 的安全性",
  mode: "run",
});

const task2 = await sessions_spawn({
  prompt: "分析 src/api.ts 的性能",
  mode: "run",
});

// 监控进度
const agents = await subagents((action = "list"));

// 收集结果
const log1 = await subagents((action = "log"), (target = "0"));
```

### 8.4 工具权限控制

| 工具               | 权限          | 跨 Agent | Sandbox |
| ------------------ | ------------- | -------- | ------- |
| `session_status`   | Owner/Allowed | 需要 A2A | 受限    |
| `sessions_list`    | Owner/Allowed | 需要 A2A | 受限    |
| `sessions_history` | Owner only    | 禁止     | 禁止    |
| `sessions_spawn`   | Owner only    | 禁止     | 受限    |
| `sessions_send`    | Owner only    | 禁止     | 受限    |

---

## 9. 核心类梳理

### 9.1 存储层

| 类/模块                | 文件路径                             | 功能                                               |
| ---------------------- | ------------------------------------ | -------------------------------------------------- |
| **SessionStore**       | `src/config/sessions.ts`             | Session 索引管理，sessions.json 的加载、更新、缓存 |
| **SessionEntry**       | `src/config/sessions/store-entry.ts` | Session 元数据类型定义，100+ 字段                  |
| **loadSessionStore**   | `src/config/sessions/store-load.ts`  | Session store 加载、迁移、标准化、缓存管理         |
| **updateSessionStore** | `src/config/sessions.ts`             | Session store 更新、持久化                         |
| **resolveStorePath**   | `src/config/sessions.ts`             | 根据配置和 agentId 解析 store 文件路径             |

### 9.2 Transcript 层

| 类/模块                                       | 文件路径                            | 功能                                                   |
| --------------------------------------------- | ----------------------------------- | ------------------------------------------------------ |
| **SessionManager**                            | `@mariozechner/pi-coding-agent`     | Session 树结构管理，append-only 保证，context 构建     |
| **buildSessionContext**                       | `@mariozechner/pi-coding-agent`     | 从 leaf 回溯到 root，处理 compaction，生成 LLM context |
| **resolveSessionTranscriptFile**              | `src/config/sessions/transcript.ts` | Transcript 文件路径解析                                |
| **appendAssistantMessageToSessionTranscript** | `src/config/sessions/transcript.ts` | 追加 assistant 消息到 transcript                       |

### 9.3 压缩层

| 类/模块                            | 文件路径                                                      | 功能                                         |
| ---------------------------------- | ------------------------------------------------------------- | -------------------------------------------- |
| **compactEmbeddedPiSession**       | `src/agents/pi-embedded-runner/compact.ts`                    | 主压缩流程，调用 LLM 生成 summary            |
| **CompactionEntry**                | `@mariozechner/pi-coding-agent`                               | 压缩节点类型，包含 summary、firstKeptEntryId |
| **resolveCompactionBoundary**      | `src/agents/pi-embedded-runner/compact.ts`                    | 确定 firstKeptEntryId 边界                   |
| **hardenManualCompactionBoundary** | `src/agents/pi-embedded-runner/manual-compaction-boundary.ts` | 手动压缩的边界硬化                           |

### 9.4 Checkpoint 层

| 类/模块                                 | 文件路径                                        | 功能                                   |
| --------------------------------------- | ----------------------------------------------- | -------------------------------------- |
| **captureCompactionCheckpointSnapshot** | `src/gateway/session-compaction-checkpoints.ts` | 创建压缩前快照（复制 transcript 文件） |
| **persistSessionCompactionCheckpoint**  | `src/gateway/session-compaction-checkpoints.ts` | 保存 checkpoint 元数据到 session store |
| **listSessionCompactionCheckpoints**    | `src/gateway/session-compaction-checkpoints.ts` | 列出所有 checkpoints                   |
| **cleanupCompactionCheckpointSnapshot** | `src/gateway/session-compaction-checkpoints.ts` | 清理快照文件                           |

### 9.5 History 处理层

| 类/模块                    | 文件路径                                          | 功能                     |
| -------------------------- | ------------------------------------------------- | ------------------------ |
| **sanitizeSessionHistory** | `src/agents/pi-embedded-runner/replay-history.ts` | 8 步历史清理管道         |
| **limitHistoryTurns**      | `src/agents/pi-embedded-runner/history.ts`        | 基于轮次的历史限制       |
| **validateReplayTurns**    | `src/agents/pi-embedded-runner/replay-history.ts` | 验证 replay turns 合规性 |

**8 步清理管道**：

1. 移除 tool_result 且无 paired tool_call 的消息
2. 移除 tool_call 且无 paired tool_result 的消息
3. 移除空内容消息
4. 移除重复 user 消息
5. 移除重复 assistant 消息
6. 移除 system 消息（非第一条）
7. 修复不完整的 tool result pairing
8. 去除敏感信息

### 9.6 Session 转 Memory 层

| 类/模块                    | 文件路径                                             | 功能                                          |
| -------------------------- | ---------------------------------------------------- | --------------------------------------------- |
| **buildSessionEntry**      | `packages/memory-host-sdk/src/host/session-files.ts` | 解析 transcript，提取可索引内容               |
| **extractSessionText**     | `packages/memory-host-sdk/src/host/session-files.ts` | 提取和清理 session 文本                       |
| **sanitizeSessionText**    | `packages/memory-host-sdk/src/host/session-files.ts` | 清理元数据、合并空格、去除系统消息            |
| **resolveMemoryFlushPlan** | `src/plugins/memory-state.ts`                        | 解析 memory flush plan（prompt、path、model） |
| **runMemoryFlushIfNeeded** | `src/auto-reply/reply/agent-runner-memory.ts`        | 判断并执行 memory flush                       |

### 9.7 Memory Dreaming 层

| 类/模块                         | 文件路径                          | 功能                                 |
| ------------------------------- | --------------------------------- | ------------------------------------ |
| **resolveMemoryDreamingConfig** | `src/memory-host-sdk/dreaming.ts` | 解析 dreaming 配置（light/deep/rem） |
| **MemoryLightDreamingConfig**   | `src/memory-host-sdk/dreaming.ts` | Light dreaming 配置（每 6 小时）     |
| **MemoryDeepDreamingConfig**    | `src/memory-host-sdk/dreaming.ts` | Deep dreaming 配置（每天 3 点）      |
| **MemoryRemDreamingConfig**     | `src/memory-host-sdk/dreaming.ts` | REM dreaming 配置（每周日）          |

### 9.8 Memory 插件层

| 类/模块                             | 文件路径                      | 功能                                  |
| ----------------------------------- | ----------------------------- | ------------------------------------- |
| **registerMemoryCapability**        | `src/plugins/memory-state.ts` | 注册 memory 插件能力                  |
| **getMemoryCapabilityRegistration** | `src/plugins/memory-state.ts` | 获取已注册的 memory 能力              |
| **MemoryPluginRuntime**             | `src/plugins/memory-state.ts` | Memory runtime 接口（search manager） |
| **MemoryFlushPlan**                 | `src/plugins/memory-state.ts` | Memory flush plan 类型定义            |

### 9.9 Session 工具层

| 类/模块                       | 文件路径                                    | 功能                       |
| ----------------------------- | ------------------------------------------- | -------------------------- |
| **createSessionStatusTool**   | `src/agents/tools/session-status-tool.ts`   | 创建 session_status 工具   |
| **createSessionsListTool**    | `src/agents/tools/sessions-list-tool.ts`    | 创建 sessions_list 工具    |
| **createSessionsHistoryTool** | `src/agents/tools/sessions-history-tool.ts` | 创建 sessions_history 工具 |
| **createSessionsSpawnTool**   | `src/agents/tools/sessions-spawn-tool.ts`   | 创建 sessions_spawn 工具   |
| **createSessionsSendTool**    | `src/agents/tools/sessions-send-tool.ts`    | 创建 sessions_send 工具    |
| **createSubagentsTool**       | `src/agents/tools/subagents-tool.ts`        | 创建 subagents 工具        |

### 9.10 Session 辅助层

| 类/模块                          | 文件路径                               | 功能                                   |
| -------------------------------- | -------------------------------------- | -------------------------------------- |
| **resolveSessionReference**      | `src/agents/tools/sessions-helpers.ts` | 解析 session 引用（key/id/alias）      |
| **createSessionVisibilityGuard** | `src/agents/tools/sessions-helpers.ts` | 创建可见性守卫（权限检查）             |
| **createAgentToAgentPolicy**     | `src/agents/tools/sessions-helpers.ts` | 创建跨 agent 访问策略                  |
| **classifySessionKind**          | `src/agents/tools/sessions-helpers.ts` | 分类 session（direct/group/cron/hook） |

---

## 附录：关键流程图

### A. Session 生命周期

```
创建 Session
  │
  ├─ SessionManager.create(cwd)
  │   ├─ 生成 sessionId
  │   ├─ 创建 .jsonl 文件（header）
  │   └─ 返回 session
  │
  ├─ 更新 sessions.json
  │   └─ store[sessionKey] = { sessionId, sessionFile, ... }
  │
  └─ 对话进行中
      ├─ appendMessage(user) → 追加到 .jsonl
      ├─ buildSessionContext() → 构建 LLM context
      ├─ appendMessage(assistant) → 追加到 .jsonl
      ├─ 更新 totalTokens
      │
      ├─ 达到阈值 → 触发压缩
      │   ├─ 创建 checkpoint
      │   ├─ 生成 summary
      │   ├─ appendCompaction()
      │   └─ 更新 compactionCount
      │
      ├─ 达到 memory flush 阈值
      │   ├─ 运行 memory flush agent
      │   ├─ 写入 MEMORY.md
      │   └─ 更新 memoryFlushAt
      │
      └─ 用户 /reset 或 /new
          ├─ 创建新 session
          └─ archive 旧 session
```

### B. Context 构建流程

```
buildSessionContext(leafId)
  │
  ├─ 从 leaf 回溯到 root
  │   └─ path = [e1, e2, e3, c1, e4, e5]
  │
  ├─ 查找 path 上的 compaction
  │   └─ compaction = c1
  │
  ├─ 有 compaction？
  │   ├─ Yes:
  │   │   ├─ 添加 CompactionSummaryMessage(c1.summary)
  │   │   ├─ 从 firstKeptEntryId 开始添加完整消息
  │   │   │   └─ e2, e3（假设 firstKeptEntryId = e2）
  │   │   └─ 添加 compaction 之后的消息
  │   │       └─ e4, e5
  │   │
  │   └─ No:
  │       └─ 添加所有完整消息
  │           └─ e1, e2, e3, e4, e5
  │
  └─ 返回 { messages, thinkingLevel, model }
```

### C. Memory 转换流程

```
Session Transcript (.jsonl)
  │
  ├─ Memory Flush（实时）
  │   ├─ 触发：totalTokens > threshold
  │   ├─ Agent 分析当前对话
  │   ├─ 提取关键信息
  │   └─ 写入 MEMORY.md
  │
  ├─ Light Dreaming（每 6h）
  │   ├─ Sources: daily, sessions, recall
  │   ├─ 去重相似内容
  │   └─ 写入 daily/*.md
  │
  ├─ Deep Dreaming（每天 3 点）
  │   ├─ Sources: daily, memory, sessions, logs
  │   ├─ 综合分析长期记忆
  │   ├─ 提取用户偏好、常用模式
  │   └─ 写入 MEMORY.md
  │
  └─ REM Dreaming（每周日）
      ├─ Sources: memory, daily, deep
      ├─ 发现高层次模式
      └─ 更新 MEMORY.md

Embedding Index (sqlite-vec)
  │
  ├─ memory_search(query)
  │   ├─ Embedding query
  │   ├─ 向量搜索
  │   └─ 返回匹配片段
  │
  └─ memory_get(lookup)
      ├─ 定位文件
      ├─ 读取内容
      └─ 返回完整片段
```

---

## 参考资料

- SessionManager 源码：`node_modules/@mariozechner/pi-coding-agent/dist/core/session-manager.js`
- Session Store：`src/config/sessions.ts`
- Compaction：`src/agents/pi-embedded-runner/compact.ts`
- Checkpoint：`src/gateway/session-compaction-checkpoints.ts`
- Memory：`packages/memory-host-sdk/src/host/session-files.ts`
- Dreaming：`src/memory-host-sdk/dreaming.ts`
- Tools：`src/agents/tools/session-status-tool.ts`

---

_文档生成时间：2026-05-01_
_基于 OpenClaw main branch 最新代码分析_
