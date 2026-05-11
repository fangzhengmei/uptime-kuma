# Uptime Kuma 备份导入导出流程分析

---

## 第一段：备份能力现状和调用链终点

### 1.1 备份功能状态

经过对代码库的全面搜索，**当前仓库中没有完整可用的备份导入导出功能**：

| 功能 | 状态 | 代码证据 |
|------|------|----------|
| 备份导出 | ❌ 不存在 | 无 `downloadBackup`、`exportBackup`、`getBackup` 相关代码 |
| 备份导入 | ⚠️ 前端存根 | 前端有 `uploadBackup` 方法，后端无对应 handler |

### 1.2 导出功能调用链（不存在）

**搜索结果** - `server/server.js` 和 `server/socket-handlers/`：
```javascript
// 所有已注册的 Socket 事件 handler（无备份相关）
socket.on("login", ...)
socket.on("logout", ...)
socket.on("getMonitorList", ...)
socket.on("add", ...)
socket.on("editMonitor", ...)
socket.on("getMonitor", ...)
socket.on("addNotification", ...)
socket.on("deleteNotification", ...)
socket.on("addAPIKey", ...)
socket.on("getAPIKeyList", ...)
// ... 共 47 个 handler，无 downloadBackup/exportBackup/getBackup
```

**结论**：导出功能从未实现，无调用链。

### 1.3 导入功能调用链（终点为未实现）

**前端入口** - `src/mixins/socket.js:679-681`：
```javascript
/**
 * Upload the provided backup
 * @param {string} uploadedJSON JSON to upload
 * @param {string} importHandle Type of import. If set to
 * most data in database will be replaced
 * @param {socketCB} callback Callback for socket response
 * @returns {void}
 */
uploadBackup(uploadedJSON, importHandle, callback) {
    socket.emit("uploadBackup", uploadedJSON, importHandle, callback);
}
```

**调用链终点** - 后端无对应 handler：

搜索 `server/server.js` 和 `server/socket-handlers/` 所有文件：
```
- server/server.js: 39 个 socket.on() 调用
- server/socket-handlers/api-key-socket-handler.js: 5 个
- server/socket-handlers/chart-socket-handler.js: 1 个
- server/socket-handlers/cloudflared-socket-handler.js: 5 个
- server/socket-handlers/database-socket-handler.js: 2 个
- server/socket-handlers/docker-socket-handler.js: 3 个
- server/socket-handlers/general-socket-handler.js: 5 个
- server/socket-handlers/maintenance-socket-handler.js: 11 个
- server/socket-handlers/proxy-socket-handler.js: 2 个
- server/socket-handlers/remote-browser-socket-handler.js: 3 个
- server/socket-handlers/status-page-socket-handler.js: 10 个

合计：86 个 Socket 事件 handler
未找到：socket.on("uploadBackup", ...)
```

**调用链总结**：
```
前端: uploadBackup(uploadedJSON, importHandle, callback)
         │
         ▼
    socket.emit("uploadBackup", ...)
         │
         ▼
    ┌─────────────────┐
    │  后端: 无 handler │  ← 终点
    │  调用被静默忽略  │
    └─────────────────┘
```

---

## 第二段：敏感字段在已登录侧与外部通知侧的保留和脱敏边界

### 2.1 监控器 (Monitor) 边界

**核心控制参数** - `server/model/monitor.js:117-254`：
```javascript
toJSON(preloadData = {}, includeSensitiveData = true) {
    let data = {
        // 基础数据（始终包含）
        id: this.id, name: this.name, description: this.description,
        url: this.url, type: this.type, interval: this.interval,
        // ... 其他非敏感字段（共 62 个字段）
    };

    if (includeSensitiveData) {
        data = {
            ...data,
            // 敏感字段（共 29 个）
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

### 2.2 已登录侧（前端同步）- 保留所有敏感字段

**场景 1：获取监控器列表** - `server/uptime-kuma-server.js:256-277`：
```javascript
async getMonitorJSONList(userID, monitorID = null) {
    let query = " user_id = ? ";
    let queryParams = [userID];

    if (monitorID) {
        query += "AND id = ? ";
        queryParams.push(monitorID);
    }

    let monitorList = await R.find("monitor", query + "ORDER BY weight DESC, name", queryParams);

    const monitorData = monitorList.map((monitor) => ({
        id: monitor.id,
        active: monitor.active,
        name: monitor.name,
    }));
    const preloadData = await Monitor.preparePreloadData(monitorData);

    const result = {};
    // includeSensitiveData 未传入，默认为 true
    monitorList.forEach((monitor) => (result[monitor.id] = monitor.toJSON(preloadData)));
    return result;
}
```

**场景 2：获取单个监控器** - `server/server.js:987-1006`：
```javascript
socket.on("getMonitor", async (monitorID, callback) => {
    try {
        checkLogin(socket);

        log.info("monitor", `Get Monitor: ${monitorID} User ID: ${socket.userID}`);

        let monitor = await R.findOne("monitor", " id = ? AND user_id = ? ", [monitorID, socket.userID]);
        const monitorData = [{ id: monitor.id, active: monitor.active }];
        const preloadData = await Monitor.preparePreloadData(monitorData);
        callback({
            ok: true,
            monitor: monitor.toJSON(preloadData),  // includeSensitiveData = true
        });
    } catch (e) {
        callback({
            ok: false,
            msg: e.message,
        });
    }
});
```

**场景 3：登录后完整同步** - `server/server.js:1804-1836`：
```javascript
async function afterLogin(socket, user) {
    socket.userID = user.id;
    socket.join(user.id);

    let monitorList = await server.sendMonitorList(socket);  // 包含所有敏感字段
    await Promise.allSettled([
        sendInfo(socket),
        server.sendMaintenanceList(socket),
        sendNotificationList(socket),      // 完整通知配置
        sendProxyList(socket),
        sendDockerHostList(socket),
        sendAPIKeyList(socket),            // 不包含实际密钥值
        sendRemoteBrowserList(socket),
        sendMonitorTypeList(socket),
    ]);

    await StatusPage.sendStatusPageList(io, socket);
}
```

### 2.3 外部通知侧 - 脱敏所有敏感字段

**场景：发送通知消息** - `server/model/monitor.js:1541-1547`：
```javascript
await Notification.send(
    JSON.parse(notification.config),
    msg,
    monitor.toJSON(preloadData, false),  // includeSensitiveData = false
    heartbeatJSON
);
```

### 2.4 API 密钥 (APIKey) 边界

**模型定义** - `server/model/api_key.js:24-52`：
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

**已登录侧（列表）** - `server/client.js:123-137`：
```javascript
async function sendAPIKeyList(socket) {
    const timeLogger = new TimeLogger();

    let result = [];
    const list = await R.find("api_key", "user_id=?", [socket.userID]);

    for (let bean of list) {
        result.push(bean.toPublicJSON());  // 不包含 key 字段
    }

    io.to(socket.userID).emit("apiKeyList", result);
    timeLogger.print("Sent API Key List");

    return list;
}
```

**已登录侧（创建瞬间）** - `server/socket-handlers/api-key-socket-handler.js:18-52`：
```javascript
socket.on("addAPIKey", async (key, callback) => {
    try {
        checkLogin(socket);

        let bean = await APIKey.save(key, socket.userID);
        await sendAPIKeyList(socket);

        callback({
            ok: true,
            key: bean.toJSON(),  // 包含完整 key 值（仅显示一次）
        });
    } catch (e) {
        callback({
            ok: false,
            msg: e.message,
        });
    }
});
```

### 2.5 通知配置 (Notification) 边界

**存储机制** - `server/notification.js:260-264`：
```javascript
bean.name = notification.name;
bean.user_id = userID;
bean.config = JSON.stringify(notification);  // 完整配置 JSON 化存储
bean.is_default = notification.isDefault || false;
```

**已登录侧** - `server/client.js:18-36`：
```javascript
async function sendNotificationList(socket) {
    const timeLogger = new TimeLogger();

    let result = [];
    let list = await R.find("notification", " user_id = ? ", [socket.userID]);

    for (let bean of list) {
        let notificationObject = bean.export();  // RedBeanNode 完整导出所有字段
        notificationObject.isDefault = notificationObject.isDefault === 1;
        notificationObject.active = notificationObject.active === 1;
        result.push(notificationObject);
    }

    io.to(socket.userID).emit("notificationList", result);

    timeLogger.print("Send Notification List");

    return list;
}
```

### 2.6 边界总结表

| 数据类型 | 已登录侧 | 外部通知侧 | 控制机制 |
|----------|----------|------------|----------|
| **监控器** | | | |
| HTTP 认证 | ✅ 包含 | ❌ 排除 | `includeSensitiveData` |
| OAuth 2.0 | ✅ 包含 | ❌ 排除 | `includeSensitiveData` |
| 数据库连接串 | ✅ 包含 | ❌ 排除 | `includeSensitiveData` |
| TLS 证书/私钥 | ✅ 包含 | ❌ 排除 | `includeSensitiveData` |
| RADIUS 认证 | ✅ 包含 | ❌ 排除 | `includeSensitiveData` |
| MQTT 认证 | ✅ 包含 | ❌ 排除 | `includeSensitiveData` |
| Push Token | ✅ 包含 | ❌ 排除 | `includeSensitiveData` |
| **API 密钥** | | | |
| 密钥值 (key) | ⚠️ 仅创建时显示一次 | N/A | `toJSON()` vs `toPublicJSON()` |
| **通知配置** | | | |
| Webhook URL | ✅ 包含 | N/A（发送方不需要） | `bean.export()` |
| Bot Token | ✅ 包含 | N/A | `bean.export()` |
| SMTP 密码 | ✅ 包含 | N/A | `bean.export()` |

### 2.7 边界代码路径

```
┌─────────────────────────────────────────────────────────────┐
│                    已登录侧 (Socket 房间)                      │
│                                                             │
│  登录后同步: afterLogin()                                    │
│  ├── 监控器: toJSON(preloadData) → includeSensitiveData=true │
│  ├── API 密钥: toPublicJSON() → 移除 key 字段                │
│  └── 通知配置: bean.export() → 完整导出                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  外部通知侧 (Webhook/Email 等)                 │
│                                                             │
│  发送通知: Notification.send()                               │
│  └── 监控器: toJSON(preloadData, false) → 敏感字段全排除      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 第三段：导入能力缺失时可参考的校验与隔离机制

由于备份导入功能未实现，以下是现有数据写入操作中采用的校验与隔离机制，可作为备份导入实现的参考。

### 3.1 认证校验机制

**checkLogin 定义** - `server/util-server.js:631-641`：
```javascript
/**
 * Check if a user is logged in
 * @param {Socket} socket Socket instance
 * @returns {void}
 * @throws The user is not logged in
 */
exports.checkLogin = (socket) => {
    if (!socket.userID) {
        throw new Error("You are not logged in.");
    }
};
```

**使用示例** - `server/server.js:725-733`：
```javascript
socket.on("add", async (monitor, callback) => {
    try {
        checkLogin(socket);  // 第一行必须调用
        let bean = R.dispense("monitor");
        // ... 后续操作
    } catch (e) {
        log.error("monitor", `Error adding Monitor: ${monitor.id} User ID: ${socket.userID}`);
        callback({
            ok: false,
            msg: e.message,
        });
    }
});
```

### 3.2 Socket 房间隔离

**登录时加入房间** - `server/server.js:1804-1806`：
```javascript
async function afterLogin(socket, user) {
    socket.userID = user.id;
    socket.join(user.id);  // 加入用户专属房间
    // ...
}
```

**消息发送到房间** - `server/client.js:31`：
```javascript
io.to(socket.userID).emit("notificationList", result);
```

### 3.3 数据库查询隔离

**查询用户的监控器** - `server/server.js:993`：
```javascript
let monitor = await R.findOne(
    "monitor", 
    " id = ? AND user_id = ? ",  // 双重条件确保所有权
    [monitorID, socket.userID]
);
```

**查询用户的通知** - `server/client.js:22`：
```javascript
let list = await R.find("notification", " user_id = ? ", [socket.userID]);
```

**查询用户的 API 密钥** - `server/client.js:127`：
```javascript
const list = await R.find("api_key", "user_id=?", [socket.userID]);
```

**查询用户的监控器列表** - `server/uptime-kuma-server.js:256-265`：
```javascript
async getMonitorJSONList(userID, monitorID = null) {
    let query = " user_id = ? ";  // 始终通过 user_id 过滤
    let queryParams = [userID];

    if (monitorID) {
        query += "AND id = ? ";
        queryParams.push(monitorID);
    }

    let monitorList = await R.find("monitor", query + "ORDER BY weight DESC, name", queryParams);
    // ...
}
```

### 3.4 数据写入隔离

**强制绑定当前用户** - `server/server.js:767`：
```javascript
bean.user_id = socket.userID;  // 强制绑定当前用户，防止用户注入
```

**通知保存时验证所有权** - `server/notification.js:241-286`：
```javascript
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

    bean.name = notification.name;
    bean.user_id = userID;
    bean.config = JSON.stringify(notification);
    bean.is_default = notification.isDefault || false;
    await R.store(bean);

    return bean;
}
```

### 3.5 数据类型校验

**状态码类型校验** - `server/server.js:734-736`：
```javascript
// Ensure status code ranges are strings
if (!monitor.accepted_statuscodes.every((code) => typeof code === "string")) {
    throw new Error("Accepted status codes are not all strings");
}
```

**JSON 序列化（隐式格式校验）** - `server/server.js:737-745`：
```javascript
monitor.accepted_statuscodes_json = JSON.stringify(monitor.accepted_statuscodes);
delete monitor.accepted_statuscodes;

monitor.kafkaProducerBrokers = JSON.stringify(monitor.kafkaProducerBrokers);
monitor.kafkaProducerSaslOptions = JSON.stringify(monitor.kafkaProducerSaslOptions);
monitor.conditions = JSON.stringify(monitor.conditions);
monitor.rabbitmqNodes = JSON.stringify(monitor.rabbitmqNodes);
```

### 3.6 前端专属字段清理

**防止无效字段污染数据库** - `server/server.js:747-760`：
```javascript
/*
 * List of frontend-only properties that should not be saved to the database.
 * Should clean up before saving to the database.
 */
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
```

### 3.7 RedBeanNode 数据绑定与验证

**数据导入与模型验证** - `server/server.js:762-771`：
```javascript
bean.import(monitor);  // RedBeanNode 数据绑定

// Map camelCase frontend property to snake_case database column
if (monitor.retryOnlyOnStatusCodeFailure !== undefined) {
    bean.retry_only_on_status_code_failure = monitor.retryOnlyOnStatusCodeFailure;
}
bean.user_id = socket.userID;

bean.validate();  // 模型验证

await R.store(bean);  // 持久化
```

### 3.8 校验与隔离机制总结

| 层级 | 机制 | 代码位置 | 作用 |
|------|------|----------|------|
| 连接层 | `checkLogin(socket)` | `server/util-server.js:637` | 未登录直接抛出异常 |
| 房间层 | `socket.join(user.id)` | `server/server.js:1806` | Socket 消息仅发送到用户房间 |
| 查询层 | `user_id = ?` 条件 | `server/server.js:993` | 所有查询强制过滤用户 |
| 写入层 | `bean.user_id = socket.userID` | `server/server.js:767` | 强制绑定当前用户 |
| 数据层 | `bean.import()` + `bean.validate()` | `server/server.js:762, 769` | 数据结构和字段验证 |
| 清理层 | 前端专用字段删除 | `server/server.js:751-760` | 防止无效字段污染数据库 |
| 类型层 | `typeof code === "string"` 校验 | `server/server.js:734-736` | 类型安全检查 |

### 3.9 备份导入实现建议（基于现有机制）

如果未来实现备份导入功能，应遵循以下校验隔离流程：

```
1. 认证校验: checkLogin(socket)
2. 数据解析: JSON.parse(backupData)，验证格式
3. 权限校验: 验证数据中的 user_id 与当前用户匹配
4. 数据清理: 删除前端专属字段
5. 字段映射: camelCase → snake_case
6. 数据绑定: bean.import(data)
7. 强制隔离: bean.user_id = socket.userID（覆盖导入数据）
8. 模型验证: bean.validate()
9. 持久化: R.store(bean)
```
