# 工具系统详解

## 概述

Claude Code 包含 44 个工具，通过 `Tool.ts` 基类和 `tools.ts` 注册中心统一管理。每个工具遵循标准的目录结构和接口规范。

## 工具注册中心 (tools.ts)

### 核心函数

- `getAllBaseTools()` — 返回当前环境下所有可能的工具
- `getTools(permissionContext)` — 根据权限过滤后返回可用工具
- `assembleToolPool(permissionContext, mcpTools)` — 合并内置工具和 MCP 工具
- `filterToolsByDenyRules()` — 根据拒绝规则过滤工具

### 工具过滤条件

| 过滤维度 | 说明 |
|---------|------|
| Permission Mode | auto/manual 等模式 |
| Environment Variables | CLAUDE_CODE_SIMPLE 精简模式 |
| Feature Flags | COORDINATOR_MODE, WORKFLOW_SCRIPTS 等 |
| REPL Mode | REPL 启用时隐藏部分基础工具 |
| DCE | 内部工具通过 USER_TYPE 条件加载 |

## 工具内部结构

每个工具目录遵循统一的标准结构：

```
<ToolName>/
├── <ToolName>.ts      ← 主文件，使用 buildTool() 工厂函数
├── constants.ts       ← 工具名称和配置常量
├── prompt.ts          ← 给 Claude 看的工具描述和使用说明
├── UI.tsx             ← React 组件，渲染工具执行过程和结果
└── [辅助文件]         ← 验证逻辑、类型定义等
```

### 工具接口

每个工具实现以下关键方法：

```typescript
interface Tool {
  isConcurrencySafe(): boolean  // 是否可并行执行
  isReadOnly(): boolean          // 是否只读
  shouldDefer: boolean           // 是否延迟加载
  inputSchema: ZodSchema         // 输入参数 Schema
  outputSchema: ZodSchema        // 输出参数 Schema
  execute(input, context): Promise<ToolResult>
}
```

## 完整工具列表

### 文件系统操作 (6 个)

| 工具 | 目录 | 功能 |
|------|------|------|
| **FileReadTool** | `tools/FileReadTool/` | 读取文件，支持图片、PDF（分页）、Token 限制 |
| **FileWriteTool** | `tools/FileWriteTool/` | 创建/覆写文件，含 diff 追踪 |
| **FileEditTool** | `tools/FileEditTool/` | 精确字符串替换编辑，含 hunk 编辑模式 |
| **GlobTool** | `tools/GlobTool/` | 文件模式匹配搜索 |
| **GrepTool** | `tools/GrepTool/` | 正则内容搜索（基于 ripgrep） |
| **NotebookEditTool** | `tools/NotebookEditTool/` | Jupyter Notebook 编辑 |

### Shell 执行 (3 个)

| 工具 | 功能 |
|------|------|
| **BashTool** | 执行 Shell 命令，含安全验证、沙箱支持、后台执行 |
| **PowerShellTool** | 执行 PowerShell 命令（Windows 平台） |
| **REPLTool** | Read-Eval-Print Loop 交互式会话（内部专用） |

### 任务与项目管理 (11 个)

| 工具 | 功能 |
|------|------|
| **TaskCreateTool** | 创建后台任务 |
| **TaskGetTool** | 获取任务详情 |
| **TaskListTool** | 列出所有任务 |
| **TaskUpdateTool** | 更新任务状态/描述 |
| **TaskOutputTool** | 获取后台任务输出 |
| **TaskStopTool** | 停止后台任务 |
| **TodoWriteTool** | 会话任务清单管理 |
| **AgentTool** | 启动子代理处理复杂任务 |
| **TeamCreateTool** | 创建多代理团队 |
| **TeamDeleteTool** | 删除团队 |
| **SendMessageTool** | 代理间消息传递 |

### MCP 集成 (4 个)

| 工具 | 功能 |
|------|------|
| **MCPTool** | 通用 MCP 工具执行封装 |
| **McpAuthTool** | MCP 服务 OAuth 认证 |
| **ListMcpResourcesTool** | 列出 MCP 资源 |
| **ReadMcpResourceTool** | 读取 MCP 资源 |

### 工作流与模式管理 (4 个)

| 工具 | 功能 |
|------|------|
| **EnterPlanModeTool** | 进入计划模式 |
| **ExitPlanModeTool** | 退出计划模式并实施 |
| **EnterWorktreeTool** | 创建隔离的 Git Worktree 会话 |
| **ExitWorktreeTool** | 退出 Worktree（保留或删除） |

### 调度与自动化 (4 个)

| 工具 | 功能 |
|------|------|
| **CronCreateTool** | 创建定时任务 |
| **CronDeleteTool** | 删除定时任务 |
| **CronListTool** | 列出定时任务 |
| **RemoteTriggerTool** | 远程代理触发器管理 |

### 通信与用户交互 (3 个)

| 工具 | 功能 |
|------|------|
| **AskUserQuestionTool** | 向用户提问（多选，支持预览） |
| **BriefTool** | 向用户发送主动消息 |
| **SkillTool** | 执行用户定义的 Skill |

### Web 与网络 (2 个)

| 工具 | 功能 |
|------|------|
| **WebFetchTool** | 抓取网页内容 |
| **WebSearchTool** | Web 搜索（支持域名过滤） |

### 代码智能 (2 个)

| 工具 | 功能 |
|------|------|
| **LSPTool** | LSP 语言服务集成（定义跳转、引用查找等） |
| **ToolSearchTool** | 搜索延迟加载的工具 |

### 配置与系统 (3 个)

| 工具 | 功能 |
|------|------|
| **ConfigTool** | 运行时配置修改 |
| **SyntheticOutputTool** | 结构化 JSON 输出（非交互模式） |
| **TungstenTool** | 内部分析工具 |

### 测试 (1 个)

| 工具 | 功能 |
|------|------|
| **TestingPermissionTool** | 测试相关权限管理 |

### 共享工具

| 文件 | 功能 |
|------|------|
| `shared/gitOperationTracking.ts` | 跨工具 Git 操作追踪 |
| `shared/spawnMultiAgent.ts` | 多代理启动逻辑 |

## 安全设计

### 工具权限检查流程

```
工具调用请求
    │
    ▼
checkReadPermissionForTool() / checkWritePermissionForTool()
    │
    ▼
useCanUseTool Hook
    │
    ├── 自动允许 (auto mode + 只读工具)
    ├── 自动拒绝 (deny rules)
    ├── 用户确认 (交互模式)
    └── Hook 回调 (pre-tool hooks)
```

### BashTool 安全验证

- 破坏性命令检测 (rm -rf, git reset --hard 等)
- Git 安全协议 (force push, main 分支保护)
- 沙箱支持 (文件系统隔离)
- 后台执行管理
- 只读模式
