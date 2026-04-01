# 核心引擎详解

## QueryEngine.ts (1295行) — 对话生命周期管理器

### 职责

- 管理 mutable 对话状态（消息、文件缓存、使用量）
- 工具执行 + 权限检查
- 处理用户输入，构建 prompt
- 管理会话持久化
- 预算追踪（token + 成本）
- 中断控制器

### 核心接口

```typescript
class QueryEngine {
  constructor(config: QueryEngineConfig)
  submitMessage(prompt, options): AsyncGenerator  // 主入口
}
```

### QueryEngine 配置

```typescript
type QueryEngineConfig = {
  model: string
  tools: Tools
  commands: Command[]
  systemPrompt: SystemPrompt
  messages: Message[]
  mcpServers: MCPServerConnection[]
  agentDefinition?: AgentDefinition
  // ... 更多配置
}
```

### 工作流

```
┌─────────────────────────────────────────────────────────┐
│                  QueryEngine 工作流                      │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐   │
│  │ 构建系统  │───▶│ 调用 API │───▶│ 解析响应         │   │
│  │ Prompt   │    │ (stream) │    │ (tool_use/text)  │   │
│  └──────────┘    └──────────┘    └────────┬─────────┘   │
│                                           │              │
│                              ┌────────────┼──────────┐   │
│                              │            │          │   │
│                         ┌────▼───┐  ┌─────▼────┐ ┌──▼─────┐
│                         │执行工具 │  │文本输出  │ │需要压缩 │
│                         │调用    │  │渲染给用户│ │上下文  │
│                         └────┬───┘  └──────────┘ └──┬─────┘
│                              │                      │      │
│                              ▼                      ▼      │
│                    ┌──────────────────┐    ┌─────────────┐ │
│                    │ 将结果追加到消息  │    │ Compact     │ │
│                    │ 历史，继续循环    │    │ 自动压缩    │ │
│                    └──────────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## query.ts (1729行) — 核心查询循环算法

### 职责

实现核心对话循环算法：消息格式化、流式响应处理、上下文窗口管理、compact 压缩。

### 核心函数

```typescript
function query(params: QueryParams): AsyncGenerator<StreamEvent>
```

### queryLoop 主循环

```
queryLoop():
  1. 构建系统 prompt（含上下文）
  2. 调用 Anthropic API
  3. 处理 tool_use blocks
  4. 执行工具 → 获取结果
  5. 处理特殊场景:
     - Max tokens 自动续接
     - Reactive compaction（上下文过长）
     - History snipping
     - 斜杠命令处理
     - Thinking blocks 处理
  6. 继续循环直到完成或达到最大轮次
```

### 上下文管理策略

| 策略 | 触发条件 | 行为 |
|------|---------|------|
| Auto Compact | Token 数接近上下文窗口限制 | 使用 forked agent 压缩历史 |
| Reactive Compact | API 返回 prompt_too_long 错误 | 立即压缩后重试 |
| Microcompact | 工具调用结果过大 | 局部压缩单个工具结果 |
| Max Tokens 续接 | API 返回 max_tokens 停止 | 自动续接消息继续生成 |

### 消息类型处理

```typescript
type Message =
  | UserMessage           // 用户输入
  | AssistantMessage      // Claude 响应
  | ToolUseSummaryMessage // 工具执行摘要
  | AttachmentMessage     // 文件附件
  | TombstoneMessage      // 已删除消息占位
```

---

## context.ts (190行) — 上下文管理

### 系统上下文 (getSystemContext)

- Git 状态（当前分支、主分支、Git 用户名）
- 截断的状态输出（最大 2000 字符）
- 最近 5 个提交记录
- 缓存破坏器（cache breaker）

### 用户上下文 (getUserContext)

- CLAUDE.md 文件内容
- 当前日期（ISO 格式）
- 项目记忆文件

### 特性

- 两个函数都使用 `memoize` 缓存，每次会话只计算一次
- 支持 `--bare` 模式跳过自动发现
- 可通过 `CLAUDE_CODE_DISABLE_CLAUDE_MDS` 环境变量禁用

---

## 多 API 提供商支持

```
API Client (services/api/client.ts)
├── Direct API (Anthropic 直接调用)
│   └── API Key / OAuth 认证
├── AWS Bedrock
│   └── AWS Credentials
├── Azure Foundry
│   └── Azure 认证
└── GCP Vertex AI
    └── GCP Credentials
```

每个提供商支持不同的认证方式和区域配置，通过统一的 `client.ts` 接口创建 Anthropic SDK 客户端实例。
