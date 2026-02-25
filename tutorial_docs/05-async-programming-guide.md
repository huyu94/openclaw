# JavaScript/TypeScript 异步编程指南

## 问题 1: 钉钉是否有 WebSocket 连接?

### 答案
OpenClaw 项目中**目前没有钉钉(DingTalk)集成**。从扩展列表中可以看到:

**已支持的消息平台**:
- ✅ Feishu/Lark (飞书) - 支持 WebSocket 和 Webhook
- ✅ Telegram (电报)
- ✅ WhatsApp
- ✅ Discord
- ✅ Slack
- ✅ Signal
- ✅ IRC
- ✅ Matrix
- ✅ Microsoft Teams
- ❌ **钉钉(DingTalk)** - 未集成

### 钉钉开放平台的连接方式

如果要集成钉钉,根据钉钉开放平台文档:

**钉钉支持的连接方式**:
1. **Stream 模式** (类似 WebSocket 长连接)
   - 使用 HTTP/2 Server-Sent Events (SSE)
   - 实时接收事件推送
   - 无需公网 IP

2. **HTTP 回调模式** (类似 Webhook)
   - 钉钉推送事件到你的服务器
   - 需要公网可访问的 HTTPS 地址
   - 需要配置回调 URL

所以钉钉**没有传统的 WebSocket**,但有类似功能的 **Stream 模式**。

---

## 问题 2: 什么时候使用异步(async/await)?

## 异步 vs 同步的核心区别

### 同步操作 (Synchronous)

```typescript
// 同步: 代码按顺序执行,一行执行完才执行下一行
function calculateSum(a: number, b: number): number {
  const result = a + b;  // 立即完成
  return result;
}

const sum = calculateSum(1, 2);  // 立即得到结果
console.log(sum);  // 3
```

**特点**:
- ⚡ 操作立即完成(通常 < 1ms)
- 🔒 阻塞执行(必须等待完成)
- 💻 只涉及 CPU 和内存

### 异步操作 (Asynchronous)

```typescript
// 异步: 操作可能需要时间,不阻塞后续代码执行
async function fetchUserData(userId: string): Promise<User> {
  const response = await fetch(`https://api.example.com/users/${userId}`);
  const data = await response.json();
  return data;
}

// 调用异步函数
const user = await fetchUserData("123");  // 可能需要 100ms - 几秒
console.log(user);
```

**特点**:
- 🐌 操作需要时间(通常 > 10ms)
- 🔓 非阻塞(可以同时做其他事)
- 🌐 涉及 I/O 操作(网络、磁盘、数据库)

---

## 必须使用 async/await 的场景

### 1. 网络请求 (Network I/O)

```typescript
// ✅ 正确: 使用 async/await
async function sendMessage(text: string) {
  const response = await fetch("https://api.feishu.cn/message/send", {
    method: "POST",
    body: JSON.stringify({ text })
  });
  return await response.json();
}

// ❌ 错误: 不能同步化网络请求
function sendMessageSync(text: string) {
  // 这样写是错的!网络请求必须是异步的
  const response = fetch(url);  // 返回 Promise,不是实际数据
  return response.json();  // 错误!还是 Promise
}
```

**原因**: 网络请求需要等待服务器响应,可能需要几百毫秒到几秒。

### 2. 文件操作 (File I/O)

```typescript
import fs from "node:fs/promises";

// ✅ 正确: 异步读取文件
async function readConfig(): Promise<Config> {
  const content = await fs.readFile("./config.json", "utf8");
  return JSON.parse(content);
}

// ❌ 危险: 同步读取(阻塞主线程)
function readConfigSync(): Config {
  const content = fs.readFileSync("./config.json", "utf8");
  return JSON.parse(content);  // 如果文件大,会卡住整个程序
}
```

**原因**: 磁盘 I/O 比内存慢 10000 倍,大文件可能需要几秒。

### 3. 数据库查询 (Database Operations)

```typescript
// ✅ 正确: 异步查询
async function getUserById(id: string): Promise<User | null> {
  const user = await db.users.findOne({ id });
  return user;
}

// ❌ 错误: 数据库没有同步 API
function getUserByIdSync(id: string): User | null {
  // 无法同步查询数据库!
}
```

**原因**: 数据库查询涉及网络通信或磁盘访问,必须异步。

### 4. 定时器和延迟 (Timers)

```typescript
// ✅ 正确: 异步延迟
async function retryWithDelay(fn: () => Promise<void>, delayMs: number) {
  await new Promise(resolve => setTimeout(resolve, delayMs));
  await fn();
}

// ❌ 错误: 没有"同步的等待"
function sleep(ms: number) {
  // 这样写不会等待!
  setTimeout(() => {}, ms);  // 立即返回,不会阻塞
}
```

**原因**: JavaScript 是单线程的,定时器必须通过事件循环实现。

### 5. 外部服务调用 (External Services)

```typescript
// ✅ 正确: 调用飞书 API
async function sendFeishuMessage(cfg: Config, to: string, text: string) {
  const client = createFeishuClient(cfg);
  const result = await client.im.message.create({
    receive_id: to,
    msg_type: "text",
    content: JSON.stringify({ text })
  });
  return result;
}
```

**原因**: 调用第三方 API 需要网络请求,必须异步。

---

## 应该保持同步的场景

### 1. 纯计算操作 (Pure Computation)

```typescript
// ✅ 保持同步
function calculateTotal(items: number[]): number {
  return items.reduce((sum, item) => sum + item, 0);
}

// ❌ 不要无谓地异步化
async function calculateTotalAsync(items: number[]): Promise<number> {
  return items.reduce((sum, item) => sum + item, 0);  // 完全没必要!
}
```

**原因**: 纯计算不涉及 I/O,立即完成,异步化只会增加复杂度。

### 2. 数据转换 (Data Transformation)

```typescript
// ✅ 保持同步
function normalizePhoneNumber(phone: string): string {
  return phone.replace(/[^0-9]/g, "");
}

// ✅ 保持同步
function formatTimestamp(timestamp: number): string {
  return new Date(timestamp).toISOString();
}
```

**原因**: 字符串操作和简单数据处理是同步的,非常快。

### 3. 对象创建和配置 (Object Creation)

```typescript
// ✅ 保持同步
function createConfig(options: ConfigOptions): Config {
  return {
    apiKey: options.apiKey,
    endpoint: options.endpoint ?? "https://api.default.com",
    timeout: options.timeout ?? 30000,
  };
}
```

**原因**: 对象创建是内存操作,立即完成。

### 4. 简单的条件判断 (Simple Logic)

```typescript
// ✅ 保持同步
function isValidEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// ✅ 保持同步
function shouldRetry(attempt: number, maxAttempts: number): boolean {
  return attempt < maxAttempts;
}
```

**原因**: 逻辑判断不涉及 I/O,瞬间完成。

---

## 判断标准速查表

| 操作类型 | 使用 async? | 原因 |
|---------|-----------|------|
| **网络请求** (fetch, API 调用) | ✅ 必须 | 等待服务器响应 |
| **文件读写** (fs.readFile, fs.writeFile) | ✅ 推荐 | 磁盘 I/O 较慢 |
| **数据库操作** (查询、插入、更新) | ✅ 必须 | 涉及网络或磁盘 |
| **定时器** (setTimeout, setInterval) | ✅ 必须 | 需要等待时间 |
| **WebSocket/Stream** (连接、监听) | ✅ 必须 | 长期等待事件 |
| **加密/解密** (大数据) | ✅ 推荐 | CPU 密集,可能卡住 |
| **纯计算** (数学运算) | ❌ 保持同步 | 立即完成 |
| **字符串处理** (split, replace) | ❌ 保持同步 | 内存操作,极快 |
| **对象操作** (创建、读取属性) | ❌ 保持同步 | 内存操作 |
| **条件判断** (if, switch) | ❌ 保持同步 | CPU 操作 |
| **数组操作** (map, filter, reduce) | ❌ 保持同步* | 除非回调是异步的 |

**注**: 如果数组操作的回调函数是异步的,需要使用 `Promise.all`。

---

## OpenClaw 中的实际例子

### 例子 1: 飞书消息发送 (必须异步)

```typescript
// extensions/feishu/src/send.ts
export async function sendMessageFeishu(params: {
  cfg: Config;
  to: string;
  text: string;
}): Promise<SendResult> {
  // 1. 创建客户端 (同步)
  const client = createFeishuClient(params.cfg);
  
  // 2. 调用 API (异步 - 网络请求)
  const response = await client.im.message.create({
    receive_id: params.to,
    msg_type: "text",
    content: JSON.stringify({ text: params.text })
  });
  
  // 3. 检查结果 (同步)
  if (response.code !== 0) {
    throw new Error(response.msg);
  }
  
  // 4. 返回结果 (同步)
  return {
    success: true,
    messageId: response.data?.message_id
  };
}
```

**为什么是异步?** 因为 `client.im.message.create()` 需要通过网络发送 HTTP 请求到飞书服务器。

### 例子 2: 配对码规范化 (保持同步)

```typescript
// src/pairing/pairing-store.ts
function normalizeAllowEntry(channel: string, entry: string): string {
  // 1. 去除空格 (同步)
  const trimmed = entry.trim();
  
  // 2. 检查通配符 (同步)
  if (trimmed === "*") {
    return "";
  }
  
  // 3. 获取适配器 (同步)
  const adapter = getPairingAdapter(channel);
  
  // 4. 调用规范化函数 (同步)
  const normalized = adapter?.normalizeAllowEntry 
    ? adapter.normalizeAllowEntry(trimmed) 
    : trimmed;
  
  // 5. 返回结果 (同步)
  return String(normalized).trim();
}
```

**为什么是同步?** 所有操作都是字符串处理和简单的函数调用,不涉及 I/O。

### 例子 3: 读取配对请求 (必须异步)

```typescript
// src/pairing/pairing-store.ts
async function readPairingRequests(filePath: string): Promise<PairingRequest[]> {
  // 读取 JSON 文件 (异步 - 文件 I/O)
  const { value } = await readJsonFile<PairingStore>(filePath, {
    version: 1,
    requests: [],
  });
  
  // 验证和返回 (同步)
  return Array.isArray(value.requests) ? value.requests : [];
}
```

**为什么是异步?** 因为 `readJsonFile` 需要从磁盘读取文件。

---

## async/await 的最佳实践

### 1. 并行执行多个异步操作

```typescript
// ❌ 不好: 串行执行(慢)
async function fetchAllUsers() {
  const user1 = await fetchUser("1");  // 等待 100ms
  const user2 = await fetchUser("2");  // 等待 100ms
  const user3 = await fetchUser("3");  // 等待 100ms
  return [user1, user2, user3];
  // 总时间: 300ms
}

// ✅ 好: 并行执行(快)
async function fetchAllUsersParallel() {
  const [user1, user2, user3] = await Promise.all([
    fetchUser("1"),  // 同时发起
    fetchUser("2"),  // 同时发起
    fetchUser("3"),  // 同时发起
  ]);
  return [user1, user2, user3];
  // 总时间: 100ms (最慢的那个)
}
```

### 2. 错误处理

```typescript
// ✅ 使用 try-catch
async function sendMessageSafely(text: string) {
  try {
    const result = await sendMessage(text);
    return { success: true, result };
  } catch (error) {
    console.error("发送失败:", error);
    return { success: false, error };
  }
}
```

### 3. 避免"async 地狱"

```typescript
// ❌ 不好: 不必要的嵌套
async function processUser(userId: string) {
  return await getUser(userId).then(async (user) => {
    return await updateUser(user.id, { lastSeen: Date.now() }).then(async (updated) => {
      return await sendNotification(updated.email);
    });
  });
}

// ✅ 好: 扁平化
async function processUserBetter(userId: string) {
  const user = await getUser(userId);
  const updated = await updateUser(user.id, { lastSeen: Date.now() });
  return await sendNotification(updated.email);
}
```

### 4. 顶层 await (Top-level await)

```typescript
// ✅ 在 ESM 模块中可以直接使用 await
const config = await loadConfig();
const client = createClient(config);

// 不需要包装在 async 函数中
```

---

## 常见误区

### 误区 1: 过度使用 async

```typescript
// ❌ 错误: 没有异步操作,不需要 async
async function add(a: number, b: number): Promise<number> {
  return a + b;  // 同步操作,不需要 async!
}

// ✅ 正确: 保持同步
function add(a: number, b: number): number {
  return a + b;
}
```

### 误区 2: 忘记 await

```typescript
// ❌ 错误: 忘记 await
async function sendTwoMessages() {
  sendMessage("Hello");   // 返回 Promise,不会等待!
  sendMessage("World");   // 返回 Promise,不会等待!
  // 函数立即结束,消息可能还没发出去
}

// ✅ 正确: 使用 await
async function sendTwoMessages() {
  await sendMessage("Hello");   // 等待第一条发送完成
  await sendMessage("World");   // 再发送第二条
}
```

### 误区 3: 在构造函数中使用 async

```typescript
// ❌ 错误: 构造函数不能是 async
class Client {
  constructor() {
    await this.connect();  // 语法错误!
  }
}

// ✅ 正确: 使用静态工厂方法
class Client {
  private constructor() {}
  
  static async create(): Promise<Client> {
    const client = new Client();
    await client.connect();
    return client;
  }
  
  private async connect() {
    // 异步连接逻辑
  }
}

// 使用
const client = await Client.create();
```

---

## 总结

### 何时使用 async/await?

**简单判断规则**:

```
是否涉及以下任何一项?
├─ 网络请求? ────→ ✅ 使用 async
├─ 文件操作? ────→ ✅ 使用 async
├─ 数据库查询? ──→ ✅ 使用 async
├─ 定时器/延迟? ─→ ✅ 使用 async
├─ 第三方 API? ──→ ✅ 使用 async
└─ 纯计算/转换? ─→ ❌ 保持同步
```

### 核心原则

1. **I/O 操作必须异步** - 网络、文件、数据库
2. **纯计算保持同步** - 不要无谓地添加 async
3. **并行优于串行** - 多个独立操作用 `Promise.all`
4. **正确处理错误** - 使用 try-catch
5. **保持代码扁平** - 避免深层嵌套

### 记住这个口诀

```
I/O 慢，必须 async
计算快，同步就好
```

异步编程的本质是**让程序在等待慢速操作时不被阻塞**,可以同时做其他事情,提高整体效率!
