# 核心启动流程与工作流

## 启动流程

```
用户执行 `claude` 命令
      │
      ▼
┌─────────────┐
│  cli.tsx     │  快速路径判断 (--version, --dump-system-prompt,
│  入口点      │  --claude-in-chrome-mcp, --daemon-worker 等)
└──────┬──────┘
       │ 正常 REPL 启动
       ▼
┌─────────────┐
│  main.tsx    │  注册所有 Commander 命令, 初始化配置
│  主模块      │  加载 Auth → Policy → MCP → Plugins → Skills
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  init.ts     │  enableConfigs() → applySafeConfigEnvironmentVariables()
│  初始化      │  → applyExtraCACerts() → setupGracefulShutdown()
│              │  → detectCurrentRepository() → configureGlobalAgents()
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  setup.ts    │  信任对话框 (Trust Dialog), Auth 认证
│  配置向导    │  API Key 验证, 订阅状态检查
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ launchRepl() │  进入 REPL 交互循环
│ replLauncher │  Ink (React Terminal UI) 渲染
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  REPL Loop   │  用户输入 → processUserInput() → QueryEngine
│  交互主循环  │  → API 调用 → 工具执行 → 输出渲染
└─────────────┘
```

## 完整工作流（一次用户交互的生命周期）

```
用户输入 "帮我修一个 bug"
│
▼
┌─────────────────────────────────────────────────────────────┐
│ 1. INPUT PROCESSING                                         │
│    processUserInput() → slash command 识别 → 附件处理        │
│    → 文件 @-mention 解析 → 系统提示注入                      │
└──────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. QUERY ENGINE LOOP                                        │
│                                                             │
│    ┌───────────────────────────────────────────────────┐    │
│    │ a. 构建 System Prompt                              │    │
│    │    - CLAUDE.md 文件 + 记忆 + 工具描述 + 权限规则  │    │
│    │    - 用户上下文 + 项目上下文                       │    │
│    └─────────────────────┬─────────────────────────────┘    │
│                          │                                   │
│    ┌─────────────────────▼─────────────────────────────┐    │
│    │ b. 调用 Anthropic API (Messages API, streaming)    │    │
│    │    - query.ts 构建请求体                            │    │
│    │    - 处理流式响应 (ContentBlock stream)             │    │
│    │    - 实时渲染到终端 (Ink/React)                     │    │
│    └─────────────────────┬─────────────────────────────┘    │
│                          │                                   │
│    ┌─────────────────────▼─────────────────────────────┐    │
│    │ c. 解析响应                                        │    │
│    │    ├─ text → 直接输出给用户                         │    │
│    │    ├─ tool_use → 进入工具执行流程                   │    │
│    │    └─ thinking → 内部推理（可能展示）               │    │
│    └─────────────────────┬─────────────────────────────┘    │
│                          │                                   │
│    ┌─────────────────────▼─────────────────────────────┐    │
│    │ d. 工具执行                                        │    │
│    │    - 权限检查 (useCanUseTool hook)                  │    │
│    │    - Hook 触发 (pre/post tool execution)            │    │
│    │    - 工具执行 → 返回 tool_result                    │    │
│    │    - 结果追加到消息历史                              │    │
│    │    → 返回步骤 b (继续循环)                          │    │
│    └───────────────────────────────────────────────────┘    │
│                                                             │
│    退出条件: 无更多 tool_use 或 用户中断                      │
└──────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. POST-PROCESSING                                          │
│    - 成本计算 (cost-tracker.ts)                              │
│    - 历史记录 (history.ts)                                   │
│    - 会话存储 (sessionStorage.ts)                            │
│    - 遥测上报 (analytics)                                    │
│    - 上下文窗口检查 → 可能触发 auto-compact                  │
└──────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
                   等待下一次用户输入
```

## 初始化阶段详解 (init.ts)

```
init() [memoized]
  │
  ├── enableConfigs()                    # 启用配置系统
  ├── applySafeConfigEnvironmentVariables() # 安全环境变量（信任前）
  ├── applyExtraCACertsFromConfig()       # TLS 证书配置
  ├── setupGracefulShutdown()             # 优雅关闭
  ├── detectCurrentRepository()           # 仓库检测
  ├── configureGlobalAgents()             # HTTP 代理
  ├── configureGlobalMTLS()               # mTLS 配置
  ├── setShellIfWindows()                 # Windows Shell 设置
  ├── applyConfigEnvironmentVariables()   # 完整环境变量（信任后）
  ├── initializeTelemetry()               # OpenTelemetry 初始化
  ├── populateOAuthAccountInfoIfNeeded()  # OAuth 账户信息
  ├── initializePolicyLimitsLoadingPromise() # 策略限制加载
  └── initializeRemoteManagedSettingsLoadingPromise() # 远程设置加载
```

## main.tsx 启动序列 (4683行)

main.tsx 是整个应用的编排中心，按以下顺序执行：

1. **早期初始化** (行 1-130)
   - 启动性能分析
   - MDM 原始读取（并行）
   - Keychain 预取（并行）
   - 导入所有核心模块

2. **命令注册** (行 200-600)
   - 注册 Commander 命令和选项
   - 设置全局标志 (--model, --permission-mode 等)

3. **主操作处理** (行 2300-3800)
   - 信任对话框流程
   - MCP 初始化
   - LSP 设置
   - 插件和技能加载
   - 根据不同入口点启动 REPL

4. **辅助功能** (行 4500-4683)
   - 多代理选项提取
   - 主动模式激活
   - Brief 模式处理
   - 游标重置
