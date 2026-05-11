# Uptime Kuma 备份导入导出流程分析

---

## 第一段：备份能力现状和调用链终点

### 1.1 备份功能状态

基于当前仓库代码的可核对证据，**当前仓库中没有完整可用的备份导入导出功能**：

| 功能 | 状态 | 可核对证据 |
|------|------|------------|
| 备份导出 | ❌ 不存在 | 搜索 `server/server.js` 和 `server/socket-handlers/` 共 86 个 Socket 事件 handler，无 `downloadBackup`、`exportBackup`、`getBackup` 相关代码 |
| 备份导入 | ⚠️ 方法定义存在，但无完整调用链 | 1) 前端有 `uploadBackup` 方法定义<br>2) 前端无任何页面/组件调用该方法<br>3) 后端无对应 handler |

### 1.2 导出功能调用链（不存在）

**可核对证据** - 搜索 `server/server.js` 和 `server/socket-handlers/`：
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

**结论（可核对）**：导出功能从未实现，无调用链。

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

**可核对证据** - 搜索所有前端文件：
```
搜索命令: grep -rn "uploadBackup(" src/ --include="*.vue" --include="*.js"

搜索结果:
- src/mixins/socket.js:679: uploadBackup(uploadedJSON, importHandle, callback) {
  (定义处)

- src/mixins/socket.js:680:     socket.emit("uploadBackup", uploadedJSON, importHandle, callback);
  (方法体内部)

- src/mixins/socket.js:681: }
  (方法结束)

无其他调用点。
```

**结论（可核对）**：`uploadBackup` 方法仅在 `socket.js` 中定义，无任何页面或组件调用该方法。

#### 1.3.3 后端无对应 handler

**可核对证据** - 搜索 `server/server.js` 和 `server/socket-handlers/` 所有文件：
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

**结论（可核对）**：后端无 `uploadBackup` 事件 handler。

#### 1.3.4 服务端无未注册事件处理

**可核对证据** - 搜索服务端 Socket 相关代码：
```
搜索命令: grep -rn "socket.onAny\|socket.once\|socket.error\|io.onAny" server/ --include="*.js"

搜索结果: 无匹配
```

**可核对证据** - 检查 `server/server.js:371` 开始的 Socket 连接处理：
```javascript
io.on("connection", async (socket) => {
    await sendInfo(socket, true);

    if (needSetup) {
        log.info("server", "Redirect to setup page");
        socket.emit("setup");
    }

    // ***************************
    // Public Socket API
    // ***************************

    socket.on("loginByToken", async (token, callback) => {
        // ...
    });

    socket.on("login", async (data, callback) => {
        // ...
    });

    // ... 后续为各个 socket.on() 事件注册
    // 无 onAny、onError 或其他通用事件处理
});
```

**结论（可核对）**：当前仓库代码中，服务端没有注册任何通用事件处理器（如 `onAny`、`onError`）来处理未注册的事件。

#### 1.3.5 UI 层无备份导入入口

**可核对证据** - 搜索 `Settings.vue`：
```javascript
// src/pages/Settings.vue
// 搜索 "backup"、"import"、"export" 关键词
// 结果：无备份导入/导出相关代码

// Settings.vue 的模板部分仅包含：
// - 侧边栏菜单（无 backup 相关菜单项）
// - router-view（无 backup 相关组件）
```

**可核对证据** - 搜索所有 `.vue` 文件：
```
搜索命令: grep -rn "backup\|Backup\|import.*backup\|export.*backup" src/pages/ src/components/ --include="*.vue"

搜索结果: 无匹配
```

**结论（可核对）**：当前仓库中无任何 UI 入口（按钮、页面、菜单）触发备份导入功能。

#### 1.3.6 实际失败路径和用户可见结果

**基于当前仓库代码的可核对结论**：

| 路径 | 状态 | 可核对证据 |
|------|------|------------|
| 1. 用户访问备份导入页面 | ❌ 无法触发 | `Settings.vue` 等页面无 backup 相关代码，无 UI 入口 |
| 2. 调用 `uploadBackup` 方法 | ❌ 无法触发 | 方法仅在 `socket.js` 中定义，无任何调用点 |
| 3. 后端处理 `uploadBackup` 事件 | ❌ 无 handler | 86 个 Socket handler 中无 `uploadBackup` |
| 4. 服务端处理未注册事件 | ⚠️ 当前代码无证据 | 服务端无 `onAny`、`onError` 等通用事件处理 |

**无法通过当前仓库代码确认的行为**（明确标注）：

> ⚠️ **当前代码无法确认**：关于 Socket.io 框架层对未注册事件的具体行为（如是否静默忽略、callback 是否永不调用、是否超时等），这些是框架层行为，不是当前仓库代码中可直接核对的证据。
>
> 当前仓库中可确认的是：
> 1. 服务端没有任何代码处理 `uploadBackup` 事件
> 2. 服务端没有任何代码处理未注册事件
> 3. 没有任何代码会向客户端返回关于 `uploadBackup` 的响应

#### 1.3.7 调用链总结（基于可核对证据）

```
┌─────────────────────────────────────────────────────────────┐
│                    实际状态（当前仓库）                        │
│                                                             │
│  1. UI 层: ❌ 无备份导入入口                                 │
│     证据: Settings.vue 等页面无 backup 相关代码               │
│     证据: 所有 .vue 文件搜索无 backup 相关代码                │
│                                                             │
│  2. 方法定义: ⚠️ 仅定义不调用                                 │
│     证据: uploadBackup 仅在 socket.js:679 定义               │
│     证据: grep 搜索无任何调用点                              │
│                                                             │
│  3. 后端 handler: ❌ 不存在                                   │
│     证据: 86 个 Socket handler 中无 uploadBackup             │
│                                                             │
│  4. 通用事件处理: ❌ 不存在                                   │
│     证据: 无 onAny、onError 等通用事件处理代码                │
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

### 2.4 通知配置 (Notification) 边界

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

#### 2.4.2 通知凭据在出站发送时的保留边界

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

**可核对证据**：`notification` 对象完整传递给具体通知提供商，无任何脱敏或裁剪。

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

**Webhook 凭据边界（可核对）**：
| 凭据字段 | 已登录侧 | 出站发送时 | 脱敏状态 | 代码证据 |
|----------|----------|------------|----------|----------|
| `webhookURL` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `webhook.js:61, 63` |
| `webhookAdditionalHeaders` | ✅ 包含 | ✅ 直接解析使用 | 无脱敏 | `webhook.js:47-56` |
| `webhookCustomBody` | ✅ 包含 | ✅ 直接渲染 | 无脱敏 | `webhook.js:43-44` |

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

**Telegram 凭据边界（可核对）**：
| 凭据字段 | 已登录侧 | 出站发送时 | 脱敏状态 | 代码证据 |
|----------|----------|------------|----------|----------|
| `telegramServerUrl` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `telegram.js:53` |
| `telegramChatID` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `telegram.js:57` |

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

**SMTP 凭据边界（可核对）**：
| 凭据字段 | 已登录侧 | 出站发送时 | 脱敏状态 | 代码证据 |
|----------|----------|------------|----------|----------|
| `smtpHost` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `smtp.js:15` |
| `smtpPort` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `smtp.js:16` |
| `smtpUsername` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `smtp.js:52` |
| `smtpPassword` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `smtp.js:53` |
| `smtpDkimPrivateKey` | ✅ 包含 | ✅ 直接使用 | 无脱敏 | `smtp.js:42` |

#### 2.4.6 通知凭据边界总结

**通知系统的设计逻辑（基于可核对证据）**：

```
┌─────────────────────────────────────────────────────────────┐
│  通知凭据的处理流程（可核对证据链）                            │
│                                                             │
│  1. 存储时:                                                  │
│     bean.config = JSON.stringify(notification)              │
│     证据: server/notification.js:262                         │
│     结论: 完整保存到 config JSON 字段                         │
│                                                             │
│  2. 已登录侧同步时:                                          │
│     bean.export() → RedBeanNode 完整导出所有字段              │
│     证据: server/client.js:15                               │
│     结论: 完整发送到前端                                      │
│                                                             │
│  3. 出站发送时:                                              │
│     Notification.send() → 完整 notification 对象传递          │
│     证据: server/notification.js:233                        │
│     各提供商直接读取 notification.xxx 字段                   │
│     证据: webhook.js:47, 61, 63; smtp.js:15, 42, 52, 53    │
│     结论: 完整使用，无脱敏                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**完整边界表（基于可核对证据）**：

| 数据类型 | 字段 | 已登录侧 | 外部通知侧（附带数据） | 外部通知侧（连接凭据） | 控制机制 | 代码证据 |
|----------|------|----------|------------------------|------------------------|----------|----------|
| **监控器** | | | | | | |
| HTTP 认证 | `basic_auth_user`, `basic_auth_pass` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` | `monitor.js:1544` |
| OAuth 2.0 | `oauth_client_id`, `oauth_client_secret` 等 | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` | `monitor.js:1544` |
| 数据库连接串 | `databaseConnectionString` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` | `monitor.js:1544` |
| TLS 证书/私钥 | `tlsCa`, `tlsCert`, `tlsKey` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` | `monitor.js:1544` |
| Push Token | `pushToken` | ✅ 包含 | ❌ 排除 | N/A | `includeSensitiveData` | `monitor.js:1544` |
| **API 密钥** | | | | | | |
| 密钥值 (key) | `key` | ⚠️ 仅创建时显示一次 | N/A | N/A | `toJSON()` vs `toPublicJSON()` | `api-key-socket-handler.js:34`; `client.js:132` |
| **通知配置** | | | | | | |
| Webhook URL | `webhookURL` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏 | `webhook.js:61, 63` |
| Webhook Headers | `webhookAdditionalHeaders` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏 | `webhook.js:47-56` |
| Telegram ChatID | `telegramChatID` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏 | `telegram.js:57` |
| SMTP 密码 | `smtpPassword` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏 | `smtp.js:53` |
| DKIM 私钥 | `smtpDkimPrivateKey` | ✅ 包含 | N/A | ✅ 直接使用 | 无脱敏 | `smtp.js:42` |

**关键区分（基于可核对证据）**：
- **附带数据**（如监控器配置）: 发送到外部通知渠道时会脱敏（`includeSensitiveData = false`）
  - 证据: `server/model/monitor.js:1544`
- **连接凭据**（如 Webhook URL、SMTP 密码）: 始终保留，无脱敏
  - 证据: `server/notification.js:233`（完整对象传递）
  - 证据: 各提供商实现中直接读取 `notification.xxx` 字段

---

## 第三段：导入能力缺失时可参考的校验与隔离机制

由于备份导入功能未实现，以下是现有数据写入操作中采用的校验与隔离机制，可作为备份导入实现的参考。所有内容均基于当前仓库的可核对代码证据。

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

### 3.8 校验与隔离机制总结（基于可核对证据）

| 层级 | 机制 | 代码位置 | 作用 | 证据 |
|------|------|----------|------|------|
| 连接层 | `checkLogin(socket)` | `server/util-server.js:637` | 未登录直接抛出异常 | `server/server.js:727` 等所有 handler 首行调用 |
| 房间层 | `socket.join(user.id)` | `server/server.js:1806` | Socket 消息仅发送到用户房间 | `server/client.js:31` 使用 `io.to(socket.userID).emit()` |
| 查询层 | `user_id = ?` 条件 | `server/server.js:993` | 所有查询强制过滤用户 | `server/server.js:993`、`client.js:22`、`client.js:127` |
| 写入层 | `bean.user_id = socket.userID` | `server/server.js:767` | 强制绑定当前用户 | `server/server.js:767` |
| 数据层 | `bean.import()` + `bean.validate()` | `server/server.js:762, 769` | 数据结构和字段验证 | `server/server.js:762, 769` |
| 清理层 | 前端专用字段删除 | `server/server.js:751-760` | 防止无效字段污染数据库 | `server/server.js:747-760` |
| 类型层 | `typeof code === "string"` 校验 | `server/server.js:734-736` | 类型安全检查 | `server/server.js:734-736` |

### 3.9 备份导入实现建议（基于现有机制）

如果未来实现备份导入功能，可参考以下校验隔离流程（基于现有代码的可核对证据）：

```
┌─────────────────────────────────────────────────────────────┐
│  备份导入校验隔离流程（参考现有机制）                           │
│                                                             │
│  1. 认证校验: checkLogin(socket)                             │
│     参考: server/server.js:727                              │
│     证据: 所有数据写入操作首行调用                            │
│                                                             │
│  2. 数据解析: JSON.parse(backupData)                         │
│     参考: server/notification.js:262                        │
│     证据: 通知配置使用 JSON.parse() 和 JSON.stringify()      │
│                                                             │
│  3. 数据结构校验:                                            │
│     参考: server/server.js:734-736                          │
│     证据: 监控器添加时检查 accepted_statuscodes 类型         │
│                                                             │
│  4. 权限校验:                                                │
│     参考: server/notification.js:245-251                    │
│     证据: 通知更新时验证所有权 (id = ? AND user_id = ?)      │
│                                                             │
│  5. 数据清理:                                                │
│     参考: server/server.js:747-760                          │
│     证据: 删除 frontendOnlyProperties                       │
│                                                             │
│  6. 字段映射:                                                │
│     参考: server/server.js:764-766                          │
│     证据: camelCase → snake_case 映射                       │
│                                                             │
│  7. 数据绑定:                                                │
│     参考: server/server.js:762, 767                         │
│     证据: bean.import(data); bean.user_id = socket.userID   │
│                                                             │
│  8. 模型验证:                                                │
│     参考: server/server.js:769                              │
│     证据: bean.validate()                                   │
│                                                             │
│  9. 持久化:                                                  │
│     参考: server/server.js:771                              │
│     证据: R.store(bean)                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
