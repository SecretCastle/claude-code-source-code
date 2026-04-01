# 命令系统详解

## 概述

Claude Code 包含 86+ 个斜杠命令，通过 `commands.ts` 注册中心统一管理。每个命令是一个独立目录，支持懒加载。

## 命令注册中心 (commands.ts)

### 核心函数

- `getCommands(cwd)` — 加载所有可用命令
- `findCommand(name, commands)` — 按名称/别名查找命令
- `getSkillToolCommands()` — 过滤模型可调用的命令
- `filterCommandsForRemoteMode()` — 过滤远程安全命令
- `clearCommandsCache()` — 清除命令缓存

### 命令类型

```typescript
type Command =
  | { type: 'local-jsx', name: string, load: () => Promise<JSXFunction> }
  | { type: 'local', name: string, load: () => Promise<LocalFunction> }
  | { type: 'prompt', name: string, description: string }
```

### 命令来源

| 来源 | 说明 |
|------|------|
| 内置命令 | `commands/` 目录下的 86+ 个子目录 |
| 技能文件 | `.claude/skills/` 目录下的用户自定义技能 |
| 打包技能 | `skills/bundled/` 内置技能 |
| 插件命令 | 通过插件系统加载的命令 |
| 工作流命令 | 条件加载的工作流脚本 |

## 命令完整列表

### Git 工作流

| 命令 | 说明 |
|------|------|
| `/commit` | 智能提交（自动生成 commit message） |
| `/review` | 代码审查 |
| `/pr_comments` | PR 评论查看 |
| `/diff` | 查看变更 |
| `/autofix-pr` | 自动修复 PR 问题 |
| `/issue` | GitHub Issue 操作 |
| `/commit-push-pr` | 提交+推送+创建 PR 一条龙 |
| `/branch` | 分支管理 |

### 会话管理

| 命令 | 说明 |
|------|------|
| `/clear` | 清除当前对话 |
| `/compact` | 手动压缩上下文 |
| `/resume` | 恢复之前的会话 |
| `/rename` | 重命名会话 |
| `/share` | 分享会话 |
| `/session` | 会话管理 |
| `/rewind` | 回退对话 |

### 配置

| 命令 | 说明 |
|------|------|
| `/config` | 配置面板 |
| `/model` | 切换模型 |
| `/permissions` | 权限管理 |
| `/sandbox-toggle` | 沙箱开关 |
| `/effort` | 思考努力程度设置 |
| `/fast` | 快速模式切换 |
| `/env` | 环境变量查看 |
| `/output-style` | 输出风格设置 |
| `/theme` | 主题切换 |
| `/keybindings` | 键绑定配置 |
| `/terminal-setup` | 终端设置 |
| `/vim` | Vim 模式开关 |

### 认证

| 命令 | 说明 |
|------|------|
| `/login` | 登录 |
| `/logout` | 登出 |
| `/status` | 状态查看 |

### 开发辅助

| 命令 | 说明 |
|------|------|
| `/doctor` | 诊断检查 |
| `/bughunter` | Bug 猎人模式 |
| `/security-review` | 安全审查 |
| `/ctx_viz` | 上下文可视化 |
| `/debug-tool-call` | 工具调用调试 |
| `/perf-issue` | 性能问题诊断 |
| `/heapdump` | 堆转储 |

### IDE 集成

| 命令 | 说明 |
|------|------|
| `/ide` | IDE 连接管理 |
| `/vim` | Vim 模式 |
| `/desktop` | 桌面应用相关 |

### 输出与统计

| 命令 | 说明 |
|------|------|
| `/cost` | 会话成本 |
| `/usage` | 使用统计 |
| `/memory` | 记忆管理 |
| `/export` | 导出对话 |
| `/stats` | 统计信息 |
| `/tasks` | 后台任务列表 |

### MCP 与插件

| 命令 | 说明 |
|------|------|
| `/mcp` | MCP 服务器管理 |
| `/plugin` | 插件管理 |
| `/skills` | 技能管理 |
| `/reload-plugins` | 重新加载插件 |

### 团队与代理

| 命令 | 说明 |
|------|------|
| `/agents-platform` | 代理平台管理 |
| `/plan` | 计划模式 |

### 高级功能

| 命令 | 说明 |
|------|------|
| `/teleport` | 远程会话传送 |
| `/voice` | 语音模式 |
| `/proactive` | 主动模式 |
| `/assistant` | 助手模式 |
| `/bridge` | 桥接模式 |
| `/brief` | 简报模式 |
| `/onboarding` | 新手引导 |
| `/init` | 初始化项目 |
| `/help` | 帮助 |

### 其他

| 命令 | 说明 |
|------|------|
| `/add-dir` | 添加目录 |
| `/color` | 颜色设置 |
| `/copy` | 复制输出 |
| `/feedback` | 反馈 |
| `/good-claude` | 表扬 Claude |
| `/btw` | 顺便提及 |
| `/clear` | 清除对话 |
| `/files` | 文件操作 |
| `/install-github-app` | 安装 GitHub App |
| `/install-slack-app` | 安装 Slack App |
| `/memory` | 记忆管理 |
| `/passes` | 通行证管理 |
| `/privacy-settings` | 隐私设置 |
| `/release-notes` | 发行说明 |
| `/remote-env` | 远程环境 |
| `/remote-setup` | 远程设置 |
| `/tag` | 标签管理 |
| `/thinkback` | 回想 |
| `/upgrade` | 升级 |
| `/version` | 版本信息 |

## 命令过滤

命令可根据以下条件过滤：

| 过滤条件 | 说明 |
|---------|------|
| Auth State | claude-ai vs console API key |
| Remote Mode | 远程会话安全命令白名单 |
| Bridge Mode | 桥接模式安全命令白名单 |
| Feature Flags | 部分命令需要特性标志启用 |
| Hidden | 隐藏命令不出现在帮助列表中 |
