# OpenClaw 如何使用 Pi 框架

## 概述

OpenClaw 将 Pi 框架深度集成到其核心，作为与 AI 模型交互的"大脑"。本文档详细解释 OpenClaw 如何使用 Pi 的各个组件。

## 1. Pi 组件的导入和使用

### 1.1 核心导入（来自 [`src/agents/pi-embedded-runner/run/attempt.ts`](../src/agents/pi-embedded-runner/run/attempt.ts:1-11)）

```typescript
// Pi 的核心组件
import type { AgentMessage } from "@mariozechner/pi-agent-core";
import type { ImageContent } from "@mariozechner/pi-ai";
import { streamSimple } from "@mariozechner/pi-ai";
import {
  createAgentSession,
  DefaultResourceLoader,
  SessionManager,
  SettingsManager,
} from "@mariozechner/pi-coding-agent";
```

**说明**：
- **pi-agent-core**：提供 `AgentMessage` 等基础类型
- **pi-ai**：提供 `streamSimple` 流式调用 LLM、`ImageContent` 处理图片
- **pi-coding-agent**：提供 `SessionManager`（会话管理）、`createAgentSession`（创建会话）

### 1.2 工具系统导入（来自 [`src/agents/pi-tools.ts`](../src/agents/pi-tools.ts:1-7)）

```typescript
import {
  codingTools,
  createEditTool,
  createReadTool,
  createWriteTool,
  readTool,
} from "@mariozechner/pi-coding-agent";
```

**说明**：
- Pi 提供了基础的 4 个工具：Read、Write、Edit、Bash
- OpenClaw 在此基础上包装和扩展

### 1.3 模型发现（来自 [`src/agents/pi-embedded-runner/model.ts`](../src/agents/pi-embedded-runner/model.ts:1-15)）

```typescript
import type { Api, Model } from "@mariozechner/pi-ai";
import {
  discoverAuthStorage,
  discoverModels,
  type AuthStorage,
  type ModelRegistry,
} from "../pi-model-discovery.js";
```

**说明**：
- `AuthStorage`：管理 API 密钥
- `ModelRegistry`：注册和查询可用模型

## 2. OpenClaw 使用 Pi 的完整流程

### 流程图

```
用户消息（Feishu）
    ↓
Channel Plugin 接收
    ↓
Gateway 路由
    ↓
┌─────────────────────────────────────────────────────────┐
│ OpenClaw Agent Command (src/commands/agent.ts)          │
│                                                          │
│ 1. 加载配置                                               │
│ 2. 解析 sessionKey                                       │
│ 3. 调用 runEmbeddedPiAgent()  ←── 关键入口              │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ Pi Embedded Runner (src/agents/pi-embedded-runner/)     │
│                                                          │
│ run.ts: 主运行逻辑                                        │
│  ├─ resolveModel() - 解析模型                            │
│  ├─ createOpenClawCodingTools() - 创建工具集             │
│  ├─ runEmbeddedAttempt() - 执行 Agent                   │
│  └─ subscribeEmbeddedPiSession() - 订阅事件              │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ Pi Framework Components                                  │
│                                                          │
│ [SessionManager] - 管理对话历史                           │
│     ↓                                                    │
│ [pi-ai.streamSimple()] - 调用 LLM                        │
│     ↓                                                    │
│ [Tool Execution] - 执行工具（read/write/exec）            │
│     ↓                                                    │
│ [Response Streaming] - 流式返回                           │
└─────────────────────────────────────────────────────────┘
    ↓
AI 响应 → Gateway → Channel Plugin → Feishu
```

## 3. 关键步骤详解

### 步骤 1：入口 - `runEmbeddedPiAgent()`

在 [`src/commands/agent.ts:26`](../src/commands/agent.ts:26) 中导入：

```typescript
import { runEmbeddedPiAgent } from "../agents/pi-embedded.js";
```

这是 OpenClaw 调用 Pi 框架的主入口。

### 步骤 2：模型解析 - `resolveModel()`

在 [`src/agents/pi-embedded-runner/run.ts:294-299`](../src/agents/pi-embedded-runner/run.ts:294-299) 中：

```typescript
const { model, error, authStorage, modelRegistry } = resolveModel(
  provider,
  modelId,
  agentDir,
  params.config,
);
```

**内部实现**（[`src/agents/pi-embedded-runner/model.ts:46-60`](../src/agents/pi-embedded-runner/model.ts:46-60)）：

```typescript
export function resolveModel(provider: string, modelId: string, agentDir?: string, cfg?: OpenClawConfig) {
  const resolvedAgentDir = agentDir ?? resolveOpenClawAgentDir();
  
  // 使用 Pi 的模型发现机制
  const authStorage = discoverAuthStorage(resolvedAgentDir);
  const modelRegistry = discoverModels(authStorage, resolvedAgentDir);
  
  // 从 Pi 的 ModelRegistry 查找模型
  const model = modelRegistry.find(provider, modelId) as Model<Api> | null;
  
  // 返回模型对象
  return { model, authStorage, modelRegistry };
}
```

**作用**：
1. 使用 Pi 的 `AuthStorage` 管理 API 密钥
2. 使用 Pi 的 `ModelRegistry` 查找模型定义
3. 返回 Pi 的 `Model` 对象供后续使用

### 步骤 3：创建工具 - `createOpenClawCodingTools()`

在 [`src/agents/pi-tools.ts:1-7`](../src/agents/pi-tools.ts:1-7) 中：

```typescript
// 从 Pi 导入基础工具
import {
  codingTools,
  createEditTool,
  createReadTool,
  createWriteTool,
  readTool,
} from "@mariozechner/pi-coding-agent";

// OpenClaw 的增强版本
export function createOpenClawCodingTools(config, sandbox) {
  // 1. 创建 Pi 的基础工具
  const readTool = createReadTool();
  const writeTool = createWriteTool();
  const editTool = createEditTool();
  
  // 2. OpenClaw 的包装增强
  return [
    wrapToolWithBeforeToolCallHook(     // 执行前钩子
      wrapToolWithAbortSignal(          // 中止信号
        wrapToolWorkspaceRootGuard(     // 工作区保护
          readTool
        )
      )
    ),
    // ... 其他工具
  ];
}
```

**增强功能**：
- **Sandbox 隔离**：限制文件访问范围
- **权限策略**：工具白名单/黑名单
- **执行钩子**：`before_tool_call` 钩子
- **中止控制**：支持取消正在执行的工具
- **结果截断**：防止超大响应

### 步骤 4：创建会话 - `SessionManager`

在 [`src/agents/pi-embedded-runner/run/attempt.ts:7-11`](../src/agents/pi-embedded-runner/run/attempt.ts:7-11) 中：

```typescript
import {
  createAgentSession,
  DefaultResourceLoader,
  SessionManager,
  SettingsManager,
} from "@mariozechner/pi-coding-agent";

// 创建或加载会话
const sessionManager = await SessionManager.create(sessionFile);

// 或创建新会话
const session = await createAgentSession({
  sessionDir: agentDir,
  sessionId: params.sessionId,
});
```

**SessionManager 的作用**：
- 管理对话历史（`getHistory()`）
- 添加消息（`addMessage()`）
- 压缩历史（`compact()`）
- 持久化到文件系统

### 步骤 5：流式调用 LLM - `streamSimple()`

在 [`src/agents/pi-embedded-runner/run/attempt.ts:5`](../src/agents/pi-embedded-runner/run/attempt.ts:5) 中：

```typescript
import { streamSimple } from "@mariozechner/pi-ai";

// 调用 LLM
const stream = streamSimple({
  model: resolvedModel,
  api: apiConfig,
  messages: sessionManager.getHistory(),
  tools: tools,
  systemPrompt: systemPrompt,
});

// 处理流式响应
for await (const event of stream) {
  if (event.type === "text_delta") {
    // 处理文本增量
  } else if (event.type === "tool_call") {
    // 执行工具
  }
}
```

**Pi 的 streamSimple 功能**：
- 统一多个 LLM 提供商（Anthropic、OpenAI、Google）
- 流式返回（实时显示）
- 工具调用处理
- 错误处理和重试

### 步骤 6：订阅会话事件 - `subscribeEmbeddedPiSession()`

在 [`src/agents/pi-embedded-subscribe.ts:34`](../src/agents/pi-embedded-subscribe.ts:34) 中：

```typescript
export function subscribeEmbeddedPiSession(params: SubscribeEmbeddedPiSessionParams) {
  // 订阅 Pi 的会话事件
  const state = {
    assistantTexts: [],    // 累积的 AI 回复
    toolMetas: [],         // 工具执行元数据
    blockReplyBreak: params.blockReplyBreak ?? "text_end",
    reasoningMode: params.reasoningMode ?? "off",
    // ...
  };
  
  // 返回事件处理器
  return (event) => {
    if (event.type === "text_delta") {
      // 处理文本流
      state.assistantTexts.push(event.text);
      params.onBlockReply?.(event.text);
    } else if (event.type === "tool_execution_start") {
      // 工具开始执行
      params.onToolStart?.(event.toolName);
    } else if (event.type === "tool_execution_end") {
      // 工具执行完成
      params.onToolEnd?.(event.result);
    }
  };
}
```

**作用**：
- 将 Pi 的底层事件转换为 OpenClaw 的事件
- 实现流式文本输出
- 处理工具执行生命周期
- 支持推理模式（Reasoning）

## 4. 核心概念的使用

### 4.1 SessionManager（会话管理）

**Pi 提供的能力**：
```typescript
class SessionManager {
  getHistory(): AgentMessage[]     // 获取历史
  addMessage(msg: AgentMessage)    // 添加消息
  async compact()                  // 压缩历史
  async save()                     // 保存到磁盘
}
```

**OpenClaw 的使用**：
```typescript
// 1. 加载现有会话
const sessionManager = await SessionManager.create(sessionFile);

// 2. 添加用户消息
sessionManager.addMessage({
  role: "user",
  content: "帮我分析 src/app.ts"
});

// 3. 调用 LLM
const response = await streamSimple({
  messages: sessionManager.getHistory(),
  // ...
});

// 4. 添加 AI 响应
sessionManager.addMessage({
  role: "assistant",
  content: response.text
});

// 5. 如果历史太长，自动压缩
if (sessionManager.getHistory().length > 100) {
  await sessionManager.compact();
}
```

### 4.2 Tool Execution（工具执行）

**Pi 的工具接口**：
```typescript
interface AgentTool {
  name: string;
  description: string;
  parameters: JSONSchema;
  execute(args: any, context: any): Promise<AgentToolResult>;
}
```

**OpenClaw 的增强**：
```typescript
// 1. Pi 的基础工具
const baseTool = createReadTool();

// 2. OpenClaw 包装
const enhancedTool = {
  ...baseTool,
  execute: async (args, context) => {
    // 前置检查（权限、路径等）
    if (!isPathAllowed(args.path)) {
      throw new Error("Access denied");
    }
    
    // 调用 Pi 的工具
    const result = await baseTool.execute(args, context);
    
    // 后置处理（截断、格式化等）
    return truncateResult(result);
  }
};
```

### 4.3 Model Selection（模型选择）

**Pi 的 ModelRegistry**：
```typescript
const authStorage = discoverAuthStorage(agentDir);
const modelRegistry = discoverModels(authStorage, agentDir);

// 查找模型
const model = modelRegistry.find("anthropic", "claude-opus-4-6");
```

**OpenClaw 的增强**：
```typescript
// 1. 支持别名
const model = resolveModel("opus", undefined, agentDir, config);
// "opus" 别名映射到 "anthropic/claude-opus-4-6"

// 2. 支持 Fallback
const models = [
  { provider: "anthropic", model: "claude-opus-4-6" },
  { provider: "openai", model: "gpt-5.3-codex" },
];
// 如果第一个模型失败，自动切换到第二个

// 3. 支持自定义提供商
// 可以在 config 中定义新的提供商，不需要修改 Pi
```

### 4.4 Compaction（压缩）

**Pi 的压缩机制**：
```typescript
// 当历史太长时
await sessionManager.compact();

// Pi 会：
// 1. 调用 LLM 生成历史摘要
// 2. 保留最近的消息
// 3. 用摘要替换旧消息
```

**OpenClaw 的增强**：
```typescript
// 自动触发压缩
if (isLikelyContextOverflowError(error)) {
  await compactEmbeddedPiSessionDirect(sessionManager);
  // 压缩后重试
}

// 配置压缩策略
const compactionSettings = {
  reserveTokens: 8000,        // 保留多少 token
  keepRecentTokens: 4000,     // 保留最近多少 token
  mode: "auto"                // 自动/手动
};
```

## 5. OpenClaw 对 Pi 的扩展点

### 5.1 Hooks 系统

```typescript
// Pi 原始：只有工具执行
await tool.execute(args, context);

// OpenClaw 增强：生命周期钩子
hookRunner.runBeforeToolCall({ toolName, args });
await tool.execute(args, context);
hookRunner.runAfterToolCall({ toolName, result });
```

### 5.2 多渠道适配

```typescript
// Pi 原始：终端输出
console.log(response.text);

// OpenClaw：多渠道分发
if (channel === "feishu") {
  await sendFeishuMessage(response.text);
} else if (channel === "discord") {
  await sendDiscordMessage(response.text);
}
```

### 5.3 并发管理

```typescript
// Pi 原始：单会话
const session = await SessionManager.create(sessionFile);

// OpenClaw：多会话并发
const sessionKey = `${channel}:${userId}:${roomId}`;
const session = await getOrCreateSession(sessionKey);
```

### 5.4 权限控制

```typescript
// Pi 原始：无限制
await tool.execute({ path: "/etc/passwd" });

// OpenClaw：Pairing + 白名单
if (!isPaired(userId) || !isToolAllowed(toolName, userId)) {
  throw new Error("Access denied");
}
```

## 6. 实际调用链

当用户在 Feishu 发消息"帮我分析 src/app.ts"：

```typescript
// 1. Channel Plugin 接收
feishuPlugin.onMessage(message) {
  // 2. Gateway 路由
  gateway.routeMessage({
    channel: "feishu",
    userId: "user123",
    roomId: "room456",
    text: "帮我分析 src/app.ts"
  });
}

// 3. Agent Command
runAgentCommand({
  prompt: "帮我分析 src/app.ts",
  sessionKey: "feishu:user123:room456"
}) {
  // 4. Pi Embedded Runner
  const result = await runEmbeddedPiAgent({
    prompt: "帮我分析 src/app.ts",
    sessionId: generateId(),
    sessionKey: "feishu:user123:room456",
    
    // 5. 解析模型（使用 Pi 的 ModelRegistry）
    provider: "anthropic",
    model: "claude-opus-4-6",
    
    // 6. 创建工具（基于 Pi 的工具）
    tools: createOpenClawCodingTools(),
    
    // 7. 加载会话（使用 Pi 的 SessionManager）
    sessionFile: "~/.openclaw/sessions/feishu:user123:room456.json"
  });
  
  // 8. Pi Framework 处理
  const sessionManager = await SessionManager.create(sessionFile);
  sessionManager.addMessage({
    role: "user",
    content: "帮我分析 src/app.ts"
  });
  
  // 9. 调用 LLM（使用 Pi 的 streamSimple）
  const stream = streamSimple({
    model: resolvedModel,
    messages: sessionManager.getHistory(),
    tools: tools
  });
  
  // 10. LLM 返回：需要调用 read_file 工具
  for await (const event of stream) {
    if (event.type === "tool_call" && event.name === "read_file") {
      // 11. 执行工具（Pi 的工具 + OpenClaw 的包装）
      const result = await tools.read_file.execute({
        path: "src/app.ts"
      });
      
      // 12. 返回结果给 LLM
      sessionManager.addMessage({
        role: "tool",
        toolCallId: event.id,
        content: result
      });
    } else if (event.type === "text_delta") {
      // 13. 流式返回文本
      onBlockReply(event.text);
    }
  }
  
  // 14. 返回完整响应
  return {
    text: "根据代码分析，这是一个 Express 应用..."
  };
}

// 15. Gateway 发送回复
gateway.sendMessage({
  channel: "feishu",
  userId: "user123",
  roomId: "room456",
  text: result.text
});
```

## 7. 总结

### OpenClaw 使用 Pi 的方式

| Pi 组件 | OpenClaw 使用方式 | 增强内容 |
|---------|------------------|---------|
| **SessionManager** | 管理对话历史 | + 多会话并发 + 自动清理 |
| **streamSimple** | 调用 LLM | + Fallback + 重试 + 钩子 |
| **工具系统** | Read/Write/Edit/Bash | + 20+ 工具 + 权限控制 + Sandbox |
| **ModelRegistry** | 模型发现 | + 别名 + 自定义提供商 + 前向兼容 |
| **Compaction** | 历史压缩 | + 自动触发 + 配置策略 |

### 核心关系

```
OpenClaw = Pi Framework（核心引擎）+ OpenClaw Extensions（企业增强）

Pi Framework 提供：
- SessionManager（会话管理）
- streamSimple（LLM 调用）
- 基础工具（Read/Write/Edit/Bash）
- ModelRegistry（模型管理）

OpenClaw 增强：
- 多渠道支持（Feishu/Discord/Telegram...）
- 并发管理（多用户多会话）
- 权限控制（Pairing + 白名单）
- Hooks 系统（生命周期钩子）
- 插件系统（Channel Plugins）
```

### 关键文件

| 文件 | 作用 |
|-----|------|
| [`src/agents/pi-embedded.ts`](../src/agents/pi-embedded.ts) | Pi 封装入口 |
| [`src/agents/pi-embedded-runner/run.ts`](../src/agents/pi-embedded-runner/run.ts) | 主运行逻辑 |
| [`src/agents/pi-embedded-runner/model.ts`](../src/agents/pi-embedded-runner/model.ts) | 模型解析 |
| [`src/agents/pi-embedded-runner/run/attempt.ts`](../src/agents/pi-embedded-runner/run/attempt.ts) | Agent 执行 |
| [`src/agents/pi-tools.ts`](../src/agents/pi-tools.ts) | 工具系统 |
| [`src/agents/pi-embedded-subscribe.ts`](../src/agents/pi-embedded-subscribe.ts) | 事件订阅 |
| [`src/commands/agent.ts`](../src/commands/agent.ts) | CLI 命令入口 |

**核心要点**：OpenClaw 将 Pi 框架作为"大脑"，在其基础上构建了完整的多渠道 AI 网关。Pi 提供核心的 Agent 能力（会话管理、LLM 调用、工具执行），OpenClaw 提供企业级功能（多渠道、并发、权限、插件）。
