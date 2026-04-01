# 基础设施层详解

## Bridge 系统 (bridge/)

### 概述

Claude Cloud Remote (CCR) 的核心基础设施，允许在云端托管 Claude Code 会话。

### 架构

```
┌──────────────┐   Bridge (WebSocket/HTTP)   ┌──────────────┐
│  IDE /       │◄────────────────────────────►│  Cloud       │
│  CLI Client  │   JWT 认证, 会话管理         │  Session     │
│              │   最多 32 并发会话            │  Pool        │
└──────────────┘                              └──────────────┘
```

### 核心文件

| 文件 | 大小 | 功能 |
|------|------|------|
| `bridgeMain.ts` | 115KB | 主桥接逻辑，会话派生和管理 |
| `replBridge.ts` | 100KB | REPL 特定桥接逻辑 |
| `remoteBridgeCore.ts` | 39KB | 核心桥接功能 |
| `bridgeApi.ts` | - | JWT 认证的 HTTP 客户端 |
| `bridgeMessaging.ts` | - | 消息协议 |
| `bridgeUI.ts` | - | 桥接状态 UI 组件 |
| `bridgeConfig.ts` | - | 桥接配置 |
| `bridgeDebug.ts` | - | 调试工具 |
| `jwtUtils.ts` | - | JWT 令牌管理 |
| `trustedDevice.ts` | - | 受信设备认证 |
| `sessionRunner.ts` | - | 会话执行器 |
| `initReplBridge.ts` | - | REPL 桥接初始化 |

### 关键特性

- **多会话**: 支持最多 32 个并发会话
- **退避策略**: 指数退避重连配置
- **容量管理**: 容量感知的会话调度
- **Work Secret**: 工作负载密钥管理
- **受信设备**: 设备级认证

---

## CLI 传输层 (cli/)

### 概述

CLI 基础设施，处理 I/O、传输协议和协议适配。

### 传输协议

```
cli/transports/
├── SSETransport.ts       — Server-Sent Events
├── WebSocketTransport.ts — WebSocket 协议
├── HybridTransport.ts    — 混合传输层
├── ccrClient.ts          — CCR (Claude Cloud Remote) 客户端
├── SerialBatchEventUploader.ts — 批量事件上传
└── WorkerStateUploader.ts — 工作节点状态同步
```

### 协议处理器

```
cli/handlers/
├── auth.ts      — 认证处理
├── agents.ts    — 代理操作
├── plugins.ts   — 插件管理
├── mcp.tsx      — MCP 协议
└── autoMode.ts  — 自动模式
```

### 关键文件

| 文件 | 功能 |
|------|------|
| `remoteIO.ts` | SDK 模式双向流（WebSocket/SSE 传输） |
| `structuredIO.ts` | 结构化 I/O 基类 |
| `print.ts` | 输出格式化（200KB+） |
| `update.ts` | 更新管理 |

---

## 远程会话 (remote/)

### 架构

```
┌───────────────────┐                    ┌───────────────────┐
│  RemoteSession     │  WebSocket + HTTP  │  Remote Server    │
│  Manager           │◄──────────────────►│                   │
│                    │  权限桥接 + 重连    │                   │
└───────────────────┘                    └───────────────────┘
```

### 核心组件

| 组件 | 功能 |
|------|------|
| `RemoteSessionManager.ts` | WebSocket 订阅接收 + HTTP POST 发送 |
| `SessionsWebSocket.ts` | WebSocket 封装（连接、重连、退避） |
| `sdkMessageAdapter.ts` | SDK 消息 ↔ REPL 消息转换 |
| `remotePermissionBridge.ts` | 远程权限处理 |

### 消息适配

```
SDKAssistantMessage       → AssistantMessage
SDKPartialAssistantMessage → StreamEvent
SDKResultMessage          → SystemMessage
SDKSystemMessage          → SystemMessage
```

---

## 协调器 (coordinator/)

### 概述

多代理编排协调器，管理多个子代理的协作。

### 特性

- 多代理任务分配
- 状态同步
- 权限传播
- 通过 `feature('COORDINATOR_MODE')` 门控

---

## 助手模式 (assistant/)

### 概述

主动式 AI 助手模式（KAIROS 特性），能够主动响应用户需求。

### 特性

- 通过 `feature('KAIROS')` 门控
- 主动消息推送
- 与 BriefTool 配合
- 与 SleepTool 配合实现定时响应

---

## 伙伴精灵 (buddy/)

### 概述

终端中的动画伙伴精灵，带有稀有度系统。

### 物种系统

18 种物种：duck, goose, blob, cat, dragon, fox, rabbit, robot, ghost, owl, penguin, frog, unicorn, phoenix, kraken, slime, mushroom, alien

### 稀有度

| 等级 | 权重 | 星级 |
|------|------|------|
| Common (普通) | 60% | ★ |
| Uncommon (优秀) | 25% | ★★ |
| Rare (稀有) | 10% | ★★★ |
| Epic (史诗) | 4% | ★★★★ |
| Legendary (传说) | 1% | ★★★★★ |

### 属性

5 个属性：DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK

### 生成

- 基于种子的确定性 PRNG
- 帧动画渲染
- 语音气泡和效果
- 空闲动画序列

---

## Server (server/)

本地 HTTP 服务器，支持 IDE 直连会话：

- `createDirectConnectSession.ts` — 创建直连会话
- `directConnectManager.ts` — 管理直连会话
- 支持 WebSocket URL 分发
- 认证和权限处理

---

## Vim 模式 (vim/)

完整的 Vim 编辑器状态机实现：

| 文件 | 功能 |
|------|------|
| `types.ts` | Vim 状态机完整类型 |
| `operators.ts` | delete, change, yank 操作 |
| `motions.ts` | 移动命令 (h/j/k/l/w/b/e 等) |
| `textObjects.ts` | 文本对象 (词、引号、括号) |
| `transitions.ts` | INSERT/NORMAL 模式状态转换 |

---

## 语音模式 (voice/)

语音输入/输出支持，通过 `feature('VOICE_MODE')` 门控。

---

## 键绑定系统 (keybindings/)

| 文件 | 功能 |
|------|------|
| `defaultBindings.ts` | 默认键绑定（Global, Chat, Autocomplete 等上下文） |
| `parser.ts` | 键绑定语法解析 (ctrl+c, shift+tab 等) |
| `resolver.ts` | 按键匹配到绑定 |
| `schema.ts` | 验证 Schema |
| `useKeybinding.ts` | React Hook |

平台感知处理（Windows vs Unix）。

---

## 技能系统 (skills/)

| 文件 | 功能 |
|------|------|
| `loadSkillsDir.ts` | 从目录加载技能 |
| `bundledSkills.ts` | 内置技能 |
| `mcpSkillBuilders.ts` | MCP 服务集成构建器 |

技能来源：管理目录、用户目录、项目目录。支持动态发现和条件激活。

---

## 插件系统 (plugins/)

| 文件 | 功能 |
|------|------|
| `builtinPlugins.ts` | 内置插件注册表 |
| `bundled/` | 打包插件定义 |

插件可提供：技能、Hook、MCP 服务器、LSP 服务器。区分 `@builtin` 和市场插件。

---

## 记忆系统 (memdir/)

| 文件 | 功能 |
|------|------|
| `memdir.ts` | 记忆 prompt 构建、截断、加载 |
| `memoryTypes.ts` | 记忆条目类型定义 |
| `paths.ts` | 记忆目录路径 |
| `teamMemPaths.ts` | 团队记忆路径 |
| `findRelevantMemories.ts` | 相关性记忆搜索 |

支持类型：
- **user** — 用户信息
- **feedback** — 反馈指导
- **project** — 项目信息
- **reference** — 外部引用

---

## 原生 TS 模块 (native-ts/)

| 模块 | 功能 |
|------|------|
| `yoga-layout/` | CSS Flexbox 布局引擎（从 C++ 移植，2500+ 行） |
| `color-diff/` | 颜色差异计算 |
| `file-index/` | 文件索引 |

yoga-layout 是 Ink 终端 UI 框架的核心渲染引擎，包含完整的 CSS flexbox 实现、布局缓存、脏标记追踪和测量函数。
