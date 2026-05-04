# OpenClaw 学习材料

本目录包含 OpenClaw 的深度学习笔记和开发者入门指南。

---

## 快速入门指南

| 模块                   | 文档                                           | 描述                                                                    |
| ---------------------- | ---------------------------------------------- | ----------------------------------------------------------------------- |
| **Gateway**            | [gateway.md](gateway.md)                       | 核心控制平面：WebSocket 服务器、RPC 分发、通道聚合、认证授权            |
| **Channel Routing**    | [channel-routing.md](channel-routing.md)       | 通道路由与消息分发：Channel Plugin 接口、Binding 匹配、Session Key 构造 |
| **Session Management** | [session-management.md](session-management.md) | 会话管理入门：Session Store、Transcript、生命周期、重置策略             |

---

## 深度解析文档

| 主题             | 文档                                                           | 描述                                                         |
| ---------------- | -------------------------------------------------------------- | ------------------------------------------------------------ |
| **架构概览**     | [openclaw-architecture.md](openclaw-architecture.md)           | OpenClaw 整体架构深度解析（餐厅类比版）                      |
| **Session 机制** | [openclaw-session-mechanism.md](openclaw-session-mechanism.md) | Session 实现机制完整深度解析：存储、压缩、Checkpoint、转记忆 |

---

## 学习路径

| 文档                                                                                       | 描述                           |
| ------------------------------------------------------------------------------------------ | ------------------------------ |
| [nodejs-learning-path-for-java-developers.md](nodejs-learning-path-for-java-developers.md) | Java 开发者的 Node.js 学习路径 |
| [project-roadmap.md](project-roadmap.md)                                                   | 项目学习路线图                 |

---

## 学习方式

采用"索引型学习"方式，5 步流程：

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

**核心理念**：记住"入口在哪"，而非"代码是什么"。需要细节时再深入。

---

## 问题记录

### 飞书 `__dirname` 问题 (2026-05-03)

**问题**: Feishu plugin 在 ESM 环境下启动报错 `__dirname is not defined in ES module scope`

**根因**: `@larksuiteoapi/node-sdk` 官方 SDK 的 ESM 版本错误使用了 `__dirname`

**时间线**:

- 2026-03-31: 引入 `bundledPluginRuntimeDependencies` 动态机制，自动排除 SDK
- 2026-05-01: 重构移除动态机制，改为静态列表，遗漏飞书 SDK
- v2026.5.2: 问题首次暴露
- 2026-05-03: PR #76392 修复，将 SDK 添加到静态排除列表

**解决**: 在 `tsdown.config.ts` 的 `explicitNeverBundleDependencies` 添加 `@larksuiteoapi/node-sdk`

---

## 相关资源

- [官方文档](https://docs.openclaw.ai) — Mintlify 托管的完整文档
- [GitHub Repo](https://github.com/openclaw/openclaw) — 源码仓库
