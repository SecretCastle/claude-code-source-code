# Claude Code v2.1.88 架构全景分析

## 顶层架构总览

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Claude Code 架构                                │
│                                                                          │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────────────────┐  │
│  │  CLI Entry   │──▶│  Bootstrap   │──▶│     main.tsx (4683 lines)    │  │
│  │  cli.tsx     │   │  state.ts    │   │   Commander + React + Ink    │  │
│  └─────────────┘   └──────────────┘   └──────────────┬───────────────┘  │
│                                                       │                  │
│                         ┌─────────────────────────────┼────────────┐     │
│                         │                             │            │     │
│                    ┌────▼────┐                  ┌─────▼─────┐ ┌────▼──┐  │
│                    │  REPL   │                  │  Commands  │ │Dialog │  │
│                    │  Loop   │                  │  (86+)     │ │Launchers│ │
│                    └────┬────┘                  └───────────┘ └───────┘  │
│                         │                                              │
│              ┌──────────┼──────────────────────┐                       │
│              │          │                      │                       │
│         ┌────▼───┐ ┌───▼─────────┐    ┌───────▼───────┐              │
│         │Query   │ │ QueryEngine │    │   Tools       │              │
│         │Engine  │ │ (API Loop)  │    │   (44 tools)  │              │
│         └────┬───┘ └───┬─────────┘    └───────┬───────┘              │
│              │         │                      │                       │
│    ┌─────────┼─────────┼──────────────────────┼────────────┐          │
│    │         │         │                      │            │          │
│ ┌──▼───┐ ┌──▼────┐ ┌──▼──────┐ ┌────────▼───┐ ┌──────▼─────┐      │
│ │Anthropic│ │MCP    │ │Services │ │  React     │ │   State    │      │
│ │API     │ │Client │ │Layer    │ │ Components │ │  Management│      │
│ └───────┘ └───────┘ └─────────┘ │  (150+)    │ │  (Store)   │      │
│                                  └────────────┘ └────────────┘      │
│                                                                          │
│  ┌──────────┐ ┌─────────┐ ┌────────┐ ┌───────────┐ ┌──────────────┐   │
│  │  Utils   │ │  Hooks  │ │ Skills │ │  Plugins  │ │  Keybindings │   │
│  │ (329)    │ │  (80+)  │ │ System │ │  System   │ │   System     │   │
│  └──────────┘ └─────────┘ └────────┘ └───────────┘ └──────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

## 代码规模统计

| 维度 | 数量 |
|------|------|
| 入口文件 | 5 |
| 核心引擎文件 | 2 (QueryEngine + query) |
| 工具 | 44 |
| 斜杠命令 | 86+ |
| 服务模块 | 20+ |
| React 组件 | 150+ |
| 自定义 Hook | 80+ |
| 工具函数 | 329 |
| Bridge 文件 | 30+ |
| 全局状态属性 | 150+ |
| **main.tsx** | **4,683 行** |
| **总计 src/ 目录** | **37 个顶层模块** |

## 目录结构总览

```
src/
├── main.tsx              # 核心启动逻辑 (4683行)
├── QueryEngine.ts        # 查询引擎 (1295行)
├── query.ts              # API 调用层 (1729行)
├── commands.ts           # 命令注册中心
├── tools.ts              # 工具注册中心
├── context.ts            # 上下文管理
├── Tool.ts               # 工具基类
├── Task.ts               # 任务基类
├── setup.ts              # 首次运行向导
├── history.ts            # 历史记录
├── cost-tracker.ts       # 成本追踪
├── ink.ts                # Ink 框架入口
│
├── entrypoints/          # 入口点 (cli, init, mcp, sdk)
├── bootstrap/            # 全局状态引导
├── commands/             # 86+ 斜杠命令
├── tools/                # 44 个工具
├── services/             # 20+ 服务模块
├── state/                # 状态管理 (自定义 Store)
├── hooks/                # 80+ React Hook
├── components/           # 150+ UI 组件
├── ink/                  # 定制版 Ink 终端框架
├── bridge/               # IDE 桥接 (30+ 文件)
├── coordinator/          # 多代理协调器
├── assistant/            # 助手模式 (KAIROS)
├── buddy/                # 伙伴精灵
├── remote/               # 远程会话
├── server/               # 内置 HTTP 服务
├── cli/                  # CLI 传输层
├── vim/                  # Vim 编辑模式
├── voice/                # 语音模式
├── keybindings/          # 键绑定系统
├── native-ts/            # 原生 TS 模块 (Flexbox 引擎)
├── skills/               # 技能系统
├── plugins/              # 插件系统
├── utils/                # 329 个工具函数
├── types/                # 类型定义
├── constants/            # 常量
├── schemas/              # Schema 定义
├── memdir/               # 记忆目录系统
├── migrations/           # 迁移系统
├── moreright/            # UI 扩展
├── outputStyles/         # 输出样式
├── upstreamproxy/        # 上游代理
└── screens/              # 屏幕定义
```

## 技术栈

| 技术 | 用途 |
|------|------|
| TypeScript | 全局类型安全 |
| React | UI 组件和 Hook |
| Ink (定制版) | 终端 UI 渲染 |
| Anthropic SDK | Claude API 集成 |
| MCP SDK | Model Context Protocol 客户端 |
| Bun | 运行时和打包器 |
| lodash-es | 工具函数 |
| Zod | Schema 验证 |
| OpenTelemetry | 遥测追踪 |
| Commander.js | CLI 命令解析 |

## 关键架构特征

### 1. 特性标志系统 (`feature()` gate)

```typescript
feature('KAIROS')           // 助手模式
feature('PROACTIVE')        // 主动模式
feature('COORDINATOR_MODE') // 协调器
feature('DAEMON')           // 守护进程
feature('BRIDGE_MODE')      // IDE 桥接
feature('VOICE_MODE')       // 语音模式
feature('AGENT_TRIGGERS')   // 定时触发
feature('REACTIVE_COMPACT') // 响应式压缩
```

大量功能通过 `bun:bundle` 的 `feature()` 进行条件编译，外部版本自动移除内部功能代码（Dead Code Elimination）。

### 2. 死代码消除 (DCE)

内部工具（REPLTool, SleepTool, SuggestBackgroundPRTool 等）通过 `process.env.USER_TYPE === 'ant'` 条件加载，确保外部构建不包含这些工具。

### 3. 性能优化

- **快速路径**: cli.tsx 对 `--version` 等常用参数零模块加载
- **延迟加载**: 大量模块使用 `require()` 动态加载
- **Memoize**: context.ts 等关键函数使用缓存
- **并行初始化**: MDM 读取、Keychain 预取与模块加载并行执行
- **启动性能分析**: startupProfiler 追踪每个阶段耗时
