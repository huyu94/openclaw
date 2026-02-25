# 飞书频道插件详解 (channel.ts)

## 概述

`channel.ts` 是飞书频道插件的核心配置文件,它定义了 OpenClaw 系统如何与飞书平台进行消息通信。这个文件通过实现 `ChannelPlugin` 接口,为系统提供了完整的飞书集成能力。

## 文件结构

```
extensions/feishu/src/channel.ts
├── 元数据定义 (meta)
├── 频道插件配置 (feishuPlugin)
│   ├── 基础信息 (meta, id)
│   ├── 配对功能 (pairing)
│   ├── 能力声明 (capabilities)
│   ├── Agent 提示 (agentPrompt)
│   ├── 群组配置 (groups)
│   ├── 配置架构 (configSchema)
│   ├── 配置管理 (config)
│   ├── 安全策略 (security)
│   ├── 设置流程 (setup)
│   ├── 引导流程 (onboarding)
│   ├── 消息功能 (messaging)
│   ├── 通讯录 (directory)
│   ├── 出站消息 (outbound)
│   ├── 状态监控 (status)
│   └── 网关管理 (gateway)
```

---

## 1. 元数据定义 (Meta)

**位置**: 第 30-39 行

```typescript
const meta: ChannelMeta = {
  id: "feishu",
  label: "Feishu",
  selectionLabel: "Feishu/Lark (飞书)",
  docsPath: "/channels/feishu",
  docsLabel: "feishu",
  blurb: "飞书/Lark enterprise messaging.",
  aliases: ["lark"],
  order: 70,
};
```

**功能**: 定义频道的基本信息
- `id`: 唯一标识符
- `label`: 显示名称
- `selectionLabel`: 选择界面显示的名称
- `docsPath`: 文档路径
- `aliases`: 别名支持(支持 "lark" 作为别名)
- `order`: UI 中的显示顺序

---

## 2. 配对功能 (Pairing)

**位置**: 第 46-56 行

```typescript
pairing: {
  idLabel: "feishuUserId",
  normalizeAllowEntry: (entry) => entry.replace(/^(feishu|user|open_id):/i, ""),
  notifyApproval: async ({ cfg, id }) => {
    await sendMessageFeishu({
      cfg,
      to: id,
      text: PAIRING_APPROVED_MESSAGE,
    });
  },
}
```

**功能**: 管理用户配对和授权流程
- `idLabel`: 用户 ID 标签
- `normalizeAllowEntry`: 规范化允许列表条目,移除前缀
- `notifyApproval`: 当用户配对被批准时,发送通知消息

---

## 3. 能力声明 (Capabilities)

**位置**: 第 57-65 行

```typescript
capabilities: {
  chatTypes: ["direct", "channel"],  // 支持私聊和群聊
  polls: false,                      // 不支持投票
  threads: true,                     // 支持消息线程
  media: true,                       // 支持媒体文件
  reactions: true,                   // 支持表情回应
  edit: true,                        // 支持编辑消息
  reply: true,                       // 支持回复消息
}
```

**功能**: 声明频道支持的功能特性
- 帮助系统了解该频道的能力边界
- 用于 UI 显示和功能启用判断

---

## 4. Agent 提示 (Agent Prompt)

**位置**: 第 66-71 行

```typescript
agentPrompt: {
  messageToolHints: () => [
    "- Feishu targeting: omit `target` to reply to the current conversation (auto-inferred). Explicit targets: `user:open_id` or `chat:chat_id`.",
    "- Feishu supports interactive cards for rich messages.",
  ],
}
```

**功能**: 为 AI Agent 提供使用提示
- 告诉 AI 如何正确地发送飞书消息
- 说明目标格式和卡片消息支持

---

## 5. 群组配置 (Groups)

**位置**: 第 72-74 行

```typescript
groups: {
  resolveToolPolicy: resolveFeishuGroupToolPolicy,
}
```

**功能**: 配置群组消息的工具使用策略
- 控制在群组中哪些工具可用

---

## 6. 配置架构 (Config Schema)

**位置**: 第 76-132 行

```typescript
configSchema: {
  schema: {
    type: "object",
    properties: {
      enabled: { type: "boolean" },
      appId: { type: "string" },
      appSecret: { type: "string" },
      // ... 更多配置项
    }
  }
}
```

**功能**: 定义配置文件的 JSON Schema
- **连接配置**: appId, appSecret, domain, connectionMode
- **Webhook 配置**: webhookPath, webhookHost, webhookPort
- **安全策略**: dmPolicy, allowFrom, groupPolicy, groupAllowFrom
- **消息配置**: requireMention, historyLimit, textChunkLimit
- **渲染配置**: renderMode, mediaMaxMb
- **多账号支持**: accounts 对象

---

## 7. 配置管理 (Config)

**位置**: 第 133-224 行

### 7.1 账号列表管理

```typescript
listAccountIds: (cfg) => listFeishuAccountIds(cfg)
```
列出所有配置的飞书账号 ID

### 7.2 账号解析

```typescript
resolveAccount: (cfg, accountId) => resolveFeishuAccount({ cfg, accountId })
```
根据 ID 解析具体账号配置

### 7.3 默认账号

```typescript
defaultAccountId: (cfg) => resolveDefaultFeishuAccountId(cfg)
```
获取默认账号 ID

### 7.4 启用/禁用账号

```typescript
setAccountEnabled: ({ cfg, accountId, enabled }) => {
  // 更新账号的 enabled 状态
  // 区分默认账号和命名账号的处理方式
}
```

### 7.5 删除账号

```typescript
deleteAccount: ({ cfg, accountId }) => {
  // 删除指定账号的配置
  // 默认账号删除整个 feishu 配置
  // 命名账号只删除 accounts 中的对应条目
}
```

### 7.6 账号描述

```typescript
describeAccount: (account) => ({
  accountId: account.accountId,
  enabled: account.enabled,
  configured: account.configured,
  name: account.name,
  appId: account.appId,
  domain: account.domain,
})
```
返回账号的摘要信息

### 7.7 允许列表管理

```typescript
resolveAllowFrom: ({ cfg, accountId }) => { /* 解析允许列表 */ }
formatAllowFrom: ({ allowFrom }) => { /* 格式化允许列表 */ }
```

---

## 8. 安全策略 (Security)

**位置**: 第 225-240 行

```typescript
security: {
  collectWarnings: ({ cfg, accountId }) => {
    // 检查配置的安全性
    // 如果群组策略为 "open",返回安全警告
    return [
      `- Feishu[${account.accountId}] groups: groupPolicy="open" allows any member to trigger...`
    ];
  },
}
```

**功能**: 收集配置相关的安全警告
- 检测不安全的配置(如开放的群组策略)
- 提供改进建议

---

## 9. 设置流程 (Setup)

**位置**: 第 241-277 行

```typescript
setup: {
  resolveAccountId: () => DEFAULT_ACCOUNT_ID,
  applyAccountConfig: ({ cfg, accountId }) => {
    // 应用账号配置
    // 为新账号启用 enabled: true
  },
}
```

**功能**: 处理账号初始化设置
- 确定账号 ID
- 应用初始配置

---

## 10. 引导流程 (Onboarding)

**位置**: 第 278 行

```typescript
onboarding: feishuOnboardingAdapter
```

**功能**: 提供新用户引导流程
- 帮助用户完成首次配置
- 引导创建机器人和获取凭证

---

## 11. 消息功能 (Messaging)

**位置**: 第 279-285 行

```typescript
messaging: {
  normalizeTarget: (raw) => normalizeFeishuTarget(raw) ?? undefined,
  targetResolver: {
    looksLikeId: looksLikeFeishuId,
    hint: "<chatId|user:openId|chat:chatId>",
  },
}
```

**功能**: 处理消息目标解析
- `normalizeTarget`: 规范化目标格式(如 "user:ou_xxx" 或 "chat:oc_xxx")
- `looksLikeId`: 判断字符串是否像飞书 ID
- `hint`: 为用户提供目标格式提示

---

## 12. 通讯录 (Directory)

**位置**: 第 286-316 行

```typescript
directory: {
  self: async () => null,
  listPeers: async ({ cfg, query, limit, accountId }) => 
    listFeishuDirectoryPeers({ cfg, query, limit, accountId }),
  listGroups: async ({ cfg, query, limit, accountId }) => 
    listFeishuDirectoryGroups({ cfg, query, limit, accountId }),
  listPeersLive: async ({ cfg, query, limit, accountId }) => 
    listFeishuDirectoryPeersLive({ cfg, query, limit, accountId }),
  listGroupsLive: async ({ cfg, query, limit, accountId }) => 
    listFeishuDirectoryGroupsLive({ cfg, query, limit, accountId }),
}
```

**功能**: 提供通讯录查询能力
- `self`: 获取机器人自身信息(飞书不提供)
- `listPeers`: 列出用户(从缓存)
- `listGroups`: 列出群组(从缓存)
- `listPeersLive`: 实时查询用户(调用 API)
- `listGroupsLive`: 实时查询群组(调用 API)

**使用场景**:
- 自动补全联系人
- 搜索用户和群组
- 显示可用的聊天目标

---

## 13. 出站消息 (Outbound)

**位置**: 第 317 行

```typescript
outbound: feishuOutbound
```

**功能**: 处理从系统发往飞书的消息
- 消息发送
- 文件上传
- 卡片消息
- 消息编辑
- 表情回应

---

## 14. 状态监控 (Status)

**位置**: 第 318-341 行

```typescript
status: {
  defaultRuntime: createDefaultChannelRuntimeState(DEFAULT_ACCOUNT_ID, { port: null }),
  buildChannelSummary: ({ snapshot }) => ({
    ...buildBaseChannelStatusSummary(snapshot),
    port: snapshot.port ?? null,
    probe: snapshot.probe,
    lastProbeAt: snapshot.lastProbeAt ?? null,
  }),
  probeAccount: async ({ account }) => await probeFeishu(account),
  buildAccountSnapshot: ({ account, runtime, probe }) => ({
    accountId: account.accountId,
    enabled: account.enabled,
    configured: account.configured,
    name: account.name,
    appId: account.appId,
    domain: account.domain,
    running: runtime?.running ?? false,
    lastStartAt: runtime?.lastStartAt ?? null,
    lastStopAt: runtime?.lastStopAt ?? null,
    lastError: runtime?.lastError ?? null,
    port: runtime?.port ?? null,
    probe,
  }),
}
```

**功能**: 管理频道运行状态
- `defaultRuntime`: 创建默认运行时状态
- `buildChannelSummary`: 构建频道状态摘要
- `probeAccount`: 探测账号连接状态(检查 API 可用性)
- `buildAccountSnapshot`: 构建账号状态快照

**监控信息包括**:
- 运行状态(running/stopped)
- 启动/停止时间
- 错误信息
- 端口信息
- 探测结果

---

## 15. 网关管理 (Gateway)

**位置**: 第 342-358 行

```typescript
gateway: {
  startAccount: async (ctx) => {
    const { monitorFeishuProvider } = await import("./monitor.js");
    const account = resolveFeishuAccount({ cfg: ctx.cfg, accountId: ctx.accountId });
    const port = account.config?.webhookPort ?? null;
    ctx.setStatus({ accountId: ctx.accountId, port });
    ctx.log?.info(
      `starting feishu[${ctx.accountId}] (mode: ${account.config?.connectionMode ?? "websocket"})`,
    );
    return monitorFeishuProvider({
      config: ctx.cfg,
      runtime: ctx.runtime,
      abortSignal: ctx.abortSignal,
      accountId: ctx.accountId,
    });
  },
}
```

**功能**: 启动和管理飞书消息监听服务
- 动态导入 `monitor.js` 模块
- 解析账号配置
- 设置运行状态(包括端口信息)
- 记录启动日志
- 启动消息监听器(支持 WebSocket 和 Webhook 两种模式)

**连接模式**:
- **websocket**: 长连接模式,实时接收消息
- **webhook**: HTTP 回调模式,需要配置公网可访问的地址

---

## 与工具注册的关系

### Channel (channel.ts) 的作用
- **注册消息频道**: 让 OpenClaw 能够与飞书进行**双向消息通信**
- **接收消息**: 监听来自飞书的消息事件
- **发送消息**: 向飞书用户/群组发送消息
- **管理连接**: 维护与飞书服务器的连接状态

### Tools (工具注册) 的作用
- **注册操作工具**: 让 AI Agent 能够**主动调用飞书 API**
- **文档操作**: 读写飞书文档
- **知识库管理**: 操作飞书知识库
- **云盘操作**: 管理飞书云盘文件
- **权限管理**: 控制文档和文件的访问权限

### 协同工作流程

```
用户在飞书发消息
    ↓
[Channel] 接收消息 (monitor.js)
    ↓
[Gateway] 处理消息
    ↓
[AI Agent] 理解需求
    ↓
[Tools] 调用飞书 API 执行操作
    ↓
[Channel] 发送结果消息 (send.js)
    ↓
用户在飞书看到回复
```

---

## 配置示例

```yaml
channels:
  feishu:
    enabled: true
    appId: "cli_xxx"
    appSecret: "xxx"
    domain: "feishu"  # 或 "lark"
    connectionMode: "websocket"  # 或 "webhook"
    
    # 安全策略
    dmPolicy: "pairing"  # 私聊策略: open/pairing/allowlist
    allowFrom:
      - "ou_xxxxx"  # 允许的用户 open_id
    
    groupPolicy: "allowlist"  # 群聊策略: open/allowlist/disabled
    groupAllowFrom:
      - "oc_xxxxx"  # 允许的群聊 chat_id
    
    # 消息配置
    requireMention: true  # 群聊中需要 @机器人
    historyLimit: 20  # 历史消息数量
    renderMode: "auto"  # 渲染模式: auto/raw/card
    
    # 多账号支持
    accounts:
      production:
        enabled: true
        name: "生产环境"
        appId: "cli_yyy"
        appSecret: "yyy"
```

---

## 总结

`channel.ts` 是飞书集成的**核心枢纽**,它:

1. **定义接口**: 实现 `ChannelPlugin` 接口,标准化频道功能
2. **声明能力**: 告诉系统飞书支持哪些功能特性
3. **管理配置**: 处理账号、安全策略、连接模式等配置
4. **处理通信**: 接收和发送消息,维护连接状态
5. **提供查询**: 支持通讯录查询和状态监控
6. **确保安全**: 实施访问控制和安全策略

配合工具注册函数,形成完整的飞书集成方案:
- **Channel 负责通信**: 消息的收发
- **Tools 负责操作**: API 的调用

这种分离设计使得系统既能被动响应消息,也能主动执行操作,实现了灵活强大的飞书集成能力。
