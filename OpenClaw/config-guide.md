# OpenClaw 配置系统完全指南

> **注意**：本文档已脱敏处理，所有敏感信息（模型名称、文件路径、账号标识等）均使用示例值。

---

## 目录

1. [配置架构概述](#1-配置架构概述)
2. [核心概念](#2-核心概念)
3. [字段详解](#3-字段详解)
4. [配置示例](#4-配置示例)
5. [运行原理](#5-运行原理)
6. [故障排查](#6-故障排查)

---

## 1. 配置架构概述

### 1.1 配置文件位置

主配置文件位于 `~/.claw.json`，采用 JSON 格式。

openclaw/open### 1.2 顶层结构

```json
{
  "meta": { },           // 元信息
  "auth": { },           // 认证配置
  "models": { },          // 模型配置
  "agents": { },          // Agent 列表
  "bindings": [ ],        // 消息路由
  "channels": { },        // 通道配置
  "gateway": { },         // 网关配置
  "session": { },         // 会话配置
  "plugins": { },         // 插件配置
  "tools": { },          // 工具配置
  "commands": { },        // 命令配置
  "messages": { },       // 消息配置
  "browser": { },        // 浏览器配置
  "web": { },            // Web 配置
  "wizard": { }          // 向导配置
}
```

---

## 2. 核心概念

### 2.1 Agent（代理）

**定义**：一个完整的 AI 代理实例，拥有独立的：
- Workspace（工作目录）
- State 目录（认证、模型注册表）
- Session 存储（聊天历史）

**核心作用**：处理用户消息并生成响应。

### 2.2 Channel（通道）

**定义**：消息输入/输出的渠道类型。

**支持的通道**：
- Telegram
- WhatsApp
- Discord
- Signal
- Feishu（飞书）
- Web

每个 Channel 可配置多个 Account（账号）。

### 2.3 Session（会话）

**定义**：一次对话的上下文。

**Session Key 格式**：
```
agent:<agentId>:<channel>:<accountId>:<peerKind>:<peerId>
```

**示例**：
- `agent:alpha:telegram:account1:user:123456` - 私聊
- `agent:beta:telegram:account1:group:-987654321` - 群组

### 2.4 Binding（绑定/路由）

**定义**：消息路由规则，决定由哪个 Agent 处理消息。

---

## 3. 字段详解

### 3.1 agents（Agent 配置）

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "minimax/MiniMax-M2.1"  // 主模型
      },
      "workspace": "/path/to/workspace",    // 默认工作目录
      "compaction": {
        "mode": "safeguard"                 // 内存压缩模式
      },
      "maxConcurrent": 4                   // 最大并发数
    },
    "list": [
      {
        "id": "alpha",                      // Agent 唯一标识
        "name": "alpha",                     // 显示名称
        "workspace": "/path/to/workspace",  // 独立工作目录（可选）
        "agentDir": "/path/to/state"        // 独立状态目录（可选）
      }
    ]
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `defaults.model.primary` | string | 默认使用的模型 |
| `defaults.workspace` | string | 默认工作目录 |
| `defaults.compaction.mode` | string | 内存压缩模式：`safeguard` / `eager` |
| `defaults.maxConcurrent` | number | 最大并发会话数 |
| `list[].id` | string | **必填** Agent 唯一标识 |
| `list[].name` | string | 显示名称 |
| `list[].workspace` | string | 独立工作目录 |
| `list[].agentDir` | string | 独立状态目录 |

### 3.2 bindings（消息路由）

```json
{
  "bindings": [
    {
      "agentId": "alpha",
      "match": {
        "channel": "telegram",
        "accountId": "account1",
        "peer": {
          "kind": "group",
          "id": "-1234567890"
        }
      }
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `agentId` | string | 目标 Agent ID，必须在 `agents.list` 中定义 |
| `match.channel` | string | 通道类型：`telegram`, `whatsapp`, `discord` 等 |
| `match.accountId` | string | 账号 ID，对应 `channels.<channel>.accounts` |
| `match.peer.kind` | string | 对端类型：`user`（私聊）, `group`（群组）, `channel`（频道） |
| `match.peer.id` | string | 对端 ID（群组/频道为数字 ID） |

**路由优先级**：从上到下匹配，第一个匹配的 binding 生效。

### 3.3 channels（通道配置）

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",          // DM 策略
      "groupPolicy": "open",           // 群组策略
      "streaming": "partial",           // 流式输出
      "accounts": {
        "account1": {
          "botToken": "123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11",
          "dmPolicy": "pairing",
          "groupPolicy": "open"
        }
      }
    }
  }
}
```

#### 通道级配置

| 字段 | 类型 | 说明 |
|------|------|------|
| `enabled` | boolean | 是否启用该通道 |
| `dmPolicy` | string | DM 策略 |
| `groupPolicy` | string | 群组策略 |
| `streaming` | string | 流式输出模式 |

#### 策略说明

**dmPolicy / groupPolicy**：
- `all` - 允许所有人
- `allowlist` - 仅白名单
- `blocklist` - 黑名单
- `pairing` - 配对模式（首次对话需确认）
- `open` - 开放模式

**streaming**：
- `full` - 完整流式输出
- `partial` - 分段输出
- `none` - 非流式

#### 账号级配置 (accounts)

| 字段 | 类型 | 说明 |
|------|------|------|
| `botToken` | string | Bot Token（Telegram/Discord） |
| `phoneNumber` | string | 电话号码（WhatsApp） |
| `dmPolicy` | string | 该账号的 DM 策略 |
| `groupPolicy` | string | 该账号的群组策略 |

### 3.4 gateway（网关配置）

```json
{
  "gateway": {
    "http": {
      "host": "0.0.0.0",
      "port": 8080
    },
    "publicUrl": "https://example.com",
    "cors": {
      "enabled": true,
      "allowOrigins": ["*"]
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `http.host` | string | HTTP 监听地址 |
| `http.port` | number | HTTP 监听端口 |
| `publicUrl` | string | 公开访问 URL |
| `cors.enabled` | boolean | 是否启用 CORS |
| `cors.allowOrigins` | array | 允许的跨域来源 |

### 3.5 session（会话配置）

```json
{
  "session": {
    "dmScope": "per-channel-peer",
    "systemPrompt": "你是一个助手。",
    "maxMessages": 100
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `dmScope` | string | DM 会话作用域 |
| `systemPrompt` | string | 系统提示词 |
| `maxMessages` | number | 最大消息数 |

### 3.6 models（模型配置）

```json
{
  "models": {
    "defaults": {
      "provider": "minimax",
      "primary": "MiniMax-M2.1"
    },
    "providers": {
      "minimax": {
        "apiKey": "sk-xxx",
        "endpoint": "https://api.minimax.chat/v1"
      },
      "openai": {
        "apiKey": "sk-xxx"
      }
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `defaults.provider` | string | 默认模型提供商 |
| `defaults.primary` | string | 默认模型名称 |
| `providers.<name>.apiKey` | string | API Key |
| `providers.<name>.endpoint` | string | API 端点 |

### 3.7 plugins（插件配置）

```json
{
  "plugins": {
    "enabled": ["feishu", "browser"],
    "feishu": {
      "enabled": true,
      "appId": "cli_xxx",
      "appSecret": "xxx"
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `enabled` | array | 启用的插件列表 |
| `<pluginName>.enabled` | boolean | 是否启用 |
| `<pluginName>.*` | object | 插件特定配置 |

### 3.8 cron（定时任务）

```json
{
  "cron": [
    {
      "id": "uuid",
      "name": "任务名称",
      "enabled": true,
      "schedule": {
        "kind": "cron",
        "expr": "0 7 * * *",
        "tz": "Asia/Hong_Kong"
      },
      "sessionTarget": "alpha",
      "payload": {
        "kind": "systemEvent",
        "text": "任务消息内容"
      },
      "delivery": {
        "mode": "none"
      }
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `schedule.kind` | string | 调度类型：`cron` |
| `schedule.expr` | string | Cron 表达式 |
| `schedule.tz` | string | 时区 |
| `sessionTarget` | string | 目标 Session |
| `payload.kind` | string | 负载类型 |
| `delivery.mode` | string | 投递模式 |

---

## 4. 配置示例

### 4.1 完整 Mock 配置

```json
{
  "meta": {
    "version": "2026.3"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "minimax/MiniMax-M2.1"
      },
      "workspace": "/home/user/workspace",
      "compaction": {
        "mode": "safeguard"
      },
      "maxConcurrent": 4
    },
    "list": [
      {
        "id": "alpha",
        "name": "alpha"
      },
      {
        "id": "beta",
        "name": "beta",
        "workspace": "/home/user/workspace-beta",
        "agentDir": "/home/user/.openclaw/agents/beta/agent"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "alpha",
      "match": {
        "channel": "telegram",
        "accountId": "default"
      }
    },
    {
      "agentId": "beta",
      "match": {
        "channel": "telegram",
        "accountId": "default",
        "peer": {
          "kind": "group",
          "id": "-1234567890"
        }
      }
    }
  ],
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "groupPolicy": "open",
      "streaming": "partial",
      "accounts": {
        "default": {
          "botToken": "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz"
        }
      }
    },
    "whatsapp": {
      "enabled": false,
      "dmPolicy": "allowlist",
      "accounts": {
        "work": {
          "phoneNumber": "+1234567890"
        }
      }
    }
  },
  "gateway": {
    "http": {
      "host": "0.0.0.0",
      "port": 8080
    },
    "publicUrl": "https://myagent.example.com"
  },
  "session": {
    "dmScope": "per-channel-peer",
    "maxMessages": 100
  },
  "models": {
    "defaults": {
      "provider": "minimax",
      "primary": "MiniMax-M2.1"
    },
    "providers": {
      "minimax": {
        "apiKey": "sk-xxxxx"
      }
    }
  },
  "plugins": {
    "enabled": ["browser", "feishu"],
    "browser": {
      "enabled": true
    },
    "feishu": {
      "enabled": true,
      "appId": "cli_xxxxx",
      "appSecret": "xxxxx"
    }
  }
}
```

---

## 5. 运行原理

### 5.1 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Gateway (网关)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Bindings (路由规则)                    │   │
│  │   message → channel → account → peer → agent       │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────┬──────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
    ┌─────────┐       ┌─────────┐       ┌─────────┐
    │ Agent α │       │ Agent β │       │ Agent γ │
    │ (alpha) │       │ (beta)  │       │ (gamma) │
    └────┬────┘       └────┬────┘       └────┬────┘
         │                 │                 │
         ▼                 ▼                 ▼
    ┌─────────┐       ┌─────────┐       ┌─────────┐
    │ Session │       │ Session │       │ Session │
    │ 列表    │       │ 列表    │       │ 列表    │
    └─────────┘       └─────────┘       └─────────┘
```

### 5.2 消息流转流程

```
1. 用户发送消息
       │
       ▼
2. Channel 接收 (Telegram/WhatsApp/Discord)
       │
       ▼
3. Gateway 解析消息
   - channel: telegram
   - account: default
   - peer kind: user/group
   - peer id: 123456789
       │
       ▼
4. Bindings 匹配
   遍历 bindings 列表，找到第一个匹配的 binding
       │
       ▼
5. 确定 Agent
   - 从 matching binding 获取 agentId
   - 查找或创建对应的 Session
       │
       ▼
6. Agent 处理
   - 加载会话历史
   - 调用模型生成响应
   - 执行工具（如有需要）
       │
       ▼
7. 返回响应
   - 通过同一 Channel 发送
```

### 5.3 Session 生命周期

```
创建 Session
     │
     ▼
存储在 ~/.openclaw/agents/<agentId>/sessions/
     │
     ├── session-001.jsonl
     ├── session-002.jsonl
     └── ...
     │
     ▼
定期压缩 (compaction mode: safeguard/eager)
     │
     ▼
超过 maxMessages 时自动清理
```

### 5.4 Cron 任务执行

```
定时触发
     │
     ▼
检查 sessionTarget 是否有效
     │
     ├── 有效 → 创建/使用 session
     │            │
     │            ▼
     │         执行 payload
     │            │
     │            ▼
     │         返回 delivery 结果
     │
     └── 无效 → 任务超时/失败
```

---

## 6. 故障排查

### 6.1 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 消息未路由到正确 Agent | bindings 顺序或匹配条件错误 | 检查 bindings 配置 |
| Cron 任务超时 | sessionTarget 无效 | 确认为已定义的 agent |
| Channel 无法接收消息 | 账号配置错误 | 检查 botToken 等 |
| 模型调用失败 | API Key 配置错误 | 检查 models.providers |

### 6.2 调试命令

```bash
# 查看 bindings
openclaw bindings list

# 查看 agents
openclaw agents list

# 查看 sessions
openclaw sessions list

# 查看 cron 状态
openclaw cron list

# 查看 gateway 状态
openclaw gateway status
```

---

## 附录：A. 字段索引表

| 顶层字段 | 说明 |
|----------|------|
| `meta` | 元信息 |
| `auth` | 认证配置 |
| `models` | 模型配置 |
| `agents` | Agent 列表 |
| `bindings` | 消息路由 |
| `channels` | 通道配置 |
| `gateway` | 网关配置 |
| `session` | 会话配置 |
| `plugins` | 插件配置 |
| `tools` | 工具配置 |
| `commands` | 命令配置 |
| `messages` | 消息配置 |
| `browser` | 浏览器配置 |
| `web` | Web 配置 |
| `wizard` | 向导配置 |

---

*文档版本: 2026-03-12*
*基于 OpenClaw 官方文档*
