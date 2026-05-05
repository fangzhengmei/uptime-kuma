# Uptime Kuma API Key 鉴权机制分析报告

## 1. 概述

本报告详细分析 Uptime Kuma 项目中 API Key 的鉴权、调用频率限制和权限范围实现机制。分析基于以下核心文件：

- `server/model/api_key.js` - API Key 数据模型
- `server/auth.js` - 认证中间件和鉴权逻辑
- `server/rate-limiter.js` - 速率限制实现
- `server/socket-handlers/api-key-socket-handler.js` - API Key 管理

---

## 2. API Key 数据模型与存储

### 2.1 数据库表结构

API Key 存储在 `api_key` 表中，结构如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER | 主键，自增 |
| `key` | VARCHAR(255) | 哈希后的 API Key（bcrypt） |
| `name` | VARCHAR(255) | API Key 名称 |
| `user_id` | INTEGER | 关联的用户 ID（外键） |
| `created_date` | DATETIME | 创建时间 |
| `active` | BOOLEAN | 是否激活（默认 true） |
| `expires` | DATETIME | 过期时间（可为 null） |

**关键代码位置**: `db/knex_init_db.js:442-457`

### 2.2 API Key 生成格式

API Key 生成逻辑位于 `server/socket-handlers/api-key-socket-handler.js`：

```javascript
// 生成 40 位随机字符串作为明文密钥
let clearKey = nanoid(40);
// 使用 bcrypt 哈希存储
let hashedKey = await passwordHash.generate(clearKey);
// 最终格式: uk{id}_{clearKey}
let formattedKey = "uk" + bean.id + "_" + clearKey;
```

**格式说明**:
- 前缀 `uk` (Uptime Kuma)
- Key ID（用于快速定位数据库记录）
- 下划线分隔符
- 40 位随机字符串（实际密钥）

**示例**: `uk1_aB3cD5eF7gH9iJ1kL2mN3oP4qR6sT8uV0wX2yZ4`

**关键代码位置**: `server/socket-handlers/api-key-socket-handler.js:22-32`

### 2.3 API Key 状态管理

API Key 有三种状态，通过 `getStatus()` 方法判断：

```javascript
getStatus() {
    let current = dayjs();
    let expiry = dayjs(this.expires);
    if (expiry.diff(current) < 0) {
        return "expired";  // 已过期
    }
    return this.active ? "active" : "inactive";  // 激活/未激活
}
```

**关键代码位置**: `server/model/api_key.js:10-18`

---

## 3. 鉴权流程分析

### 3.1 认证方式：HTTP Basic Auth

Uptime Kuma 使用 `express-basic-auth` 中间件实现 API Key 认证。客户端需要在请求头中携带：

```
Authorization: Basic <base64(username:api_key)>
```

**重要**: `username` 字段实际上未被使用，认证只验证 `password` 字段（即 API Key）。

### 3.2 鉴权中间件 `apiAuth`

`apiAuth` 是核心鉴权中间件，位于 `server/auth.js:155-176`：

```javascript
exports.apiAuth = async function (req, res, next) {
    if (!(await Settings.get("disableAuth"))) {
        let usingAPIKeys = await Settings.get("apiKeysEnabled");
        let middleware;
        if (usingAPIKeys) {
            // 使用 API Key 认证
            middleware = basicAuth({
                authorizer: apiAuthorizer,
                authorizeAsync: true,
                challenge: true,
            });
        } else {
            // 降级使用用户名密码认证
            middleware = basicAuth({
                authorizer: userAuthorizer,
                authorizeAsync: true,
                challenge: true,
            });
        }
        middleware(req, res, next);
    } else {
        next();  // 认证已禁用，直接通过
    }
};
```

**配置开关**:
- `disableAuth`: 全局禁用认证（开发环境使用）
- `apiKeysEnabled`: 是否启用 API Key 认证（创建第一个 API Key 时自动设为 true）

### 3.3 API Key 验证逻辑 `verifyAPIKey`

验证函数位于 `server/auth.js:41-63`：

```javascript
async function verifyAPIKey(key) {
    if (typeof key !== "string") {
        return false;
    }

    // 解析 Key ID: uk{id}_{clearKey}
    let index = key.substring(2, key.indexOf("_"));  // 提取 id
    let clear = key.substring(key.indexOf("_") + 1, key.length);  // 提取明文密钥

    // 根据 ID 从数据库获取哈希
    let hash = await R.findOne("api_key", " id=? ", [index]);

    if (hash === null) {
        return false;
    }

    // 检查过期时间和激活状态
    let current = dayjs();
    let expiry = dayjs(hash.expires);
    if (expiry.diff(current) < 0 || !hash.active) {
        return false;
    }

    // 验证 bcrypt 哈希
    return hash && passwordHash.verify(clear, hash.key);
}
```

**验证步骤**:
1. **格式检查**: 确保输入是字符串
2. **解析 Key**: 从 `uk{id}_{clearKey}` 格式中提取 ID 和明文密钥
3. **数据库查询**: 根据 ID 查找对应的哈希记录
4. **状态检查**: 验证是否过期、是否激活
5. **哈希验证**: 使用 bcrypt 验证明文与存储的哈希是否匹配

### 3.4 完整的鉴权授权器 `apiAuthorizer`

位于 `server/auth.js:79-97`，集成了速率限制和鉴权：

```javascript
function apiAuthorizer(username, password, callback) {
    // 1. 先检查速率限制
    apiRateLimiter.pass(null, 0).then((pass) => {
        if (pass) {
            // 2. 验证 API Key
            verifyAPIKey(password).then((valid) => {
                if (!valid) {
                    log.warn("api-auth", "Failed API auth attempt: invalid API Key");
                }
                callback(null, valid);
                // 3. 验证成功后消耗一个令牌
                apiRateLimiter.removeTokens(1);
            });
        } else {
            log.warn("api-auth", "Failed API auth attempt: rate limit exceeded");
            callback(null, false);
        }
    });
}
```

**关键设计**:
- 速率限制检查在鉴权之前进行，防止暴力破解
- 只有验证成功后才消耗令牌
- 失败尝试会记录日志

### 3.5 受保护的 API 端点

目前只有 `/metrics` 端点明确使用 `apiAuth` 中间件：

```javascript
// server/server.js:337
app.get("/metrics", apiAuth, prometheusAPIMetrics());
```

**注意**: `api-router.js` 中的其他端点（如 `/api/push/:pushToken`）使用独立的认证机制（push token），不通过 API Key 鉴权。

---

## 4. 调用频率限制（Rate Limiting）

### 4.1 实现架构

速率限制基于 `limiter` 库实现，位于 `server/rate-limiter.js`。

核心类 `KumaRateLimiter` 封装了速率限制逻辑：

```javascript
class KumaRateLimiter {
    constructor(config) {
        this.errorMessage = config.errorMessage;
        this.rateLimiter = new RateLimiter(config);
    }

    async pass(callback, num = 1) {
        const remainingRequests = await this.removeTokens(num);
        log.info("rate-limit", "remaining requests: " + remainingRequests);
        if (remainingRequests < 0) {
            if (callback) {
                callback({ ok: false, msg: this.errorMessage });
            }
            return false;
        }
        return true;
    }

    async removeTokens(num = 1) {
        return await this.rateLimiter.removeTokens(num);
    }
}
```

### 4.2 速率限制配置

系统定义了三个独立的速率限制器：

| 限制器 | 令牌数/间隔 | 用途 |
|--------|-------------|------|
| `loginRateLimiter` | 20/分钟 | 登录尝试限制 |
| `apiRateLimiter` | **60/分钟** | API Key 调用限制 |
| `twoFaRateLimiter` | 30/分钟 | 2FA 验证限制 |

**配置代码**:
```javascript
const apiRateLimiter = new KumaRateLimiter({
    tokensPerInterval: 60,
    interval: "minute",
    fireImmediately: true,
    errorMessage: "Too frequently, try again later.",
});
```

**关键代码位置**: `server/rate-limiter.js:57-62`

### 4.3 速率限制工作流程

在 `apiAuthorizer` 中的执行顺序：

1. **预检查**: `apiRateLimiter.pass(null, 0)` - 检查是否有可用令牌（不消耗）
2. **鉴权**: 执行 `verifyAPIKey()` 验证 API Key
3. **消耗令牌**: 验证成功后调用 `apiRateLimiter.removeTokens(1)`

**安全优势**:
- 无效的 API Key 不会消耗令牌（防止暴力破解时浪费资源）
- 只有验证通过的请求才会计入速率限制
- 速率限制失败会记录日志：`"Failed API auth attempt: rate limit exceeded"`

---

## 5. 权限范围（Permissions/Scopes）

### 5.1 当前实现状态

**重要发现**: Uptime Kuma 当前**没有实现细粒度的权限范围（Scopes）机制**。

#### 证据分析:

1. **数据库表结构**:
   - `api_key` 表中**没有** `scope`、`permission` 或类似字段
   - 只有 `user_id` 外键关联到用户

2. **模型定义**:
   - `APIKey` 类（`server/model/api_key.js`）没有任何权限相关属性
   - `toJSON()` 和 `toPublicJSON()` 方法不包含权限信息

3. **鉴权逻辑**:
   - `verifyAPIKey()` 只验证 Key 的有效性、激活状态和过期时间
   - 不进行任何权限检查

### 5.2 权限模型

当前的权限模型非常简单：

```
API Key → 关联 User → 继承 User 的全部权限
```

**特点**:
- 每个 API Key 严格绑定到一个用户（`user_id`）
- API Key 拥有该用户的**所有权限**
- 没有只读、只写等细粒度权限控制
- 没有针对特定端点的权限限制

### 5.3 改进建议（非实现需求）

如果未来需要添加权限范围功能，建议考虑：

1. **数据库扩展**:
   - 添加 `api_key_scope` 表存储权限范围
   - 或在 `api_key` 表添加 `scopes` JSON 字段

2. **支持的 Scope 类型**:
   - `read:monitors` - 读取监控信息
   - `write:monitors` - 创建/修改监控
   - `read:status_pages` - 读取状态页
   - `admin:settings` - 管理设置
   - `metrics:read` - 读取 Prometheus 指标

3. **鉴权增强**:
   - 在 `apiAuth` 中间件后添加 scope 检查中间件
   - 每个受保护的端点声明所需的 scope

---

## 6. 外部自动化客户端请求流程

### 6.1 完整请求流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    外部自动化客户端请求流程                            │
└─────────────────────────────────────────────────────────────────────┘

  客户端                          服务器
    │                               │
    │  1. 构造 HTTP 请求            │
    │     Authorization: Basic     │
    │     base64(anything:api_key) │
    │──────────────────────────────>│
    │                               │
    │                               │  2. apiAuth 中间件
    │                               │     ├─ 检查 disableAuth 配置
    │                               │     └─ 检查 apiKeysEnabled 配置
    │                               │
    │                               │  3. apiAuthorizer
    │                               │     ├─ apiRateLimiter.pass()
    │                               │     │   └─ 检查是否有可用令牌
    │                               │     │
    │                               │     └─ verifyAPIKey()
    │                               │         ├─ 解析 key: uk{id}_{clear}
    │                               │         ├─ 数据库查询哈希
    │                               │         ├─ 检查 active 状态
    │                               │         ├─ 检查 expires 过期时间
    │                               │         └─ bcrypt 哈希验证
    │                               │
    │  4. 速率限制/鉴权失败         │
    │  <───────────────────────────│
    │  HTTP 401 Unauthorized       │
    │                               │
    │                               │  5. 鉴权成功
    │                               │     └─ apiRateLimiter.removeTokens(1)
    │                               │
    │                               │  6. 执行业务逻辑
    │                               │     (如 /metrics 端点)
    │                               │
    │  7. 返回响应                  │
    │  <───────────────────────────│
    │  HTTP 200 OK                 │
```

### 6.2 客户端代码示例

#### 使用 curl:
```bash
# API Key 作为密码，用户名可以任意
curl -u any_username:uk1_aB3cD5eF7gH9iJ1kL2mN3oP4qR6sT8uV0wX2yZ4 \
  http://localhost:3001/metrics
```

#### 使用 Python requests:
```python
import requests
from requests.auth import HTTPBasicAuth

api_key = "uk1_aB3cD5eF7gH9iJ1kL2mN3oP4qR6sT8uV0wX2yZ4"

# username 可以是任意值，只有 password (api_key) 会被验证
response = requests.get(
    "http://localhost:3001/metrics",
    auth=HTTPBasicAuth("automation_client", api_key)
)

print(response.text)
```

#### 使用 Node.js axios:
```javascript
const axios = require('axios');

const apiKey = "uk1_aB3cD5eF7gH9iJ1kL2mN3oP4qR6sT8uV0wX2yZ4";

axios.get('http://localhost:3001/metrics', {
    auth: {
        username: 'any_user',  // 任意值
        password: apiKey       // 实际 API Key
    }
})
.then(response => console.log(response.data))
.catch(error => console.error(error));
```

### 6.3 错误处理

| 错误场景 | HTTP 状态码 | 日志信息 |
|----------|-------------|----------|
| 无效的 API Key | 401 | `"Failed API auth attempt: invalid API Key"` |
| 速率限制超限 | 401 | `"Failed API auth attempt: rate limit exceeded"` |
| API Key 已过期 | 401 | 同上（无效 Key） |
| API Key 未激活 | 401 | 同上（无效 Key） |

---

## 7. 安全特性分析

### 7.1 安全设计亮点

1. **哈希存储**:
   - API Key 使用 bcrypt 哈希存储，不是明文
   - 即使数据库泄露，攻击者也无法直接使用 Key

2. **Key ID 分离**:
   - 使用 `uk{id}_{key}` 格式，ID 用于快速查询
   - 实际密钥是 40 位随机字符串，熵值足够

3. **过期机制**:
   - 支持设置过期时间 (`expires` 字段)
   - 过期的 Key 自动失效

4. **激活控制**:
   - 支持临时禁用/启用 Key (`active` 字段)
   - 无需删除即可吊销

5. **速率限制**:
   - 每分钟 60 次请求限制
   - 无效请求不消耗令牌，防止暴力破解

### 7.2 潜在风险与建议

1. **缺乏细粒度权限**:
   - **现状**: API Key 拥有用户全部权限
   - **建议**: 添加 scope 机制，限制 Key 的使用范围

2. **无使用审计**:
   - **现状**: 没有记录 API Key 的使用历史
   - **建议**: 添加 `api_key_usage` 表记录每次调用

3. **无 IP 白名单**:
   - **现状**: 任何 IP 都可以使用有效 Key
   - **建议**: 可选的 IP 绑定功能

4. **无密钥轮换提醒**:
   - **现状**: 过期前无提醒机制
   - **建议**: 即将过期时发送通知

---

## 8. 关键代码位置索引

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| API Key 模型 | `server/model/api_key.js` | 全文件 |
| 鉴权中间件 | `server/auth.js` | 155-176 |
| API Key 验证 | `server/auth.js` | 41-63 |
| 授权器 | `server/auth.js` | 79-97 |
| 速率限制实现 | `server/rate-limiter.js` | 全文件 |
| API Key 创建 | `server/socket-handlers/api-key-socket-handler.js` | 18-52 |
| 数据库表定义 | `db/knex_init_db.js` | 442-457 |
| /metrics 端点 | `server/server.js` | 337 |

---

## 9. 总结

Uptime Kuma 的 API Key 鉴权机制实现了基本的安全功能：

- **✅ 已实现**:
  - bcrypt 哈希存储
  - 过期时间和激活状态控制
  - HTTP Basic Auth 集成
  - 速率限制（60次/分钟）
  - 基于用户的权限模型

- **❌ 未实现**:
  - 细粒度权限范围（Scopes）
  - API 使用审计日志
  - IP 白名单绑定
  - 密钥轮换提醒

当前实现适用于简单的自动化集成场景，但对于需要更精细访问控制的企业级应用，建议扩展权限范围和审计功能。
