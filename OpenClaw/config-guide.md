# OpenClaw 配置系统详解

本文档详细解释 OpenClaw 的配置系统，帮助你理解如何配置 Agent、Channel、Session 以及它们之间的关系。

---

## 1. 核心概念

### 1.1 配置文件的结构

OpenClaw 的主配置文件位于 `~/.openclaw/openclaw.json`，包含以下主要部分：

```json
{
  "meta": { ... },           // 元信息
  "auth": { ... },           // 认证配置
  "models": { ... },         // 模型配置
  "agents": { ... },         // Agent 列表
  "bindings": [ ... ],       // 消息路由绑定
  "channels": { ... },        // 通道配置
  "gateway": { ... },         // 网关配置
  "plugins": { ... }          // 插件配置
}
```

---

## 2. Agent (代理)

### 2.1 什么是 Agent？

Agent 是一个完整的 AI 代理实例，拥有：
- **独立的 Workspace** - 文件存储目录
- **独立的 State 目录** - 认证配置、模型注册表
- **独立的 Session 存储** - 聊天历史

### 2.2 Agent 配置字段

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "minimax/MiniMax-M2.1" },
      "workspace": "/Users/linus/.openclaw/workspace",
      "compaction": { "mode": "safeguard" },
      "maxConcurrent": 4
    },
    "list": [
      {
        "id": "main",                    // Agent ID（必须唯一）
        "name": "main",                   // 显示名称（可选）
        "workspace": "/path/to/workspace", // 独立工作目录（可选）
        "agentDir": "/path/to/agent/state" // 独立状态目录（可选）
      },
      {
        "id": "pulse",
        "name": "pulse",
        "workspace": "/Users/linus/.openclaw/workspace-pulse",
        "agentDir": "/Users/linus/.openclaw/agents/pulse/agent"
      }
    ]
  }
}
```

### 2.3 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | Agent 的唯一标识符 |
| `name` | string | 否 | 显示名称，默认等于 `id` |
| `workspace` | string | 否 | 独立工作目录，默认 `~/.openclaw/workspace-<id>` |
| `agentDir` | string | 否 | 独立状态目录，默认 `~/.openclaw/agents/<id>/agent` |

### 2.4 Agent 的目录结构

```
~/.openclaw/
├── agents/
│   ├── main/
│   │   ├── agent/           # state 目录
│   │   │   └── auth-profiles.json
│   │   └── sessions/        # 会话历史
│   └── pulse/
│       ├── agent/
│       └── sessions/
├── workspace/               # main agent 的工作目录
├── workspace-pulse/         # pulse agent 的工作目录
└── openclaw.json           # 主配置文件
```

### 2.5 创建新的 Agent

```bash
# 使用向导创建
openclaw agents add coding
```

---

## 3. Channel (通道)

### 3.1 什么是 Channel？

Channel 是消息的输入/输出渠道，如 Telegram、WhatsApp、Discord 等。

### 3.2 Channel 配置字段

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "groupPolicy": "open",
      "streaming": "partial",
      "accounts": {
        "nora": { "botToken": "xxx" },
        "default": { "botToken": "yyy" }
      }
    }
  }
}
```

---

## 4. Bindings (绑定/路由)

### 4.1 什么是 Bindings？

Bindings 定义了消息如何路由到正确的 Agent。

### 4.2 match 字段说明

| 字段 | 说明 |
|------|------|
| `channel` | 通道类型 (telegram, whatsapp, discord) |
| `accountId` | 账号 ID |
| `peer.kind` | 对端类型 (user, group, channel) |
| `peer.id` | 对端 ID |

---

## 5. Cron 定时任务

### 关键字段

| 字段 | 说明 |
|------|------|
| `sessionTarget` | 会话目标：main / isolated |
| `agentId` | Agent ID |

---

*文档版本: 2026-03-12*
*基于 OpenClaw 2026.3.8*
