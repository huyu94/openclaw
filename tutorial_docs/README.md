# OpenClaw 项目学习教程

> 本目录包含 OpenClaw 项目的详细技术文档和教程

## 📚 文档列表

### 1. [Gateway 架构设计分析](./01-gateway-architecture.md)
深入剖析 OpenClaw Gateway 的核心架构、设计模式和实现细节。

**包含内容**:
- 整体架构图
- 核心模块详解（WebSocket、HTTP、插件系统）
- 启动流程完整分析
- WebSocket 协议握手与消息流转
- Hooks 系统（Webhook 路由机制）
- 设计模式（依赖注入、插件架构、状态管理）
- 配置管理与热重载
- 性能优化与安全设计

**适合人群**: 想深入了解 Gateway 实现原理的开发者

---

### 2. [飞书渠道集成详解](./02-feishu-integration.md)
完整介绍 OpenClaw 如何集成飞书消息平台作为渠道插件。

**包含内容**:
- 飞书插件架构设计
- 核心模块（监听、处理、发送、回复）
- 消息流转完整流程图
- 工具系统（知识库/文档/云盘/多维表格）
- 配置指南（基本配置、环境变量、完整示例）
- 高级特性（话题隔离、动态 Agent、富文本卡片、权限处理）
- 双连接模式（WebSocket vs Webhook）

**适合人群**: 想集成新渠道或了解渠道插件机制的开发者

---

### 3. [TypeScript 基础语法详解](./03-typescript-basics.md)
通过项目实际代码，讲解 TypeScript 常用语法。

**包含内容**:
- 可选链操作符 `?.`（安全访问嵌套属性）
- 空值合并操作符 `??`（精确的默认值处理）
- 数组方法 `map`（数组转换）
- 箭头函数（简洁函数写法）
- 类型系统（基本类型、接口、泛型）
- 异步编程（Promise、async/await）
- 解构赋值（对象/数组解构）
- 模板字符串（字符串插值）

**适合人群**: TypeScript 初学者或需要语法参考的开发者

---

## 🎯 学习路径建议

### 对于新手

1. **先学 TypeScript 基础** → [03-typescript-basics.md](./03-typescript-basics.md)
   - 了解项目中常用的语法
   - 能看懂代码的基本结构

2. **再看 Gateway 架构** → [01-gateway-architecture.md](./01-gateway-architecture.md)
   - 理解项目的核心设计
   - 掌握整体架构思路

3. **最后研究具体实现** → [02-feishu-integration.md](./02-feishu-integration.md)
   - 通过飞书案例学习插件机制
   - 了解如何扩展新功能

### 对于有经验的开发者

1. **直接看架构** → [01-gateway-architecture.md](./01-gateway-architecture.md)
   - 快速了解设计模式
   - 找到关键代码入口

2. **研究插件机制** → [02-feishu-integration.md](./02-feishu-integration.md)
   - 学习如何集成新渠道
   - 参考工具开发模式

3. **需要时查语法** → [03-typescript-basics.md](./03-typescript-basics.md)
   - 作为语法速查手册
   - 了解项目编码风格

---

## 🔍 快速查找

### 按功能查找

| 功能 | 文档位置 |
|------|---------|
| Gateway 启动流程 | [01-gateway-architecture.md § 启动流程](./01-gateway-architecture.md#启动流程) |
| WebSocket 握手 | [01-gateway-architecture.md § WebSocket 协议](./01-gateway-architecture.md#websocket-协议) |
| Hooks 路由 | [01-gateway-architecture.md § Hooks 系统](./01-gateway-architecture.md#hooks-系统) |
| 飞书消息监听 | [02-feishu-integration.md § 消息监听](./02-feishu-integration.md#3-消息监听-monitorts) |
| 飞书工具开发 | [02-feishu-integration.md § 工具系统](./02-feishu-integration.md#工具系统) |
| 可选链语法 | [03-typescript-basics.md § 可选链操作符](./03-typescript-basics.md#可选链操作符-) |
| 异步编程 | [03-typescript-basics.md § 异步编程](./03-typescript-basics.md#异步编程) |

### 按文件查找

| 源文件 | 对应文档 |
|--------|---------|
| `src/gateway/server.impl.ts` | [01-gateway-architecture.md](./01-gateway-architecture.md) |
| `src/gateway/hooks-mapping.ts` | [01-gateway-architecture.md § Hooks 系统](./01-gateway-architecture.md#hooks-系统) |
| `extensions/feishu/src/channel.ts` | [02-feishu-integration.md § 渠道插件定义](./02-feishu-integration.md#2-渠道插件定义-channelts) |
| `extensions/feishu/src/bot.ts` | [02-feishu-integration.md § 消息处理核心](./02-feishu-integration.md#4-消息处理核心-botts) |
| `extensions/feishu/src/wiki.ts` | [02-feishu-integration.md § 知识库工具](./02-feishu-integration.md#1-知识库工具-feishu_wiki) |

---

## 💡 问答集

### Q1: 配置文件在哪里？
**A**: 默认位置是 `~/.openclaw/openclaw.json`，详见 [01-gateway-architecture.md § 配置管理](./01-gateway-architecture.md#配置管理)

### Q2: 如何添加新的消息渠道？
**A**: 参考飞书插件的实现，详见 [02-feishu-integration.md](./02-feishu-integration.md)

### Q3: 如何理解这段代码？
```typescript
const spaces = res.data?.items?.map((s) => ({ ... })) ?? [];
```
**A**: 详细解析见 [03-typescript-basics.md § 可选链操作符](./03-typescript-basics.md#可选链操作符-)

### Q4: Gateway 如何启动？
**A**: 详见 [01-gateway-architecture.md § 启动流程](./01-gateway-architecture.md#启动流程)

### Q5: 飞书如何接收消息？
**A**: 支持 WebSocket 和 Webhook 两种模式，详见 [02-feishu-integration.md § 消息监听](./02-feishu-integration.md#3-消息监听-monitorts)

---

## 🔗 相关链接

### 官方文档
- [OpenClaw 官方文档](https://docs.openclaw.ai/)
- [飞书开放平台](https://open.feishu.cn/)
- [TypeScript 官方文档](https://www.typescriptlang.org/)

### 项目资源
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [Discord 社区](https://discord.gg/openclaw)

### 技术参考
- [WebSocket 协议](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSockets_API)
- [JSON Schema](https://json-schema.org/)
- [Node.js 文档](https://nodejs.org/docs/)

---

## 📝 文档更新记录

| 日期 | 版本 | 更新内容 |
|------|------|---------|
| 2026-02-25 | v1.0 | 初始版本，包含 Gateway 架构、飞书集成、TypeScript 基础三份文档 |

---

## 🤝 贡献指南

发现文档有误或需要补充？欢迎提交 PR 或 Issue！

**文档编写规范**:
1. 使用 Markdown 格式
2. 代码示例必须可运行或来自项目实际代码
3. 包含清晰的目录结构
4. 添加代码文件链接（格式：`[文件名](../path/to/file.ts)`）
5. 配图使用 ASCII 艺术或 Mermaid 图表

---

## 📄 许可证

本教程文档遵循与 OpenClaw 项目相同的许可证。

---

**Happy Learning! 🚀**
