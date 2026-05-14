---
title: "OpenClaw 安装与使用指南：你的个人 AI 助手"
date: 2026-05-14
draft: false
tags: ["OpenClaw", "AI助手", "开源", "个人助手", "Linux"]
categories: ["AI工具"]
summary: "OpenClaw 是 37万 Star 的开源个人 AI 助手，跑在你电脑上，通过聊天软件控制，能做邮件、日历、代码、浏览等几乎所有事。"
showToc: true
TocOpen: true
weight: 1
series: ["AI工具"]
---

## 一、OpenClaw 是什么

OpenClaw 🦞 是一个**开源个人 AI 助手**，运行在你自己的电脑上，通过 WhatsApp/Telegram/Discord/微信等聊天软件控制。

| 维度 | 说明 |
|------|------|
| GitHub Stars | 371,644 |
| 开源协议 | MIT |
| 语言 | TypeScript |
| 创始人 | Peter Steinberger (@steipete) |
| 当前版本 | 2026.5.7 |
| 支持 OS | macOS / Linux / Windows |

**核心特点**：
- 🏠 **本地运行** — 数据在你自己的电脑上，不传到第三方
- 💬 **聊天控制** — 通过 Telegram/Discord/WhatsApp 等控制
- 🧠 **持久记忆** — 记住你的偏好和上下文，24/7 运行
- 🔧 **技能扩展** — 5400+ 社区技能，还能让 AI 自己写新技能
- 🌐 **浏览器控制** — 能帮你浏览网页、填表、抓数据

---

## 二、安装

### 2.1 环境要求

| 要求 | 说明 |
|------|------|
| Node.js | ≥ 22.14（推荐 24+） |
| npm | 最新版 |
| 磁盘 | ~500MB |
| AI 模型 API Key | Claude / GPT / 本地模型 |

### 2.2 一键安装

```bash
# 方式 1：官方安装脚本（推荐）
curl -fsSL https://openclaw.ai/install.sh | bash

# 方式 2：npm 安装
npm i -g openclaw

# 方式 3：从源码安装
git clone https://github.com/openclaw/openclaw.git
cd openclaw && corepack enable && pnpm install
pnpm openclaw onboard
```

### 2.3 验证安装

```bash
openclaw --version
# OpenClaw 2026.5.7
```

### 2.4 初始化配置

```bash
openclaw onboard
# 交互式引导，设置：
# 1. AI 模型 API Key
# 2. 聊天渠道（Telegram/Discord 等）
# 3. 个人偏好
```

---

## 三、核心概念

### 3.1 架构

```
你的手机（Telegram/WhatsApp/Discord）
  ↓ 消息
OpenClaw Gateway（本地 WebSocket 服务）
  ↓ 调度
Agent（AI 大脑，Claude/GPT）
  ↓ 调用
Skills（技能插件，5400+）
  ↓ 执行
你的电脑（文件/浏览器/终端/邮件/日历）
```

### 3.2 关键组件

| 组件 | 说明 |
|------|------|
| **Gateway** | 本地 WebSocket 网关，所有消息的中枢 |
| **Agent** | AI 代理，理解你的意图并调度技能 |
| **Skills** | 技能插件，每个技能做一类事 |
| **Channels** | 聊天渠道，连接 Telegram/Discord 等 |
| **Memory** | 持久记忆，跨会话保存上下文 |
| **Cron** | 定时任务，主动提醒和执行 |

---

## 四、常用命令

### 4.1 基本操作

```bash
# 启动 Gateway（后台运行）
openclaw gateway

# 查看状态
openclaw status

# 健康检查
openclaw health

# 诊断问题
openclaw doctor

# 打开控制面板
openclaw dashboard

# 交互式终端聊天
openclaw chat
```

### 4.2 配置管理

```bash
# 交互式配置
openclaw configure

# 查看配置
openclaw config get <key>

# 设置配置
openclaw config set <key> <value>

# 查看配置文件路径
openclaw config file
```

### 4.3 模型管理

```bash
# 查看可用模型
openclaw models list

# 设置默认模型
openclaw config set agent.model claude-sonnet-4-20250514
```

### 4.4 渠道管理

```bash
# 登录 Telegram
openclaw channels login --channel telegram

# 登录 Discord
openclaw channels login --channel discord

# 登录 WhatsApp
openclaw channels login --channel whatsapp

# 查看渠道状态
openclaw channels status
```

### 4.5 技能管理

```bash
# 列出所有技能
openclaw skills list

# 搜索技能
openclaw skills list --filter "gmail"

# 安装社区技能（从 ClawHub）
openclaw skills install <skill-name>

# 让 AI 自己创建技能
# 在聊天中告诉它："帮我创建一个技能，每天早上8点检查我的日历并提醒我"
```

### 4.6 定时任务

```bash
# 查看定时任务
openclaw cron list

# 添加定时任务
openclaw cron add --schedule "0 8 * * *" --message "早上好，今天有什么安排？"

# 删除定时任务
openclaw cron remove <id>
```

### 4.7 记忆管理

```bash
# 搜索记忆
openclaw memory search "我的偏好"

# 查看所有记忆
openclaw memory list
```

---

## 五、技能（Skills）全览

### 5.1 内置技能（52个）

| 技能 | 功能 | 平台 |
|------|------|------|
| 🧩 coding-agent | 委托编码任务给 Claude Code/Codex | 全平台 |
| 🎮 discord | Discord 消息管理 | 全平台 |
| 📦 github | GitHub Issues/PR/CI 管理 | 全平台 |
| 📦 gh-issues | 获取 GitHub Issue 并分配修复 | 全平台 |
| 🎮 gog | Gmail/Calendar/Drive 管理 | 全平台 |
| 📧 himalaya | 邮件收发搜索 | 全平台 |
| 💎 obsidian | Obsidian 笔记管理 | 全平台 |
| 📝 notion | Notion 页面管理 | 全平台 |
| 📸 camsnap | RTSP/ONVIF 摄像头 | 全平台 |
| 🌐 browser | 浏览器控制 | 全平台 |
| 🎤 whisper | 语音转文字 | 全平台 |
| 🔐 1password | 密码管理 | macOS |
| 📝 apple-notes | Apple 笔记 | macOS |
| ⏰ apple-reminders | Apple 提醒 | macOS |
| 🧲 gifgrep | GIF 搜索 | 全平台 |
| 📄 nano-pdf | PDF 编辑 | 全平台 |
| 📰 blogwatcher | 博客/RSS 监控 | 全平台 |
| 🧊 mcp | MCP 服务器管理 | 全平台 |

### 5.2 社区技能（5400+）

通过 ClawHub 安装：

```bash
# 搜索社区技能
openclaw skills search "spotify"

# 安装
openclaw skills install spotify-skill

# 让 AI 自己写技能
# 在聊天中说："帮我写一个技能，检查天气预报并每天早上告诉我"
```

### 5.3 技能状态

| 状态 | 说明 |
|------|------|
| ✓ ready | 已就绪，可直接使用 |
| △ needs setup | 需要配置 API Key 或安装依赖 |
| ✗ error | 有错误，需要修复 |

---

## 六、实战场景

### 6.1 日常助手

```
你（Telegram）: "帮我看看明天有什么会议"
OpenClaw: → 调用 gog 技能 → 读取 Google Calendar → 回复会议列表

你: "帮我回复张三的邮件，说我同意方案A"
OpenClaw: → 调用 himalaya 技能 → 找到邮件 → 起草回复 → 发送

你: "提醒我下午3点给老板发周报"
OpenClaw: → 创建 cron 任务 → 下午3点主动提醒
```

### 6.2 开发助手

```
你（Discord）: "帮我修一下 src/auth.ts 的类型错误"
OpenClaw: → 调用 coding-agent → 启动 Claude Code → 修复 → 提交 PR

你: "查看 GitHub 上 my-project 的最新 CI 结果"
OpenClaw: → 调用 github 技能 → 获取 CI 状态 → 报告结果

你: "写一个脚本，每天凌晨2点备份数据库"
OpenClaw: → 调用 coding-agent → 编写脚本 → 测试 → 部署 cron
```

### 6.3 信息助手

```
你: "帮我追踪这个博客的更新"
OpenClaw: → 调用 blogwatcher 技能 → 订阅 RSS → 有更新时通知

你: "帮我查一下明天北京的天气"
OpenClaw: → 浏览器控制 → 搜索天气 → 回复

你: "把这个网页的内容总结一下"
OpenClaw: → 浏览器控制 → 抓取内容 → AI 总结 → 回复
```

### 6.4 生活助手

```
你: "帮我订一张明天去上海的机票"
OpenClaw: → 浏览器控制 → 打开携程 → 搜索航班 → 确认后下单

你: "帮我取消 Netflix 订阅"
OpenClaw: → 浏览器控制 → 登录 Netflix → 取消订阅

你: "控制空气净化器，把空气质量调到优"
OpenClaw: → 调用智能家居技能 → 控制 Winix 净化器
```

---

## 七、配置详解

### 7.1 配置文件位置

```
~/.openclaw/
├── config.yaml          # 主配置
├── memory/              # 持久记忆
├── skills/              # 技能配置
├── channels/            # 渠道配置
└── state/               # 运行状态
```

### 7.2 核心 API Key 配置

```yaml
# config.yaml
models:
  default: claude-sonnet-4-20250514
  providers:
    anthropic:
      api_key: sk-ant-xxx
    openai:
      api_key: sk-xxx
```

### 7.3 渠道配置

```yaml
channels:
  telegram:
    bot_token: "123456:ABC-xxx"
    allowed_users: ["your_telegram_id"]
  discord:
    bot_token: "xxx"
    allowed_guilds: ["your_server_id"]
  whatsapp:
    # WhatsApp Web 扫码登录，无需配置
```

### 7.4 安全配置

```yaml
security:
  exec_policy: "ask"    # ask / allow / deny
  sandbox: true          # 在沙箱中执行命令
  allowed_paths:
    - /home/user/projects
    - /home/user/documents
```

---

## 八、MCP 集成

OpenClaw 原生支持 MCP（Model Context Protocol），可以接入任何 MCP 服务器：

```bash
# 查看 MCP 配置
openclaw mcp list

# 添加 MCP 服务器
openclaw mcp add --name blender --command "blender-mcp"

# 与 opencode 的关系
# OpenClaw 可以通过 MCP 控制 opencode！
openclaw mcp add --name opencode --command "opencode"
```

---

## 九、常见问题

| 问题 | 解决 |
|------|------|
| Gateway 启动失败 | `openclaw doctor` 检查 |
| Telegram 收不到消息 | `openclaw channels login --channel telegram --verbose` |
| 技能无法使用 | `openclaw skills list` 检查状态，needs setup 的需要配置 |
| AI 模型调用失败 | 检查 API Key 是否正确、额度是否用完 |
| 内存占用高 | 减少 cron 任务、限制技能数量 |
| 4核8G 够用吗 | Gateway 够用，本地模型不够（建议用 API） |

---

## 十、与 opencode 的区别

| 维度 | opencode | OpenClaw |
|------|----------|----------|
| 定位 | CLI 编程助手 | 个人 AI 助手 |
| 交互方式 | 终端 | 聊天软件（Telegram 等） |
| 核心能力 | 读写代码、执行命令 | 邮件/日历/代码/浏览器/智能家居 |
| 运行模式 | 交互式会话 | 24/7 后台运行 |
| 记忆 | 仅当前会话 | 持久记忆，跨会话 |
| 技能 | skill | 5400+ 社区技能 |
| MCP | 被控制方 | 既能控制也能被控制 |

**可以一起用**：OpenClaw 通过 MCP 控制 opencode，在 Telegram 里说"帮我改一下 auth 模块"→ OpenClaw → opencode → 代码修改完成。

---

## 十一、快速上手步骤

```bash
# 1. 安装
curl -fsSL https://openclaw.ai/install.sh | bash

# 2. 配置 AI 模型（需要 API Key）
openclaw configure

# 3. 启动 Gateway
openclaw gateway &

# 4. 连接聊天渠道
openclaw channels login --channel telegram

# 5. 在 Telegram 里和你的 AI 助手聊天
# 发送: "你好，帮我看看今天的日程"

# 6. 添加技能
openclaw skills install github

# 7. 设置定时任务
openclaw cron add --schedule "0 8 * * *" --message "早上好，今天有什么安排？"
```

---

## 十二、参考链接

| 资源 | 链接 |
|------|------|
| 官网 | https://openclaw.ai |
| GitHub | https://github.com/openclaw/openclaw |
| 文档 | https://docs.openclaw.ai |
| ClawHub（技能市场） | https://clawhub.com |
| Discord 社区 | https://discord.gg/openclaw |
| 本机版本 | OpenClaw 2026.5.7 |
| 本机安装路径 | ~/.nvm/versions/node/v25.9.0/bin/openclaw |