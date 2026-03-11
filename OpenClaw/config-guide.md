# OpenClaw 配置系统完全指南

> **注意**：本文档已脱敏处理，所有敏感信息（模型名称、文件路径、账号标识等）均使用示例值。
> 
> **版本**：基于 OpenClaw 2026.3.8
> 
> **参考来源**：
> - 官方文档：https://docs.openclaw.ai
> - CLI 帮助：`openclaw --help`, `openclaw config --help`
> - 源码分析：实际配置文件字段验证

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

主配置文件位于 `~/.claw.json`（也写作 `~/.openclaw/openclaw.json`）。

### 1.2 顶层结构 (15个字段)

```json
{
  "meta": { },           // 元信息
  "auth": { },           // 认证配置
  "models": { },         // 模型配置
  "agents": { },         // Agent 列表
  "bindings": [ ],       // 消息路由
  "channels": { },       // 通道配置
  "gateway": { },        // 网关配置
  "session": { },        // 会话配置
  "plugins": { },        // 插件配置
  "tools": { },         // 工具配置
  "commands": { },       // 命令配置
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
- **Workspace** - 文件存储目录
- **AgentDir/State 目录** - 认证配置、模型注册表
- **Session 存储** - 聊天历史

**示例目录结构**：
```
~/.claw/
├── agents/
│   ├── main/
│   │   ├── agent/           # state 目录
│   │   │   └── auth-profiles.json
│   │   └── sessions/        # 会话历史 (*.jsonl)
│   └── pulse/
│       ├── agent/
│       └── sessions/
├── workspace/               # main agent 的工作目录
├── workspace-pulse/         # pulse agent 的工作目录
└── openclaw.json           # 主配置文件
```

### 2.2 Channel（通道）

**定义**：消息输入/输出的渠道类型。

**支持的通道**：
| 通道 | 说明 |
|------|------|
| `telegram` | Telegram Bot |
| `whatsapp` | WhatsApp Business |
| `discord` | Discord Bot |
| `signal` | Signal |
| `feishu` | 飞书 |
| `web` | Web Chat |

### 2.3 Session（会话）

**定义**：一次对话的上下文。

**Session Key 格式**：
```
agent:<agentId>:<channel>:<accountId>:<peerKind>:<peerId>
```

**示例**：
- `agent:main:telegram:nora:user:123456789` - 私聊
- `agent:pulse:telegram:default:group:-987654321` - 群组

### 2.4 Binding（绑定/路由）

**定义**：消息路由规则，决定由哪个 Agent 处理消息。

**匹配优先级**：从上到下匹配，第一个匹配的 binding 生效。

---

## 3. 字段详解

### 3.1 agents（Agent 配置）

**位置**：`agents`

**说明**：定义系统中所有的 Agent 实例。

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "minimax/MiniMax-M2.1"
      },
      "models": {
        "minimax/MiniMax-M2.1": {
          "alias": "Minimax"
        }
      },
      "workspace": "/path/to/workspace",
      "compaction": {
        "mode": "safeguard"
      },
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      }
    },
    "list": [
      {
        "id": "main",
        "name": "main"
      },
      {
        "id": "pulse",
        "name": "pulse",
        "workspace": "/path/to/workspace-pulse",
        "agentDir": "/path/to/agents/pulse/agent"
      }
    ]
  }
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `defaults.model.primary` | string | 默认主模型 | `minimax/MiniMax-M2.1` |
| `defaults.models.<modelId>.alias` | string | 模型别名 | `"Minimax"` |
| `defaults.workspace` | string | 默认工作目录 | `/home/user/workspace` |
| `defaults.compaction.mode` | string | 内存压缩模式 | `safeguard`, `eager` |
| `defaults.maxConcurrent` | number | 最大并发会话数 | `4` |
| `defaults.subagents.maxConcurrent` | number | 子 Agent 最大并发 | `8` |
| `list[].id` | string | **必填** Agent 唯一标识 | `"main"`, `"pulse"` |
| `list[].name` | string | 显示名称 | `"pulse"` |
| `list[].workspace` | string | 独立工作目录 | `"/path/to/workspace"` |
| `list[].agentDir` | string | 独立状态目录 | `"/path/to/agent/state"` |

**Feature 映射**：
- `compaction.mode`: 控制内存压缩策略，影响长期运行的会话内存使用
- `maxConcurrent`: 限制同时处理的会话数，防止资源耗尽
- `subagents.maxConcurrent`: 限制子 Agent 并发数

---

### 3.2 bindings（消息路由）

**位置**：`bindings`

**说明**：定义消息如何路由到不同的 Agent。

```json
{
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "telegram",
        "accountId": "default",
        "peer": {
          "kind": "group",
          "id": "-1234567890"
        }
      }
    }
  ]
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `agentId` | string | **必填** 目标 Agent ID | `"main"`, `"pulse"` |
| `match.channel` | string | 通道类型 | `"telegram"`, `"whatsapp"`, `"discord"` |
| `match.accountId` | string | 账号 ID | `"default"`, `"nora"` |
| `match.peer.kind` | string | 对端类型 | `"user"`（私聊）, `"group"`（群组）, `"channel"`（频道） |
| `match.peer.id` | string | 对端 ID | `"-1234567890"`（群组 ID） |

**Feature 映射**：
- 消息路由：接收消息 → 解析 channel/account/peer → 匹配 binding → 路由到对应 Agent
- 支持一个消息同时匹配多个 binding 时按顺序优先处理

---

### 3.3 channels（通道配置）

**位置**：`channels.<channelName>`

**说明**：配置每个通道的行为和政策。

#### 3.3.1 Telegram 配置示例

```json
{
  "channels": {
    "telegram": {
      "markdown": {
        "tables": "code"
      },
      "enabled": true,
      "commands": {
        "native": true,
        "nativeSkills": true
      },
      "configWrites": true,
      "dmPolicy": "pairing",
      "replyToMode": "all",
      "groups": {
        "*": {
          "requireMention": false
        }
      },
      "allowFrom": ["*"],
      "groupPolicy": "open",
      "chunkMode": "newline",
      "streaming": "partial",
      "blockStreaming": false,
      "actions": {
        "reactions": true,
        "sendMessage": true,
        "poll": true,
        "deleteMessage": true,
        "sticker": true
      },
      "reactionNotifications": "own",
      "reactionLevel": "extensive",
      "accounts": {
        "default": {
          "botToken": "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz",
          "dmPolicy": "pairing",
          "groupPolicy": "open",
          "streaming": "partial"
        }
      }
    }
  }
}
```

#### 3.3.2 通道级字段说明

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `enabled` | boolean | 是否启用该通道 | `true` |
| `dmPolicy` | string | DM 策略 | `pairing`, `allowlist`, `blocklist`, `all` |
| `groupPolicy` | string | 群组策略 | `open`, `allowlist`, `blocklist` |
| `streaming` | string | 流式输出模式 | `full`, `partial`, `none` |
| `blockStreaming` | boolean | 是否阻止流式输出 | `false` |
| `replyToMode` | string | 回复模式 | `all`, `thread`, `channel` |
| `chunkMode` | string | 分块模式 | `newline`, `sentence`, `word` |
| `requireMention` | boolean | 群组中是否需要 @ | `false` |
| `allowFrom` | array | 允许的消息来源 | `["*"]`, `["user1", "user2"]` |
| `markdown.tables` | string | Markdown 表格格式 | `code`, `unicode` |
| `commands.native` | boolean | 启用原生命令 | `true` |
| `commands.nativeSkills` | boolean | 启用 Skill 命令 | `true` |
| `configWrites` | boolean | 允许配置写入 | `true` |
| `actions.reactions` | boolean | 启用表情反应 | `true` |
| `actions.sendMessage` | boolean | 启用发送消息 | `true` |
| `actions.poll` | boolean | 启用投票 | `true` |
| `actions.deleteMessage` | boolean | 启用删除消息 | `true` |
| `actions.sticker` | boolean | 启用贴纸 | `true` |
| `reactionNotifications` | string | 反应通知范围 | `own`, `all`, `none` |
| `reactionLevel` | string | 反应级别 | `extensive`, `minimal` |

#### 3.3.3 账号级配置 (accounts.<accountId>)

| 字段 | 类型 | 说明 |
|------|------|------|
| `botToken` | string | Telegram/Discord Bot Token |
| `phoneNumber` | string | WhatsApp 电话号码 |
| `dmPolicy` | string | 该账号的 DM 策略（覆盖全局） |
| `groupPolicy` | string | 该账号的群组策略（覆盖全局） |
| `streaming` | string | 该账号的流式输出模式（覆盖全局） |

**Feature 映射**：
- `dmPolicy` / `groupPolicy`: 控制谁能与 Agent 对话
- `streaming`: 控制响应如何发送（实时 vs 完整后发送）
- `actions`: 控制 Agent 可以在通道中执行哪些操作
- `reactionLevel`: 控制反应的使用频率

---

### 3.4 gateway（网关配置）

**位置**：`gateway`

**说明**：配置 Gateway 的网络和行为。

```json
{
  "gateway": {
    "port": 8080,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "testtest1",
      "password": "",
      "rateLimit": {
        "exemptLoopback": false
      }
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    }
  }
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `port` | number | 监听端口 | `8080`, `18789` |
| `mode` | string | 运行模式 | `local`, `service` |
| `bind` | string | 绑定地址 | `loopback`, `0.0.0.0` |
| `auth.mode` | string | 认证模式 | `token`, `password` |
| `auth.token` | string | 认证 Token | - |
| `auth.rateLimit.exemptLoopback` | boolean | 跳过本地速率限制 | `false` |
| `tailscale.mode` | string | Tailscale 模式 | `off`, `login`, `serve` |
| `tailscale.resetOnExit` | boolean | 退出时重置 | `false` |

**Feature 映射**：
- `port` / `bind`: 网络监听配置
- `auth`: 访问控制
- `tailscale`: 远程访问配置

---

### 3.5 session（会话配置）

**位置**：`session`

**说明**：配置会话行为。

```json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `dmScope` | string | DM 会话作用域 | `per-channel-peer`, `per-account-peer`, `global` |

**Feature 映射**：
- `dmScope`: 控制私聊会话是否按通道/账号隔离

---

### 3.6 models（模型配置）

**位置**：`models`

**说明**：配置模型提供商和模型列表。

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "minimax": {
        "baseUrl": "https://api.minimax.io/anthropic",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "MiniMax-M2.1",
            "name": "MiniMax M2.1",
            "reasoning": false,
            "input": ["text"],
            "cost": {
              "input": 15,
              "output": 60,
              "cacheRead": 2,
              "cacheWrite": 10
            },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `mode` | string | 模式：`merge` 或 `replace` |
| `providers.<name>.baseUrl` | string | API 基础 URL |
| `providers.<name>.api` | string | API 类型 |
| `providers.<name>.models[].id` | string | 模型 ID |
| `providers.<name>.models[].name` | string | 模型显示名称 |
| `providers.<name>.models[].reasoning` | boolean | 是否支持推理 |
| `providers.<name>.models[].input` | array | 输入类型 |
| `providers.<name>.models[].cost.*` | number | 成本（每百万 token） |
| `providers.<name>.models[].contextWindow` | number | 上下文窗口大小 |
| `providers.<name>.models[].maxTokens` | number | 最大输出 tokens |

**Feature 映射**：
- 模型选择：Agent 根据配置选择使用哪个模型
- 成本计算：用于追踪使用成本

---

### 3.7 plugins（插件配置）

**位置**：`plugins`

**说明**：配置插件系统。

```json
{
  "plugins": {
    "allow": ["whatsapp", "feishu", "telegram"],
    "entries": {
      "feishu": {
        "enabled": true
      },
      "whatsapp": {
        "enabled": true
      }
    },
    "installs": {
      "feishu": {
        "source": "npm",
        "spec": "@m1heng-clawd/feishu",
        "version": "0.1.16"
      }
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `allow` | array | 允许的插件列表 |
| `entries.<name>.enabled` | boolean | 是否启用 |
| `installs.<name>.source` | string | 安装来源：`npm` |
| `installs.<name>.spec` | string | 包规范 |
| `installs.<name>.version` | string | 版本号 |

**Feature 映射**：
- 插件扩展：系统功能通过插件扩展
- 动态加载：插件可以在运行时加载

---

### 3.8 tools（工具配置）

**位置**：`tools`

**说明**：配置工具系统。

```json
{
  "tools": {
    "profile": "coding"
  }
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `profile` | string | 工具配置集 | `coding`, `default` |

**Feature 映射**：
- 控制哪些工具可用

---

### 3.9 commands（命令配置）

**位置**：`commands`

**说明**：配置命令系统。

```json
{
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  }
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `native` | string | 原生命令 | `auto`, `on`, `off` |
| `nativeSkills` | string | Skill 命令 | `auto`, `on`, `off` |
| `restart` | boolean | 允许重启 | `true` |
| `ownerDisplay` | string | 所有者显示 | `raw`, `name` |

---

### 3.10 messages（消息配置）

**位置**：`messages`

**说明**：配置消息处理。

```json
{
  "messages": {
    "ackReactionScope": "group-mentions"
  }
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `ackReactionScope` | string | 确认反应范围 | `group-mentions`, `all`, `none` |

---

### 3.11 browser（浏览器配置）

**位置**：`browser`

**说明**：配置浏览器工具。

```json
{
  "browser": {
    "attachOnly": true
  }
}
```

| 字段 | 类型 | 说明 | 示例值 |
|------|------|------|--------|
| `attachOnly` | boolean | 仅附加模式 | `true` |

---

### 3.12 web（Web 配置）

**位置**：`web`

**说明**：配置 Web 功能。

```json
{
  "web": {
    "enabled": true
  }
}
```

---

### 3.13 wizard（向导配置）

**位置**：`wizard`

**说明**：配置安装向导。

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
      "models": {
        "minimax/MiniMax-M2.1": {
          "alias": "Minimax"
        }
      },
      "workspace": "/home/user/workspace",
      "compaction": {
        "mode": "safeguard"
      },
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      }
    },
    "list": [
      {
        "id": "main",
        "name": "main"
      },
      {
        "id": "pulse",
        "name": "pulse",
        "workspace": "/home/user/workspace-pulse"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "telegram",
        "accountId": "default"
      }
    },
    {
      "agentId": "pulse",
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
      "actions": {
        "reactions": true,
        "sendMessage": true
      },
      "accounts": {
        "default": {
          "botToken": "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz"
        }
      }
    }
  },
  "gateway": {
    "port": 8080,
    "mode": "local",
    "bind": "loopback"
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "models": {
    "mode": "merge",
    "providers": {
      "minimax": {
        "baseUrl": "https://api.minimax.io/anthropic",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "MiniMax-M2.1",
            "name": "MiniMax M2.1",
            "cost": { "input": 15, "output": 60 },
            "contextWindow": 200000
          }
        ]
      }
    }
  }
}
```

---

## 5. 运行原理

### 5.1 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Gateway (网关)                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Bindings (路由规则)                    │   │
│  │   message → channel → account → peer → agent       │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────┬──────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐       ┌─────────┐       ┌─────────┐
   │ Agent α │       │ Agent β │       │ Agent γ │
   │ (main)  │       │ (pulse) │       │         │
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
存储在 ~/.claw/agents/<agentId>/sessions/
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
| Cron 任务超时 | sessionTarget 设置为不存在的 agent | 确认为已定义的 agent（如 `main`） |
| Channel 无法接收消息 | 账号配置错误 | 检查 botToken 等 |
| 模型调用失败 | API Key 配置错误 | 检查 models.providers |

### 6.2 调试命令

```bash
# 查看当前版本
openclaw --version

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

# 验证配置
openclaw config validate
```

---

## 附录：A. 字段索引表

| 顶层字段 | 说明 | 章节 |
|----------|------|------|
| `meta` | 元信息 | - |
| `auth` | 认证配置 | - |
| `models` | 模型配置 | 3.6 |
| `agents` | Agent 列表 | 3.1 |
| `bindings` | 消息路由 | 3.2 |
| `channels` | 通道配置 | 3.3 |
| `gateway` | 网关配置 | 3.4 |
| `session` | 会话配置 | 3.5 |
| `plugins` | 插件配置 | 3.7 |
| `tools` | 工具配置 | 3.8 |
| `commands` | 命令配置 | 3.9 |
| `messages` | 消息配置 | 3.10 |
| `browser` | 浏览器配置 | 3.11 |
| `web` | Web 配置 | 3.12 |
| `wizard` | 向导配置 | 3.13 |

---

*文档版本: 2026-03-12*
*基于 OpenClaw 2026.3.8 官方文档和源码分析*
*参考来源：https://docs.openclaw.ai, CLI 帮助文档*
