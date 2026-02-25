# OpenClaw Gateway 架构设计分析

> 本文档详细分析 OpenClaw Gateway 的核心架构、设计模式和实现细节

## 目录
- [概述](#概述)
- [整体架构](#整体架构)
- [核心模块](#核心模块)
- [启动流程](#启动流程)
- [WebSocket 协议](#websocket-协议)
- [Hooks 系统](#hooks-系统)
- [设计模式](#设计模式)
- [配置管理](#配置管理)

---

## 概述

Gateway 是 OpenClaw 的**核心服务器组件**，负责：
- WebSocket 连接管理和消息路由
- 多渠道消息集成（Telegram/Discord/Slack/Signal/飞书等）
- Agent 运行时协调和会话管理
- HTTP API 端点（OpenAI 兼容接口、Hooks 等）
- Node 节点注册和远程调用

**核心代码位置**: [`src/gateway/`](../src/gateway/)

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          OpenClaw Gateway                                │
│                       (Core Coordination Hub)                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐          ┌────────────────┐         ┌─────────────────┐
│   WebSocket   │          │  HTTP Server   │         │  Plugin System  │
│   Server      │          │                │         │                 │
└───────────────┘          └────────────────┘         └─────────────────┘
        │                           │                           │
        │                           │                           │
   ┌────┴─────┐              ┌──────┴───────┐          ┌───────┴────────┐
   │          │              │              │          │                │
   ▼          ▼              ▼              ▼          ▼                ▼
┌──────┐  ┌────────┐   ┌─────────┐  ┌──────────┐  ┌────────┐   ┌────────┐
│Client│  │ Node   │   │Control  │  │  Hooks   │  │Gateway │   │Channel │
│(UI)  │  │(远程)  │   │   UI    │  │(Webhook) │  │Plugins │   │Plugins │
└──────┘  └────────┘   └─────────┘  └──────────┘  └────────┘   └────────┘
```

---

## 核心模块

### 1. 入口层 (Entry Points)

**代码位置**: [`src/gateway/server.impl.ts:155`](../src/gateway/server.impl.ts#L155)

```
┌─────────────────────────────────────────────────────┐
│ WebSocket (主通道)                                  │
│  ├─ 握手认证 (connect.challenge)                    │
│  ├─ 方法调用 (RequestFrame)                         │
│  └─ 事件推送 (EventFrame)                           │
├─────────────────────────────────────────────────────┤
│ HTTP 端点                                           │
│  ├─ 控制 UI (静态资源 + API)                        │
│  ├─ OpenAI 兼容接口 (/v1/chat/completions)          │
│  ├─ OpenResponses API (/v1/responses)               │
│  ├─ Hooks (webhook → wake/agent)                    │
│  └─ Canvas Host (A2UI)                              │
└─────────────────────────────────────────────────────┘
```

### 2. 协议层 (Protocol Layer)

**代码位置**: [`src/gateway/protocol/index.ts`](../src/gateway/protocol/index.ts)

```typescript
// Frame 类型定义
type RequestFrame = {
  id: string;           // 请求 ID
  method: string;       // 方法名
  params: unknown;      // 参数
};

type ResponseFrame = {
  id: string;           // 对应请求 ID
  result?: unknown;     // 成功结果
  error?: ErrorShape;   // 错误信息
};

type EventFrame = {
  event: string;        // 事件名
  payload: unknown;     // 事件载荷
  seq?: number;         // 序列号
};
```

**Schema 验证**: 使用 Ajv 进行严格的 JSON Schema 验证

### 3. 权限层 (Authorization)

**代码位置**: [`src/gateway/server-methods.ts:93`](../src/gateway/server-methods.ts#L93)

```
┌──────────────────────────────────────────┐
│ Roles:                                   │
│  ├─ operator (人类操作者)                │
│  └─ node (远程节点)                      │
├──────────────────────────────────────────┤
│ Scopes (operator):                       │
│  ├─ admin    (完全控制)                  │
│  ├─ read     (只读查询)                  │
│  ├─ write    (执行操作)                  │
│  ├─ approvals (命令审批)                 │
│  └─ pairing  (设备配对)                  │
└──────────────────────────────────────────┘
```

### 4. 核心服务层

**代码位置**: 分散在 [`src/gateway/server-*.ts`](../src/gateway/)

```
┌────────────────────┬─────────────────────┬──────────────────┐
│  Method Router     │   Channel Manager   │  Agent Runtime   │
│  ┌──────────────┐  │  ┌───────────────┐  │  ┌────────────┐  │
│  │ agent        │  │  │ Telegram      │  │  │ Chat State │  │
│  │ send         │  │  │ Discord       │  │  │ Run Queue  │  │
│  │ wake         │  │  │ Slack         │  │  │ Buffers    │  │
│  │ config.*     │  │  │ Signal        │  │  │ Aborts     │  │
│  │ sessions.*   │  │  │ WhatsApp      │  │  └────────────┘  │
│  │ nodes.*      │  │  │ iMessage      │  │                  │
│  │ models.*     │  │  │ Feishu        │  │                  │
│  │ skills.*     │  │  │ (plugins...)  │  │                  │
│  │ cron.*       │  │  └───────────────┘  │                  │
│  └──────────────┘  │                     │                  │
└────────────────────┴─────────────────────┴──────────────────┘
```

### 5. 广播系统 (Broadcast System)

**代码位置**: [`src/gateway/server-broadcast.ts:34`](../src/gateway/server-broadcast.ts#L34)

```typescript
// 智能广播特性
- Scope 过滤 (权限检查)
- 慢消费者检测 (bufferedAmount > 256KB)
- 可选丢弃策略 (dropIfSlow)
- 定向推送 (broadcastToConnIds)
- 事件序列号 (单调递增 seq)
```

---

## 启动流程

**主入口**: [`startGatewayServer(port, opts)`](../src/gateway/server.impl.ts#L155)

```
┌─────────────────────────────────────────────────────────┐
│ 1. 配置加载与迁移                                       │
│    ├─ 读取 ~/.openclaw/openclaw.json                   │
│    ├─ 检测旧配置 → 自动迁移                             │
│    └─ 验证配置 schema                                   │
├─────────────────────────────────────────────────────────┤
│ 2. 插件系统初始化                                       │
│    ├─ 加载 gateway 插件                                 │
│    ├─ 加载 channel 插件 (Telegram/Discord/...)         │
│    └─ 收集所有可用 methods                              │
├─────────────────────────────────────────────────────────┤
│ 3. 运行时配置解析                                       │
│    ├─ 绑定地址策略 (loopback/LAN/tailnet)              │
│    ├─ 端口配置                                          │
│    ├─ 认证配置 (token/password/tailscale)              │
│    ├─ TLS 设置                                          │
│    └─ Hooks 配置                                        │
├─────────────────────────────────────────────────────────┤
│ 4. HTTP/WebSocket 服务器创建                            │
│    ├─ 创建 HTTP(S) server                               │
│    ├─ 挂载 WebSocket server                             │
│    └─ 绑定端口监听                                      │
├─────────────────────────────────────────────────────────┤
│ 5. 核心服务启动                                         │
│    ├─ Discovery (mDNS/广域DNS)                          │
│    ├─ Channel Manager (启动各渠道)                     │
│    ├─ Cron Service (定时任务)                          │
│    ├─ Heartbeat Runner (心跳)                           │
│    └─ Maintenance Timers (维护定时器)                  │
├─────────────────────────────────────────────────────────┤
│ 6. WebSocket Handlers 挂载                              │
│    ├─ 连接处理器                                        │
│    ├─ 消息处理器                                        │
│    └─ 事件广播器                                        │
├─────────────────────────────────────────────────────────┤
│ 7. 辅助服务启动                                         │
│    ├─ Browser Control Server                            │
│    ├─ Plugin Services                                   │
│    ├─ Tailscale Exposure                                │
│    └─ Config Hot Reload                                 │
└─────────────────────────────────────────────────────────┘
```

**关键代码片段**:

```typescript
// src/gateway/server.impl.ts
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer> {
  // 1. 配置加载
  const configSnapshot = await readConfigFileSnapshot();
  const cfgAtStart = loadConfig();
  
  // 2. 插件加载
  const { pluginRegistry, gatewayMethods } = loadGatewayPlugins({
    cfg: cfgAtStart,
    coreGatewayHandlers,
    baseMethods,
  });
  
  // 3. 运行时配置
  const runtimeConfig = await resolveGatewayRuntimeConfig({
    cfg: cfgAtStart,
    port,
    bind: opts.bind,
    auth: opts.auth,
  });
  
  // 4. 创建服务器
  const { httpServer, wss, clients, broadcast } = 
    await createGatewayRuntimeState({ ... });
  
  // 5. 启动服务
  await startGatewayDiscovery({ ... });
  await startChannels();
  cron.start();
  startHeartbeatRunner();
  
  // 6. 挂载 handlers
  attachGatewayWsHandlers({ wss, clients, ... });
  
  // 7. 热重载
  startGatewayConfigReloader({ ... });
  
  return { close: () => { ... } };
}
```

---

## WebSocket 协议

### 握手流程

**代码位置**: [`src/gateway/server/ws-connection.ts:19`](../src/gateway/server/ws-connection.ts#L19)

```
客户端连接
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. 服务器发送 connect.challenge         │
│    { nonce, ts }                        │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ 2. 客户端发送 connect 请求              │
│    { token, password, client, ... }     │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ 3. 服务器验证认证信息                   │
│    - 检查 token/password                │
│    - 验证 Tailscale (如果启用)          │
│    - 检查 role 和 scopes                │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ 4. 服务器返回 hello                     │
│    { version, methods, events, ... }    │
└────────────┬────────────────────────────┘
             │
             ▼
    持久连接建立 ✓
```

### 消息流转

```
┌──────────────────────────────────────────────────────┐
│ Request (客户端 → 服务器)                            │
│  {                                                   │
│    type: "request",                                  │
│    id: "req-123",                                    │
│    method: "agent",                                  │
│    params: { message: "Hello" }                     │
│  }                                                   │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │  Method Router       │
        │  (鉴权 + 分发)       │
        └──────────┬───────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────┐
│ Response (服务器 → 客户端)                           │
│  {                                                   │
│    type: "response",                                 │
│    id: "req-123",                                    │
│    result: { ... }                                   │
│  }                                                   │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ Event (服务器主动推送)                               │
│  {                                                   │
│    type: "event",                                    │
│    event: "agent",                                   │
│    seq: 42,                                          │
│    payload: { ... }                                  │
│  }                                                   │
└──────────────────────────────────────────────────────┘
```

---

## Hooks 系统

### 架构概览

**代码位置**: [`src/gateway/hooks-mapping.ts`](../src/gateway/hooks-mapping.ts)

```
外部系统 (GitHub/CI/IoT)
    │
    │ POST /hooks/github
    │ { payload: {...} }
    │
    ▼
┌────────────────────────────────────┐
│  HTTP Hooks Handler                │
│  1. Token 验证                     │
│  2. Payload 解析                   │
│  3. Mapping 匹配                   │
└────────────┬───────────────────────┘
             │
             ▼
┌────────────────────────────────────┐
│  Hook Mapping                      │
│  - matchPath: "github"             │
│  - matchSource: "push"             │
│  - transform: (ctx) => {...}       │
│  - action: "agent"                 │
└────────────┬───────────────────────┘
             │
             ▼
┌────────────────────────────────────┐
│  Template Rendering                │
│  {{payload.repo}}                  │
│  {{headers.x-github-event}}        │
│  {{now}}                           │
└────────────┬───────────────────────┘
             │
             ▼
┌────────────────────────────────────┐
│  Action Dispatch                   │
│  ├─ wake: 唤醒心跳                 │
│  └─ agent: 启动 agent 运行         │
└────────────────────────────────────┘
```

### 配置示例

```json
{
  "hooks": {
    "enabled": true,
    "token": "your-secret-token",
    "path": "/hooks",
    "mappings": [
      {
        "id": "github-push",
        "match": { "path": "github" },
        "action": "agent",
        "messageTemplate": "GitHub push to {{payload.repository.name}}: {{payload.head_commit.message}}",
        "sessionKey": "hook:github:{{payload.repository.name}}",
        "channel": "telegram"
      }
    ],
    "presets": ["gmail"]
  }
}
```

---

## 设计模式

### 1. 依赖注入模式

```typescript
// 通过上下文对象传递依赖
type GatewayRequestContext = {
  deps: CliDeps;
  cron: CronService;
  nodeRegistry: NodeRegistry;
  broadcast: BroadcastFn;
  // ... 其他依赖
};

// 便于测试和模块解耦
function handleRequest(ctx: GatewayRequestContext) {
  ctx.broadcast("event", payload);
}
```

### 2. 插件架构

```typescript
// Gateway 插件扩展方法和事件
type GatewayPlugin = {
  id: string;
  gatewayHandlers?: GatewayRequestHandlers;
  gatewayMethods?: string[];
};

// Channel 插件扩展渠道
type ChannelPlugin = {
  id: string;
  gateway?: {
    startAccount: (ctx) => Promise<void>;
    stopAccount?: (ctx) => Promise<void>;
  };
  outbound: ChannelOutboundAdapter;
};
```

### 3. 状态管理分层

```typescript
// Runtime State: WebSocket 连接
type RuntimeState = {
  httpServer: Server;
  wss: WebSocketServer;
  clients: Set<GatewayWsClient>;
  broadcast: BroadcastFn;
};

// Chat State: Agent 运行状态
type ChatRunState = {
  registry: ChatRunRegistry;
  buffers: Map<string, string>;
  abortControllers: Map<string, AbortController>;
};

// Channel State: 渠道运行时快照
type ChannelRuntimeSnapshot = {
  channels: Record<ChannelId, ChannelAccountSnapshot>;
};
```

### 4. 事件驱动架构

```typescript
// 订阅 Agent 事件
onAgentEvent((event) => {
  broadcast("agent", event);
});

// 订阅心跳事件
onHeartbeatEvent((event) => {
  broadcast("heartbeat", event);
});

// 订阅配置变更
onConfigChange((newCfg) => {
  applyHotReload(newCfg);
});
```

---

## 配置管理

### 配置文件路径

**默认位置**: `~/.openclaw/openclaw.json`

**代码位置**: [`src/config/paths.ts`](../src/config/paths.ts)

```typescript
// 优先级顺序
1. OPENCLAW_CONFIG_PATH 环境变量
2. OPENCLAW_STATE_DIR/openclaw.json
3. ~/.openclaw/openclaw.json
4. ~/.clawdbot/clawdbot.json (旧版兼容)
```

### 热重载机制

**代码位置**: [`src/gateway/config-reload.ts`](../src/gateway/config-reload.ts)

```
文件监听 (fs.watch)
    │
    ▼
检测变更 (hash 对比)
    │
    ▼
┌─────────────────────────────┐
│ 判断重载类型                │
│  ├─ Hot Reload (无需重启)   │
│  └─ Restart Required        │
└──────────┬──────────────────┘
           │
           ▼
    ┌─────────────────┐
    │  Hot Reload     │
    │  - Hooks 配置   │
    │  - Channel 配置 │
    │  - Cron 配置    │
    └─────────────────┘
```

---

## 关键代码入口索引

| 模块 | 文件 | 说明 |
|------|------|------|
| 启动入口 | [`server.impl.ts:155`](../src/gateway/server.impl.ts#L155) | `startGatewayServer()` |
| WebSocket | [`ws-connection.ts:19`](../src/gateway/server/ws-connection.ts#L19) | 连接处理 |
| 协议层 | [`protocol/index.ts`](../src/gateway/protocol/index.ts) | Frame 定义和验证 |
| 方法路由 | [`server-methods.ts:93`](../src/gateway/server-methods.ts#L93) | 授权和分发 |
| 广播系统 | [`server-broadcast.ts:34`](../src/gateway/server-broadcast.ts#L34) | 事件推送 |
| Hooks | [`hooks-mapping.ts:137`](../src/gateway/hooks-mapping.ts#L137) | Webhook 路由 |
| 渠道管理 | [`server-channels.ts:64`](../src/gateway/server-channels.ts#L64) | 渠道生命周期 |
| Agent事件 | [`server-chat.ts`](../src/gateway/server-chat.ts) | Agent 运行状态 |

---

## 性能优化

### 1. WebSocket 慢消费者处理

```typescript
// 检测缓冲区大小
if (socket.bufferedAmount > MAX_BUFFERED_BYTES) {
  if (dropIfSlow) {
    continue; // 跳过此客户端
  } else {
    socket.close(1008, "slow consumer"); // 断开连接
  }
}
```

### 2. 事件去重

```typescript
// 使用 Map 存储已处理的消息 ID
const processedMessages = new Map<string, number>();

function tryRecordMessage(id: string): boolean {
  if (processedMessages.has(id)) return false;
  processedMessages.set(id, Date.now());
  return true;
}
```

### 3. Delta 缓冲

```typescript
// 累积文本增量，批量发送
const buffers = new Map<string, string>();
buffers.set(runId, (buffers.get(runId) || "") + delta);

// 定期刷新
if (Date.now() - lastSentAt > FLUSH_INTERVAL) {
  flush(runId);
}
```

---

## 安全设计

### 1. 多层认证

```typescript
// Token 认证
if (connectAuth.token !== resolvedAuth.token) {
  return { ok: false, error: "invalid token" };
}

// Tailscale 认证
if (resolvedAuth.allowTailscale) {
  const tailscaleAuth = await verifyTailscaleClient(req);
  if (tailscaleAuth.ok) return { ok: true };
}

// 本地直连免认证
if (isLocalDirectRequest(req, trustedProxies)) {
  return { ok: true };
}
```

### 2. Scope 权限控制

```typescript
const APPROVAL_METHODS = new Set(["exec.approval.request", "exec.approval.resolve"]);

function authorizeMethod(method: string, scopes: string[]) {
  if (APPROVAL_METHODS.has(method) && !scopes.includes("operator.approvals")) {
    return errorShape(ErrorCodes.INVALID_REQUEST, "missing scope");
  }
}
```

### 3. Origin 检查 (CSRF 防护)

```typescript
function isOriginAllowed(origin: string, allowedOrigins: string[]): boolean {
  return allowedOrigins.some(pattern => matchOriginPattern(origin, pattern));
}
```

---

## 总结

Gateway 采用**事件驱动 + 插件化 + 分层架构**设计：

1. **高可扩展性**: 插件系统支持无限扩展渠道和功能
2. **高可用性**: 热重载、优雅降级、错误隔离
3. **高性能**: 慢消费者处理、事件去重、Delta 缓冲
4. **高安全性**: 多层认证、Scope 权限、Origin 检查
5. **易维护性**: 依赖注入、状态分层、清晰的模块边界

**设计哲学**: "Everything is a plugin, everything is an event"
