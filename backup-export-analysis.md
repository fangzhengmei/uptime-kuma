# Uptime Kuma 备份导入导出流程分析

## 1. 备份能力状态确认

### 1.1 当前仓库状态

经过对代码库的全面搜索，**当前仓库中没有完整可用的备份功能**：

| 功能 | 状态 | 说明 |
|------|------|------|
| 备份导出 | ❌ 不存在 | 无 `downloadBackup`、`exportBackup`、`getBackup` 等相关代码 |
| 备份导入 | ⚠️ 前端存根 | 前端有 `uploadBackup` 方法，但后端无对应 handler |

### 1.2 代码证据

**前端存根方法** - `src/mixins/socket.js:679-681`:
```javascript
uploadBackup(uploadedJSON, importHandle, callback) {
    socket.emit("uploadBackup", uploadedJSON, importHandle, callback);
}
```

**后端 Socket Handler 完整列表** - `server/server.js` 和 `server/socket-handlers/`:
- `login`, `logout`, `prepare2FA`, `save2FA` 等认证相关
- `add`, `editMonitor`, `deleteMonitor` 等监控器管理
- `addNotification`, `deleteNotification` 等通知管理
- `addAPIKey`, `getAPIKeyList`, `deleteAPIKey` 等 API 密钥管理
- **无 `uploadBackup` handler**

### 1.3 结论

当前代码库的备份功能处于**未完成状态**：
- 前端保留了调用接口的存根
- 后端没有对应的实现逻辑
- 实际的数据备份操作需要通过其他方式（如直接导出数据库）

---

## 2. 登录后数据同步流程（误判为备份导出的机制）

### 2.1 核心同步函数

用户登录后，系统通过 `afterLogin()` 函数同步所有用户数据：

**位置**: `server/server.js:1804-1836`

```javascript
async function afterLogin(socket, user) {
    socket.userID = user.id;
    socket.join(user.id);

    let monitorList = await server.sendMonitorList(socket);
    await Promise.allSettled([
        sendInfo(socket),
        server.sendMaintenanceList(socket),
        sendNotificationList(socket),
        sendProxyList(socket),
        sendDockerHostList(socket),
        sendAPIKeyList(socket),
        sendRemoteBrowserList(socket),
        sendMonitorTypeList(socket),
    ]);

    await StatusPage.sendStatusPageList(io, socket);
    // ... 发送心跳历史等
}
```

### 2.2 数据同步边界

所有发送到前端的数据都通过 **Socket.io** 通道，且**仅发送给已认证用户**：

1. **认证检查**: 所有数据操作前调用 `checkLogin(socket)`
2. **用户隔离**: 数据库查询通过 `user_id = ?` 条件过滤
3. **Socket 房间**: 登录后加入 `user.id` 房间，消息仅发送到该房间

---

## 3. 导出包含的敏感字段

### 3.1 监控器 (Monitor) 敏感字段

**位置**: `server/model/monitor.js:117-254`

监控器的 `toJSON()` 方法通过 `includeSensitiveData` 参数控制敏感字段暴露：

```javascript
toJSON(preloadData = {}, includeSensitiveData = true) {
    // 基础数据（始终包含）
    let data = {
        id: this.id, name: this.name, description: this.description,
        url: this.url, type: this.type, interval: this.interval,
        // ... 其他非敏感字段
    };

    if (includeSensitiveData) {
        data = {
            ...data,
            headers: this.headers,
            body: this.body,
            grpcBody: this.grpcBody,
            grpcMetadata: this.grpcMetadata,
            basic_auth_user: this.basic_auth_user,
            basic_auth_pass: this.basic_auth_pass,
            oauth_client_id: this.oauth_client_id,
            oauth_client_secret: this.oauth_client_secret,
            oauth_token_url: this.oauth_token_url,
            oauth_scopes: this.oauth_scopes,
            oauth_audience: this.oauth_audience,
            oauth_auth_method: this.oauth_auth_method,
            pushToken: this.pushToken,
            databaseConnectionString: this.databaseConnectionString,
            radiusUsername: this.radiusUsername,
            radiusPassword: this.radiusPassword,
            radiusSecret: this.radiusSecret,
            mqttUsername: this.mqttUsername,
            mqttPassword: this.mqttPassword,
            mqttWebsocketPath: this.mqttWebsocketPath,
            authWorkstation: this.authWorkstation,
            authDomain: this.authDomain,
            tlsCa: this.tlsCa,
            tlsCert: this.tlsCert,
            tlsKey: this.tlsKey,
            kafkaProducerSaslOptions: JSON.parse(this.kafkaProducerSaslOptions),
            rabbitmqUsername: this.rabbitmqUsername,
            rabbitmqPassword: this.rabbitmqPassword,
        };
    }

    data.includeSensitiveData = includeSensitiveData;
    return data;
}
```

### 3.2 监控器敏感字段分类

| 类别 | 字段 | 敏感度 | 说明 |
|------|------|--------|------|
| HTTP 认证 | `basic_auth_user`, `basic_auth_pass` | 高 | HTTP Basic Auth 凭证 |
| OAuth 2.0 | `oauth_client_id`, `oauth_client_secret`, `oauth_token_url`, `oauth_scopes`, `oauth_audience`, `oauth_auth_method` | 高 | OAuth 认证完整配置 |
| HTTP 自定义 | `headers`, `body`, `grpcBody`, `grpcMetadata` | 中 | 可能包含认证 Token |
| 数据库 | `databaseConnectionString` | 极高 | 完整连接字符串含密码 |
| RADIUS | `radiusUsername`, `radiusPassword`, `radiusSecret` | 高 | RADIUS 认证完整信息 |
| MQTT | `mqttUsername`, `mqttPassword`, `mqttWebsocketPath` | 高 | MQTT 连接认证 |
| TLS/SSL | `tlsCa`, `tlsCert`, `tlsKey` | 极高 | 证书和私钥 |
| Windows 认证 | `authWorkstation`, `authDomain` | 中 | NTLM/Kerberos 相关 |
| Kafka | `kafkaProducerSaslOptions` | 高 | SASL 认证配置 |
| RabbitMQ | `rabbitmqUsername`, `rabbitmqPassword` | 高 | RabbitMQ 连接凭证 |
| Push Token | `pushToken` | 高 | 被动监控的认证令牌 |

### 3.3 API 密钥 (APIKey) 敏感字段

**位置**: `server/model/api_key.js:24-52`

```javascript
toJSON() {
    return {
        id: this.id,
        key: this.key,        // 完整密钥
        name: this.name,
        userID: this.user_id,
        createdDate: this.created_date,
        active: this.active,
        expires: this.expires,
        status: this.getStatus(),
    };
}

toPublicJSON() {
    return {
        id: this.id,
        // key: this.key,      // 已移除
        name: this.name,
        userID: this.user_id,
        createdDate: this.created_date,
        active: this.active,
        expires: this.expires,
        status: this.getStatus(),
    };
}
```

### 3.4 通知配置 (Notification) 敏感字段

**位置**: `server/notification.js:260-264`

```javascript
bean.name = notification.name;
bean.user_id = userID;
bean.config = JSON.stringify(notification);  // 完整配置 JSON 化存储
bean.is_default = notification.isDefault || false;
```

通知配置的 `config` 字段包含**完整的通知提供商配置**，根据不同通知类型可能包含：

| 通知类型 | 可能包含的敏感字段 |
|----------|-------------------|
| Webhook | Webhook URL, Auth Token, Header 认证 |
| Telegram | Bot Token |
| Slack / Discord | Webhook URL |
| Email (SMTP) | SMTP Host, Port, Username, Password |
| Twilio | Account SID, Auth Token, Phone Number |
| Teams | Webhook URL |
| 短信服务 (SMS) | API Key, API Secret |

### 3.5 代理配置 (Proxy)

**位置**: `server/client.js:104-116`

```javascript
async function sendProxyList(socket) {
    const list = await R.find("proxy", " user_id = ? ", [socket.userID]);
    io.to(socket.userID).emit(
        "proxyList",
        list.map((bean) => bean.export())  // RedBeanNode 完整导出
    );
}
```

代理配置使用 RedBeanNode 的 `export()` 方法，**导出所有数据库字段**，包括代理的认证信息。

---

## 4. 敏感字段的脱敏/保留边界

### 4.1 监控器 (Monitor) 边界

**保留场景** (`includeSensitiveData = true`):

```javascript
// server/uptime-kuma-server.js:275
monitorList.forEach((monitor) => (
    result[monitor.id] = monitor.toJSON(preloadData)
));

// server/server.js:998
callback({
    ok: true,
    monitor: monitor.toJSON(preloadData),
});
```

**脱敏场景** (`includeSensitiveData = false`):

```javascript
// server/model/monitor.js:1544
// 通知消息中的监控器数据（发送到外部通知渠道）
await Notification.send(
    JSON.parse(notification.config),
    msg,
    monitor.toJSON(preloadData, false),  // includeSensitiveData = false
    heartbeatJSON
);
```

**边界总结**:

| 调用场景 | includeSensitiveData | 敏感字段 | 说明 |
|----------|---------------------|----------|------|
| `getMonitorList` | `true` (默认) | ✅ 包含 | 前端展示/编辑监控器 |
| `getMonitor` | `true` (默认) | ✅ 包含 | 编辑单个监控器 |
| 通知消息 | `false` | ❌ 排除 | 防止敏感数据泄露到外部通知渠道 |
| 公开状态页面 | N/A | ❌ 排除 | 使用 `toPublicJSON()` 方法 |

### 4.2 API 密钥 (APIKey) 边界

**保留场景** (`toJSON()`):

```javascript
// server/socket-handlers/api-key-socket-handler.js:18-52
// 添加 API 密钥后返回完整密钥（仅显示一次）
socket.on("addAPIKey", async (key, callback) => {
    // ...
    callback({
        ok: true,
        key: bean.toJSON(),  // 包含完整 key 值
    });
});
```

**脱敏场景** (`toPublicJSON()`):

```javascript
// server/client.js:123-137
async function sendAPIKeyList(socket) {
    let result = [];
    const list = await R.find("api_key", "user_id=?", [socket.userID]);

    for (let bean of list) {
        result.push(bean.toPublicJSON());  // 移除 key 字段
    }

    io.to(socket.userID).emit("apiKeyList", result);
}
```

**边界总结**:

| 场景 | 使用方法 | 密钥值 | 说明 |
|------|----------|--------|------|
| 创建 API 密钥 | `toJSON()` | ✅ 包含 | 仅创建瞬间可见一次 |
| API 密钥列表 | `toPublicJSON()` | ❌ 不包含 | 列表中永不显示密钥 |

### 4.3 状态页面 (StatusPage) 边界

**保留场景** (`toJSON()`):

```javascript
// server/model/status_page.js:361-371
// 登录用户获取管理后台数据
static async sendStatusPageList(io, socket) {
    let result = {};
    let list = await R.findAll("status_page", " ORDER BY title ");
    for (let item of list) {
        result[item.id] = await item.toJSON();  // 完整数据
    }
    io.to(socket.userID).emit("statusPageList", result);
}
```

**脱敏场景** (`toPublicJSON()`):

```javascript
// server/model/status_page.js:309-340
// 公开访问的数据
static async getStatusPageData(statusPage) {
    const config = await statusPage.toPublicJSON();  // 公开数据
    // ...
}
```

### 4.4 通知配置 (Notification) 边界

通知配置**没有分级保护机制**，始终使用完整导出：

```javascript
// server/client.js:18-36
async function sendNotificationList(socket) {
    let result = [];
    let list = await R.find("notification", " user_id = ? ", [socket.userID]);

    for (let bean of list) {
        let notificationObject = bean.export();  // RedBeanNode 完整导出
        // ...
        result.push(notificationObject);
    }

    io.to(socket.userID).emit("notificationList", result);
}
```

**边界总结**:

| 场景 | 使用方法 | 敏感配置 | 说明 |
|------|----------|----------|------|
| 通知列表 | `bean.export()` | ✅ 包含 | 前端展示/编辑通知 |
| 发送通知 | 直接解析 `config` | ✅ 使用 | 运行时需要完整配置 |

---

## 5. 数据写入时的校验与隔离

虽然没有专门的备份导入功能，但所有数据写入操作（如添加/编辑监控器、保存通知配置）都有统一的校验和隔离机制。

### 5.1 认证校验

**位置**: `server/util-server.js:637-641`

```javascript
exports.checkLogin = (socket) => {
    if (!socket.userID) {
        throw new Error("You are not logged in.");
    }
};
```

所有 Socket 事件 handler 在执行操作前都会调用 `checkLogin()`：

```javascript
// server/server.js:725-797
socket.on("add", async (monitor, callback) => {
    try {
        checkLogin(socket);  // 第一行检查
        // ... 后续操作
    } catch (e) {
        // ...
    }
});
```

### 5.2 用户隔离机制

#### 5.2.1 Socket 房间隔离

```javascript
// server/server.js:1804-1806
async function afterLogin(socket, user) {
    socket.userID = user.id;
    socket.join(user.id);  // 加入用户专属房间
}
```

#### 5.2.2 数据库查询隔离

```javascript
// server/server.js:993
let monitor = await R.findOne(
    "monitor", 
    " id = ? AND user_id = ? ",  // 双重条件确保所有权
    [monitorID, socket.userID]
);

// server/uptime-kuma-server.js:257-258
async getMonitorJSONList(userID, monitorID = null) {
    let query = " user_id = ? ";  // 始终通过 user_id 过滤
    let queryParams = [userID];
    // ...
}
```

#### 5.2.3 数据写入隔离

```javascript
// server/server.js:767
bean.user_id = socket.userID;  // 强制绑定当前用户
```

### 5.3 数据校验机制

#### 5.3.1 监控器字段校验

```javascript
// server/server.js:734-760
// 类型校验
if (!monitor.accepted_statuscodes.every((code) => typeof code === "string")) {
    throw new Error("Accepted status codes are not all strings");
}

// JSON 序列化（隐式格式校验）
monitor.accepted_statuscodes_json = JSON.stringify(monitor.accepted_statuscodes);
monitor.kafkaProducerBrokers = JSON.stringify(monitor.kafkaProducerBrokers);
monitor.kafkaProducerSaslOptions = JSON.stringify(monitor.kafkaProducerSaslOptions);
monitor.conditions = JSON.stringify(monitor.conditions);
monitor.rabbitmqNodes = JSON.stringify(monitor.rabbitmqNodes);

// 清理前端专属字段（防止注入）
const frontendOnlyProperties = [
    "humanReadableInterval",
    "globalpingdnsresolvetypeoptions",
    "responsecheck",
];
for (const prop of frontendOnlyProperties) {
    if (prop in monitor) {
        delete monitor[prop];
    }
}

// RedBeanNode 导入（数据绑定）
bean.import(monitor);

// 字段名映射
if (monitor.retryOnlyOnStatusCodeFailure !== undefined) {
    bean.retry_only_on_status_code_failure = monitor.retryOnlyOnStatusCodeFailure;
}

// 模型验证
bean.validate();

await R.store(bean);
```

#### 5.3.2 通知配置校验

```javascript
// server/notification.js:241-271
static async save(notification, notificationID, userID) {
    if (notificationID) {
        // 更新现有通知 - 验证所有权
        let bean = await R.findOne(
            "notification", 
            " id = ? AND user_id = ? ",
            [notificationID, userID]
        );
        if (!bean) {
            throw new Error("notification not found");
        }
    } else {
        bean = R.dispense("notification");
    }

    // 序列化存储
    bean.config = JSON.stringify(notification);

    await R.store(bean);
    return bean;
}
```

### 5.4 隔离机制总结

| 层级 | 机制 | 代码位置 | 说明 |
|------|------|----------|------|
| 连接层 | `checkLogin(socket)` | `server/util-server.js:637` | 未登录直接抛出异常 |
| 房间层 | `socket.join(user.id)` | `server/server.js:1806` | 消息仅发送到用户房间 |
| 查询层 | `user_id = ?` 条件 | `server/uptime-kuma-server.js:257` | 所有查询强制过滤用户 |
| 写入层 | `bean.user_id = socket.userID` | `server/server.js:767` | 强制绑定当前用户 |
| 数据层 | `bean.import()` + `bean.validate()` | `server/server.js:762, 769` | 数据结构和字段验证 |
| 清理层 | 前端专用字段删除 | `server/server.js:751-760` | 防止无效字段污染数据库 |

---

## 6. 安全架构总结

### 6.1 保护层级

```
┌─────────────────────────────────────────────────────────────┐
│                    已认证用户 (Socket 房间)                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              数据同步 (afterLogin)                      │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  监控器 (includeSensitiveData = true)            │  │  │
│  │  │  ✅ 包含: 密码、密钥、连接字符串、证书            │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  API 密钥 (toPublicJSON)                         │  │  │
│  │  │  ❌ 移除: 实际密钥值 (仅创建时显示一次)          │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  通知配置 (bean.export)                          │  │  │
│  │  │  ✅ 包含: 完整 config (含 Webhook Token 等)     │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              └──────────── 隔离边界
                              │
┌─────────────────────────────────────────────────────────────┐
│                    公开访问 (HTTP API)                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  状态页面 (toPublicJSON)                               │  │
│  │  ❌ 移除: 内部管理数据                                  │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  推送接口 (pushToken 认证)                              │  │
│  │  ✅ 仅验证 Token, 无敏感数据返回                        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              └──────────── 外部通知
                              │
┌─────────────────────────────────────────────────────────────┐
│              通知渠道 (Webhook/Email/Telegram 等)              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  监控器数据 (includeSensitiveData = false)              │  │
│  │  ❌ 排除: 所有敏感字段                                   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 关键发现

1. **已认证用户可获取完整敏感配置**: 登录后，监控器的密码、密钥、证书等都会发送到前端
2. **API 密钥有额外保护**: 密钥值仅在创建瞬间显示一次，列表中永不显示
3. **通知配置无分级保护**: 完整 `config` 始终发送到前端（包含 Webhook Token 等）
4. **无专门备份功能**: 当前仓库没有备份导出/导入的实现
5. **用户隔离是核心保护**: 所有操作通过 `user_id` 严格隔离，配合 `checkLogin()` 认证

### 6.3 代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| 登录后数据同步 | `server/server.js` | 1804-1836 |
| 监控器 `toJSON()` | `server/model/monitor.js` | 117-254 |
| API 密钥 `toPublicJSON()` | `server/model/api_key.js` | 42-52 |
| 通知列表发送 | `server/client.js` | 18-36 |
| 认证检查 | `server/util-server.js` | 637-641 |
| 添加监控器校验 | `server/server.js` | 725-797 |
| 通知保存 | `server/notification.js` | 241-271 |
