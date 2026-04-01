# 服务层详解

## 概述

服务层包含 20+ 个专用服务模块，每个模块负责一个特定的功能域。服务之间通过 Forked Agent、后台轮询、设置同步等模式协作。

## 核心服务

### API 客户端 (services/api/)

| 文件 | 功能 |
|------|------|
| `client.ts` | 创建 Anthropic SDK 客户端（支持 4 个提供商） |
| `claude.ts` | Claude API 调用和用量追踪 |
| `bootstrap.ts` | 启动引导数据获取 |
| `errors.ts` | API 错误分类和处理 |
| `withRetry.ts` | 自动重试逻辑 |
| `filesApi.ts` | 文件 API（上传/下载） |
| `logging.ts` | API 请求日志 |
| `metricsOptOut.ts` | 指标选择退出 |

### MCP 客户端 (services/mcp/)

| 文件 | 功能 |
|------|------|
| `client.ts` | MCP 连接管理（stdio, SSE, HTTP 传输） |
| `MCPConnectionManager.tsx` | 连接生命周期管理 |
| `auth.ts` | MCP OAuth 认证 |
| `config.ts` | MCP 配置管理 |
| `officialRegistry.ts` | 官方 MCP 服务注册表 |
| `types.ts` | 类型定义 |

### OAuth 服务 (services/oauth/)

- OAuth 2.0 授权码流程（带 PKCE）
- 自动（浏览器跳转）和手动（复制粘贴）认证
- 令牌交换和刷新

### 分析服务 (services/analytics/)

- 事件日志公共 API
- 事件队列（等待 sink 附加）
- 路由到 Datadog 和 1P 事件日志
- PII 验证类型防止敏感数据泄漏
- GrowthBook A/B 测试集成

## 后台服务（Forked Agent 模式）

```
┌─────────────────────────────────────────────────────────┐
│  Forked Agent 模式                                      │
│                                                          │
│  主对话线程                                              │
│  ┌─────────────────────────────────────────────────┐    │
│  │ 用户 ↔ QueryEngine ↔ API                       │    │
│  └───────────────────────┬─────────────────────────┘    │
│                          │ fork (共享 prompt cache)      │
│         ┌────────────────┼──────────────────┐           │
│         │                │                  │           │
│    ┌────▼─────┐   ┌─────▼──────┐   ┌──────▼───────┐   │
│    │Auto Dream│   │Session     │   │  Compact     │   │
│    │记忆整合  │   │Memory      │   │  上下文压缩  │   │
│    └──────────┘   │会话笔记    │   └──────────────┘   │
│                   └────────────┘                        │
│    ┌──────────┐   ┌──────────────┐                     │
│    │Magic Docs│   │Extract       │                     │
│    │文档维护  │   │Memories      │                     │
│    │          │   │记忆提取      │                     │
│    └──────────┘   └──────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

### Auto Dream (services/autoDream/)

- 后台记忆整合服务
- 使用 `/dream` prompt 进行跨会话学习
- 当时间门控通过且有足够会话积累时触发
- 运行为 forked subagent

### Session Memory (services/SessionMemory/)

- 自动维护当前对话笔记的 Markdown 文件
- 定期在后台运行（forked subagent）
- 不中断主对话流

### Magic Docs (services/MagicDocs/)

- 自动维护带有特殊头部的 Markdown 文档
- 检测 `# MAGIC DOC: [title]` 模式的文件
- 后台更新文档内容

### Compact (services/compact/)

- 对话压缩以减少上下文窗口占用
- 微压缩 (micro-compaction) 和自动压缩策略
- 使用 forked subagent 创建压缩摘要
- 处理 prompt 缓存和边界消息

### Extract Memories (services/extractMemories/)

- 从会话记录中提取持久记忆
- 写入自动记忆目录 (`~/.claude/projects/<path>/memory/`)
- 在每个查询循环结束时通过 stop hooks 运行

## 同步服务

### Settings Sync (services/settingsSync/)

- 跨环境同步用户设置
- Interactive CLI：上传本地设置到远程（增量）
- CCR：下载远程设置
- 包含冲突解决和记忆文件同步

### Remote Managed Settings (services/remoteManagedSettings/)

- 获取企业管理设置
- 基于校验和的验证（减少网络流量）
- 失败时优雅降级（fail-open）

### Policy Limits (services/policyLimits/)

- 获取组织级策略限制
- 根据策略禁用 CLI 功能
- ETag 缓存和后台轮询

### Team Memory Sync (services/teamMemorySync/)

- 在本地文件系统和服务器 API 之间同步团队记忆
- 按 repo 范围限定（通过 git remote hash 标识）
- 组织内所有认证成员共享

## 代码智能服务

### LSP (services/lsp/)

- 管理 LSP 服务器生命周期
- 初始化和被动反馈
- 诊断和代码补全

## 其他服务

### Plugin Service (services/plugins/)

- 核心插件操作（安装、卸载、启用、禁用、更新）
- 依赖解决和市场集成

### Prompt Suggestion (services/PromptSuggestion/)

- 使用 forked agent 生成主动提示建议
- 在用户输入时后台运行推测
- 改善用户意图解读

### Tool Use Summary (services/toolUseSummary/)

- 生成已完成工具批次的可读摘要
- 使用 Haiku 创建简洁进度标签
- 用于 SDK 移动端进度更新

### Tips (services/tips/)

- 管理上下文提示和 Hint
- 基于用户行为和会话上下文调度
- 包含 IDE 特定提示和键盘快捷键

### Tool Execution (services/tools/)

- 核心工具执行引擎
- 工具调用、进度追踪、错误处理
- 权限检查和拒绝追踪

## 服务交互模式总结

| 模式 | 使用服务 | 特点 |
|------|---------|------|
| **Forked Agent** | Auto Dream, Session Memory, Magic Docs, Compact, Extract Memories | 共享 prompt cache，不阻塞主对话 |
| **后台轮询** | Remote Settings, Policy Limits, Team Memory | 指数退避重试，fail-open |
| **设置变更检测** | Settings Sync | 文件监视器 + 响应式更新 |
| **权限管线** | Tool Execution, Policy Limits | 三模式：interactive/coordinator/swarm |
