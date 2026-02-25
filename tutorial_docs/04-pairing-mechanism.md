# OpenClaw 配对(Pairing)机制详解

## 概述

配对(Pairing)是 OpenClaw 的**安全访问控制机制**,用于在用户首次联系机器人时进行身份验证和授权。它确保只有经过批准的用户才能与机器人交互。

## 配对流程

### 完整流程图

```
┌─────────────────┐
│ 1. 用户发消息   │
│   给机器人      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. 检查访问权限 │
│   (dmPolicy)    │
└────────┬────────┘
         │
         ├─── allowFrom 中? ───> 允许访问 ✓
         │
         ├─── dmPolicy="open"? ───> 允许访问 ✓
         │
         ├─── dmPolicy="pairing"?
         │        │
         │        ▼
         │   ┌──────────────────┐
         │   │ 3. 生成配对码    │
         │   │   (8位随机码)    │
         │   └────────┬─────────┘
         │            │
         │            ▼
         │   ┌──────────────────┐
         │   │ 4. 存储待审批    │
         │   │   请求到文件     │
         │   └────────┬─────────┘
         │            │
         │            ▼
         │   ┌──────────────────┐
         │   │ 5. 发送配对提示  │
         │   │   给用户         │
         │   └────────┬─────────┘
         │            │
         │            ▼
         │   ┌──────────────────┐
         │   │ 管理员审批       │
         │   │ openclaw pairing │
         │   │ approve          │
         │   └────────┬─────────┘
         │            │
         │            ▼
         │   ┌──────────────────┐
         │   │ 6. 添加到        │
         │   │   allowFrom      │
         │   └────────┬─────────┘
         │            │
         │            ▼
         │   ┌──────────────────┐
         │   │ 7. 发送通知      │
         │   │   (可选)         │
         │   └──────────────────┘
         │
         └─── dmPolicy="allowlist"? ───> 拒绝访问 ✗
```

---

## 核心组件

### 1. 频道配对适配器 (Channel Pairing Adapter)

定义在 [`channel.ts`](extensions/feishu/src/channel.ts:46-56) 中:

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

**功能**:
- **idLabel**: 定义用户 ID 的显示标签(如 "feishuUserId", "telegramUserId")
- **normalizeAllowEntry**: 规范化允许列表条目,移除前缀(如 `"user:ou_xxx"` → `"ou_xxx"`)
- **notifyApproval**: 批准后发送通知消息给用户

---

### 2. 配对存储 (Pairing Store)

位置: [`src/pairing/pairing-store.ts`](src/pairing/pairing-store.ts:1)

#### 2.1 数据结构

```typescript
type PairingRequest = {
  id: string;              // 用户 ID (如 open_id)
  code: string;            // 8位随机配对码
  createdAt: string;       // 创建时间 (ISO 8601)
  lastSeenAt: string;      // 最后活跃时间
  meta?: Record<string, string>;  // 元数据 (如 accountId, name)
};

type PairingStore = {
  version: 1;
  requests: PairingRequest[];
};
```

#### 2.2 存储位置

配对请求存储在: `~/.openclaw/credentials/<channel>-pairing.json`

例如:
- `~/.openclaw/credentials/feishu-pairing.json`
- `~/.openclaw/credentials/telegram-pairing.json`
- `~/.openclaw/credentials/whatsapp-pairing.json`

#### 2.3 配对码生成

来源: [`src/pairing/pairing-store.ts:186-204`](src/pairing/pairing-store.ts:186-204)

```typescript
const PAIRING_CODE_LENGTH = 8;
const PAIRING_CODE_ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";

function randomCode(): string {
  // 生成 8 位人类友好的随机码
  // 排除易混淆字符 (0O1I)
  let out = "";
  for (let i = 0; i < PAIRING_CODE_LENGTH; i++) {
    const idx = crypto.randomInt(0, PAIRING_CODE_ALPHABET.length);
    out += PAIRING_CODE_ALPHABET[idx];
  }
  return out;
}
```

**特点**:
- 8 个字符,全大写
- 使用 32 个字符的字母表(排除 0、O、1、I 等易混淆字符)
- 加密安全的随机数生成(`crypto.randomInt`)
- 自动去重,最多重试 500 次

#### 2.4 请求管理

```typescript
// 创建或更新配对请求
await upsertChannelPairingRequest({
  channel: "feishu",
  id: "ou_xxxxx",           // 用户 ID
  accountId: "default",     // 账号 ID (多账号支持)
  meta: { name: "张三" }    // 附加信息
});

// 列出待审批请求
const requests = await listChannelPairingRequests("feishu");

// 批准配对码
const approved = await approveChannelPairingCode({
  channel: "feishu",
  code: "ABCD1234",
  accountId: "default"
});
```

#### 2.5 过期和限制

- **过期时间**: 1 小时 (`PAIRING_PENDING_TTL_MS = 60 * 60 * 1000`)
- **最大待审批数**: 3 个 (`PAIRING_PENDING_MAX = 3`)
- 自动清理过期请求
- 超过限制时保留最近活跃的请求

---

### 3. AllowFrom 存储

#### 3.1 数据结构

```typescript
type AllowFromStore = {
  version: 1;
  allowFrom: string[];  // 已授权的用户 ID 列表
};
```

#### 3.2 存储位置

**默认账号**:
- `~/.openclaw/credentials/<channel>-allowFrom.json`

**命名账号**:
- `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json`

例如:
- `~/.openclaw/credentials/feishu-allowFrom.json` (默认账号)
- `~/.openclaw/credentials/feishu-production-allowFrom.json` (production 账号)

#### 3.3 操作方法

```typescript
// 读取 allowFrom 列表
const allowList = await readChannelAllowFromStore("feishu");

// 添加条目
await addChannelAllowFromStoreEntry({
  channel: "feishu",
  entry: "ou_xxxxx",
  accountId: "default"
});

// 移除条目
await removeChannelAllowFromStoreEntry({
  channel: "feishu",
  entry: "ou_xxxxx",
  accountId: "default"
});
```

---

### 4. 访问控制检查

示例来自 WhatsApp ([`src/web/inbound/access-control.ts:151-214`](src/web/inbound/access-control.ts:151-214)):

```typescript
// 检查是否在 allowFrom 中
const allowed = dmHasWildcard || normalizedAllowFrom.includes(userId);

if (!allowed) {
  if (dmPolicy === "pairing") {
    // 生成配对码并存储
    const { code, created } = await upsertChannelPairingRequest({
      channel: "whatsapp",
      id: userId,
      accountId: account.accountId,
      meta: { name: userName }
    });
    
    if (created) {
      // 发送配对提示给用户
      await sock.sendMessage(jid, {
        text: buildPairingReply({
          channel: "whatsapp",
          idLine: `Your WhatsApp phone number: ${userId}`,
          code
        })
      });
    }
  }
  
  // 拒绝访问
  return { allowed: false, shouldMarkRead: false };
}

// 允许访问
return { allowed: true, shouldMarkRead: true };
```

---

## 配对策略 (dmPolicy)

在配置文件中设置:

```yaml
channels:
  feishu:
    dmPolicy: "pairing"  # open | pairing | allowlist | disabled
    allowFrom:           # 预配置的允许列表
      - "ou_xxxxx"
    groupPolicy: "allowlist"  # 群组策略
    groupAllowFrom:
      - "oc_xxxxx"
```

### 策略说明

| dmPolicy | 行为 | 适用场景 |
|----------|------|---------|
| **pairing** (默认) | 首次联系时生成配对码,需要管理员批准 | **推荐**:安全且用户友好 |
| **open** | 允许任何人直接访问 | 测试环境、公开机器人 |
| **allowlist** | 仅允许 allowFrom 中的用户,不生成配对码 | 严格控制的环境 |
| **disabled** | 禁用所有 DM | 仅用于群组的机器人 |

---

## CLI 命令

### 列出待审批请求

```bash
# 列出所有待审批请求
openclaw pairing list feishu

# 列出特定账号的请求
openclaw pairing list feishu --account production

# JSON 格式输出
openclaw pairing list feishu --json
```

输出示例:
```
Pairing requests (2)
┌──────────┬─────────────┬──────────────┬──────────────────────┐
│ Code     │ feishuUserId│ Meta         │ Requested            │
├──────────┼─────────────┼──────────────┼──────────────────────┤
│ ABCD1234 │ ou_xxxxx    │ {"name":"张三"}│ 2026-02-25T08:30:00Z │
│ EFGH5678 │ ou_yyyyy    │ {"name":"李四"}│ 2026-02-25T08:35:00Z │
└──────────┴─────────────┴──────────────┴──────────────────────┘
```

### 批准配对

```bash
# 方式 1: 显式指定频道
openclaw pairing approve feishu ABCD1234

# 方式 2: 使用 --channel 选项
openclaw pairing approve --channel feishu ABCD1234

# 方式 3: 指定账号
openclaw pairing approve feishu ABCD1234 --account production

# 批准并通知用户
openclaw pairing approve feishu ABCD1234 --notify
```

批准成功后:
```
✓ Approved feishu sender ou_xxxxx.
```

---

## 批准流程详解

### 步骤 1: 用户发送消息

用户首次给机器人发消息,系统检测到该用户不在 allowFrom 中。

### 步骤 2: 生成配对码

系统调用 [`upsertChannelPairingRequest`](src/pairing/pairing-store.ts:471-479):

```typescript
const { code, created } = await upsertChannelPairingRequest({
  channel: "feishu",
  id: "ou_xxxxx",
  accountId: "default",
  meta: { name: "张三" }
});
```

- 检查是否已有该用户的待审批请求
- 如果已存在,重用现有的配对码(`created: false`)
- 如果不存在,生成新的 8 位随机码(`created: true`)

### 步骤 3: 发送配对提示

系统通过 [`buildPairingReply`](src/pairing/pairing-messages.ts:1) 构建提示消息:

```typescript
const message = buildPairingReply({
  channel: "feishu",
  idLine: "Your Feishu open_id: ou_xxxxx",
  code: "ABCD1234"
});
```

生成的消息示例:
```
OpenClaw access request pending.

Your Feishu open_id: ou_xxxxx
Pairing code: ABCD1234

Ask the bot owner to approve with:
openclaw pairing approve feishu ABCD1234
```

### 步骤 4: 管理员批准

管理员在终端运行:

```bash
openclaw pairing approve feishu ABCD1234
```

系统执行 [`approveChannelPairingCode`](src/pairing/pairing-store.ts:408-417):

1. 读取配对请求文件
2. 查找匹配的配对码
3. 提取用户 ID
4. 将用户 ID 添加到 allowFrom 存储
5. 删除配对请求记录

### 步骤 5: 发送批准通知(可选)

如果使用 `--notify` 选项,系统调用频道的 `notifyApproval`:

```typescript
await notifyApproval({
  cfg: config,
  id: "ou_xxxxx",
  runtime: runtime
});
```

飞书示例 ([`channel.ts:49-54`](extensions/feishu/src/channel.ts:49-54)):
```typescript
await sendMessageFeishu({
  cfg,
  to: "ou_xxxxx",
  text: "✅ OpenClaw access approved. Send a message to start chatting."
});
```

---

## 多账号支持

OpenClaw 支持同一频道的多个账号,每个账号有独立的配对请求和 allowFrom 列表。

### 配置示例

```yaml
channels:
  feishu:
    enabled: true
    appId: "cli_default"
    dmPolicy: "pairing"
    
    accounts:
      production:
        enabled: true
        name: "生产环境"
        appId: "cli_prod"
        dmPolicy: "pairing"
      
      staging:
        enabled: true
        name: "测试环境"
        appId: "cli_staging"
        dmPolicy: "open"
```

### 存储结构

```
~/.openclaw/credentials/
├── feishu-pairing.json                    # 默认账号的配对请求
├── feishu-allowFrom.json                  # 默认账号的 allowFrom
├── feishu-production-pairing.json         # production 账号的配对请求
├── feishu-production-allowFrom.json       # production 账号的 allowFrom
├── feishu-staging-pairing.json            # staging 账号的配对请求
└── feishu-staging-allowFrom.json          # staging 账号的 allowFrom
```

### 命令使用

```bash
# 列出 production 账号的配对请求
openclaw pairing list feishu --account production

# 批准 production 账号的配对
openclaw pairing approve feishu ABCD1234 --account production
```

---

## 安全特性

### 1. 加密安全的随机码

使用 Node.js 的 `crypto.randomInt()` 生成配对码,确保不可预测性。

### 2. 自动过期

配对请求 1 小时后自动过期,防止旧请求累积。

### 3. 数量限制

最多保留 3 个待审批请求,防止滥用。

### 4. 文件锁

使用文件锁(`withFileLock`)防止并发修改导致数据损坏。

### 5. 路径安全

频道名和账号 ID 经过清理,防止路径遍历攻击:

```typescript
function safeChannelKey(channel: string): string {
  const safe = channel.replace(/[\\/:*?"<>|]/g, "_").replace(/\.\./g, "_");
  if (!safe || safe === "_") {
    throw new Error("invalid pairing channel");
  }
  return safe;
}
```

### 6. 规范化处理

通过 `normalizeAllowEntry` 确保 ID 格式一致,防止绕过:

```typescript
normalizeAllowEntry: (entry) => entry.replace(/^(feishu|user|open_id):/i, "")
```

这样 `"user:ou_xxx"`, `"feishu:ou_xxx"`, `"ou_xxx"` 都会被规范化为 `"ou_xxx"`。

---

## 实现要点

### 1. 幂等性

重复调用 `upsertChannelPairingRequest` 不会创建重复的配对码,而是返回现有的码。

### 2. 原子性

使用文件锁确保读-修改-写操作的原子性,防止竞态条件。

### 3. 向后兼容

支持读取旧的 channel-level allowFrom 文件,同时使用新的 account-scoped 文件。

### 4. 可扩展性

频道可以通过 `pairingAdapter` 参数传递自定义适配器,支持扩展频道。

---

## 总结

配对机制是 OpenClaw 的核心安全特性,它:

1. **生成唯一的 8 位配对码**
2. **存储待审批请求到 JSON 文件**
3. **发送配对提示给用户**
4. **管理员通过 CLI 批准**
5. **将用户 ID 添加到 allowFrom**
6. **可选发送批准通知**

通过这个机制,OpenClaw 实现了:
- ✅ 安全的访问控制
- ✅ 用户友好的审批流程
- ✅ 多账号隔离
- ✅ 自动化的生命周期管理

配对机制平衡了安全性和易用性,是 OpenClaw 推荐的默认 DM 策略。
