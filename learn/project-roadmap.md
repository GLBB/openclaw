# OpenClaw 项目精通路线图

## 目标

从"能把项目跑起来"到"成为项目Owner"，具备独立维护、扩展和决策的能力。

---

## 阶段一：建立全局视角（1-2周）

### 目标

理解项目"是什么"和"为什么这样设计"，建立mental model。

### 学习材料

1. **书籍**：《OpenClaw架构与源码解读》
   - 第3章：整体架构鸟瞰（必读）
   - 第11章：Gateway服务器架构（必读）
   - 其他章节按需阅读

2. **官方文档**：
   - `README.md` - 项目定位和快速上手
   - `docs/concepts/architecture.md` - 架构概览
   - `docs/gateway/protocol.md` - Gateway协议

3. **视频/文章**：
   - 知乎专栏"从零打造个人专属的OpenClaw"
   - 火龙果软件的"Gateway架构全解析"

### 关键概念（必须理解）

| 概念            | 定义                               | 为什么重要                                 |
| --------------- | ---------------------------------- | ------------------------------------------ |
| Gateway         | 控制平面，类似K8s Control Plane    | 所有消息流经这里，理解它就理解了系统的中枢 |
| Agent Runtime   | AI对话引擎，Prompt+Tools+Session   | 这是AI能力的核心实现                       |
| Channel         | 消息渠道插件（Telegram/Discord等） | 这是用户交互的入口                         |
| Plugin/Provider | 扩展系统，支持模型、渠道、工具     | 这是系统的可扩展性核心                     |
| Session         | 会话状态和历史                     | 这是Agent的记忆机制                        |
| Node            | iOS/Android/macOS设备节点          | 这是远程执行能力                           |

### 验收标准

- 能画出完整的架构图（五层结构）
- 能解释一条消息从Telegram发出去到返回的完整流程
- 能说出Gateway、Agent、Channel、Plugin各自的职责边界

---

## 阶段二：深入核心模块（2-3周）

### 模块A：Gateway（最关键）

**阅读顺序**：

```
1. src/gateway/server.ts           # 入口
2. src/gateway/server-http.ts      # HTTP/WebSocket服务
3. src/gateway/server-methods.ts   # RPC方法定义
4. src/gateway/server-chat.ts      # Chat处理流程
5. src/gateway/client.ts           # 客户端协议
6. src/gateway/protocol/           # 协议schema
```

**必须理解**：

- WebSocket连接生命周期（connect → req/res → event → close）
- 消息路由机制（如何分发到正确的Agent）
- 认证和权限控制（shared-secret、Tailscale、trusted-proxy）
- 配置热重载（reload机制）

**实践任务**：

- 用 `--verbose` 跑Gateway，观察一条消息的完整日志
- 修改日志级别，理解不同模块的日志输出

### 模块B：Plugin/Provider系统

**阅读顺序**：

```
1. src/plugins/types.ts            # 类型定义（70KB，慢慢读）
2. src/plugins/loader.ts           # 加载器（93KB）
3. src/plugins/manifest.ts         # Manifest验证
4. src/plugins/registry.ts         # 注册表
5. src/plugins/hooks.ts            # Hook系统
6. src/plugin-sdk/                 # 公共SDK（理解边界）
```

**必须理解**：

- 插件边界：`plugin-sdk/*`（公共） vs `plugins/*`（内部）
- Manifest结构：id、entrypoint、capabilities、hooks
- Provider注册：auth、models、tools
- Hook系统：before-agent-start、before-tool-call等

**实践任务**：

- 读一个完整的插件：`extensions/telegram/`
- 理解它如何注册、处理消息、调用SDK

### 模块C：Agent引擎

**阅读顺序**：

```
1. src/agents/pi-embedded-runner/run.ts  # 运行入口
2. src/agents/pi-core.ts                 # 核心逻辑
3. src/agents/pi-prompt.ts               # Prompt构建
4. src/agents/pi-tools.ts                # 工具调用
5. src/agents/pi-session.ts              # 会话管理
6. src/agents/acp-spawn.ts               # 子Agent派发
```

**必须理解**：

- Prompt构建流程（system prompt + tools + history）
- 工具调用循环（LLM → tool_use → execute → continue）
- Session压缩和compaction机制
- 子Agent派发场景

**实践任务**：

- 用 `openclaw agent --message "xxx" --verbose` 观察Prompt构建
- 添加一个自定义工具，测试调用流程

### 模块D：Channel实现

**阅读顺序**：

```
1. src/channels/plugins/types.plugin.ts  # 渠道类型
2. src/channels/plugins/types.core.ts    # 核心类型
3. src/routing/                          # 消息路由
```

**选一个渠道深入**：

- `extensions/telegram/src/index.ts`
- `extensions/discord/src/index.ts`
- `extensions/whatsapp/src/index.ts`

**必须理解**：

- 消息入站流程（inbound → Gateway → Agent）
- 消息出站流程（Agent → Gateway → Channel）
- DM配对机制（pairing、allowlist）

---

## 阘段三：动手实践（2-4周）

### 任务1：修复一个小Bug

在GitHub Issues找一个简单的bug，尝试修复：

- 定位问题 → 理解代码 → 提出方案 → 实现修复 → 跑测试

### 任务2：添加一个小功能

建议从以下选择：

- 给某个Channel添加新命令（如 `/weather`）
- 给Agent添加新工具（如查询天气）
- 添加新的Provider支持（如某个新模型API）

### 任务3：理解测试体系

```
pnpm test                  # 全量测试
pnpm test src/agents/...   # 模块测试
pnpm test:live             # Live测试
```

理解：

- Vitest框架和测试组织
- Mock策略
- Live测试的运行条件

---

## 阶段四：架构决策能力（持续）

### 目标

具备独立判断"能否改动"、"如何改动"的能力。

### 必须掌握的边界规则（AGENTS.md）

```
AGENTS.md                  # 项目总规（44KB，重点读）
src/plugin-sdk/AGENTS.md   # SDK边界
src/plugins/AGENTS.md      # 插件规则
src/channels/AGENTS.md     # 渠道规则
src/gateway/AGENTS.md      # Gateway规则
```

### 关键决策点

1. **新功能应该放在哪里？**
   - Core？Plugin？Extension？Skill？
   - 判断依据：是否通用、是否需要独立部署

2. **是否可以引入新依赖？**
   - 判断依据：是否必要、是否有替代方案

3. **是否可以修改公共接口？**
   - 判断依据：向后兼容、版本迁移

### 验收标准

- 能判断一个PR是否符合架构边界
- 能评估一个改动的影响范围
- 能提出符合项目风格的实现方案

---

## 阶段五：Owner能力（持续积累）

### 维护能力

- Review PR：判断正确性、架构合规性
- 处理Issue：快速定位问题、给出解决方案
- 发布管理：版本号、CHANGELOG、release流程

### 扩展能力

- 新Channel接入：理解插件模板，快速实现
- 新Provider接入：理解auth流程、模型注册
- 新工具添加：理解Tool Policy、安全控制

### 决策能力

- 架构演进方向：判断哪些改进值得做
- 技术选型：新依赖、新框架的选择
- 用户反馈处理：优先级判断、路线规划

---

## 学习资源汇总

### 必读材料

| 类型 | 内容                             | 优先级 |
| ---- | -------------------------------- | ------ |
| 书籍 | 《OpenClaw架构与源码解读》完整版 | P0     |
| 源码 | AGENTS.md + src核心模块          | P0     |
| 文档 | docs/concepts/、docs/gateway/    | P1     |
| 测试 | \*.test.ts 文件                  | P1     |

### 辅助材料

- 知乎专栏：架构总览与设计哲学
- 火龙果软件：Gateway架构全解析
- GitHub Issues：历史讨论和决策记录
- CHANGELOG：演进历史

---

## 时间规划建议

| 阶段   | 时间  | 重点               |
| ------ | ----- | ------------------ |
| 阶段一 | 1-2周 | 读书籍、建立视角   |
| 阶段二 | 2-3周 | 读核心源码         |
| 阶段三 | 2-4周 | 实践改动           |
| 阶段四 | 持续  | 边界规则、决策能力 |
| 阶段五 | 持续  | Owner能力积累      |

---

## AI时代的学习策略

### 不要记住所有代码

- 记住**架构边界**（哪些不能互相调用）
- 记住**关键概念**（Gateway、Agent、Channel、Plugin）
- 建立**检索能力**（知道去哪里找）

### AI辅助阅读

- 让AI解释复杂代码段
- 让AI画出调用流程图
- 让AI对比不同实现方案

### 实践优先

- 读代码不如改代码
- 理论不如实操
- 观察不如调试

---

## 验收清单（成为Owner的标志）

1. ✅ 能画出完整架构图并解释每层职责
2. ✅ 能Trace一条消息的完整流程
3. ✅ 能读懂任意一个插件的完整实现
4. ✅ 能判断一个改动应该放在哪个模块
5. ✅ 能独立修复bug并跑通测试
6. ✅ 能独立添加新功能并符合架构边界
7. ✅ 能Review PR并给出专业意见
8. ✅ 能处理用户Issue并定位问题
9. ✅ 能规划技术演进方向

---

生成时间：2026-04-23
