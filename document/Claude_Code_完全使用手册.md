# Claude Code 完全使用手册

> 作者：Claude Code (claude-sonnet-4-6) | 生成日期：2026-03-17
> 本手册覆盖：基础对话 · 斜杠命令 · 代码能力 · 高级功能 · Hooks · MCP · Skills · 最佳实践

---

## 目录

1. [Claude Code 是什么](#1-claude-code-是什么)
2. [基础聊天用法](#2-基础聊天用法)
3. [斜杠命令（Slash Commands）](#3-斜杠命令slash-commands)
4. [代码独有能力](#4-代码独有能力)
5. [Plan Mode（计划模式）](#5-plan-mode计划模式)
6. [Worktree（隔离工作区）](#6-worktree隔离工作区)
7. [Hooks（自动化钩子）](#7-hooks自动化钩子)
8. [MCP（模型上下文协议）](#8-mcp模型上下文协议)
9. [Skills（用户技能）](#9-skills用户技能)
10. [权限系统](#10-权限系统)
11. [Settings 配置详解](#11-settings-配置详解)
12. [快捷键与键盘操作](#12-快捷键与键盘操作)
13. [记忆系统（Memory）](#13-记忆系统memory)
14. [最佳实践与精髓](#14-最佳实践与精髓)

---

## 1. Claude Code 是什么

Claude Code 是 Anthropic 官方出品的 **CLI（命令行界面）AI 编程助手**，不是网页版 Claude。它直接运行在你的终端里，能：

- **读写你本地的任何文件**
- **执行终端命令**（npm、gradle、git 等）
- **搜索整个代码库**
- **操作 git**（commit、push、PR）
- **接入外部服务**（通过 MCP）
- **记忆跨会话信息**

```
# 启动方式
claude          # 普通启动
claude "帮我分析这个文件"   # 带初始提示启动
claude --model claude-opus-4-6   # 指定模型
```

---

## 2. 基础聊天用法

### 2.1 直接提问

最基本的用法就是用自然语言描述你的需求：

```
你：帮我看看 app/build.gradle.kts 里的依赖配置
Claude：（自动读文件并分析）
```

### 2.2 引用文件

可以直接在对话中提及路径，Claude 会自动理解：

```
你：分析一下 sdk/src/main/java/com/m4399/minigame/sdk/MiniSDK.kt 的入口流程
```

### 2.3 模糊描述也能理解

```
你：我的测试里有个 bug，权限弹窗没有正确处理多语言
Claude：（自动定位相关文件，找到问题，给出修复方案）
```

### 2.4 多轮对话（上下文保持）

Claude Code 在一个会话内记住所有上下文：

```
你：帮我重构 handlePermissionDialogs 方法
Claude：（完成修改）

你：再加个日志
Claude：（基于刚才的修改继续操作，不需要重复解释）
```

### 2.5 让 Claude 先分析再行动

```
你：先分析一下整个 sdk 模块的架构，再告诉我怎么添加新功能
```

---

## 3. 斜杠命令（Slash Commands）

在对话框输入 `/` 开头的命令，触发特定功能。

### 3.1 内置命令一览

| 命令 | 功能 | 示例 |
|------|------|------|
| `/help` | 查看帮助文档 | `/help` |
| `/clear` | 清空当前对话上下文 | `/clear` |
| `/compact` | 压缩上下文（节省 token） | `/compact` |
| `/config` | 查看/修改配置（主题、模型等） | `/config` |
| `/cost` | 查看本次会话的 token 消耗 | `/cost` |
| `/doctor` | 诊断 Claude Code 环境是否正常 | `/doctor` |
| `/exit` | 退出 Claude Code | `/exit` |
| `/fast` | 切换快速模式（更快响应） | `/fast` |
| `/feedback` | 提交反馈到官方 | `/feedback` |
| `/ide` | 管理 IDE 集成 | `/ide` |
| `/init` | 在当前项目初始化 CLAUDE.md | `/init` |
| `/login` | 登录 Anthropic 账号 | `/login` |
| `/logout` | 登出 | `/logout` |
| `/memory` | 查看/管理记忆文件 | `/memory` |
| `/model` | 切换使用的 Claude 模型 | `/model claude-opus-4-6` |
| `/permissions` | 查看当前权限设置 | `/permissions` |
| `/pr_comments` | 查看 PR 评论 | `/pr_comments 123` |
| `/release-notes` | 查看最新版本更新日志 | `/release-notes` |
| `/reset` | 重置整个会话 | `/reset` |
| `/review` | Code Review 当前 diff | `/review` |
| `/status` | 查看当前状态 | `/status` |
| `/terminal-setup` | 配置终端状态栏 | `/terminal-setup` |
| `/vim` | 切换 vim 键位模式 | `/vim` |

### 3.2 重点命令详解

#### `/clear` vs `/compact`

```
/clear   → 完全清空，Claude 忘记所有对话内容（适合开始新任务）
/compact → 智能压缩，保留关键信息但缩短上下文（适合长会话节省费用）
```

#### `/init` — 项目初始化

在项目根目录运行，Claude 会自动分析项目结构，生成 `CLAUDE.md`：

```
你：/init

Claude：（扫描项目，生成 CLAUDE.md，内含项目架构说明、构建命令、注意事项）
```

生成后，之后每次启动都会自动读取这个文件，无需重复介绍项目背景。

#### `/review` — 代码审查

```
你：/review

Claude：（分析当前 git diff，给出代码质量、潜在 bug、安全问题的完整报告）
```

---

## 4. 代码独有能力

这是 Claude Code 与普通 Claude 最大的区别：它有一套**真实执行**的工具集。

### 4.1 文件系统操作

**读文件（Read）**
```
你：读一下 app/build.gradle.kts
```
Claude 用内置 Read 工具读取，显示带行号的内容，后续编辑时精确定位。

**编辑文件（Edit）**
```
你：把第 45 行的 minSdk 改成 23
```
Claude 使用精确字符串替换，只改目标内容，不破坏其他代码。

**写新文件（Write）**
```
你：帮我新建一个 NetworkUtils.kt，实现 HTTP GET 请求的封装
```

**搜索文件（Glob）**
```
你：找出所有 build.gradle.kts 文件
Claude：（返回所有匹配路径，按修改时间排序）
```

**搜索内容（Grep）**
```
你：在整个项目中找所有调用了 MiniSDK.init 的地方
Claude：（正则搜索，返回文件名+行号+匹配行内容）
```

### 4.2 终端执行（Bash）

Claude 可以直接运行命令：

```
你：帮我运行单元测试
Claude：（执行 ./gradlew :app:connectedAndroidTest，返回完整输出）

你：看看当前 git 状态
Claude：（执行 git status，分析输出）
```

**后台运行**（适合长时间任务）：
```
你：在后台帮我 build 一下 release 包
Claude：（run_in_background=true，你可以继续聊天，完成后通知你）
```

### 4.3 Git 操作

Claude Code 深度集成 git：

```
你：帮我提交刚才的修改，写个合适的 commit message
Claude：
  1. git status（查看变更）
  2. git diff（查看具体内容）
  3. git log（参考历史风格）
  4. git add <具体文件>（不用 git add -A，避免误提交）
  5. git commit -m "修复权限弹窗多语言适配问题"
```

```
你：帮我创建一个 PR
Claude：
  1. 分析 branch 上所有 commit
  2. 生成 PR 标题 + 正文（含 Summary 和 Test Plan）
  3. gh pr create（调用 GitHub CLI）
  4. 返回 PR 链接
```

### 4.4 多文件并行操作

Claude Code 会智能并行处理：

```
你：帮我在所有 module 的 build.gradle.kts 里把 compileSdk 从 33 改成 34

Claude：（同时读取 sdk/、runtime/、app/、platform_api/ 的 build.gradle.kts，
         并行修改，一次性完成）
```

### 4.5 代码库探索（Agent/Explore）

对于复杂探索任务，Claude 会启动专用子 Agent：

```
你：帮我分析 SDK 模块和 runtime 模块是如何通信的，找出所有关键接口

Claude：（启动 Explore Agent，深度扫描 platform_api 目录，
         分析 AIDL/接口定义，返回完整调用链分析）
```

---

## 5. Plan Mode（计划模式）

Plan Mode 是 Claude Code 的"先想后做"模式，适合**复杂、多文件、有风险**的改动。

### 5.1 触发方式

Claude 会主动判断是否需要进入 Plan Mode，也可以手动触发：

```
你：帮我重构整个 SDK 的初始化流程

Claude：（主动提示）这个改动涉及多个文件，我建议先制定计划，您审批后再执行？
你：好的

# 进入 Plan Mode：
# 1. Claude 深度探索代码库
# 2. 制定详细步骤计划
# 3. 展示给你审批
# 4. 你确认后才开始执行
```

### 5.2 Plan Mode 的价值

```
❌ 不用 Plan Mode：Claude 直接改代码 → 改了 8 个文件后你发现方向错了 → 回滚痛苦

✅ 用 Plan Mode：
   - Step 1: 修改 MiniSDK.kt 的 init 签名
   - Step 2: 更新 MiniSdkConfig.kt 的 Builder
   - Step 3: 同步修改 app 模块的调用处
   → 你看完计划说"第二步方向不对，改成XXX"
   → Claude 只需调整计划，不用回滚代码
```

---

## 6. Worktree（隔离工作区）

Worktree 让 Claude 在**独立的 git 分支副本**上工作，不影响你当前的工作。

### 6.1 使用场景

```
你：用 worktree 帮我试验一下把 minSdk 降到 19 会不会有问题

Claude：（创建新 worktree → 在隔离环境里修改 → 尝试构建 → 报告结果）
# 实验结束：
# - 没问题 → keep worktree，合并到主分支
# - 有问题 → remove worktree，主分支完全不受影响
```

### 6.2 命令

```
你：开一个 worktree 叫 experiment-minsdk

Claude：（创建 .claude/worktrees/experiment-minsdk/，切换到独立分支）

你：退出 worktree，保留改动
# Claude 执行 ExitWorktree(action="keep")

你：退出 worktree，放弃所有改动
# Claude 执行 ExitWorktree(action="remove")
```

---

## 7. Hooks（自动化钩子）

Hooks 是 Claude Code 最强大的自动化功能，让你定义**"每次XX之前/之后，自动执行YY"**。

### 7.1 什么是 Hooks

Hooks 是配置在 `settings.json` 里的 shell 脚本，在特定事件发生时**由 Claude Code 宿主程序自动执行**（不是 Claude 自己执行）。

### 7.2 Hook 事件类型

| 事件 | 触发时机 |
|------|---------|
| `PreToolUse` | 工具调用**之前** |
| `PostToolUse` | 工具调用**之后** |
| `Notification` | Claude 发出通知时 |
| `Stop` | Claude 完成响应时 |
| `SubagentStop` | 子 Agent 完成时 |

### 7.3 配置示例

**场景：每次 Claude 写完代码，自动运行 ktlint 格式化**

在 `settings.json` 中：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "ktlint --format \"${file}\""
          }
        ]
      }
    ]
  }
}
```

**场景：Claude 停止响应后，播放声音提醒你**

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -c \"[console]::beep(800,200)\""
          }
        ]
      }
    ]
  }
}
```

**场景：禁止 Claude 直接 push 到 main**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '$TOOL_INPUT' | grep -q 'push.*main' && echo 'BLOCKED: 禁止直接 push 到 main' && exit 1 || exit 0"
          }
        ]
      }
    ]
  }
}
```

### 7.4 用 /update-config 技能配置

```
你：/update-config 每次 Claude 停止时，在终端显示"任务完成"
Claude：（自动修改 settings.json，添加 Stop hook）
```

---

## 8. MCP（模型上下文协议）

MCP（Model Context Protocol）是让 Claude Code **接入外部工具和服务**的协议，相当于给 Claude 装插件。

### 8.1 MCP 能做什么

| 类型 | 例子 |
|------|------|
| 数据库 | 直接查询 MySQL、PostgreSQL |
| 云服务 | 操作 AWS S3、GitHub、Jira |
| 搜索 | Brave Search、Google |
| 文件服务 | Google Drive、Notion |
| 监控 | Datadog、Sentry |
| 自定义 | 你自己写的任何服务 |

### 8.2 配置 MCP Server

在 `settings.json` 中添加：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_token_here"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "your_key"
      }
    }
  }
}
```

### 8.3 使用 MCP 的效果

```
# 接入 GitHub MCP 后：
你：查一下我们仓库最近 5 个 PR 的状态
Claude：（直接调用 GitHub API，无需你提供 token，返回真实数据）

# 接入数据库 MCP 后：
你：查一下生产数据库里最近注册的 10 个用户
Claude：（直接执行 SQL，返回结果）

# 接入 Jira MCP 后：
你：把刚才修复的 bug 关联到 JIRA-1234
Claude：（直接操作 Jira，更新 issue）
```

### 8.4 查看已接入的 MCP

```
你：/mcp
Claude：（列出当前所有已连接的 MCP server 及其提供的工具）
```

---

## 9. Skills（用户技能）

Skills 是预定义的**复杂任务模板**，通过 `/技能名` 调用，相当于一键执行一套复杂流程。

### 9.1 当前可用 Skills

#### `/commit` — 智能提交

```
你：/commit

Claude 自动：
  1. git status（找所有变更）
  2. git diff（分析改了什么）
  3. git log（学习历史 commit 风格）
  4. 起草 commit message（聚焦"为什么"，不只是"改了什么"）
  5. git add <具体文件>（安全暂存）
  6. git commit（附 Co-Authored-By: Claude）
  7. git status（确认成功）
```

#### `/review` — 代码审查

```
你：/review

Claude 输出：
  ✅ 优点分析
  🐛 潜在 Bug
  🔒 安全隐患（SQL注入、XSS等）
  📈 性能建议
  🏗️ 架构改进点
  💡 具体修改建议（附代码示例）
```

#### `/update-config` — 配置管理

```
你：/update-config 允许 Claude 自动运行 npm 命令
你：/update-config 每次文件修改后自动运行 lint
你：/update-config 设置环境变量 DEBUG=true
```

#### `/simplify` — 代码简化

```
你：/simplify

Claude：（审查刚写的代码，找出过度工程化的地方，删除不必要的抽象，简化逻辑）
```

#### `/loop` — 循环任务

```
你：/loop 5m 检查构建状态
Claude：（每 5 分钟自动检查一次，持续报告，最长 3 天）

你：/loop 10m /review
Claude：（每 10 分钟自动做一次代码审查）
```

#### `claude-api` — Claude API 开发助手

当你的代码 import 了 anthropic SDK 时自动触发，提供专业的 API 开发指导：

```kotlin
// 你的代码里有：
import com.anthropic.sdk.Anthropic

// Claude Code 自动进入 API 开发专家模式
```

### 9.2 调用技巧

```
你：/commit -m "自定义消息覆盖自动生成的"
你：/review-pr 123        （审查指定 PR 号）
```

---

## 10. 权限系统

Claude Code 有精细的权限管理，控制 Claude 可以做什么。

### 10.1 权限模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **默认模式** | 危险操作前询问用户 | 日常开发 |
| **自动批准** | 配置白名单，无需每次确认 | 流水线/CI |
| **只读模式** | 只能读，不能写/执行 | 代码审查 |

### 10.2 配置权限白名单

在 `settings.json` 中：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(./gradlew *)",
      "Bash(git status)",
      "Bash(git diff*)",
      "Edit(**/src/**/*.kt)",
      "Read(**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)"
    ]
  }
}
```

### 10.3 运行时动态授权

```
Claude 想执行：./gradlew assembleRelease
提示：[允许一次] [始终允许] [拒绝]

→ 选"始终允许" → 自动加入白名单
```

### 10.4 不同级别的配置

```
# 用户级（全局生效，所有项目）
~/.claude/settings.json

# 项目级（只对当前项目生效，可提交到 git）
.claude/settings.json

# 项目本地级（不提交，个人本地覆盖）
.claude/settings.local.json
```

---

## 11. Settings 配置详解

### 11.1 完整配置结构

```json
{
  "model": "claude-sonnet-4-6",
  "theme": "dark",
  "verbose": false,

  "permissions": {
    "allow": [],
    "deny": []
  },

  "env": {
    "MY_VAR": "value"
  },

  "hooks": {
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
    "Notification": []
  },

  "mcpServers": {}
}
```

### 11.2 常用配置项

```json
{
  "model": "claude-opus-4-6",          // 默认使用 Opus（最强）
  "theme": "dark",                      // 主题：dark/light/auto
  "verbose": true,                      // 显示详细工具调用日志
  "autoUpdates": false,                 // 关闭自动更新
  "includeCoAuthoredBy": true          // commit 中添加 Claude 署名
}
```

---

## 12. 快捷键与键盘操作

### 12.1 输入框快捷键

| 快捷键 | 功能 |
|--------|------|
| `Enter` | 发送消息 |
| `Shift + Enter` | 换行（不发送） |
| `↑ / ↓` | 历史消息导航 |
| `Ctrl + C` | 中断当前 Claude 响应 |
| `Ctrl + L` | 清空屏幕（不清上下文） |
| `Ctrl + R` | 搜索历史命令 |

### 12.2 Vim 模式

```
你：/vim   （开启 Vim 键位模式）

# 开启后：
i          → 进入插入模式
Esc        → 返回普通模式
dd         → 删除当前行
gg/G       → 跳到开头/结尾
/keyword   → 搜索历史
```

### 12.3 多行输入技巧

```
你：（输入时按 Shift+Enter 换行）
帮我：
1. 分析 OptimizedGameTest.kt
2. 找出所有 bug
3. 逐一修复并解释原因
```

---

## 13. 记忆系统（Memory）

Claude Code 有**跨会话记忆**能力，可以在不同对话之间保留知识。

### 13.1 记忆文件结构

```
~/.claude/projects/<project-path>/memory/
├── MEMORY.md          ← 自动加载，每次会话都会读取（前200行）
├── debugging.md       ← 调试经验
├── patterns.md        ← 代码模式
└── preferences.md     ← 个人偏好
```

### 13.2 让 Claude 记住重要事项

```
你：记住，我们项目里所有网络请求都要加 retry 逻辑，不要问我每次

Claude：（写入 MEMORY.md，下次会话自动遵守）
```

```
你：记住我的代码风格：函数不超过 30 行，禁止嵌套超过 3 层

Claude：（保存到记忆，永久生效）
```

### 13.3 CLAUDE.md（项目级指令）

项目根目录的 `CLAUDE.md` 是给 Claude 的**项目说明书**，每次启动自动读取：

```markdown
# CLAUDE.md 内容示例：

## 项目规范
- 必须使用 Kotlin，禁止 Java
- 使用 Kotlin DSL (build.gradle.kts)

## 构建命令
- 构建 SDK: ./gradlew :sdk:bundleReleaseAar
- 运行测试: ./gradlew :app:connectedAndroidTest

## 注意事项
- 签名文件在根目录 m4399.keystore
- minSdk=21，不要使用 21 以下的 API
```

---

## 14. 最佳实践与精髓

### 14.1 精准描述，事半功倍

❌ **模糊**：
```
你：帮我优化代码
```

✅ **精准**：
```
你：OptimizedGameTest.kt 的 waitForLoadingToDisappear 方法里，
   如果加载页面从来没出现（比如网络快速加载完成），
   当前只是 sleep 3 秒，这不够可靠。
   帮我改成：先等待游戏主界面某个特定控件出现，再认为加载完成。
   游戏主界面有一个 contentDescription 为"游戏画布"的 View。
```

### 14.2 CLAUDE.md 是你最重要的投资

花 30 分钟写好 `CLAUDE.md`，之后每次 Claude 都会自动遵守规范，不用重复解释背景。

```markdown
## 禁止事项
- 禁止自动 push 代码
- 禁止使用 Java
- 禁止添加我没要求的功能

## 代码风格
- 注释用中文
- 变量名用英文驼峰
- 每个函数必须有 KDoc
```

### 14.3 利用 Plan Mode 做高风险改动

```
凡是涉及：
- 重构核心模块
- 修改公共接口
- 影响多个模块的改动
- 数据库 schema 变更

→ 先让 Claude 出计划，你审批，再执行
```

### 14.4 任务拆分，效果更好

❌：
```
你：帮我完成整个用户认证模块
```

✅：
```
你：先帮我设计用户认证的数据模型
（确认后）
你：好，现在实现登录接口
（确认后）
你：现在加上 token 刷新逻辑
```

### 14.5 让 Claude 先读后改

```
你：先把 MiniSDK.kt 完整读一遍，理解现有架构，再告诉我应该在哪里加新功能

# 不要：你：在 MiniSDK.kt 里加一个 xxx 方法（Claude 可能没读过这个文件）
```

### 14.6 充分利用并行能力

```
你：同时帮我：
1. 分析 sdk 模块的依赖树
2. 检查 runtime 模块的 ProGuard 配置
3. 查找所有未使用的资源文件

# Claude 会并行执行这三个任务，速度是串行的 3 倍
```

### 14.7 用 /cost 控制费用

```
你：/cost
# 输出：本次会话消耗 45,231 input tokens，12,847 output tokens，约 $0.18

# 提示：超过 100k tokens 时用 /compact 压缩一下
```

### 14.8 利用 Hooks 实现全自动化工作流

```json
// 终极自动化配置示例：
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": "cd ${workdir} && ./gradlew ktlintFormat 2>/dev/null || true"
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "powershell -c \"[System.Media.SystemSounds]::Asterisk.Play()\""
        }]
      }
    ]
  }
}
```

效果：Claude 每次改完代码 → 自动格式化 → 完成后响铃提醒。

---

## 附录：常用命令速查

```bash
# 启动
claude                          # 普通启动
claude --model claude-opus-4-6  # 用 Opus 模型

# 会话内
/clear          # 清空上下文
/compact        # 压缩上下文
/cost           # 查费用
/review         # 代码审查
/commit         # 智能提交
/vim            # 开启 Vim 模式
/memory         # 管理记忆
/model          # 切换模型

# 让 Claude 执行
"帮我运行测试"              → ./gradlew test
"帮我看看错误日志"          → 读取并分析
"帮我创建 PR"              → gh pr create
"记住这个规范..."          → 写入 MEMORY.md
"用 worktree 试验一下..."  → 创建隔离环境
```

---

*本文档由 Claude Code (claude-sonnet-4-6) 生成 | 如有疑问请在对话中直接提问*
