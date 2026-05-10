# Uptime Kuma 监控类型配置共享机制分析

## 一、整体架构概览

Uptime Kuma 的监控系统采用了**混合架构模式**：

### 1.1 两种监控类型实现方式

1. **内联实现（Legacy 模式）**：HTTP、Ping、Push、Steam、Docker、Radius 等类型直接在 `Monitor` 模型的 `start()` 方法中处理
2. **类继承实现（New 模式）**：TCP、DNS、gRPC、SMTP、SNMP、Redis 等通过继承 `MonitorType` 基类实现

### 1.2 核心类结构

```
MonitorType (基础类)
├── TCPMonitorType (tcp.js)
├── DnsMonitorType (dns.js)
├── GrpcKeywordMonitorType (grpc.js)
├── SMTPMonitorType (smtp.js)
├── SNMPMonitorType (snmp.js)
├── RedisMonitorType (redis.js)
├── MqttMonitorType (mqtt.js)
├── GameDigMonitorType (gamedig.js)
├── GroupMonitorType (group.js)
├── GlobalpingMonitorType (globalping.js)
├── SystemServiceMonitorType (system-service.js)
├── TailscalePing (tailscale-ping.js)
├── WebSocketMonitorType (websocket-upgrade.js)
├── RealBrowserMonitorType (real-browser-monitor-type.js)
├── ManualMonitorType (manual.js)
├── SIPMonitorType (sip-options.js)
├── RabbitMqMonitorType (rabbitmq.js)
├── MongodbMonitorType (mongodb.js)
├── PostgresMonitorType (postgres.js)
├── MssqlMonitorType (mssql.js)
├── MysqlMonitorType (mysql.js)
└── OracleDbMonitorType (oracledb.js)
```

---

## 二、通用字段与类型专属字段的边界

### 2.1 通用字段（所有监控类型共享）

根据 `server/model/monitor.js` 的 `toJSON()` 方法（第 117-254 行），通用字段包括：

#### 基础标识字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `id` | integer | 监控ID |
| `name` | string | 监控名称 |
| `description` | string | 描述 |
| `type` | string | 监控类型（http/port/ping/dns 等） |
| `subtype` | string | 子类型（如 globalping 的 ping/http/dns） |
| `active` | boolean | 是否激活 |

#### 调度与重试字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `interval` | integer | 检查间隔（秒） |
| `timeout` | integer | 超时时间（秒） |
| `maxretries` | integer | 最大重试次数 |
| `retryInterval` | integer | 重试间隔（秒） |
| `retryOnlyOnStatusCodeFailure` | boolean | 仅在状态码失败时重试 |
| `resendInterval` | integer | 通知重发间隔 |

#### 行为控制字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `ignoreTls` | boolean | 忽略 TLS 错误 |
| `upsideDown` | boolean | 倒置模式（UP 变 DOWN） |
| `expiryNotification` | boolean | 证书过期通知 |
| `domainExpiryNotification` | boolean | 域名过期通知 |
| `proxyId` | integer | 代理ID |

#### 响应保存配置
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `saveResponse` | boolean | 保存成功响应 |
| `saveErrorResponse` | boolean | 保存错误响应 |
| `responseMaxLength` | integer | 响应最大长度 |

#### 分组与标签
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `parent` | integer | 父监控ID（用于分组） |
| `tags` | array | 标签列表 |
| `notificationIDList` | object | 通知ID列表 |

#### 条件判断（支持条件的监控类型）
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `conditions` | JSON | 条件表达式组（用于 DNS 等支持条件的类型） |

### 2.2 类型专属字段边界分析

#### HTTP 类型专属字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `url` | string | 目标 URL |
| `method` | string | HTTP 方法（GET/POST 等） |
| `headers` | JSON | 请求头 |
| `body` | string | 请求体 |
| `httpBodyEncoding` | string | 编码格式（json/form/xml） |
| `maxredirects` | integer | 最大重定向次数 |
| `accepted_statuscodes` | array | 接受的状态码列表 |
| `ipFamily` | string | IP 家族（ipv4/ipv6） |
| `cacheBust` | boolean | 缓存破坏 |

#### HTTP Keyword 类型额外字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `keyword` | string | 要检查的关键词 |
| `invertKeyword` | boolean | 反转关键词匹配 |

#### HTTP JSON Query 类型额外字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `jsonPath` | string | JSON 路径表达式 |
| `jsonPathOperator` | string | 比较操作符 |
| `expectedValue` | string | 期望值 |

#### 认证相关字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `authMethod` | string | 认证方式（basic/oauth2-cc/ntlm/mtls） |
| `basic_auth_user` | string | 基础认证用户名 |
| `basic_auth_pass` | string | 基础认证密码 |
| `oauth_client_id` | string | OAuth2 客户端ID |
| `oauth_client_secret` | string | OAuth2 客户端密钥 |
| `oauth_token_url` | string | OAuth2 Token URL |
| `oauth_scopes` | string | OAuth2 作用域 |
| `oauth_audience` | string | OAuth2 Audience |
| `oauth_auth_method` | string | OAuth2 认证方法 |
| `tlsCa` | string | mTLS CA 证书 |
| `tlsCert` | string | mTLS 客户端证书 |
| `tlsKey` | string | mTLS 客户端密钥 |
| `authWorkstation` | string | NTLM 工作站 |
| `authDomain` | string | NTLM 域 |

#### TCP 类型专属字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `hostname` | string | 主机名 |
| `port` | integer | 端口号 |
| `expected_tls_alert` | string | 期望的 TLS 警报（用于 mTLS 验证） |
| `smtpSecurity` | string | SMTP 安全类型（secure/starttls） |

#### Ping 类型专属字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `hostname` | string | 主机名 |
| `packetSize` | integer | 数据包大小 |
| `ping_count` | integer | 发送次数 |
| `ping_numeric` | boolean | 仅输出数字IP |
| `ping_per_request_timeout` | integer | 每次请求超时 |

#### DNS 类型专属字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `hostname` | string | 要查询的主机名 |
| `dns_resolve_type` | string | 记录类型（A/AAAA/MX/CNAME 等） |
| `dns_resolve_server` | string | DNS 解析服务器 |
| `port` | integer | DNS 服务器端口 |
| `dns_last_result` | string | 最后一次查询结果 |
| `conditions` | JSON | 条件表达式（用于记录内容验证） |

#### 数据库类型通用字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `databaseConnectionString` | string | 数据库连接字符串 |
| `databaseQuery` | string | 数据库查询语句 |

#### MQTT 类型专属字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `hostname` | string | MQTT 服务器主机名 |
| `mqttTopic` | string | MQTT 主题 |
| `mqttSuccessMessage` | string | 成功消息 |
| `mqttCheckType` | string | 检查类型 |
| `mqttUsername` | string | MQTT 用户名 |
| `mqttPassword` | string | MQTT 密码 |
| `mqttWebsocketPath` | string | WebSocket 路径 |

#### gRPC 类型专属字段
| 字段名 | 类型 | 说明 |
|--------|------|------|
| `grpcUrl` | string | gRPC URL |
| `grpcProtobuf` | string | Protobuf 定义 |
| `grpcServiceName` | string | 服务名 |
| `grpcMethod` | string | 方法名 |
| `grpcBody` | string | 请求体 |
| `grpcMetadata` | JSON | 元数据 |
| `grpcEnableTls` | boolean | 启用 TLS |
| `keyword` | string | 关键词检查 |

### 2.3 字段边界划分策略

Uptime Kuma 采用以下策略划分字段边界：

1. **单表存储策略**：所有监控类型的字段都存储在同一个 `monitor` 表中，通过 `type` 字段区分
2. **条件显示策略**：前端根据 `type` 字段动态显示/隐藏表单字段
3. **前端过滤**：前端通过 `v-if="monitor.type === 'xxx'"` 条件渲染不同的表单组件
4. **后端映射**：后端在保存时手动映射字段到数据库列
5. **增量迁移**：新增字段通过数据库迁移文件添加，不影响现有数据

---

## 三、前端表单配置到后端执行的映射机制

### 3.1 数据流概览

```
EditMonitor.vue (前端表单)
    ↓
Socket.io "add" / "editMonitor" 事件
    ↓
server.js (字段映射与验证)
    ↓
monitor.js (Monitor 模型 validate() 方法)
    ↓
R.store() (存储到数据库)
    ↓
start() / beat() (执行监控检查)
    ↓
MonitorType.check() 或内联逻辑
```

### 3.2 前端表单结构（EditMonitor.vue）

#### 表单动态渲染机制

前端采用 **条件渲染** 策略，根据 `monitor.type` 显示不同字段：

```vue
<!-- 通用字段 - 始终显示 -->
<div class="my-3">
    <label for="name">{{ $t("Friendly Name") }}</label>
    <input v-model="monitor.name" />
</div>

<!-- HTTP 相关类型字段 -->
<div v-if="monitor.type === 'http' || monitor.type === 'keyword' || 
          monitor.type === 'json-query' || monitor.type === 'real-browser'">
    <input v-model="monitor.url" />
</div>

<!-- TCP/Ping/DNS 共享字段 -->
<div v-if="monitor.type === 'port' || monitor.type === 'ping' || 
          monitor.type === 'dns' || ...">
    <input v-model="monitor.hostname" />
</div>

<!-- 类型专属字段 -->
<div v-if="monitor.type === 'dns'">
    <select v-model="monitor.dns_resolve_type">
        <option value="A">A</option>
        <option value="AAAA">AAAA</option>
        ...
    </select>
</div>
```

#### 关键文件位置
- 主表单文件：`src/pages/EditMonitor.vue`
- 条件组件：`src/components/EditMonitorConditions.vue`

### 3.3 字段命名转换映射

Uptime Kuma 存在 **camelCase ↔ snake_case** 转换：

#### 前端 camelCase → 后端 snake_case

| 前端字段 | 后端字段 | 映射位置 |
|---------|---------|---------|
| `retryOnlyOnStatusCodeFailure` | `retry_only_on_status_code_failure` | server.js:764-766 |
| `saveResponse` | `save_response` | server.js:876 |
| `saveErrorResponse` | `save_error_response` | server.js:877 |
| `responseMaxLength` | `response_max_length` | server.js:878 |
| `expectedTlsAlert` | `expected_tls_alert` | server.js:933 |

#### JSON 字段序列化

部分字段在存储前需要序列化为 JSON：

| 前端字段 | 处理方式 | 代码位置 |
|---------|---------|---------|
| `accepted_statuscodes` | JSON.stringify() | server.js:737 |
| `kafkaProducerBrokers` | JSON.stringify() | server.js:740 |
| `kafkaProducerSaslOptions` | JSON.stringify() | server.js:741 |
| `conditions` | JSON.stringify() | server.js:743 |
| `rabbitmqNodes` | JSON.stringify() | server.js:745 |

### 3.4 添加监控流程（add 事件）

#### 代码位置：`server/server.js:725-797`

```javascript
socket.on("add", async (monitor, callback) => {
    // 1. 认证检查
    checkLogin(socket);
    
    // 2. 创建 bean
    let bean = R.dispense("monitor");
    
    // 3. 处理通知列表
    let notificationIDList = monitor.notificationIDList;
    delete monitor.notificationIDList;
    
    // 4. 状态码验证与转换
    monitor.accepted_statuscodes_json = JSON.stringify(monitor.accepted_statuscodes);
    delete monitor.accepted_statuscodes;
    
    // 5. JSON 字段序列化
    monitor.kafkaProducerBrokers = JSON.stringify(monitor.kafkaProducerBrokers);
    monitor.kafkaProducerSaslOptions = JSON.stringify(monitor.kafkaProducerSaslOptions);
    monitor.conditions = JSON.stringify(monitor.conditions);
    monitor.rabbitmqNodes = JSON.stringify(monitor.rabbitmqNodes);
    
    // 6. 删除前端仅用字段
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
    
    // 7. 导入字段到 bean
    bean.import(monitor);
    
    // 8. 特殊字段映射（camelCase → snake_case）
    if (monitor.retryOnlyOnStatusCodeFailure !== undefined) {
        bean.retry_only_on_status_code_failure = monitor.retryOnlyOnStatusCodeFailure;
    }
    
    // 9. 设置用户ID
    bean.user_id = socket.userID;
    
    // 10. 验证
    bean.validate();
    
    // 11. 存储
    await R.store(bean);
    
    // 12. 更新通知关联
    await updateMonitorNotification(bean.id, notificationIDList);
    
    // 13. 启动监控
    if (monitor.active !== false) {
        await startMonitor(socket.userID, bean.id);
    }
});
```

### 3.5 编辑监控流程（editMonitor 事件）

#### 代码位置：`server/server.js:800-969`

与添加不同，编辑采用**显式字段赋值**方式，而不是 `bean.import()`：

```javascript
socket.on("editMonitor", async (monitor, callback) => {
    let bean = await R.findOne("monitor", " id = ? ", [monitor.id]);
    
    // 显式赋值每个字段
    bean.name = monitor.name;
    bean.description = monitor.description;
    bean.parent = monitor.parent;
    bean.type = monitor.type;
    bean.subtype = monitor.subtype;
    bean.url = monitor.url;
    bean.method = monitor.method;
    bean.body = monitor.body;
    bean.ipFamily = monitor.ipFamily;
    bean.headers = monitor.headers;
    bean.timeout = monitor.timeout;
    bean.interval = monitor.interval;
    bean.retryInterval = monitor.retryInterval;
    bean.hostname = monitor.hostname;
    bean.port = parseInt(monitor.port);
    bean.keyword = monitor.keyword;
    bean.accepted_statuscodes_json = JSON.stringify(monitor.accepted_statuscodes);
    bean.dns_resolve_type = monitor.dns_resolve_type;
    bean.dns_resolve_server = monitor.dns_resolve_server;
    // ... 更多字段赋值
    
    // JSON 字段序列化
    bean.kafkaProducerBrokers = JSON.stringify(monitor.kafkaProducerBrokers);
    bean.conditions = JSON.stringify(monitor.conditions);
    bean.rabbitmqNodes = JSON.stringify(monitor.rabbitmqNodes);
    
    // Ping 高级选项
    bean.ping_numeric = monitor.ping_numeric;
    bean.ping_count = monitor.ping_count;
    bean.ping_per_request_timeout = monitor.ping_per_request_timeout;
    
    bean.validate();
    await R.store(bean);
    
    // 重启监控
    if (await Monitor.isActive(bean.id, bean.active)) {
        await restartMonitor(socket.userID, bean.id);
    }
});
```

### 3.6 验证机制（validate 方法）

#### 代码位置：`server/model/monitor.js:1659-1804`

验证方法按类型分组：

```javascript
validate() {
    // 通用验证
    if (this.interval > MAX_INTERVAL_SECOND) {
        throw new Error(`Interval cannot be more than ${MAX_INTERVAL_SECOND} seconds`);
    }
    
    // JSON 格式验证
    if (this.headers) {
        try {
            JSON.parse(this.headers);
        } catch (e) {
            throw new Error(`Headers must be valid JSON: ${e.message}`);
        }
    }
    
    // Ping 类型专属验证
    if (this.type === "ping") {
        if (this.packetSize && (this.packetSize < PING_PACKET_SIZE_MIN || 
            this.packetSize > PING_PACKET_SIZE_MAX)) {
            throw new Error(`Packet size must be between...`);
        }
        if (this.ping_count && ...) {
            // ...
        }
    }
    
    // Real Browser 类型专属验证
    if (this.type === "real-browser") {
        const delay = Number(this.screenshot_delay);
        if (isNaN(delay) || delay < 0) {
            throw new Error("Screenshot delay must be a non-negative number");
        }
    }
    
    // MongoDB 类型专属验证
    if (this.type === "mongodb" && this.databaseQuery) {
        try {
            JSON.parse(this.databaseQuery);
        } catch (error) {
            throw new Error(`Invalid JSON in database query: ${error.message}`);
        }
    }
}
```

### 3.7 后端执行映射（监控检查）

#### 内联实现的监控类型

代码位置：`server/model/monitor.js:468-714`

```javascript
// HTTP 类型
if (this.type === "http" || this.type === "keyword" || this.type === "json-query") {
    // 使用 axios 发送 HTTP 请求
    let res = await this.makeAxiosRequest(options);
    
    if (this.type === "http") {
        bean.status = UP;
    } else if (this.type === "keyword") {
        // 检查关键词
        let keywordFound = data.includes(this.keyword);
        // ...
    } else if (this.type === "json-query") {
        // 执行 JSON 查询
        const { status, response } = await evaluateJsonQuery(
            data, this.jsonPath, this.jsonPathOperator, this.expectedValue
        );
    }
}
// Ping 类型
else if (this.type === "ping") {
    bean.ping = await ping(
        this.hostname,
        this.ping_count,
        "",
        this.ping_numeric,
        this.packetSize,
        this.timeout,
        this.ping_per_request_timeout
    );
    bean.status = UP;
}
```

#### 类继承实现的监控类型

代码位置：`server/model/monitor.js:905-918`

```javascript
else if (this.type in UptimeKumaServer.monitorTypeList) {
    let startTime = dayjs().valueOf();
    const monitorType = UptimeKumaServer.monitorTypeList[this.type];
    await monitorType.check(this, bean, UptimeKumaServer.getInstance());
    
    // 验证实现正确性
    if (!monitorType.allowCustomStatus && bean.status !== UP) {
        throw new Error("The monitor implementation is incorrect...");
    }
    
    if (bean.ping === undefined || bean.ping === null) {
        bean.ping = dayjs().valueOf() - startTime;
    }
}
```

#### MonitorType 基类接口

代码位置：`server/monitor-types/monitor-type.js:1-40`

```javascript
class MonitorType {
    name = undefined;
    
    // 是否支持条件判断（控制 UI 显示）
    supportsConditions = false;
    
    // 支持的条件变量（用于条件表达式）
    conditionVariables = [];
    
    // 是否允许设置自定义状态（非 UP 状态）
    allowCustomStatus = false;
    
    /**
     * 执行监控检查
     * @param {Monitor} monitor Monitor 对象
     * @param {Heartbeat} heartbeat 心跳对象
     * @param {UptimeKumaServer} server 服务器实例
     */
    async check(monitor, heartbeat, server) {
        throw new Error("You need to override check()");
    }
}
```

#### 具体实现示例（DNS 类型）

代码位置：`server/monitor-types/dns.js:12-100`

```javascript
class DnsMonitorType extends MonitorType {
    name = "dns";
    supportsConditions = true;
    conditionVariables = [new ConditionVariable("record", defaultStringOperators)];
    
    async check(monitor, heartbeat, _server) {
        let startTime = dayjs().valueOf();
        
        // 解析 DNS
        let dnsRes = await this.dnsResolve(
            monitor.hostname, 
            resolverServers, 
            monitor.port, 
            monitor.dns_resolve_type
        );
        heartbeat.ping = dayjs().valueOf() - startTime;
        
        // 条件评估
        const conditions = ConditionExpressionGroup.fromMonitor(monitor);
        let conditionsResult = true;
        
        switch (monitor.dns_resolve_type) {
            case "A":
            case "AAAA":
            case "PTR":
                dnsMessage = `Records: ${dnsRes.join(" | ")}`;
                conditionsResult = dnsRes.some((record) => 
                    handleConditions({ record })
                );
                break;
            // ... 其他记录类型
        }
        
        if (!conditionsResult) {
            throw new Error(dnsMessage);
        }
        
        heartbeat.msg = dnsMessage;
        heartbeat.status = UP;
    }
}
```

### 3.8 监控类型注册机制

代码位置：`server/uptime-kuma-server.js:112-135`

```javascript
constructor() {
    // 注册所有 MonitorType 实现
    UptimeKumaServer.monitorTypeList["real-browser"] = new RealBrowserMonitorType();
    UptimeKumaServer.monitorTypeList["tailscale-ping"] = new TailscalePing();
    UptimeKumaServer.monitorTypeList["websocket-upgrade"] = new WebSocketMonitorType();
    UptimeKumaServer.monitorTypeList["dns"] = new DnsMonitorType();
    UptimeKumaServer.monitorTypeList["postgres"] = new PostgresMonitorType();
    UptimeKumaServer.monitorTypeList["mqtt"] = new MqttMonitorType();
    UptimeKumaServer.monitorTypeList["smtp"] = new SMTPMonitorType();
    UptimeKumaServer.monitorTypeList["group"] = new GroupMonitorType();
    UptimeKumaServer.monitorTypeList["snmp"] = new SNMPMonitorType();
    UptimeKumaServer.monitorTypeList["grpc-keyword"] = new GrpcKeywordMonitorType();
    UptimeKumaServer.monitorTypeList["mongodb"] = new MongodbMonitorType();
    UptimeKumaServer.monitorTypeList["rabbitmq"] = new RabbitMqMonitorType();
    UptimeKumaServer.monitorTypeList["sip-options"] = new SIPMonitorType();
    UptimeKumaServer.monitorTypeList["gamedig"] = new GameDigMonitorType();
    UptimeKumaServer.monitorTypeList["port"] = new TCPMonitorType();
    UptimeKumaServer.monitorTypeList["manual"] = new ManualMonitorType();
    UptimeKumaServer.monitorTypeList["globalping"] = new GlobalpingMonitorType(this.getUserAgent());
    UptimeKumaServer.monitorTypeList["redis"] = new RedisMonitorType();
    UptimeKumaServer.monitorTypeList["system-service"] = new SystemServiceMonitorType();
    UptimeKumaServer.monitorTypeList["sqlserver"] = new MssqlMonitorType();
    UptimeKumaServer.monitorTypeList["mysql"] = new MysqlMonitorType();
    UptimeKumaServer.monitorTypeList["oracledb"] = new OracleDbMonitorType();
}
```

---

## 四、数据库表结构分析

### 4.1 单表设计策略

Uptime Kuma 采用**单表设计**，所有监控类型的字段都存储在 `monitor` 表中。

### 4.2 字段演进（数据库迁移）

通过 `db/knex_migrations/` 目录下的迁移文件可以看到字段的增量添加：

#### 基础字段（初始迁移）
- 通用配置字段：`name`, `type`, `url`, `hostname`, `port`, `interval`, `timeout`
- HTTP 相关：`method`, `headers`, `body`, `accepted_statuscodes_json`
- DNS 相关：`dns_resolve_type`, `dns_resolve_server`

#### 2024-08-24：条件功能
```javascript
// conditions.js
table.text("conditions").notNullable().defaultTo("[]");
```

#### 2025-03-04：Ping 高级选项
```javascript
// ping-advanced-options.js
table.integer("ping_count").defaultTo(1).notNullable();
table.boolean("ping_numeric").defaultTo(true).notNullable();
table.integer("ping_per_request_timeout").defaultTo(2).notNullable();
```

#### 2025-10-15：响应保存配置
```javascript
// add-monitor-response-config.js
table.boolean("save_response").notNullable().defaultTo(false);
table.boolean("save_error_response").notNullable().defaultTo(true);
table.integer("response_max_length").notNullable().defaultTo(1024);
```

#### 2026-01-05：TLS 监控
```javascript
// add-tls-monitor.js
table.string("expected_tls_alert", 50).defaultTo(null);
```

---

## 五、设计特点与优缺点分析

### 5.1 设计优点

1. **简单直接**：单表设计避免了复杂的表关联和 JOIN 操作
2. **灵活扩展**：通过条件渲染和增量迁移轻松添加新字段
3. **向后兼容**：旧监控数据不依赖新字段，默认值处理完善
4. **统一接口**：MonitorType 基类为新类型提供标准化实现方式

### 5.2 设计缺点

1. **字段冗余**：单表包含所有类型字段，大量字段对特定类型无意义
2. **编辑逻辑复杂**：`editMonitor` 中手动赋值超过 80 个字段，容易遗漏
3. **混合架构**：内联实现和类继承实现并存，增加维护复杂度
4. **类型安全差**：JavaScript 动态类型，字段错误在运行时才发现

### 5.3 潜在改进方向

1. **统一架构**：将 HTTP、Ping 等内联实现迁移到 MonitorType 类体系
2. **配置驱动**：使用配置文件定义类型字段和验证规则
3. **自动映射**：实现通用的字段映射机制，减少手动赋值
4. **分表或 JSON 字段**：考虑使用类型专用表或 JSON 字段存储专属配置

---

## 六、关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| MonitorType 基类 | `server/monitor-types/monitor-type.js` | 1-40 |
| Monitor 模型 | `server/model/monitor.js` | 1-1900+ |
| toJSON 方法（字段导出） | `server/model/monitor.js` | 117-254 |
| start() 方法（监控执行） | `server/model/monitor.js` | 409-1149 |
| validate() 方法 | `server/model/monitor.js` | 1659-1804 |
| 添加监控 | `server/server.js` | 725-797 |
| 编辑监控 | `server/server.js` | 800-969 |
| 监控类型注册 | `server/uptime-kuma-server.js` | 112-135 |
| DNS 类型实现 | `server/monitor-types/dns.js` | 1-186 |
| TCP 类型实现 | `server/monitor-types/tcp.js` | 1-416 |
| 前端表单 | `src/pages/EditMonitor.vue` | 1-... |
