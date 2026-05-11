# Uptime Kuma 备份导入导出流程分析

## 1. 概述

Uptime Kuma 的备份导入导出功能是基于 Socket.io 事件实现的。系统通过精心设计的数据模型和序列化机制来保护敏感配置信息，确保在数据导出和传输过程中的安全性。

## 2. 架构概览

### 2.1 备份功能位置

- **前端触发**: `src/mixins/socket.js` 中的 `uploadBackup()` 方法
- **数据来源**: 前端通过 Socket.io 从服务器获取各类数据
- **数据模型**: 核心模型位于 `server/model/` 目录

### 2.2 数据传输机制

系统使用 Socket.io 进行实时数据传输，主要数据类型包括：
- 监控器列表 (monitorList)
- 通知配置列表 (notificationList)
- 代理配置列表 (proxyList)
- API 密钥列表 (apiKeyList)
- 维护计划列表 (maintenanceList)
- 状态页面配置 (statusPageList)

## 3. 敏感数据保护机制

### 3.1 监控器模型 (Monitor)

**文件位置**: `server/model/monitor.js`

#### 3.1.1 核心保护方法

监控器模型提供了三个关键的序列化方法：

1. **`toPublicJSON(showTags, certExpiry)`** - 用于公开状态页面的最小化数据
2. **`toJSON(preloadData, includeSensitiveData)`** - 用于内部使用的完整数据
3. **`includeSensitiveData`** 标志控制敏感字段的暴露

#### 3.1.2 敏感字段列表

当 `includeSensitiveData = false` 时，以下字段会被**排除**：

| 字段名 | 描述 | 用途 |
|--------|------|------|
| `headers` | HTTP 请求头 | 自定义 HTTP 头信息 |
| `body` | HTTP 请求体 | POST/PUT 请求数据 |
| `grpcBody` | gRPC 请求体 | gRPC 服务请求数据 |
| `grpcMetadata` | gRPC 元数据 | gRPC 认证和元数据 |
| `basic_auth_user` | 基础认证用户名 | HTTP Basic Auth 用户名 |
| `basic_auth_pass` | 基础认证密码 | HTTP Basic Auth 密码 |
| `oauth_client_id` | OAuth 客户端 ID | OAuth 2.0 认证 |
| `oauth_client_secret` | OAuth 客户端密钥 | OAuth 2.0 认证 |
| `oauth_token_url` | OAuth Token URL | OAuth 2.0 认证地址 |
| `oauth_scopes` | OAuth 权限范围 | OAuth 2.0 权限 |
| `oauth_audience` | OAuth 受众 | OAuth 2.0 受众标识 |
| `oauth_auth_method` | OAuth 认证方法 | OAuth 2.0 认证方式 |
| `pushToken` | Push 监控 Token | 被动监控的认证令牌 |
| `databaseConnectionString` | 数据库连接字符串 | 数据库监控的连接信息 |
| `radiusUsername` | RADIUS 用户名 | RADIUS 认证监控 |
| `radiusPassword` | RADIUS 密码 | RADIUS 认证监控 |
| `radiusSecret` | RADIUS 共享密钥 | RADIUS 协议密钥 |
| `mqttUsername` | MQTT 用户名 | MQTT 监控认证 |
| `mqttPassword` | MQTT 密码 | MQTT 监控认证 |
| `mqttWebsocketPath` | MQTT WebSocket 路径 | MQTT 连接配置 |
| `authWorkstation` | NTLM 认证工作站 | Windows 认证 |
| `authDomain` | NTLM 认证域 | Windows 域认证 |
| `tlsCa` | TLS CA 证书 | TLS/SSL 认证 |
| `tlsCert` | TLS 客户端证书 | TLS/SSL 认证 |
| `tlsKey` | TLS 私钥 | TLS/SSL 认证 |
| `kafkaProducerSaslOptions` | Kafka SASL 配置 | Kafka 安全认证 |
| `rabbitmqUsername` | RabbitMQ 用户名 | RabbitMQ 监控认证 |
| `rabbitmqPassword` | RabbitMQ 密码 | RabbitMQ 监控认证 |

#### 3.1.3 非敏感字段（始终包含）

以下字段无论 `includeSensitiveData` 如何设置都会包含：
- 基础信息: `id`, `name`, `description`, `type`, `url`, `interval`
- 状态信息: `active`, `forceInactive`, `maxretries`, `timeout`
- 标签和分组: `tags`, `notificationIDList`, `maintenance`
- 协议配置: `method`, `hostname`, `port`, `protocol`

### 3.2 API 密钥模型 (APIKey)

**文件位置**: `server/model/api_key.js`

#### 3.2.1 保护策略

API 密钥模型采用了两层保护：

1. **`toJSON()`** - 包含完整密钥（仅在创建时使用一次）
2. **`toPublicJSON()`** - 移除密钥值，仅保留元数据

#### 3.2.2 字段对比

| 字段 | toJSON() | toPublicJSON() | 说明 |
|------|----------|----------------|------|
| `id` | ✅ | ✅ | 数据库 ID |
| `name` | ✅ | ✅ | 密钥名称 |
| `key` | ✅ | ❌ | 实际密钥值（敏感） |
| `userID` | ✅ | ✅ | 所属用户 |
| `createdDate` | ✅ | ✅ | 创建日期 |
| `active` | ✅ | ✅ | 激活状态 |
| `expires` | ✅ | ✅ | 过期时间 |
| `status` | ✅ | ✅ | 当前状态 |

#### 3.2.3 发送到客户端的处理

在 `server/client.js` 的 `sendAPIKeyList()` 方法中：

```javascript
async function sendAPIKeyList(socket) {
    let result = [];
    const list = await R.find("api_key", "user_id=?", [socket.userID]);

    for (let bean of list) {
        result.push(bean.toPublicJSON());  // 使用 toPublicJSON 移除密钥
    }

    io.to(socket.userID).emit("apiKeyList", result);
}
```

### 3.3 通知配置模型 (Notification)

**文件位置**: `server/notification.js`

#### 3.3.1 存储机制

通知配置存储在 `notification` 表的 `config` 字段中，采用 JSON 字符串格式：

```javascript
bean.config = JSON.stringify(notification);
```

#### 3.3.2 敏感字段

通知配置包含多种通知提供商的敏感信息，包括但不限于：

| 通知类型 | 敏感字段示例 |
|----------|--------------|
| Webhook | Webhook URL、认证令牌 |
| Telegram | Bot Token |
| Slack | Webhook URL |
| Email (SMTP) | SMTP 密码、API Key |
| 短信服务 | API Key、Secret |
| Teams | Webhook URL |

#### 3.3.3 传输到客户端

在 `server/client.js` 中，通知使用 RedBeanNode 的 `export()` 方法：

```javascript
async function sendNotificationList(socket) {
    let result = [];
    let list = await R.find("notification", " user_id = ? ", [socket.userID]);

    for (let bean of list) {
        let notificationObject = bean.export();  // 导出所有字段
        // ... 处理布尔值转换
        result.push(notificationObject);
    }

    io.to(socket.userID).emit("notificationList", result);
}
```

**注意**: RedBeanNode 的 `export()` 方法会导出所有数据库字段，包括包含敏感配置的 `config` 字段。这意味着通知的完整配置（包括密钥和密码）会被发送到已登录用户的前端。

### 3.4 状态页面模型 (StatusPage)

**文件位置**: `server/model/status_page.js`

#### 3.4.1 公开和私有数据分离

状态页面提供了两个序列化方法：

1. **`toJSON()`** - 管理后台使用的完整数据
2. **`toPublicJSON()`** - 公开访问使用的最小化数据

#### 3.4.2 字段对比

| 字段 | toJSON() | toPublicJSON() | 说明 |
|------|----------|----------------|------|
| `id` | ✅ | ❌ | 数据库 ID（内部使用） |
| `slug` | ✅ | ✅ | URL 友好标识符 |
| `title` | ✅ | ✅ | 页面标题 |
| `description` | ✅ | ✅ | 页面描述 |
| `icon` | ✅ | ✅ | 图标 |
| `theme` | ✅ | ✅ | 主题设置 |
| `published` | ✅ | ✅ | 发布状态 |
| `domainNameList` | ✅ | ❌ | 自定义域名（内部管理） |

### 3.5 其他模型

#### 3.5.1 维护计划 (Maintenance)

**文件位置**: `server/model/maintenance.js`

- `toPublicJSON()` - 用于状态页面的公开数据
- `toJSON()` - 完整的管理数据

#### 3.5.2 心跳记录 (Heartbeat)

**文件位置**: `server/model/heartbeat.js`

- `toPublicJSON()` - 移除敏感的响应体信息
- `toJSON()` / `toJSONAsync()` - 完整数据，用于调试和管理

#### 3.5.3 事件记录 (Incident)

**文件位置**: `server/model/incident.js`

- `toPublicJSON()` - 公开显示的事件信息

## 4. 数据流向分析

### 4.1 登录后数据同步流程

用户登录后，系统通过 `afterLogin()` 函数（`server/server.js:1804`）发送所有必要数据：

```javascript
async function afterLogin(socket, user) {
    socket.userID = user.id;
    socket.join(user.id);

    let monitorList = await server.sendMonitorList(socket);
    await Promise.allSettled([
        sendInfo(socket),
        server.sendMaintenanceList(socket),
        sendNotificationList(socket),      // 包含完整通知配置
        sendProxyList(socket),
        sendDockerHostList(socket),
        sendAPIKeyList(socket),            // 使用 toPublicJSON，无密钥
        sendRemoteBrowserList(socket),
        sendMonitorTypeList(socket),
    ]);
}
```

### 4.2 监控器列表获取

**位置**: `server/uptime-kuma-server.js:256-277`

```javascript
async getMonitorJSONList(userID, monitorID = null) {
    // ... 查询数据库 ...
    monitorList.forEach((monitor) => (
        result[monitor.id] = monitor.toJSON(preloadData)  // includeSensitiveData = true
    ));
    return result;
}
```

**关键发现**: 
- 默认情况下，`toJSON()` 的第二个参数 `includeSensitiveData` 默认为 `true`
- 这意味着监控器的完整敏感配置会被发送到已登录用户的前端

### 4.3 单个监控器获取

**位置**: `server/server.js:987-1006`

```javascript
socket.on("getMonitor", async (monitorID, callback) => {
    // ...
    let monitor = await R.findOne("monitor", " id = ? AND user_id = ? ", [monitorID, socket.userID]);
    callback({
        ok: true,
        monitor: monitor.toJSON(preloadData),  // 同样默认包含敏感数据
    });
});
```

## 5. 备份导入流程

### 5.1 前端触发

**位置**: `src/mixins/socket.js:679-681`

```javascript
uploadBackup(uploadedJSON, importHandle, callback) {
    socket.emit("uploadBackup", uploadedJSON, importHandle, callback);
}
```

### 5.2 导入处理策略

备份导入功能允许用户上传 JSON 格式的备份数据进行恢复。系统支持两种导入模式：
1. **合并模式**: 将备份数据与现有数据合并
2. **替换模式**: 替换数据库中的大部分数据

## 6. 安全考量与建议

### 6.1 当前实现的安全特性

1. **认证保护**: 所有数据操作都需要通过 `checkLogin(socket)` 验证
2. **用户隔离**: 每个用户只能访问自己的数据（通过 `user_id` 过滤）
3. **API 密钥保护**: API 密钥值在列表中不显示，只在创建时显示一次
4. **公开页面隔离**: 状态页面使用 `toPublicJSON()` 限制公开数据
5. **密码哈希**: 用户密码使用 bcrypt 哈希存储（`server/password-hash.js`）
6. **JWT 认证**: 使用 JWT 令牌进行无状态认证
7. **2FA 支持**: 支持双因素认证增强安全性

### 6.2 潜在风险点

1. **通知配置完全暴露**: 通知的完整配置（包括 API 密钥）会发送到前端
2. **监控器敏感数据**: 监控器的密码、密钥等敏感信息在前端可访问
3. **备份文件安全**: 备份 JSON 文件包含所有敏感配置，需要妥善保管

### 6.3 安全最佳实践

1. **备份文件加密**: 建议对导出的备份文件进行加密存储
2. **访问控制**: 限制能够访问备份功能的用户权限
3. **传输安全**: 始终使用 HTTPS 加密所有通信
4. **密钥轮换**: 定期轮换 API 密钥和访问令牌
5. **审计日志**: 记录所有备份导入导出操作

## 7. 关键代码位置汇总

| 功能模块 | 文件路径 | 关键方法/函数 |
|----------|----------|--------------|
| 监控器模型 | `server/model/monitor.js` | `toJSON()`, `toPublicJSON()` |
| API 密钥模型 | `server/model/api_key.js` | `toJSON()`, `toPublicJSON()` |
| 通知模型 | `server/notification.js` | `save()`, `delete()` |
| 状态页面模型 | `server/model/status_page.js` | `toJSON()`, `toPublicJSON()` |
| 客户端数据发送 | `server/client.js` | `send*List()` 系列函数 |
| 监控器列表 | `server/uptime-kuma-server.js` | `getMonitorJSONList()` |
| 登录后同步 | `server/server.js` | `afterLogin()` |
| Socket 事件处理 | `server/server.js` | `socket.on()` 事件监听器 |
| 前端 Socket | `src/mixins/socket.js` | `uploadBackup()` 等 |
| 密码哈希 | `server/password-hash.js` | `generate()`, `verify()` |
| 认证检查 | `server/util-server.js` | `checkLogin()`, `doubleCheckPassword()` |

## 8. 总结

Uptime Kuma 采用了多层级的敏感数据保护策略：

1. **模型层面**: 通过 `toJSON()` 和 `toPublicJSON()` 方法分离公开和私有数据
2. **API 密钥**: 创建后永不完整显示，只在创建瞬间可见
3. **访问控制**: 通过用户认证和 `user_id` 过滤确保数据隔离
4. **传输层**: 依赖 HTTPS 进行加密传输

对于备份功能，系统设计上允许导出完整配置（包括敏感信息），这是为了确保备份的可恢复性。用户在使用备份功能时应注意：
- 备份文件包含所有敏感配置，请妥善保管
- 建议在安全环境中处理备份文件
- 导入前验证备份来源的可信度

系统通过提供精细粒度的控制（如 `includeSensitiveData` 参数），在功能性和安全性之间取得了平衡。
