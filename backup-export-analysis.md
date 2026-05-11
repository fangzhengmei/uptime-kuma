# Uptime Kuma 备份导入导出流程分析

---

## 第一段：备份能力现状和调用链终点

### 1.1 备份功能状态

经过对代码库的全面搜索，**当前仓库中没有完整可用的备份导入导出功能**：

| 功能 | 状态 | 代码证据 |
|------|------|----------|
| 备份导出 | ❌ 不存在 | 无 `downloadBackup`、`exportBackup`、`getBackup` 相关代码 |
| 备份导入 | ⚠️ 前端存根，无调用链 | 前端有 `uploadBackup` 方法定义，但：<br>1) 后端无对应 handler<br>2) 前端无任何页面/组件调用该方法 |

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
// ... 共 86 个 handler，无 downloadBackup/exportBackup/getBackup
```

**结论**：导出功能从未实现，无调用链。

### 1.3 导入功能调用链（完整缺失）

#### 1.3.1 前端方法定义（仅存在，未被调用）

**前端存根方法** - `src/mixins/socket.js:672-681`：
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

#### 1.3.2 前端调用点搜索（无调用）

搜索所有前端文件：
```
- src/mixins/socket.js: 定义 uploadBackup 方法
- src/pages/*.vue: 无任何页面使用 uploadBackup
- src/components/*.vue: 无任何组件使用 uploadBackup

搜索结果：仅在 socket.js 中找到定义，无调用点。
```

**代码证据** - `src/mixins/socket.js` 方法定义但无引用：
```
定义位置: src/mixins/socket.js:679
搜索调用点: grep -rn "uploadBackup(" src/ --include="*.vue" --include="*.js"
结果: 仅在定义处出现一次，无调用
```

#### 1.3.3 后端无对应 handler

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

#### 1.3.4 实际失败路径和用户可见结果

**完整调用链分析**：

```
用户视角（当前无法触发）:
┌─────────────────────────────────────────────────────────────┐
│  UI 页面: 无备份导入按钮/界面                                  │
│  原因: Settings.vue 等页面无 backup 相关代码                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ (用户无法到达此步骤)
┌─────────────────────────────────────────────────────────────┐
│  前端方法: this.uploadBackup(uploadedJSON, importHandle, cb)  │
│  位置: src/mixins/socket.js:679-681                          │
│  状态: 方法定义存在，但无调用                                  │
└─────────────────────────────────────────────────────────────┘
         │
         ▼ (假设通过控制台手动调用)
┌─────────────────────────────────────────────────────────────┐
│  Socket.emit: socket.emit("uploadBackup", ...)               │
│  行为: 向服务端发送事件                                        │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  服务端处理: 无 handler                                       │
│  Socket.io 行为:                                             │
│    - 未注册的事件被静默忽略                                   │
│    - 不会抛出异常                                             │
│    - callback 永远不会被调用                                  │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  用户可见结果（假设调用）:                                     │
│    1. 无任何错误提示                                          │
│    2. 无任何成功提示                                          │
│    3. 页面无任何变化                                          │
│    4. 数据库无任何变化                                        │
│    5. 前端 callback 永远等待（超时或挂起）                     │
└─────────────────────────────────────────────────────────────┘
```

**Socket.io 对未注册事件的行为说明**：

Socket.io 设计上，当客户端发送一个服务端未注册的事件时：
1. 服务端不会抛出异常
2. 服务端不会记录日志（除非自定义了未处理事件的监听）
3. 客户端的 callback 永远不会被执行
4. 客户端会一直等待 callback，直到超时或连接断开

**代码证据 - 服务端无错误处理** - 搜索所有 socket 相关代码：
```javascript
// 所有 handler 都有 try-catch，但未注册的事件不会进入任何 handler
socket.on("add", async (monitor, callback) => {
    try {
        // ... 处理逻辑
        callback({ ok: true, ... });
    } catch (e) {
        callback({ ok: false, msg: e.message });  // 只有已注册事件才会走到这里
    }
});

// 未注册的 "uploadBackup" 事件:
// - 不会进入任何 try-catch
// - 不会调用 callback
// - 静默失败
```

#### 1.3.5 调用链总结

```
┌─────────────────────────────────────────────────────────────┐
│                    实际状态（当前仓库）                        │
│                                                             │
│  1. UI 层: ❌ 无备份导入按钮/页面                             │
│     证据: Settings.vue 等页面无 backup 相关代码               │
│                                                             │
│  2. 方法定义: ⚠️ 仅定义不调用                                 │
│     证据: uploadBackup 仅在 socket.js 中定义，无调用点        │
│                                                             │
│  3. 后端 handler: ❌ 不存在                                   │
│     证据: 86 个 Socket handler 中无 uploadBackup             │
│                                                             │
│  4. 假设调用行为: ⚠️ 静默失败                                 │
│     证据: Socket.io 未注册事件被静默忽略，callback 永不调用    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 第二段：敏感字段在已登录侧与外部通知侧的保留和脱敏边界

### 2.1 数据流向总览

```
┌─────────────────────────────────────────────────────────────┐
│                    数据存储层 (数据库)                         │
│                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │
│  │   monitor     │  │  notification │  │   api_key     │    │
│  │ (所有字段)     │  │  (config JSON)│  │  (key 字段)   │    │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘    │
│          │                  │                  │            │
└──────────┼──────────────────┼──────────────────┼────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│                    已登录侧 (Socket 房间)                      │
│                                                             │
│  监控器: toJSON(preloadData) → includeSensitiveData=true     │
│    ✅ 包含: HTTP 认证、OAuth、数据库连接串、TLS 证书等        │
│                                                             │
│  通知配置: bean.export() → 完整导出                          │
│    ✅ 包含: Webhook URL、Bot Token、SMTP 密码等              │
│                                                             │
│  API 密钥: toPublicJSON() → 移除 key 字段                    │
│    ❌ 不包含: 实际密钥值 (仅创建时显示一次)                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           │                  │
           │                  │ (通知凭据用于发送)
           │                  │
           ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│                 外部通知侧 (出站发送)                          │
│                                                             │
│  监控器数据 (附带在通知消息中):                                │
│    ❌ 排除: monitor.toJSON(preloadData, false)              │
│    敏感字段全排除                                            │
│                                                             │
│  通知凭据 (用于连接外部服务):                                  │
│    ✅ 保留: 直接从 notification 对象读取                      │
│    无脱敏、无裁剪、直接使用                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 监控器 (Monitor) 边界

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

**已登录侧（前端同步）- 保留所有敏感字段** - `server/uptime-kuma-server.js:256-277`：
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

**外部通知侧（出站发送）- 脱敏所有敏感字段** - `server/model/monitor.js:1541-1547`：
```javascript
await Notification.send(
    JSON.parse(notification.config),
    msg,
    monitor.toJSON(preloadData, false),  // includeSensitiveData = false
    heartbeatJSON
);
```

### 2.3 API 密钥 (APIKey) 边界

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

### 2.4 通知配置 (Notification) 边界 - 关键补充

#### 2.4.1 通知凭据的存储和已登录侧边界

**存储机制** - `server/notification.js:260-264`：
```javascript
bean.name = notification.name;
bean.user_id = userID;
bean.config = JSON.stringify(notification);  // 完整配置 JSON 化存储
bean.is_default = notification.isDefault || false;
```

**已登录侧（前端同步）** - `server/client.js:18-36`：
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

#### 2.4.2 通知凭据在出站发送时的保留边界 - 关键发现

**Notification.send 分发** - `server/notification.js:228-234`：
```javascript
static async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    if (this.providerList[notification.type]) {
        // 直接将完整 notification 对象传给具体提供商
        // notification 包含完整 config：Webhook URL、Bot Token、SMTP 密码等
        return this.providerList[notification.type].send(notification, msg, monitorJSON, heartbeatJSON);
    } else {
        throw new Error("Notification type is not supported");
    }
}
```

**关键证据**：`notification` 对象**完整传递**给具体通知提供商，无任何脱敏或裁剪。

#### 2.4.3 Webhook 提供商 - 凭据直接使用

**Webhook 实现** - `server/notification-providers/webhook.js:11-70`：
```javascript
async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    const okMsg = "Sent Successfully.";

    try {
        const httpMethod = notification.httpMethod?.toLowerCase() || "post";

        let data = {
            heartbeat: heartbeatJSON,
            monitor: monitorJSON,  // 这是已脱敏的 monitorJSON
            msg,
        };
        let config = {
            headers: {},
        };

        // ... 处理 content type

        // 直接读取 notification.webhookAdditionalHeaders（可能包含认证 Token）
        if (notification.webhookAdditionalHeaders) {
            try {
                config.headers = {
                    ...config.headers,
                    ...JSON.parse(notification.webhookAdditionalHeaders),  // 直接使用，无脱敏
                };
            } catch (err) {
                throw new Error("Additional Headers is not a valid JSON");
            }
        }

        config = this.getAxiosConfigWithProxy(config);

        // 直接使用 notification.webhookURL，无脱敏
        if (httpMethod === "get") {
            await axios.get(notification.webhookURL, config);  // webhookURL 直接使用
        } else {
            await axios.post(notification.webhookURL, data, config);  // webhookURL 直接使用
        }

        return okMsg;
    } catch (error) {
        this.throwGeneralAxiosError(error);
    }
}
```

**Webhook 凭据边界**：
| 凭据字段 | 已登录侧 | 出站发送时 | 脱敏状态 |
|----------|----------|------------|----------|
| `webhookURL` | ✅ 包含 | ✅ 直接使用 | 无脱敏 |
| `webhookAdditionalHeaders` | ✅ 包含 | ✅ 直接解析使用 | 无脱敏（可能包含 Authorization Token） |
| `webhookCustomBody` | ✅ 包含 | ✅ 直接渲染 | 无脱敏 |

#### 2.4.4 Telegram 提供商 - Bot Token 直接使用

**Telegram 实现** - `server/notification-providers/telegram.js:51-80`：
```javascript
async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    const okMsg = "Sent Successfully.";
    // 直接读取 notification.telegramServerUrl，无脱敏
    const url = notification.telegramServerUrl ?? "https://api.telegram.org";

    try {
        let params = {
            // 直接读取 notification.telegramChatID，无脱敏
            chat_id: notification.telegramChatID,
            text: msg,
            disable_notification: notification.telegramSendSilently ?? false,
            protect_content: notification.telegramProtectContent ?? false,
            link_preview_options: { is_disabled: true },
        };
        // ...

        // Bot Token 构建在 URL 中：https://api.telegram.org/bot{token}/sendMessage
        // token 从 notification 对象读取，无脱敏
    }
}
```

**Telegram 凭据边界**：
| 凭据字段 | 已登录侧 | 出站发送时 | 脱敏状态 |
|----------|----------|------------|----------|
| `telegramServerUrl` | ✅ 包含 | ✅ 直接使用 | 无脱敏 |
| `telegramBotToken` | ✅ 包含 | ✅ 直接使用 | 无脱敏（嵌入 API URL） |
| `telegramChatID` | ✅ 包含 | ✅ 直接使用 | 无脱敏 |

#### 2.4.5 SMTP 提供商 - 密码直接使用

**SMTP 实现** - `server/notification-providers/smtp.js:11-91`：
```javascript
async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    const okMsg = "Sent Successfully.";

    // 直接读取 SMTP 配置，无脱敏
    const config = {
        host: notification.smtpHost,      // 直接使用
        port: notification.smtpPort,      // 直接使用
        secure: notification.smtpSecure,  // 直接使用
    };

    // ... TLS 配置

    // DKIM 私钥直接使用
    if (notification.smtpDkimDomain) {
        config.dkim = {
            domainName: notification.smtpDkimDomain,
            keySelector: notification.smtpDkimKeySelector,
            privateKey: notification.smtpDkimPrivateKey,  // 私钥直接使用，无脱敏
            hashAlgo: notification.smtpDkimHashAlgo,
            headerFieldNames: notification.smtpDkimheaderFieldNames,
            skipFields: notification.smtpDkimskipFields,
        };
    }

    // SMTP 认证直接使用
    if (notification.smtpUsername || notification.smtpPassword) {
        config.auth = {
            user: notification.smtpUsername,  // 用户名直接使用
            pass: notification.smtpPassword,  // 密码直接使用，无脱敏
        };
    }

    // ... 发送邮件
    let transporter = nodemailer.createTransport(config);
    await transporter.sendMail({
        from: notification.smtpFrom,
        cc: notification.smtpCC,
        bcc: notification.smtpBCC,
        to: notification.smtpTo,
        // ...
    });

    return okMsg;
}
```

**SMTP 凭据边界**：
| 凭据字段 | 已登录侧 | 出站发送时 | 脱敏状态 |
|----------|----------|------------|----------|
| `smtpHost` | ✅ 包含 | ✅ 直接使用 | 无脱敏 |
| `smtpPort` | ✅ 包含 | ✅ 直接使用 | 无脱敏 |
| `smtpUsername` | ✅ 包含 | ✅ 直接使用 | 无脱敏 |
| `smtpPassword` | ✅ 包含 | ✅ 直接使用 | **无脱敏（明文传递给 nodemailer）** |
| `smtpDkimPrivateKey` | ✅ 包含 | ✅ 直接使用 | **无脱敏（私钥明文）** |

#### 2.4.6 通知凭据边界总结

**通知系统的设计逻辑**：

```
┌─────────────────────────────────────────────────────────────┐
│  通知凭据的特殊地位                                           │
│                                                             │
│  通知凭据（Webhook URL、Bot Token、SMTP 密码等）不是"附带   │
│  数据"，而是"连接凭据"。它们用于连接外部通知服务，因此：      │
│                                                             │
│  1. 存储时: 完整保存到 config JSON 字段                       │
│  2. 已登录侧: 完整发送到前端（用户需要编辑配置）              │
│  3. 出站发送时: 完整使用（必须使用真实凭据才能连接外部服务）  │
│  4. 无脱敏: 任何时候都不脱敏，因为脱敏后无法使用              │
│                                                             │
│  ⚠️ 这是合理的设计，但意味着:                                 │
│    - 通知凭据始终以明文形式存在于数据库                       │
│    - 通知凭据始终以明文形式发送到已登录用户的前端             │
│    - 通知凭据在出站发送时直接使用（这是必需的）               │
└─────────────────────────────────────────────────────────────┘
```

**完整边界表**：

| 数据类型 | 字段 | 已登录侧 | 外部通知侧（附带数据） | 外部通知侧（连接凭据） | 控制机制 |
|----------|------|----------|------------------------|------------------------|----------|
| **监控器** | | | | | |
| HTTP 认证 | `basic_auth_user`, `basic_auth_pass` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` |
| OAuth 2.0 | `oauth_client_id`, `oauth_client_secret` 等 | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` |
| 数据库连接串 | `databaseConnectionString` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` |
| TLS 证书/私钥 | `tlsCa`, `tlsCert`, `tlsKey` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` |
| Push Token | `pushToken` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` |
| **API 密钥** | | | | | |
| 密钥值 (key) | `key` | ⚠️ 仅创建时显示一次 | N/A | N/A | `toJSON()` vs `toPublicJSON()` |
| **通知配置** | | | | | |
| Webhook URL | `webhookURL` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏 |
| Webhook Headers | `webhookAdditionalHeaders` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏（可能含 Token） |
| Telegram Bot Token | `telegramBotToken` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏 |
| SMTP 密码 | `smtpPassword` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏（明文） |
| DKIM 私钥 | `smtpDkimPrivateKey` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏（明文） |

**关键区分**：
- **附带数据**（如监控器配置）: 发送到外部通知渠道时会脱敏
- **连接凭据**（如 Webhook URL、Bot Token、SMTP 密码）: 始终保留，因为它们是连接外部服务所必需的

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
┌─────────────────────────────────────────────────────────────┐
│  备份导入校验隔离流程（建议）                                 │
│                                                             │
│  1. 认证校验: checkLogin(socket)                             │
│     └── 未登录直接拒绝                                       │
│                                                             │
│  2. 数据解析: JSON.parse(backupData)                         │
│     └── 验证 JSON 格式，解析失败抛出异常                      │
│                                                             │
│  3. 数据结构校验:                                            │
│     └── 验证必需字段、数据类型、格式约束                      │
│                                                             │
│  4. 权限校验:                                                │
│     └── 验证备份数据中的 user_id 与当前用户匹配               │
│     └── 或直接忽略 user_id，使用当前 socket.userID 覆盖      │
│                                                             │
│  5. 数据清理:                                                │
│     └── 删除前端专属字段（frontendOnlyProperties）           │
│     └── 删除不应导入的字段（如 id、created_date 等）          │
│                                                             │
│  6. 字段映射:                                                │
│     └── camelCase → snake_case                              │
│                                                             │
│  7. 数据绑定:                                                │
│     └── bean.import(data)                                   │
│     └── bean.user_id = socket.userID（强制覆盖）             │
│                                                             │
│  8. 模型验证:                                                │
│     └── bean.validate()                                     │
│                                                             │
│  9. 事务处理:                                                │
│     └── 所有导入操作在一个事务中，失败回滚                    │
│                                                             │
│  10. 持久化:                                                 │
│      └── R.store(bean)                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
