# TypeScript 基础语法详解

> 本文档通过实际代码示例，讲解 OpenClaw 项目中常用的 TypeScript 语法

## 目录
- [可选链操作符 ?.](#可选链操作符-)
- [空值合并操作符 ??](#空值合并操作符-)
- [数组方法 map](#数组方法-map)
- [箭头函数](#箭头函数)
- [类型系统](#类型系统)
- [异步编程](#异步编程)
- [解构赋值](#解构赋值)
- [模板字符串](#模板字符串)

---

## 可选链操作符 `?.`

### 问题场景

访问嵌套对象属性时，如果中间某层是 `null` 或 `undefined`，会报错：

```javascript
// ❌ 传统 JavaScript - 容易报错
const name = user.profile.name;  // TypeError: Cannot read property 'name' of undefined
```

### 解决方案：`?.`

```typescript
// ✅ TypeScript - 安全访问
const name = user?.profile?.name;  // 如果 user 或 profile 不存在，返回 undefined
```

### 工作原理

```typescript
// 等价于
const name = (user !== null && user !== undefined && 
              user.profile !== null && user.profile !== undefined) 
              ? user.profile.name 
              : undefined;
```

### 实际示例

来自 [`extensions/feishu/src/wiki.ts:31`](../extensions/feishu/src/wiki.ts#L31):

```typescript
const spaces =
    res.data?.items?.map((s) => ({
      space_id: s.space_id,
      name: s.name,
      description: s.description,
      visibility: s.visibility,
    })) ?? [];
```

**逐步分析**:

```typescript
// 步骤 1: 尝试访问 res.data
res.data  
// 如果 res.data 是 null/undefined，整个表达式返回 undefined

// 步骤 2: 尝试访问 res.data.items
res.data?.items  
// 如果 items 不存在，返回 undefined

// 步骤 3: 调用 map 方法
res.data?.items?.map(...)  
// 如果 items 是 undefined，不会调用 map，直接返回 undefined
```

### 使用场景

```typescript
// 1. 访问对象属性
user?.profile?.email

// 2. 调用可选方法
obj?.method?.()

// 3. 访问数组元素
arr?.[0]?.value

// 4. 链式调用
api?.fetchUser?.()?.then?.(data => console.log(data))
```

---

## 空值合并操作符 `??`

### 问题场景

使用 `||` 运算符时，会把所有假值（0, "", false, null, undefined）都替换：

```javascript
// ❌ 使用 || 的问题
const count = 0;
const display = count || 10;  // 返回 10，但我们期望是 0

const text = "";
const message = text || "默认";  // 返回 "默认"，但我们期望是空字符串
```

### 解决方案：`??`

`??` 只处理 `null` 和 `undefined`，保留其他假值：

```typescript
// ✅ 使用 ?? 更精确
const count = 0;
const display = count ?? 10;  // 返回 0

const text = "";
const message = text ?? "默认";  // 返回 ""

const value = null;
const result = value ?? "默认";  // 返回 "默认"
```

### 对比表

| 表达式 | `||` 结果 | `??` 结果 |
|--------|----------|----------|
| `0 || 100` | `100` | `0` |
| `"" || "默认"` | `"默认"` | `""` |
| `false || true` | `true` | `false` |
| `null || "默认"` | `"默认"` | `"默认"` |
| `undefined || "默认"` | `"默认"` | `"默认"` |

### 实际示例

```typescript
// 来自 wiki.ts
const spaces = res.data?.items?.map(...) ?? [];

// 分析
res.data?.items?.map(...)  // 可能返回 undefined
undefined ?? []             // 如果是 undefined，使用 []
```

**完整流程**:

```typescript
// 情况 1: res.data.items 存在
res.data.items = [...]
res.data?.items?.map(...)  // 返回数组 [...]
[...] ?? []                // 左边不是 null/undefined，使用左边
结果: [...]

// 情况 2: res.data 不存在
res.data = undefined
res.data?.items            // 返回 undefined
undefined?.map(...)        // 返回 undefined
undefined ?? []            // 使用右边的默认值
结果: []
```

---

## 数组方法 `map`

### 基本概念

`map` 方法遍历数组，对每个元素执行转换，返回新数组（不修改原数组）。

### 语法

```typescript
array.map((element, index, array) => {
  // 返回新元素
});
```

### 实际示例

```typescript
// 来自 wiki.ts
const spaces = items.map((s) => ({
  space_id: s.space_id,
  name: s.name,
  description: s.description,
  visibility: s.visibility,
}));
```

**详细分析**:

```typescript
// 原始数据
const items = [
  {
    space_id: "s1",
    name: "技术文档",
    description: "内部技术资料",
    visibility: "private",
    extra_field: "不需要的字段",
    another_field: "也不需要"
  },
  {
    space_id: "s2",
    name: "产品规划",
    description: "产品路线图",
    visibility: "public",
    extra_field: "不需要的字段",
    another_field: "也不需要"
  }
];

// map 转换
const spaces = items.map((s) => ({
  space_id: s.space_id,
  name: s.name,
  description: s.description,
  visibility: s.visibility,
}));

// 结果（去掉了不需要的字段）
[
  {
    space_id: "s1",
    name: "技术文档",
    description: "内部技术资料",
    visibility: "private"
  },
  {
    space_id: "s2",
    name: "产品规划",
    description: "产品路线图",
    visibility: "public"
  }
]
```

### 常见用法

```typescript
// 1. 提取字段
const names = users.map(u => u.name);
// ["Alice", "Bob", "Charlie"]

// 2. 转换数据
const doubled = numbers.map(n => n * 2);
// [2, 4, 6, 8]

// 3. 构建对象
const userList = ids.map(id => ({ id, name: `User ${id}` }));
// [{ id: 1, name: "User 1" }, ...]

// 4. 调用方法
const uppercased = strings.map(s => s.toUpperCase());
// ["HELLO", "WORLD"]
```

---

## 箭头函数

### 基本语法

```typescript
// 传统函数
function add(a, b) {
  return a + b;
}

// 箭头函数
const add = (a, b) => a + b;
```

### 多种写法

```typescript
// 1. 无参数
const greet = () => "Hello";

// 2. 单参数（可省略括号）
const double = n => n * 2;
const double = (n) => n * 2;  // 也可以加括号

// 3. 多参数
const add = (a, b) => a + b;

// 4. 多行函数体（需要 return）
const complex = (x) => {
  const result = x * 2;
  return result + 1;
};

// 5. 返回对象（必须用括号包裹）
const createUser = (name) => ({ name, age: 0 });  // ✅ 正确
const createUser = (name) => { name, age: 0 };    // ❌ 错误（会被当作函数体）
```

### 为什么要用括号包裹对象？

```typescript
// JavaScript 的二义性问题
(s) => { name: s.name }  
// 这会被解析为：函数体 + 标签语句
// 相当于：
(s) => {
  name: s.name;  // 标签 "name"，值 s.name（被忽略）
}
// 返回 undefined

(s) => ({ name: s.name })
// 这会被解析为：返回对象
// 返回 { name: s.name }
```

### 实际示例

```typescript
// 来自 wiki.ts
items.map((s) => ({
  space_id: s.space_id,
  name: s.name,
}))

// 等价的完整写法
items.map((s) => {
  return {
    space_id: s.space_id,
    name: s.name,
  };
})
```

### 箭头函数的优势

```typescript
// 1. 简洁
numbers.filter(n => n > 0).map(n => n * 2);

// 2. 自动绑定 this（在类方法中很有用）
class Counter {
  count = 0;
  
  // 传统方法：this 会丢失
  increment() {
    setTimeout(function() {
      this.count++;  // ❌ this 是 undefined
    }, 1000);
  }
  
  // 箭头函数：this 自动绑定
  increment() {
    setTimeout(() => {
      this.count++;  // ✅ this 指向 Counter 实例
    }, 1000);
  }
}
```

---

## 类型系统

### 基本类型

```typescript
// 原始类型
let name: string = "Alice";
let age: number = 25;
let active: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;

// 数组
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b"];

// 元组
let tuple: [string, number] = ["Alice", 25];

// 枚举
enum Color {
  Red,
  Green,
  Blue
}
let color: Color = Color.Red;

// Any（避免使用）
let anything: any = "can be anything";

// Unknown（更安全的 any）
let unknown: unknown = "something";
if (typeof unknown === "string") {
  console.log(unknown.toUpperCase());  // 类型收窄后可以使用
}
```

### 对象类型

```typescript
// 接口
interface User {
  name: string;
  age: number;
  email?: string;  // 可选属性
  readonly id: number;  // 只读属性
}

// 类型别名
type Point = {
  x: number;
  y: number;
};

// 联合类型
type Status = "pending" | "success" | "error";

// 交叉类型
type Admin = User & { role: "admin" };
```

### 函数类型

```typescript
// 函数签名
type MathFn = (a: number, b: number) => number;

const add: MathFn = (a, b) => a + b;

// 可选参数
function greet(name: string, greeting?: string) {
  return `${greeting || "Hello"}, ${name}`;
}

// 默认参数
function greet(name: string, greeting = "Hello") {
  return `${greeting}, ${name}`;
}

// 剩余参数
function sum(...numbers: number[]) {
  return numbers.reduce((a, b) => a + b, 0);
}
```

### 泛型

```typescript
// 泛型函数
function identity<T>(value: T): T {
  return value;
}

const num = identity<number>(42);
const str = identity<string>("hello");

// 泛型接口
interface Box<T> {
  value: T;
}

const numberBox: Box<number> = { value: 42 };
const stringBox: Box<string> = { value: "hello" };

// 泛型约束
function getProperty<T, K extends keyof T>(obj: T, key: K) {
  return obj[key];
}

const user = { name: "Alice", age: 25 };
const name = getProperty(user, "name");  // ✅ 正确
const invalid = getProperty(user, "invalid");  // ❌ 编译错误
```

### 实际示例

来自项目代码：

```typescript
// extensions/feishu/src/types.ts
export type FeishuDomain = "feishu" | "lark" | (string & {});
export type FeishuConnectionMode = "websocket" | "webhook";

export type ResolvedFeishuAccount = {
  accountId: string;
  appId: string;
  appSecret: string;
  verificationToken?: string;
  domain: FeishuDomain;
  config: FeishuConfig;
};

// 使用
function createClient(account: ResolvedFeishuAccount): Lark.Client {
  return new Lark.Client({
    appId: account.appId,
    appSecret: account.appSecret,
    domain: resolveDomain(account.domain),
  });
}
```

---

## 异步编程

### Promise

```typescript
// 创建 Promise
const fetchUser = (id: number): Promise<User> => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (id > 0) {
        resolve({ id, name: "Alice" });
      } else {
        reject(new Error("Invalid ID"));
      }
    }, 1000);
  });
};

// 使用 Promise
fetchUser(1)
  .then(user => console.log(user))
  .catch(error => console.error(error));
```

### Async/Await

```typescript
// async 函数
async function getUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const user = await response.json();
  return user;
}

// 错误处理
async function getUserSafe(id: number): Promise<User | null> {
  try {
    const user = await getUser(id);
    return user;
  } catch (error) {
    console.error(error);
    return null;
  }
}

// 并行执行
async function getMultipleUsers(ids: number[]): Promise<User[]> {
  const promises = ids.map(id => getUser(id));
  return await Promise.all(promises);
}
```

### 实际示例

来自 [`extensions/feishu/src/bot.ts`](../extensions/feishu/src/bot.ts):

```typescript
export async function handleFeishuMessage(params: {
  cfg: ClawdbotConfig;
  event: FeishuMessageEvent;
  // ...
}) {
  // 1. 异步获取账号配置
  const account = resolveFeishuAccount({ cfg, accountId });
  
  // 2. 异步路由解析
  const route = await resolveOutboundSession({
    cfg,
    channel: "feishu",
    // ...
  });
  
  // 3. 异步下载媒体
  const media = await resolveFeishuMediaList({ ctx, cfg, account });
  
  // 4. 异步获取引用消息
  if (ctx.quotedMessageId) {
    const quoted = await getMessageFeishu({
      cfg,
      messageId: ctx.quotedMessageId,
      accountId
    });
  }
  
  // 5. 异步分发消息
  await dispatchMessage(payload, dispatcher);
}
```

---

## 解构赋值

### 对象解构

```typescript
// 基本用法
const user = { name: "Alice", age: 25, email: "alice@example.com" };
const { name, age } = user;
console.log(name);  // "Alice"
console.log(age);   // 25

// 重命名
const { name: userName, age: userAge } = user;
console.log(userName);  // "Alice"

// 默认值
const { name, country = "Unknown" } = user;
console.log(country);  // "Unknown"

// 嵌套解构
const user = {
  name: "Alice",
  address: { city: "Beijing", country: "China" }
};
const { address: { city } } = user;
console.log(city);  // "Beijing"

// 剩余属性
const { name, ...rest } = user;
console.log(rest);  // { age: 25, email: "..." }
```

### 数组解构

```typescript
// 基本用法
const numbers = [1, 2, 3, 4, 5];
const [first, second] = numbers;
console.log(first);  // 1
console.log(second); // 2

// 跳过元素
const [, , third] = numbers;
console.log(third);  // 3

// 剩余元素
const [head, ...tail] = numbers;
console.log(head);  // 1
console.log(tail);  // [2, 3, 4, 5]

// 交换变量
let a = 1, b = 2;
[a, b] = [b, a];
```

### 函数参数解构

```typescript
// 对象参数解构
function greet({ name, age }: { name: string; age: number }) {
  return `${name} is ${age} years old`;
}

greet({ name: "Alice", age: 25 });

// 带默认值
function greet({ name, age = 0 }: { name: string; age?: number }) {
  return `${name} is ${age} years old`;
}
```

### 实际示例

来自项目代码：

```typescript
// src/gateway/server-broadcast.ts
export function createGatewayBroadcaster(params: { clients: Set<GatewayWsClient> }) {
  // 解构 params
  const { clients } = params;
  
  // ... 使用 clients
}

// extensions/feishu/src/bot.ts
export async function handleFeishuMessage(params: {
  cfg: ClawdbotConfig;
  event: FeishuMessageEvent;
  botOpenId?: string;
  runtime?: RuntimeEnv;
  chatHistories: Map<string, HistoryEntry[]>;
  accountId: string;
}) {
  // 解构所有参数
  const { cfg, event, botOpenId, runtime, chatHistories, accountId } = params;
  
  // 直接使用变量，代码更清晰
  const account = resolveFeishuAccount({ cfg, accountId });
}
```

---

## 模板字符串

### 基本用法

```typescript
// 传统字符串拼接
const name = "Alice";
const age = 25;
const message = "Hello, " + name + "! You are " + age + " years old.";

// 模板字符串
const message = `Hello, ${name}! You are ${age} years old.`;
```

### 多行字符串

```typescript
// 传统写法
const html = "<div>\n" +
  "  <h1>Title</h1>\n" +
  "  <p>Content</p>\n" +
  "</div>";

// 模板字符串
const html = `
  <div>
    <h1>Title</h1>
    <p>Content</p>
  </div>
`;
```

### 表达式嵌入

```typescript
const a = 10;
const b = 20;

// 可以嵌入任意表达式
console.log(`Sum: ${a + b}`);  // "Sum: 30"
console.log(`Double: ${a * 2}`);  // "Double: 20"
console.log(`Is positive: ${a > 0}`);  // "Is positive: true"

// 调用函数
const format = (n: number) => n.toFixed(2);
console.log(`Formatted: ${format(3.14159)}`);  // "Formatted: 3.14"

// 三元表达式
const status = a > b ? "greater" : "less or equal";
console.log(`Status: ${status}`);
```

### 实际示例

来自项目代码：

```typescript
// extensions/feishu/src/monitor.ts
log(`feishu[${accountId}]: starting WebSocket connection...`);
log(`feishu[${accountId}]: bot open_id resolved: ${botOpenId ?? "unknown"}`);

// src/gateway/server-broadcast.ts
logWs("out", "event", {
  event,
  seq: eventSeq ?? "targeted",
  clients: params.clients.size,
});

// extensions/feishu/src/bot.ts
const feishuFrom = `feishu:${ctx.senderOpenId}`;
const feishuTo = isGroup ? `chat:${ctx.chatId}` : `user:${ctx.senderOpenId}`;
```

---

## 总结

### 常用语法速查

| 语法 | 用途 | 示例 |
|------|------|------|
| `?.` | 安全访问 | `obj?.prop?.method?.()` |
| `??` | 空值默认 | `value ?? defaultValue` |
| `.map()` | 数组转换 | `arr.map(x => x * 2)` |
| `=>` | 箭头函数 | `(a, b) => a + b` |
| `async/await` | 异步编程 | `await fetchData()` |
| `{ }` | 解构赋值 | `const { name } = user` |
| `` ` ` `` | 模板字符串 | `` `Hello ${name}` `` |

### 学习建议

1. **多看实际代码**: 在项目中查找这些语法的使用场景
2. **小步快跑**: 每次学一个新语法，立即在代码中尝试
3. **类型思维**: 始终思考变量的类型，让 TypeScript 帮你检查错误
4. **善用 IDE**: VSCode 的类型提示能帮你理解代码

### 推荐资源

- [TypeScript 官方文档](https://www.typescriptlang.org/docs/)
- [MDN JavaScript 参考](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)
- OpenClaw 项目代码（最佳学习材料）
