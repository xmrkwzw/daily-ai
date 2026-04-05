# OpenCode 使用指南

## 什么是 OpenCode？

OpenCode 是一个开源的 AI 编程助手，运行在终端中。它可以帮助你：
- 编写、编辑、调试代码
- 搜索和分析代码库
- 执行终端命令
- 管理 Git 操作
- 自动化重复性任务

---

## 安装

### 方法一：安装脚本（推荐）
```bash
curl -fsSL https://opencode.ai/install | bash
```

### 方法二：npm
```bash
npm install -g opencode-ai
```

### 验证安装
```bash
opencode --version
```

---

## 配置 AI 模型

**重要：OpenCode 需要配置 AI 模型的 API Key 才能使用。**

### 方式一：使用 Z.AI
```bash
opencode auth login
# 选择 Z.AI 或 Z.AI Coding Plan
```
然后在 [Z.AI API Console](https://z.ai) 获取 API Key。

### 方式二：Anthropic Claude
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 方式三：OpenAI
```bash
export OPENAI_API_KEY="sk-..."
```

### 方式四：API 代理（更便宜）
可以使用 APIYI、OpenRouter 等代理服务访问 400+ 模型。

---

## 基本使用

### 启动交互界面
```bash
opencode
# 或
opencode tui
```

### 指定目录启动
```bash
opencode /path/to/project
```

### 继续上次会话
```bash
opencode --continue
# 或
opencode -c
```

### 指定模型
```bash
opencode --model anthropic/claude-sonnet-4-20250514
```

### 非交互模式（直接执行命令）
```bash
opencode run "解释这段代码"
```

---

## 快捷键（默认）

> **Leader Key:** `Ctrl+X`（`<leader>` 前缀代表 Ctrl+X）

### 常用快捷键

| 功能 | 快捷键 |
|------|--------|
| 命令面板 | `Ctrl+P` |
| 退出应用 | `Ctrl+C`, `Ctrl+D`, `<leader>q` |
| 新建会话 | `<leader>n` |
| 会话列表 | `<leader>l` |
| 滚动到底部 | `End` 或 `Ctrl+Alt+G` |
| 滚动到顶部 | `Home` 或 `Ctrl+G` |
| 中断响应 | `Escape` |
| 发送消息 | `Enter` |
| 换行 | `Shift+Enter`, `Ctrl+Enter`, `Alt+Enter` |

### 应用控制

| 功能 | 快捷键 | 说明 |
|------|--------|------|
| 命令面板 | `Ctrl+P` | 搜索命令/文件/会话/项目 |
| 打开外部编辑器 | `<leader>e` | 在 $EDITOR 中编辑 |
| 切换主题 | `<leader>t` | 切换界面主题 |
| 切换侧边栏 | `<leader>b` | 显示/隐藏侧边栏 |
| 状态视图 | `<leader>s` | 查看状态信息 |
| 挂起终端 | `Ctrl+Z` | 挂起到后台 |

### 会话管理

| 功能 | 快捷键 | 说明 |
|------|--------|------|
| 新建会话 | `<leader>n` | 创建新会话 |
| 会话列表 | `<leader>l` | 浏览所有会话 |
| 会话时间线 | `<leader>g` | 查看历史会话 |
| 导出会话 | `<leader>x` | 导出到编辑器 |
| 压缩会话 | `<leader>c` | 压缩会话上下文 |
| 中断操作 | `Escape` | 停止当前操作 |
| 下一个子会话 | `<leader>Right` | 导航到子会话 |
| 上一个子会话 | `<leader>Left` | 导航到上一个子会话 |
| 父会话 | `<leader>Up` | 返回父会话 |

### 消息导航

| 功能 | 快捷键 | 说明 |
|------|--------|------|
| 滚动到顶部 | `Ctrl+G`, `Home` | 跳转到第一条消息 |
| 滚动到底部 | `Ctrl+Alt+G`, `End` | 跳转到最后一条消息 |
| 上一页 | `PageUp` | 上翻一页 |
| 下一页 | `PageDown` | 下翻一页 |
| 半页上翻 | `Ctrl+Alt+U` | 上翻半页 |
| 半页下翻 | `Ctrl+Alt+D` | 下翻半页 |
| 复制消息 | `<leader>y` | 复制到剪贴板 |
| 撤销消息 | `<leader>u` | 撤销最后一条消息 |
| 重做消息 | `<leader>r` | 重做消息 |
| 折叠代码块 | `<leader>h` | 展开/折叠代码块 |

### 模型和代理选择

| 功能 | 快捷键 | 说明 |
|------|--------|------|
| 模型列表 | `<leader>m` | 浏览可用模型 |
| 切换到下个模型 | `F2` | 切换到下一个最近使用的模型 |
| 切换到上个模型 | `Shift+F2` | 切换到上一个最近使用的模型 |
| 代理列表 | `<leader>a` | 浏览可用代理 |
| 下一个代理 | `Tab` | 切换到下一个代理 |
| 上一个代理 | `Shift+Tab` | 切换到上一个代理 |

### 输入框操作

| 功能 | 快捷键 | 说明 |
|------|--------|------|
| 发送 | `Enter` | 发送消息 |
| 换行 | `Shift+Enter`, `Ctrl+Enter`, `Alt+Enter` | 插入换行 |
| 清除输入 | `Ctrl+C` | 清除输入框 |
| 粘贴 | `Ctrl+V` | 从剪贴板粘贴 |
| 光标左移 | `Left`, `Ctrl+B` | 光标向左移动 |
| 光标右移 | `Right`, `Ctrl+F` | 光标向右移动 |
| 行首 | `Ctrl+A` | 跳转到行首 |
| 行尾 | `Ctrl+E` | 跳转到行尾 |

---

## Ctrl+P 命令面板

**快捷键：`Ctrl+P`**

这是 OpenCode 最强大的功能之一，可以快速搜索和执行各种操作。

### 功能类别

| 类别 | 说明 | 示例 |
|------|------|------|
| **Commands** | 执行内置或自定义命令 | `/init`, `/undo`, `/help` |
| **Files** | 搜索并打开项目中的文件 | `src/index.ts` |
| **Sessions** | 搜索并切换到历史会话 | 之前的对话记录 |
| **Projects** | 搜索并切换到其他项目 | 切换工作目录 |

### 使用技巧

1. **直接输入** - 输入关键词搜索匹配项
2. **输入 `>`** - 快速切换到命令模式
3. **模糊搜索** - 支持模糊匹配，输入部分字母即可找到
4. **快速切换项目** - 在多项目间快速切换

### 搜索结果分组

```
> Ctrl+P 打开后显示：
┌─────────────────────────────────────┐
│ 🔍 搜索...                        │
├─────────────────────────────────────┤
│ 📁 Projects (项目)                  │
│   └ project-a                       │
│   └ project-b                       │
├─────────────────────────────────────┤
│ 📄 Files (文件)                     │
│   └ src/index.ts                    │
│   └ src/App.vue                     │
├─────────────────────────────────────┤
│ 💬 Sessions (会话)                   │
│   └ 修复登录Bug                     │
│   └ 添加用户功能                    │
├─────────────────────────────────────┤
│ ⚡ Commands (命令)                   │
│   └ /init                           │
│   └ /help                           │
└─────────────────────────────────────┘
```

---

## Tab 键功能

`Tab` 键主要用于**自动补全**和**代理切换**：

| 功能 | 快捷键 | 说明 |
|------|--------|------|
| 代理自动补全 | `Tab` | 在输入时自动补全代理名称 |
| 切换到下一个代理 | `Tab` | 在多个代理间切换 |
| 切换到上一个代理 | `Shift+Tab` | 反向切换代理 |

---

## 内置命令

| 命令 | 说明 |
|------|------|
| `/init` | 初始化项目 |
| `/undo` | 撤销操作 |
| `/redo` | 重做操作 |
| `/share` | 分享会话 |
| `/help` | 帮助 |

---

## 自定义命令

在 `.opencode/commands/` 目录下创建 `.md` 文件：

```markdown
---
description: 运行测试
agent: build
model: anthropic/claude-sonnet-4-20250514
---

运行完整测试套件并显示结果。
```

使用方式：`/test`

---

## CLI 命令参考

| 命令 | 说明 |
|------|------|
| `opencode` | 启动 TUI 界面 |
| `opencode tui` | 启动终端界面 |
| `opencode run <prompt>` | 非交互模式执行 |
| `opencode serve` | 启动后端服务器 |
| `opencode web` | 启动 Web 界面 |
| `opencode attach` | 连接到运行中的服务器 |
| `opencode auth` | 管理 AI 凭证 |
| `opencode models` | 列出可用模型 |
| `opencode session` | 管理会话 |
| `opencode github` | GitHub 操作 |
| `opencode pr` | 获取和 checkout PR |
| `opencode export` | 导出对话数据 |
| `opencode stats` | 显示使用统计 |
| `opencode init` | 初始化项目配置 |

---

## 配置文件

OpenCode 使用 JSON 格式配置，位置：

1. 项目级：`./opencode.json`
2. 用户级：`~/.config/opencode/opencode.json`

示例配置：
```json
{
  "$schema": "https://opencode.ai/config.json",
  "theme": "opencode",
  "model": "anthropic/claude-sonnet-4-20250514",
  "autoupdate": true,
  "tools": {
    "bash": true,
    "edit": true,
    "write": true,
    "read": true
  }
}
```

---

## 常见问题

### 为什么现在可以和你聊天？

你现在能和我聊天是因为 **当前环境已经配置了 AI 模型**。OpenCode 可以通过以下方式获得 AI 能力：

1. **远程服务器** - 如果你连接到了已经配置好模型的 OpenCode 服务器
2. **本地配置** - 本地已配置了 API Key（ANTHROPIC_API_KEY、OPENAI_API_KEY 等）
3. **Z.AI 登录** - 已通过 `opencode auth login` 登录

### 如何确认自己的配置？

```bash
# 查看可用模型
opencode models

# 查看认证状态
opencode auth list
```

### 如果本地没有配置模型？

可以在远程服务器上使用 OpenCode，或者本地配置 API Key：
1. 运行 `opencode auth login` 登录 Z.AI
2. 或设置环境变量 `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`

### AGENTS.md 是什么？
在项目根目录创建 `AGENTS.md` 可以定义项目的开发规范、命令、规则等。

---

## 资源链接

- 官网：https://opencode.ai
- 文档：https://docs.z.ai
- GitHub：https://github.com/opencodeai/opencode
