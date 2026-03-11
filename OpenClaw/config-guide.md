# OpenClaw 配置系统完全指南

> **注意**：本文档已脱敏处理。
> 
> **版本**：基于 OpenClaw 2026.3.8
> 
> **参考来源**：
> - 官方文档：https://docs.openclaw.ai
> - CLI：`openclaw --help`, `openclaw config --help`
> - 实际配置验证

---

## 目录

1. [配置架构概述](#1-配置架构概述)
2. [核心概念详解](#2-核心概念详解)
3. [字段详解](#3-字段详解)
4. [配置示例](#4-配置示例)
5. [运行原理](#5-运行原理)
6. [故障排查](#6-故障排查)

---

## 1. 配置架构概述

### 1.1 顶层结构 (15个字段)

```json
{
  "meta": { },
  "auth": { },
  "models": { },
  "agents": { },
  "bindings": [ ],
  "channels": { },
  "gateway": { },
  "session": { },
  "plugins": { },
  "tools": { },
  "commands": { },
  "messages": { },
  "browser": { },
  "web": { },
  "wizard": { }
}
```

---

## 2. 核心概念详解

### 2.1 Agent vs Model vs Channel vs Session

这是四个不同的概念，容易混淆：

| 概念 | 定义 | 类比 |
|------|------|------|
| **Agent** | AI 代理实例 | 一个人脑 |
| **Model** | 底层 AI 模型 | 人的知识/能力 |
| **Channel** | 消息渠道 | 联系方式（电话/微信/Telegram） |
| **Session** | 对话上下文 | 一次聊天记录 |

**关系**：
```
User → Channel → Session → Agent → Model
  ↑                              │
  └────── 返回响应 ──────────────┘
```

### 2.2 Binding 路由规则详解

**Binding 的核心作用**：决定消息由哪个 Agent 处理。

#### 2.2.1 Binding 匹配流程

```
收到消息
    │
    ├── channel: "telegram"
    ├── accountId: "default"  
    ├── peer.kind: "group" 或 "user"
    └── peer.id: 群组ID 或 用户ID
    │
    ▼
遍历 bindings（按顺序）
    │
    ├── 检查第1个 binding 的 match 条件
    │   ├── 如果全部匹配 → 使用这个 agentId，停止匹配
    │   └── 如果不匹配 → 继续检查下一个
    │
    ├── 检查第2个 binding ...
    │
    └── 如果都不匹配 → 消息被丢弃
```

#### 2.2.2 Binding 配置字段

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
          "id": "-5003878822"
        }
      }
    }
  ]
}
```

| 字段 | 是否必填 | 说明 |
|------|----------|------|
| `agentId` | **必填** | 处理消息的 Agent ID |
| `match.channel` | 可选 | 通道类型（不填=所有通道） |
| `match.accountId` | 可选 | 账号 ID（不填=所有账号） |
| `match.peer.kind` | 可选 | `user`(私聊) / `group`(群组) / `channel`(频道) |
| `match.peer.id` | 可选 | 具体 ID（不填=所有该类型） |

#### 2.2.3 匹配优先级示例

```json
// 优先级1：最具体 - 指定群组的指定账号
{ "agentId": "pulse", "match": { "channel": "telegram", "accountId": "default", "peer": { "kind": "group", "id": "-5003878822" } } }

// 优先级2：较具体 - 任意私聊的默认账号
{ "agentId": "main", "match": { "channel": "telegram", "accountId": "default", "peer": { "kind": "user" } } }

// 优先级3：最宽泛 - 任意账号的任意消息
{ "agentId": "main", "match": { "channel": "telegram" } }
```

#### 2.2.4 常见错误：群组多 Agent 配置

**错误配置**（当前问题）：
```json
// 问题：两个 binding 指向同一个群组，但 main 在前面，会抢走所有消息
{ "agentId": "main", "match": { "peer": { "kind": "group", "id": "-5003878822" } } }
{ "agentId": "pulse", "match": { "peer": { "kind": "group", "id": "-5003878822" } } }
```

**正确方案**：

**方案 A：通过 @mention 选择**
```json
// 在 channel 配置中启用 requireMention
{ "channels": { "telegram": { "groups": { "*": { "requireMention": true } } } } }

// Binding 不需要指定具体群组
{ "agentId": "main", "match": { "channel": "telegram", "accountId": "default" } }
{ "agentId": "pulse", "match": { "channel": "telegram", "accountId": "default" } }

// 在群里 @main → main 回复
// 在群里 @pulse → pulse 回复
```

**方案 B：不同账号绑定不同 Agent**
```json
// 账号 nora 绑定 main
{ "agentId": "main", "match": { "channel": "telegram", "accountId": "nora", "peer": { "kind": "group", "id": "-5003878822" } } }

// 账号 default 绑定 pulse
{ "agentId": "pulse", "match": { "channel": "telegram", "accountId": "default", "peer": { "kind": "group", "id": "-5003878822" } } }

// 需要两个不同的 Telegram Bot 账号
```

---

## 3. 字段详解

### 3.1 agents（Agent 配置）

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "provider/model-id" },
      "models": { "provider/model-id": { "alias": "简称" } },
      "workspace": "/path/to/workspace",
      "compaction": { "mode": "safeguard" },
      "maxConcurrent": 4,
      "subagents": { "maxConcurrent": 8 }
    },
    "list": [
      {
        "id": "agent-id",
        "name": "显示名称",
        "workspace": "/独立/工作目录",
        "agentDir": "/独立/状态目录",
        "model": { "primary": "override-model-id" }
      }
    ]
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `defaults.model.primary` | string | **默认模型 ID**，格式：`provider/model-id` |
| `defaults.models.<modelId>.alias` | string | 模型别名 |
| `defaults.workspace` | string | 默认工作目录 |
| `defaults.compaction.mode` | string | 内存压缩：`safeguard`(安全) / `eager`(激进) |
| `defaults.maxConcurrent` | number | Agent 最大并发会话数 |
| `list[].id` | string | **必填** Agent 唯一 ID |
| `list[].name` | string | 显示名称 |
| `list[].workspace` | string | 独立工作目录（覆盖默认） |
| `list[].model.primary` | string | 该 Agent 专用模型（覆盖默认） |

### 3.2 models（模型配置）

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
            "name": "显示名称",
            "reasoning": false,
            "cost": { "input": 15, "output": 60 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

| 字段 | 说明 |
|------|------|
| `mode` | `merge`(合并) / `replace`(替换) 官方预设 |
| `providers.<name>.baseUrl` | API 端点 |
| `providers.<name>.api` | API 类型 |
| `providers.<name>.models[].id` | 模型 ID（供 agents 引用） |
| `providers.<name>.models[].cost.*` | 每百万 token 成本 |

**Agent 与 Model 的关系**：
```
agents.defaults.model.primary = "minimax/MiniMax-M2.1"
                          ↓
   在 models.providers.minimax.models 中查找 id = "MiniMax-M2.1"
```

### 3.3 channels（通道配置）

#### 3.3.1 Telegram 配置详解

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "groupPolicy": "open",
      "replyToMode": "all",
      "chunkMode": "newline",
      "streaming": "partial",
      "blockStreaming": false,
      "requireMention": false,
      "allowFrom": ["*"],
      "markdown": { "tables": "code" },
      "commands": { "native": true, "nativeSkills": true },
      "configWrites": true,
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
        "account-id": {
          "botToken": "xxx",
          "dmPolicy": "覆盖",
          "groupPolicy": "覆盖",
          "streaming": "覆盖"
        }
      }
    }
  }
}
```

#### 3.3.2 枚举值详解

| 字段 | 枚举值 | 含义 | 依赖/影响 |
|------|--------|------|-----------|
| **dmPolicy** | `pairing` | 首次对话需确认 | 常用于个人号 |
| | `allowlist` | 仅白名单用户 | 需要配合 `allowFrom` |
| | `blocklist` | 黑名单用户除外 | |
| | `all` | 所有人 | 需谨慎 |
| **groupPolicy** | `open` | 任意群都能用 | 最宽松 |
| | `allowlist` | 仅白名单群 | |
| | `blocklist` | 黑名单群除外 | |
| | `pairing` | 需确认 | |
| **streaming** | `full` | 实时流式输出 | 响应即时 |
| | `partial` | 分段输出 | 默认，推荐 |
| | `none` | 非流式 | 等待完整响应 |
| **chunkMode** | `newline` | 按换行分块 | 默认 |
| | `sentence` | 按句子分块 | |
| | `word` | 按单词分块（慎用） | |
| **replyToMode** | `all` | 回复所有人 | |
| | `thread` | 仅回复线程 | |
| | `channel` | 回复频道 | |
| **reactionNotifications** | `own` | 仅自己的反应 | 控制通知频率 |
| | `all` | 所有反应 | |
| | `none` | 关闭 | |
| **reactionLevel** | `extensive` | 大量使用反应 | 活跃但可能烦 |
| | `minimal` | 最小化使用 | 安静 |
| **markdown.tables** | `code` | 代码块格式 | 兼容性最好 |
| | `unicode` | Unicode 表格 | |

#### 3.3.3 字段依赖关系

```
基础开关（必须先开启）:
├── enabled: true → 才能使用其他功能
├── streaming: "partial" → chunkMode 才有意义
└── groupPolicy: "open" → allowFrom 才能生效

功能开关（依赖基础开关）:
├── actions.reactions: true → reactionNotifications, reactionLevel 才有意义
├── commands.native: true → nativeSkills 才有意义
└── allowFrom: ["*"] → 配合 dmPolicy/groupPolicy 使用
```

#### 3.3.4 场景示例

| 场景 | 配置 |
|------|------|
| **个人 Bot**（严格） | `dmPolicy: "pairing"`, `groupPolicy: "blocklist"` |
| **群组 Bot**（开放） | `dmPolicy: "all"`, `groupPolicy: "open"`, `requireMention: true` |
| **安静模式** | `streaming: "none"`, `reactionLevel: "minimal"`, `reactionNotifications: "none"` |
| **实时响应** | `streaming: "full"`, `chunkMode: "sentence"` |

---

### 3.4 session（会话配置）

```json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

| 枚举值 | 含义 |
|--------|------|
| `per-channel-peer` | 每个通道+用户=独立会话 |
| `per-account-peer` | 每个账号+用户=独立会话 |
| `global` | 全局唯一会话 |

**示例**：
```
用户 A 在 Telegram 和 WhatsApp 各发一条消息：
- per-channel-peer: 两个独立会话
- global: 同一个会话（上下文共享）
```

---

## 4. 配置示例

### 4.1 正确配置：群组多 Agent

```json
{
  "agents": {
    "list": [
      { "id": "main" },
      { "id": "pulse" }
    ]
  },
  "bindings": [
    // 私聊统一用 main
    { "agentId": "main", "match": { "channel": "telegram", "accountId": "default", "peer": { "kind": "user" } } },
    // 群里 @main → main 回复
    { "agentId": "main", "match": { "channel": "telegram", "accountId": "default", "peer": { "kind": "group", "id": "-5003878822" } } },
    // 群里 @pulse → pulse 回复
    { "agentId": "pulse", "match": { "channel": "telegram", "accountId": "default", "peer": { "kind": "group", "id": "-5003878822" } } }
  ],
  "channels": {
    "telegram": {
      "enabled": true,
      "groupPolicy": "open",
      "groups": {
        "*": { "requireMention": true }
      }
    }
  }
}
```

---

## 5. 运行原理

### 5.1 消息流转

```
用户发送消息
    │
    ▼
Channel 接收（Telegram/WhatsApp/...）
    │
    ▼
Gateway 解析
    ├── channel: "telegram"
    ├── accountId: "default"
    ├── peer.kind: "group"
    └── peer.id: "-5003878822"
    │
    ▼
Bindings 匹配（按顺序）
    │
    ▼
确定 Agent
    │
    ▼
查找/创建 Session
    │
    ▼
Agent 处理 → Model 生成 → 工具执行
    │
    ▼
返回响应
```

### 5.2 Session 隔离

| dmScope | Session Key 示例 | 说明 |
|---------|------------------|------|
| `per-channel-peer` | `agent:main:telegram:nora:user:123` | 通道+用户隔离 |
| `per-account-peer` | `agent:main::user:123` | 账号+用户隔离 |
| `global` | `agent:main::user:123` | 全局唯一 |

---

## 6. 故障排查

| 问题 | 原因 | 解决 |
|------|------|------|
| 群组里两个 Agent 抢话 | Binding 重复/顺序错误 | 确认 requireMention 或分离账号 |
| 消息未被任何 Agent 处理 | 没有匹配的 binding | 检查 channel/account/peer 配置 |
| 模型调用失败 | model ID 不在 models.providers 中 | 确认模型已定义 |
| 跨会话消息混乱 | dmScope 配置错误 | 检查 session.dmScope |

---

*文档版本: 2026-03-12 v3*
*基于 OpenClaw 2026.3.8*
