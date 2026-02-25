# 飞书 (Feishu) 渠道集成详解

> 本文档详细介绍 OpenClaw 如何集成飞书消息平台

## 目录
- [概述](#概述)
- [架构设计](#架构设计)
- [核心模块](#核心模块)
- [消息流转](#消息流转)
- [工具系统](#工具系统)
- [配置指南](#配置指南)
- [高级特性](#高级特性)

---

## 概述

飞书集成是 OpenClaw 的**渠道插件 (Channel Plugin)**，位于 `extensions/feishu/` 目录。

### 支持的功能

- ✅ 私聊 (DM) 和群聊 (Group Chat)
- ✅ 实时消息 (WebSocket) 和 Webhook 模式
- ✅ 富文本消息 (卡片/Markdown)
- ✅ 媒体附件 (图片/文件/语音)
- ✅ @提及和转发
- ✅ 表情回应
- ✅ 话题隔离会话
- ✅ 动态 Agent 创建
- ✅ 知识库/文档/云盘工具

---

## 架构设计

### 整体架构

```
飞书开放平台
    │
    ├─ WebSocket 长连接 ────────┐
    │                           │
    └─ Webhook HTTP 回调 ───────┤
                                │
                                ▼
                    ┌───────────────────────┐
                    │  Feishu Plugin        │
                    │  (Channel Plugin)     │
                    └───────────┬───────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
            ┌─────────────┐ ┌─────────┐ ┌─────────┐
            │  Monitor    │ │  Bot    │ │  Send   │
            │  (监听)     │ │ (处理)  │ │ (发送)  │
            └─────────────┘ └─────────┘ └─────────┘
                    │           │           │
                    └───────────┼───────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
            ┌─────────────┐ ┌─────────┐ ┌─────────┐
            │  Directory  │ │ Policy  │ │ Reply   │
            │  (目录)     │ │(权限)   │ │(回复)   │
            └─────────────┘ └─────────┘ └─────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
            ┌─────────────┐ ┌─────────┐ ┌─────────┐
            │   Tools     │ │ Media   │ │ Dynamic │
            │   (工具)    │ │(媒体)   │ │ Agent   │
            └─────────────┘ └─────────┘ └─────────┘
```

### 目录结构

```
extensions/feishu/
├── index.ts                    # 插件入口
├── src/
│   ├── channel.ts              # 渠道插件定义
│   ├── monitor.ts              # 消息监听 (WebSocket/Webhook)
│   ├── bot.ts                  # 消息处理核心
│   ├── send.ts                 # 消息发送
│   ├── client.ts               # 飞书 SDK 封装
│   ├── accounts.ts             # 多账号管理
│   ├── directory.ts            # 联系人/群组目录
│   ├── outbound.ts             # 出站消息适配器
│   ├── reply-dispatcher.ts     # 回复分发器
│   ├── policy.ts               # 权限策略
│   ├── dynamic-agent.ts        # 动态 Agent 创建
│   ├── mention.ts              # @提及处理
│   ├── media.ts                # 图片/文件上传下载
│   ├── reactions.ts            # 表情回应
│   ├── typing.ts               # 输入中状态
│   ├── onboarding.ts           # 配置向导
│   ├── probe.ts                # 连接探测
│   ├── types.ts                # 类型定义
│   ├── config-schema.ts        # 配置 Schema
│   ├── targets.ts              # 目标解析
│   ├── wiki.ts                 # 知识库工具
│   ├── docx.ts                 # 文档工具
│   ├── drive.ts                # 云盘工具
│   ├── bitable.ts              # 多维表格工具
│   └── perm.ts                 # 权限管理工具
```

---

## 核心模块

### 1. 插件注册 (`index.ts`)

**代码位置**: [`extensions/feishu/index.ts:52`](../extensions/feishu/index.ts#L52)

```typescript
const plugin = {
  id: "feishu",
  name: "Feishu",
  description: "Feishu/Lark channel plugin",
  
  register(api: OpenClawPluginApi) {
    // 1. 设置运行时环境
    setFeishuRuntime(api.runtime);
    
    // 2. 注册渠道插件
    api.registerChannel({ plugin: feishuPlugin });
    
    // 3. 注册工具集
    registerFeishuDocTools(api);      // 文档
    registerFeishuWikiTools(api);     // 知识库
    registerFeishuDriveTools(api);    // 云盘
    registerFeishuPermTools(api);     // 权限
    registerFeishuBitableTools(api);  // 多维表格
  },
};
```

### 2. 渠道插件定义 (`channel.ts`)

**代码位置**: [`extensions/feishu/src/channel.ts:34`](../extensions/feishu/src/channel.ts#L34)

```typescript
export const feishuPlugin: ChannelPlugin<ResolvedFeishuAccount> = {
  id: "feishu",
  
  // 元信息
  meta: {
    label: "Feishu",
    selectionLabel: "Feishu/Lark (飞书)",
    aliases: ["lark"],
    docsPath: "/channels/feishu",
  },
  
  // 能力声明
  capabilities: {
    chatTypes: ["direct", "channel"],
    polls: false,
    threads: true,      // 支持话题
    media: true,        // 支持媒体
    reactions: true,    // 支持表情
    edit: true,         // 支持编辑
    reply: true,        // 支持回复
  },
  
  // Gateway 生命周期钩子
  gateway: {
    startAccount: async (ctx) => {
      await monitorFeishuProvider({
        config: ctx.cfg,
        runtime: ctx.runtime,
        accountId: ctx.accountId,
        abortSignal: ctx.abortSignal,
      });
    },
  },
  
  // 出站消息适配器
  outbound: feishuOutbound,
  
  // 配对机制
  pairing: {
    idLabel: "feishuUserId",
    normalizeAllowEntry: (entry) => entry.replace(/^(feishu|user|open_id):/i, ""),
    notifyApproval: async ({ cfg, id }) => {
      await sendMessageFeishu({ cfg, to: id, text: PAIRING_APPROVED_MESSAGE });
    },
  },
};
```

### 3. 消息监听 (`monitor.ts`)

**代码位置**: [`extensions/feishu/src/monitor.ts:1`](../extensions/feishu/src/monitor.ts#L1)

#### 3.1 WebSocket 模式 (默认)

```typescript
async function monitorAccountWebSocket(params: {
  cfg: ClawdbotConfig;
  account: ResolvedFeishuAccount;
  runtime?: RuntimeEnv;
  abortSignal?: AbortSignal;
}) {
  const { cfg, account, runtime, abortSignal } = params;
  const accountId = account.accountId;
  
  // 1. 创建 WebSocket 客户端
  const wsClient = createFeishuWSClient(account);
  
  // 2. 创建事件分发器
  const eventDispatcher = createEventDispatcher(account);
  
  // 3. 注册事件处理器
  registerEventHandlers(eventDispatcher, {
    cfg,
    accountId,
    runtime,
    chatHistories: new Map(),
    fireAndForget: false,  // WebSocket 模式可以 await
  });
  
  // 4. 启动 WebSocket 连接
  wsClient.start({ eventDispatcher });
  
  log(`feishu[${accountId}]: WebSocket client started`);
}
```

**优点**:
- 实时性高（毫秒级延迟）
- 自动重连
- 不需要公网 IP

#### 3.2 Webhook 模式

```typescript
async function monitorAccountWebhook(params: {
  cfg: ClawdbotConfig;
  account: ResolvedFeishuAccount;
  runtime?: RuntimeEnv;
  abortSignal?: AbortSignal;
}) {
  const { cfg, account, runtime, abortSignal } = params;
  const accountId = account.accountId;
  
  const port = account.config.webhookPort ?? 3000;
  const path = account.config.webhookPath ?? "/feishu/events";
  
  // 1. 创建 HTTP 服务器
  const server = http.createServer((req, res) => {
    if (req.url === path && req.method === "POST") {
      // 2. 处理飞书回调
      eventDispatcher.invoke(req, res);
    } else {
      res.writeHead(404);
      res.end();
    }
  });
  
  // 3. 注册事件处理器
  registerEventHandlers(eventDispatcher, {
    cfg,
    accountId,
    runtime,
    chatHistories: new Map(),
    fireAndForget: true,  // Webhook 需快速响应 (<3s)
  });
  
  // 4. 启动服务器
  server.listen(port);
  
  log(`feishu[${accountId}]: Webhook server listening on port ${port}`);
}
```

**优点**:
- 简单直接
- 适合生产环境
- 可以和其他 HTTP 服务共存

**要求**:
- 需要公网 IP 或内网穿透
- 需要配置飞书后台的 Webhook URL

### 4. 消息处理核心 (`bot.ts`)

**代码位置**: [`extensions/feishu/src/bot.ts:1`](../extensions/feishu/src/bot.ts#L1)

```typescript
export async function handleFeishuMessage(params: {
  cfg: ClawdbotConfig;
  event: FeishuMessageEvent;
  botOpenId?: string;
  runtime?: RuntimeEnv;
  chatHistories: Map<string, HistoryEntry[]>;
  accountId: string;
}) {
  const { cfg, event, botOpenId, runtime, chatHistories, accountId } = params;
  
  // 1. 消息去重
  const messageId = event.message.message_id;
  if (!tryRecordMessage(messageId)) {
    log(`feishu: skipping duplicate message ${messageId}`);
    return;
  }
  
  // 2. 提取消息上下文
  const ctx = extractFeishuMessageContext(event, botOpenId);
  
  // 3. 权限检查
  const account = resolveFeishuAccount({ cfg, accountId });
  const feishuCfg = account.config;
  
  if (ctx.isGroup) {
    // 群组权限检查
    const groupPolicy = feishuCfg?.groupPolicy ?? "open";
    if (groupPolicy === "allowlist") {
      const allowed = isFeishuGroupAllowed({ cfg: feishuCfg, groupId: ctx.chatId });
      if (!allowed) return;
    }
    
    // 检查是否 @机器人
    if (feishuCfg?.requireMention && !ctx.mentionedBot) {
      // 未提及机器人，只记录历史
      recordPendingHistoryEntryIfEnabled({ ... });
      return;
    }
  } else {
    // 私聊权限检查
    const dmPolicy = feishuCfg?.dmPolicy ?? "pairing";
    const match = resolveFeishuAllowlistMatch({ ... });
    if (!match.allowed) return;
  }
  
  // 4. 解析会话路由
  const route = await resolveOutboundSession({
    cfg,
    channel: "feishu",
    accountId: account.accountId,
    peerId: ctx.senderOpenId,
    from: `feishu:${ctx.senderOpenId}`,
  });
  
  // 5. 动态 Agent 创建（如果启用）
  if (!ctx.isGroup && route.matchedBy === "default") {
    const dynamicCfg = feishuCfg?.dynamicAgentCreation;
    if (dynamicCfg?.enabled) {
      const result = await maybeCreateDynamicAgent({ cfg, senderOpenId: ctx.senderOpenId });
      if (result.created) {
        // 重新解析路由
        route = await resolveOutboundSession({ ... });
      }
    }
  }
  
  // 6. 下载媒体附件
  const media = await resolveFeishuMediaList({ ctx, cfg, account });
  
  // 7. 获取引用消息（如果有）
  let quotedContent: string | undefined;
  if (ctx.quotedMessageId) {
    const quoted = await getMessageFeishu({ cfg, messageId: ctx.quotedMessageId, accountId });
    quotedContent = extractMessageBody(quoted);
  }
  
  // 8. 构建消息载荷
  const payload = {
    CommandBody: ctx.content,
    From: `feishu:${ctx.senderOpenId}`,
    To: ctx.isGroup ? `chat:${ctx.chatId}` : `user:${ctx.senderOpenId}`,
    SessionKey: route.sessionKey,
    SenderId: ctx.senderOpenId,
    Provider: "feishu" as const,
    Surface: "feishu" as const,
    MessageSid: ctx.messageId,
    ReplyToMessageSid: ctx.quotedMessageId,
    QuotedContent: quotedContent,
    CommandAuthorized: true,
    OriginatingChannel: "feishu" as const,
    ...media,
  };
  
  // 9. 分发到 Agent
  const dispatcher = createFeishuReplyDispatcher({
    cfg,
    info: { chatId: ctx.chatId, messageId: ctx.messageId },
    account,
    runtime,
  });
  
  await dispatchMessage(payload, dispatcher);
  
  log(`feishu[${accountId}]: dispatch complete`);
}
```

### 5. 回复分发器 (`reply-dispatcher.ts`)

**代码位置**: [`extensions/feishu/src/reply-dispatcher.ts:1`](../extensions/feishu/src/reply-dispatcher.ts#L1)

```typescript
export function createFeishuReplyDispatcher(params: {
  cfg: ClawdbotConfig;
  info: { chatId: string; messageId?: string };
  account: ResolvedFeishuAccount;
  runtime?: RuntimeEnv;
}) {
  const { cfg, info, account, runtime } = params;
  const { chatId, messageId: replyToMessageId } = info;
  const accountId = account.accountId;
  
  let typingState: { messageId: string; reactionId: string | null } | null = null;
  
  return {
    // Agent 回复时调用
    async deliver(payload: { text?: string; media?: MediaAttachment[] }) {
      const text = payload.text || "";
      
      if (!text.trim()) {
        runtime.log?.(`feishu[${accountId}] deliver: empty text, skipping`);
        return;
      }
      
      // 1. 文本分块（飞书限制 4000 字符）
      const textChunkLimit = core.channel.text.resolveTextChunkLimit(
        cfg,
        "feishu",
        accountId,
        { fallbackLimit: 4000 }
      );
      const chunkMode = core.channel.text.resolveChunkMode(cfg, "feishu");
      const chunks = core.channel.text.chunkText(text, textChunkLimit, chunkMode);
      
      // 2. 渲染模式选择
      const renderMode = account.config?.renderMode ?? "auto";
      
      if (renderMode === "card" || (renderMode === "auto" && shouldUseCard(text))) {
        // 发送卡片消息
        runtime.log?.(`feishu[${accountId}] deliver: sending ${chunks.length} card chunks`);
        for (const chunk of chunks) {
          await sendCardFeishu({
            cfg,
            chatId,
            content: buildCardContent(chunk),
            accountId,
          });
        }
      } else {
        // 发送文本消息
        runtime.log?.(`feishu[${accountId}] deliver: sending ${chunks.length} text chunks`);
        for (const chunk of chunks) {
          await sendMessageFeishu({
            cfg,
            to: chatId,
            text: chunk,
            accountId,
          });
        }
      }
      
      // 3. 发送媒体附件
      if (payload.media && payload.media.length > 0) {
        for (const media of payload.media) {
          await sendMediaFeishu({
            cfg,
            chatId,
            media,
            accountId,
          });
        }
      }
    },
    
    // 显示输入中状态
    async startTyping() {
      if (!replyToMessageId) return;
      typingState = await addTypingIndicator({ cfg, messageId: replyToMessageId, accountId });
      runtime.log?.(`feishu[${accountId}]: added typing indicator reaction`);
    },
    
    // 移除输入中状态
    async stopTyping() {
      if (!typingState) return;
      await removeTypingIndicator(typingState);
      typingState = null;
      runtime.log?.(`feishu[${accountId}]: removed typing indicator reaction`);
    },
  };
}
```

---

## 消息流转

```
飞书用户发送消息
    │
    ▼
┌─────────────────────────────────────────┐
│ 飞书服务器                              │
│  ├─ WebSocket → Gateway                 │
│  └─ Webhook → Gateway                   │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ monitor.ts                              │
│  ├─ 接收 im.message.receive_v1 事件     │
│  └─ 调用 handleFeishuMessage()          │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ bot.ts                                  │
│  1. 消息去重                             │
│  2. 提取上下文 (发送者/群组/内容/@提及)  │
│  3. 权限检查 (DM policy/Group policy)   │
│  4. 路由解析 (匹配 Agent/会话)           │
│  5. 动态 Agent 创建 (如果启用)          │
│  6. 下载媒体附件                        │
│  7. 构建消息载荷                        │
│  8. 分发到 Agent                        │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Gateway dispatchMessage                 │
│  ├─ 保存到会话历史                      │
│  └─ 调用 Agent 处理                     │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Agent 处理                              │
│  ├─ 思考                                │
│  ├─ 调用工具                            │
│  └─ 生成回复                            │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ reply-dispatcher.ts                     │
│  1. 显示输入中状态 (Typing...)          │
│  2. 文本分块 (4000 字符限制)            │
│  3. 渲染模式选择 (文本/卡片)            │
│  4. 发送消息                            │
│  5. 发送媒体附件                        │
│  6. 移除输入中状态                      │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ send.ts                                 │
│  ├─ sendMessageFeishu()                 │
│  ├─ sendCardFeishu()                    │
│  └─ sendMediaFeishu()                   │
└────────────┬────────────────────────────┘
             │
             ▼
        飞书服务器
             │
             ▼
        飞书用户接收消息
```

---

## 工具系统

飞书插件提供了丰富的工具集，让 Agent 能操作飞书平台的各种资源。

### 1. 知识库工具 (`feishu_wiki`)

**代码位置**: [`extensions/feishu/src/wiki.ts`](../extensions/feishu/src/wiki.ts)

**功能**:
- `spaces`: 列出所有知识库空间
- `nodes`: 列出空间内的节点（文档树）
- `get`: 获取节点详情
- `create`: 创建新节点
- `move`: 移动节点
- `rename`: 重命名节点

**使用示例**:
```
Agent: 列出所有知识库空间
工具调用: feishu_wiki { action: "spaces" }
返回: { spaces: [{ space_id: "xxx", name: "团队知识库" }] }

Agent: 在指定空间创建文档
工具调用: feishu_wiki { 
  action: "create", 
  space_id: "xxx", 
  title: "API 设计文档",
  obj_type: "docx"
}
返回: { node_token: "yyy", title: "API 设计文档" }
```

### 2. 文档工具 (`feishu_doc`)

**代码位置**: [`extensions/feishu/src/docx.ts`](../extensions/feishu/src/docx.ts)

**功能**:
- `get`: 获取文档内容（纯文本）
- `list_blocks`: 列出文档块结构
- `create_block`: 创建新块
- `update_block`: 更新块内容
- `delete_block`: 删除块

**使用示例**:
```
Agent: 读取文档内容
工具调用: feishu_doc { action: "get", doc_id: "xxx" }
返回: { title: "技术文档", content: "..." }

Agent: 在文档末尾追加内容
工具调用: feishu_doc {
  action: "create_block",
  doc_id: "xxx",
  block_type: "text",
  text: "新增的段落内容"
}
```

### 3. 云盘工具 (`feishu_drive`)

**代码位置**: [`extensions/feishu/src/drive.ts`](../extensions/feishu/src/drive.ts)

**功能**:
- 文件列表
- 文件上传
- 文件下载
- 文件夹管理

### 4. 多维表格工具 (`feishu_bitable`)

**代码位置**: [`extensions/feishu/src/bitable.ts`](../extensions/feishu/src/bitable.ts)

**功能**:
- 列出字段
- 列出记录
- 获取记录
- 创建记录
- 更新记录

---

## 配置指南

### 基本配置

在 `~/.openclaw/openclaw.json`:

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_a1b2c3d4e5f6g7h8",
      "appSecret": "your-app-secret-here",
      "domain": "feishu",
      "connectionMode": "websocket"
    }
  }
}
```

### 完整配置示例

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      
      // 凭证配置
      "appId": "cli_xxx",
      "appSecret": "xxx",
      "encryptKey": "xxx",              // 消息加密密钥（可选）
      "verificationToken": "xxx",        // Webhook 验证 token
      
      // 连接模式
      "domain": "feishu",                // "feishu" (国内) | "lark" (国际)
      "connectionMode": "websocket",     // "websocket" | "webhook"
      "webhookPath": "/feishu/events",   // Webhook 路径
      "webhookPort": 3000,               // Webhook 端口
      
      // 私聊策略
      "dmPolicy": "pairing",             // "open" | "pairing" | "allowlist"
      "allowFrom": ["ou_xxx", "ou_yyy"], // 允许的用户 open_id
      
      // 群组策略
      "groupPolicy": "open",             // "open" | "allowlist" | "disabled"
      "groupAllowFrom": ["oc_xxx"],      // 允许的群组 chat_id
      "requireMention": true,            // 群组需要 @机器人
      
      // 会话管理
      "topicSessionMode": "enabled",     // 话题隔离会话
      "historyLimit": 10,                // 群组历史消息数量
      "dmHistoryLimit": 20,              // 私聊历史消息数量
      
      // 渲染设置
      "renderMode": "auto",              // "auto" | "raw" | "card"
      "textChunkLimit": 4000,            // 文本分块大小
      "chunkMode": "length",             // "length" | "newline"
      
      // 动态 Agent
      "dynamicAgentCreation": {
        "enabled": true,
        "maxAgents": 100,
        "workspace": "~/.openclaw/agents/feishu"
      },
      
      // 媒体设置
      "mediaMaxMb": 30,                  // 最大媒体文件大小
      
      // 工具配置
      "tools": {
        "wiki": true,
        "doc": true,
        "drive": true,
        "perm": false,
        "bitable": true
      }
    }
  }
}
```

### 环境变量支持

也可以通过环境变量配置（优先级高于配置文件）:

```bash
export FEISHU_APP_ID="cli_xxx"
export FEISHU_APP_SECRET="xxx"
```

---

## 高级特性

### 1. 话题隔离会话

**功能**: 群组中不同话题独立会话，避免上下文混淆

**配置**:
```json
{
  "channels": {
    "feishu": {
      "topicSessionMode": "enabled"
    }
  }
}
```

**效果**:
- 话题 A 的会话 ID: `feishu:group:oc_xxx:topic:om_aaa`
- 话题 B 的会话 ID: `feishu:group:oc_xxx:topic:om_bbb`

### 2. 动态 Agent 创建

**功能**: 为每个用户自动创建独立 Agent

**配置**:
```json
{
  "channels": {
    "feishu": {
      "dynamicAgentCreation": {
        "enabled": true,
        "maxAgents": 100
      }
    }
  }
}
```

**效果**:
- 用户 `ou_xxx` → Agent `feishu-ou_xxx`
- 每个用户有独立的配置和会话历史

### 3. 富文本卡片渲染

**功能**: 自动将 Markdown 渲染为飞书交互卡片

**配置**:
```json
{
  "channels": {
    "feishu": {
      "renderMode": "auto"  // 自动判断
    }
  }
}
```

**判断逻辑**:
- 包含代码块 → 使用卡片
- 包含表格 → 使用卡片
- 包含列表 → 使用卡片
- 纯文本 → 使用文本消息

### 4. 权限错误自动处理

**功能**: 检测权限错误并自动通知用户授权链接

**代码位置**: [`extensions
