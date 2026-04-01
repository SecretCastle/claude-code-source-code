# UI 层、状态管理与 Hook 系统

## 状态管理 (state/)

### 架构

Claude Code 使用自定义状态管理系统（非 Redux/Zustand）：

```
┌─────────────────────────────────────────────────────┐
│                    State Management                   │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐ │
│  │  AppStateStore    │     │  bootstrap/state.ts    │ │
│  │  (React Store)   │     │  (全局单例状态)         │ │
│  │                  │     │                        │ │
│  │ • messages       │     │ • sessionId            │ │
│  │ • toolPermissions│     │ • totalCostUSD         │ │
│  │ • model          │     │ • originalCwd          │ │
│  │ • speculation    │     │ • telemetry providers  │ │
│  │ • isProcessing   │     │ • registered hooks     │ │
│  └──────────────────┘     └────────────────────────┘ │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐ │
│  │  selectors.ts     │     │  onChangeAppState.ts   │ │
│  │  (状态选择器)     │     │  (状态变更回调)         │ │
│  └──────────────────┘     └────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 文件结构

| 文件 | 功能 |
|------|------|
| `AppState.tsx` | React Provider 包装器 |
| `AppStateStore.ts` | 主状态 Store（含完整类型定义） |
| `store.ts` | 通用 Store 实现（发布/订阅模式） |
| `selectors.ts` | 状态选择器函数 |
| `onChangeAppState.ts` | 状态变更处理函数 |
| `teammateViewHelpers.ts` | Teammate 视图辅助函数 |

### 设计原则

- **DeepImmutable**: 所有状态使用 DeepImmutable 包装
- **发布/订阅**: Store 基于监听器模式
- **按需订阅**: 通过 `useAppState` hook 按需订阅
- **无副作用**: 状态更新不产生副作用

### 状态分类

| 类别 | 属性 |
|------|------|
| 项目状态 | originalCwd, projectRoot |
| 会话状态 | sessionId, parentSessionId |
| 成本追踪 | totalCostUSD, totalAPIDuration |
| 模型状态 | mainLoopModelOverride, modelStrings |
| 遥测 | meters, counters, tracers |
| 代理状态 | agentColorMap, agentColorIndex |
| 插件状态 | inlinePlugins, sessionCreatedTeams |
| 定时任务 | scheduledTasksEnabled, sessionCronTasks |

### 全局状态 (bootstrap/state.ts)

bootstrap/state.ts 包含 150+ 个全局状态属性，是整个应用的共享状态层。关键状态包括：

- 会话标识 (sessionId)
- 工作目录 (originalCwd, projectRoot)
- 成本和计时 (totalCostUSD, totalAPIDuration)
- 遥测提供商 (meters, tracers, loggers)
- 注册的 Hook
- 代理颜色映射
- 插件状态
- Cron 任务

---

## React Hook 系统 (hooks/)

### 概述

80+ 个自定义 React Hook，覆盖权限、输入、IDE 集成、任务管理等所有方面。

### 核心分类

#### 工具/权限 Hook

| Hook | 功能 |
|------|------|
| `useCanUseTool` | 工具权限检查 |
| `useToolPermission/*` | 工具权限处理（三种模式） |

#### 输入 Hook

| Hook | 功能 |
|------|------|
| `useTextInput` | 文本输入处理 |
| `useVimInput` | Vim 模式输入 |
| `usePasteHandler` | 粘贴处理 |
| `useTypeahead` | 自动补全 |
| `useSearchInput` | 搜索输入 |
| `useInputBuffer` | 输入缓冲 |

#### IDE 集成 Hook

| Hook | 功能 |
|------|------|
| `useIDEIntegration` | IDE 集成管理 |
| `useIdeSelection` | IDE 文本选择 |
| `useIdeAtMentioned` | IDE @-mention |
| `useDiffInIDE` | IDE 中的 Diff 显示 |
| `useDirectConnect` | 直接连接 IDE |
| `useIdeConnectionStatus` | IDE 连接状态 |
| `useIdeLogging` | IDE 日志 |

#### 模型与设置 Hook

| Hook | 功能 |
|------|------|
| `useMainLoopModel` | 主循环模型选择 |
| `useSettings` | 当前设置访问 |
| `useSettingsChange` | 设置变更监听 |
| `useDynamicConfig` | 动态配置 |

#### 任务/代理 Hook

| Hook | 功能 |
|------|------|
| `useTasksV2` | 后台任务管理 |
| `useTaskListWatcher` | 任务列表监视 |
| `useSwarmInitialization` | Swarm 初始化 |
| `useSwarmPermissionPoller` | Swarm 权限轮询 |
| `useBackgroundTaskNavigation` | 后台任务导航 |
| `useScheduledTasks` | 定时任务管理 |

#### 搜索 Hook

| Hook | 功能 |
|------|------|
| `useHistorySearch` | 历史搜索 |
| `useSearchInput` | 搜索输入状态 |
| `useGlobalKeybindings` | 全局键绑定 |

#### UI 效果 Hook

| Hook | 功能 |
|------|------|
| `useTerminalSize` | 终端尺寸追踪 |
| `useVirtualScroll` | 虚拟滚动 |
| `useBlink` | 光标闪烁动画 |
| `useElapsedTime` | 经过时间 |
| `useAfterFirstRender` | 首次渲染后执行 |
| `useMinDisplayTime` | 最小显示时间 |

#### 通知 Hook

| Hook | 功能 |
|------|------|
| `notifs/useStartupNotification` | 启动通知 |
| `notifs/useAutoModeUnavailableNotification` | 自动模式不可用通知 |
| `notifs/useTeammateShutdownNotification` | Teammate 关闭通知 |

---

## UI 组件 (components/)

### 概述

150+ React 组件，基于定制版 Ink 框架渲染终端 UI。

### 核心布局

| 组件 | 功能 |
|------|------|
| `App.tsx` | 根应用组件 |
| `FullscreenLayout.tsx` | 主应用布局 |
| `VirtualMessageList.tsx` | 虚拟化消息列表（高性能滚动） |
| `MessageSelector.tsx` | 消息选择界面 |

### 消息渲染

| 组件 | 功能 |
|------|------|
| `Messages.tsx` | 消息列表渲染 |
| `Message.tsx` | 单条消息渲染 |
| `MessageRow.tsx` | 消息行 |
| `MessageResponse.tsx` | 消息响应 |
| `MessageTimestamp.tsx` | 时间戳 |
| `MessageModel.tsx` | 模型标识 |
| `Markdown.tsx` | Markdown 渲染 |
| `MarkdownTable.tsx` | Markdown 表格 |
| `HighlightedCode/` | 代码语法高亮 |

### 输入组件

| 组件 | 功能 |
|------|------|
| `TextInput.tsx` | 文本输入 |
| `VimTextInput.tsx` | Vim 模式输入 |
| `BaseTextInput.tsx` | 基础文本输入 |
| `PromptInput/` | Prompt 输入目录 |
| `SearchBox.tsx` | 搜索框 |
| `ContextSuggestions.tsx` | 上下文建议 |

### 工具显示

| 组件 | 功能 |
|------|------|
| `FileEditToolDiff.tsx` | 文件编辑 Diff |
| `ToolUseLoader.tsx` | 工具执行加载器 |
| `FallbackToolUseErrorMessage.tsx` | 错误消息 |
| `StructuredDiff/` | 结构化 Diff 显示 |

### 对话框

| 组件 | 功能 |
|------|------|
| `TrustDialog/` | 信任确认对话框 |
| `MCPServerApprovalDialog.tsx` | MCP 服务器审批 |
| `BridgeDialog.tsx` | Bridge 连接对话框 |
| `ExportDialog.tsx` | 导出对话 |
| `GlobalSearchDialog.tsx` | 全局搜索 |
| `QuickOpenDialog.tsx` | 快速打开 |
| `Onboarding.tsx` | 首次运行体验 |
| `IdeOnboardingDialog.tsx` | IDE 集成引导 |
| `WorktreeExitDialog.tsx` | Worktree 退出确认 |

### 设置面板

| 组件 | 功能 |
|------|------|
| `Settings/` | 设置面板（Status, Config, Usage 标签页） |
| `ModelPicker.tsx` | 模型选择器 |
| `ThemePicker.tsx` | 主题选择器 |
| `LanguagePicker.tsx` | 语言选择器 |
| `OutputStylePicker.tsx` | 输出风格选择器 |

### 状态指示

| 组件 | 功能 |
|------|------|
| `StatusLine.tsx` | 状态栏 |
| `DevBar.tsx` | 开发者工具栏 |
| `Spinner/` | 加载动画 (Teammate, Glimmer, Shimmer, Flashing) |
| `EffortIndicator.ts` | 思考努力指示 |
| `FastIcon.tsx` | 快速模式图标 |
| `TokenWarning.tsx` | Token 使用警告 |

### 代理与团队

| 组件 | 功能 |
|------|------|
| `AgentProgressLine.tsx` | 代理进度条 |
| `TeammateViewHeader.tsx` | Teammate 视图头部 |
| `CoordinatorAgentStatus.tsx` | 协调器代理状态 |
| `agents/` | 代理相关组件 |
| `teams/` | 团队协作组件 |

### 任务管理

| 组件 | 功能 |
|------|------|
| `TaskListV2.tsx` | 任务列表 V2 |
| `ResumeTask.tsx` | 任务恢复 |

### 特殊组件

| 组件 | 功能 |
|------|------|
| `LogoV2/` | 动画 Logo 和欢迎屏 |
| `Feedback.tsx` | 反馈收集 |
| `FeedbackSurvey/` | 调查问卷 |
| `DiagnosticsDisplay.tsx` | 诊断信息 |
| `ContextVisualization.tsx` | 上下文可视化 |
| `Teleport*.tsx` | 远程会话传送相关 |
| `ClickableImageRef.tsx` | 可点击图片引用 |

### 设计系统

| 组件 | 功能 |
|------|------|
| `design-system/` | 设计系统组件 (Button, Pane, Tabs 等) |
| `ui/` | 额外 UI 组件 (TreeSelect, OrderedList) |

---

## Ink 终端 UI 框架 (ink/)

### 概述

定制版的 React 终端 UI 框架（从 Ink fork 并深度定制）。

### 核心组件

| 组件/文件 | 功能 |
|----------|------|
| `ink.tsx` | 主入口（React reconciler 集成） |
| `reconciler.ts` | React 自定义 reconciler |
| `renderer.ts` | 终端渲染器 |
| `screen.ts` | 屏幕缓冲管理 |
| `output.ts` | 输出格式化 |
| `dom.ts` | 虚拟 DOM |
| `selection.ts` | 文本选择支持 |
| `terminal.ts` | 终端抽象层 |

### UI 组件

- `Box.tsx`, `Text.tsx` — 布局组件
- `Button.tsx`, `Link.tsx` — 交互元素
- `ErrorOverview.tsx` — 错误显示

### Hook

- `useApp` — App 实例
- `useInput` — 键盘输入
- `useStdin` — 标准输入流
- `useTerminalFocus` — 终端焦点状态
- `useAnimationFrame` — 动画帧

### 事件系统

- `keyboard-event.ts` — 键盘事件
- `click-event.ts` — 点击事件
- `input-event.ts` — 输入事件

### 特殊能力

- 鼠标追踪
- 文本选择
- 超链接支持
- 动画帧循环
- 焦点管理
