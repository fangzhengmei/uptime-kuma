# Uptime Kuma Socket.IO 实时通信机制分析

## 1. 概述

Uptime Kuma 使用 Socket.IO 实现后端与前端的实时通信。本文档从三个维度分析其实现机制：
- 后端状态变更如何通过 Socket.IO 推送到前端
- Socket.IO 连接鉴权是如何实现的
- 前端收到推送后如何更新展示状态

---

## 2. 后端状态变更推送机制

### 2.1 核心架构

后端使用 `UptimeKumaServer` 单例类管理 Socket.IO 服务，核心架构如下：

```javascript
// server/uptime-kuma-server.js:22-45
class UptimeKumaServer {
    static instance = null;
    monitorList = {};           // 监控列表
    maintenanceList = {};       // 维护列表
    io = undefined;             // Socket.IO 服务实例
    jwtSecret = null;           // JWT 密钥
}
```

### 2.2 Socket.IO 服务初始化

Socket.IO 服务在 `UptimeKumaServer` 构造函数中初始化，配置了 CORS 和 WebSocket 源检查：

```javascript
// server/uptime-kuma-server.js:144-195
this.io = new Server(this.httpServer, {
    cors,  // 开发环境允许所有源
    allowRequest: async (req, callback) => {
        const transport = req._query?.transport || "polling";
        const clientIP = await this.getClientIPwithProxy(req.connection.remoteAddress, req.headers);
        
        // WebSocket 源检查（仅适用于 websocket 传输）
        if (transport === "websocket") {
            const bypass = process.env.UPTIME_KUMA_WS_ORIGIN_CHECK === "bypass";
            if (bypass || !req.headers.origin) {
                callback(null, true);
            } else {
                // 验证 origin 与 host 是否匹配
                const originURL = new URL(req.headers.origin);
                if (req.headers.host !== originURL.host && xForwardedFor !== originURL.host) {
                    callback(null, false);  // 拒绝连接
                } else {
                    callback(null, true);   // 允许连接
                }
            }
        } else {
            // polling 传输由 CORS 保护
            callback(null, true);
        }
    },
});
```

### 2.3 房间机制（Room）

Uptime Kuma 使用 Socket.IO 的房间（Room）机制实现用户级别的消息隔离：

```javascript
// server/server.js:1804-1806
async function afterLogin(socket, user) {
    socket.userID = user.id;      // 在 socket 对象上绑定用户ID
    socket.join(user.id);          // 加入以 userID 命名的房间
}
```

**推送方式**：
- `io.to(userID).emit()`：向该用户的所有连接（多个浏览器标签页）推送消息
- `socket.emit()`：仅向当前连接推送消息

### 2.4 状态变更推送类型

#### 2.4.1 监控列表推送

**全量推送**：
```javascript
// server/uptime-kuma-server.js:219-223
async sendMonitorList(socket) {
    let list = await this.getMonitorJSONList(socket.userID);
    this.io.to(socket.userID).emit("monitorList", list);  // 向用户房间推送
    return list;
}
```

**增量更新**：
```javascript
// server/uptime-kuma-server.js:231-236
async sendUpdateMonitorIntoList(socket, monitorID) {
    let list = await this.getMonitorJSONList(socket.userID, monitorID);
    if (list && list[monitorID]) {
        this.io.to(socket.userID).emit("updateMonitorIntoList", list);
    }
}
```

**删除推送**：
```javascript
// server/uptime-kuma-server.js:244-246
async sendDeleteMonitorFromList(socket, monitorID) {
    this.io.to(socket.userID).emit("deleteMonitorFromList", monitorID);
}
```

#### 2.4.2 心跳（Heartbeat）推送

心跳推送是监控状态变更的核心机制，有两种推送方式：

**历史心跳列表**：
```javascript
// server/client.js:46-64
async function sendHeartbeatList(socket, monitorID, toUser = false, overwrite = false) {
    let list = await R.getAll(
        `SELECT * FROM heartbeat WHERE monitor_id = ? ORDER BY time DESC LIMIT 100`,
        [monitorID]
    );
    let result = list.reverse();
    
    if (toUser) {
        // 推送给用户所有连接
        io.to(socket.userID).emit("heartbeatList", monitorID, result, overwrite);
    } else {
        // 仅推送给当前连接
        socket.emit("heartbeatList", monitorID, result, overwrite);
    }
}
```

**实时心跳推送**（监控执行后）：

实时心跳推送由 Monitor 类在每次检查完成后触发，详情见 `server/model/monitor.js`。

#### 2.4.3 统计数据推送

```javascript
// 登录后推送所有监控的统计数据
// server/server.js:1823-1826
for (let monitorID in monitorList) {
    monitorPromises.push(sendHeartbeatList(socket, monitorID));
    monitorPromises.push(Monitor.sendStats(io, monitorID, user.id));
}
```

#### 2.4.4 其他状态推送

| 推送函数 | 事件名称 | 数据内容 |
|---------|---------|---------|
| `sendMaintenanceList` | maintenanceList | 维护计划列表 |
| `sendNotificationList` | notificationList | 通知方式列表 |
| `sendProxyList` | proxyList | 代理列表 |
| `sendDockerHostList` | dockerHostList | Docker 主机列表 |
| `sendAPIKeyList` | apiKeyList | API Key 列表 |
| `sendRemoteBrowserList` | remoteBrowserList | 远程浏览器列表 |
| `sendMonitorTypeList` | monitorTypeList | 监控类型配置 |
| `sendInfo` | info | 服务器信息（版本、时区等） |

---

## 3. 连接鉴权机制

### 3.1 整体鉴权流程

```
客户端连接
    ↓
WebSocket 源检查 (allowRequest)
    ↓
发送 "loginRequired" 事件 (除非禁用认证)
    ↓
客户端发起登录 (login / loginByToken)
    ↓
服务端验证凭据
    ↓
设置 socket.userID + 加入房间
    ↓
推送初始数据 (monitorList, heartbeatList 等)
```

### 3.2 连接层防护

#### 3.2.1 WebSocket 源检查

```javascript
// server/uptime-kuma-server.js:146-194
allowRequest: async (req, callback) => {
    const transport = req._query?.transport || "polling";
    const clientIP = await this.getClientIPwithProxy(req.connection.remoteAddress, req.headers);
    
    if (transport === "websocket") {
        const bypass = process.env.UPTIME_KUMA_WS_ORIGIN_CHECK === "bypass";
        if (bypass) {
            callback(null, true);  // 绕过检查（危险）
        } else if (!req.headers.origin) {
            callback(null, true);  // 无 origin 允许（非浏览器请求）
        } else {
            // 验证 origin 与 host 匹配
            const originURL = new URL(req.headers.origin);
            if (req.headers.host !== originURL.host && 
                xForwardedFor !== originURL.host) {
                callback(null, false);  // 源不匹配，拒绝
            } else {
                callback(null, true);   // 允许
            }
        }
    } else {
        callback(null, true);  // polling 由 CORS 保护
    }
}
```

#### 3.2.2 开发环境 CORS

```javascript
// server/uptime-kuma-server.js:137-142
let cors = undefined;
if (isDev) {
    cors = {
        origin: "*",  // 开发环境允许所有源
    };
}
```

### 3.3 认证方式

#### 3.3.1 连接后的认证提示

```javascript
// server/server.js:1726-1734
if (await setting("disableAuth")) {
    // 禁用认证模式：自动登录管理员
    log.info("auth", "Disabled Auth: auto login to admin");
    await afterLogin(socket, await R.findOne("user"));
    socket.emit("autoLogin");
} else {
    // 需要认证模式：发送登录要求
    socket.emit("loginRequired");
}
```

#### 3.3.2 Token 登录（JWT）

```javascript
// server/server.js:383-430
socket.on("loginByToken", async (token, callback) => {
    try {
        // 1. 验证 JWT
        let decoded = jwt.verify(token, server.jwtSecret);
        
        // 2. 查询用户
        let user = await R.findOne("user", " username = ? AND active = 1 ", [decoded.username]);
        
        if (user) {
            // 3. 检查密码是否变更（防止旧 token）
            if (decoded.h !== shake256(user.password, SHAKE256_LENGTH)) {
                throw new Error("The token is invalid due to password change or old token");
            }
            
            // 4. 登录成功
            await afterLogin(socket, user);
            callback({ ok: true });
        } else {
            callback({ ok: false, msg: "authUserInactiveOrDeleted", msgi18n: true });
        }
    } catch (error) {
        callback({ ok: false, msg: "authInvalidToken", msgi18n: true });
    }
});
```

**JWT 生成**：
```javascript
// server/model/user.js
static createJWT(user, jwtSecret) {
    return jwt.sign(
        {
            username: user.username,
            h: shake256(user.password, SHAKE256_LENGTH),  // 密码哈希用于检测密码变更
        },
        jwtSecret,
        {
            expiresIn: "30d",  // 30 天过期
        }
    );
}
```

#### 3.3.3 用户名密码登录

```javascript
// server/server.js:432-510
socket.on("login", async (data, callback) => {
    // 1. 登录速率限制
    if (!(await loginRateLimiter.pass(callback))) {
        return;
    }
    
    // 2. 验证用户名密码
    let user = await login(data.username, data.password);
    
    if (user) {
        if (user.twofa_status === 0) {
            // 无 2FA：直接登录
            await afterLogin(socket, user);
            callback({ ok: true, token: User.createJWT(user, server.jwtSecret) });
        } else if (user.twofa_status === 1 && !data.token) {
            // 需要 2FA 令牌
            callback({ tokenRequired: true });
        } else if (data.token) {
            // 验证 2FA 令牌
            let verify = notp.totp.verify(data.token, user.twofa_secret, twoFAVerifyOptions);
            if (user.twofa_last_token !== data.token && verify) {
                await afterLogin(socket, user);
                callback({ ok: true, token: User.createJWT(user, server.jwtSecret) });
            } else {
                callback({ ok: false, msg: "authInvalidToken", msgi18n: true });
            }
        }
    } else {
        callback({ ok: false, msg: "authIncorrectCreds", msgi18n: true });
    }
});
```

### 3.4 登录后处理（afterLogin）

```javascript
// server/server.js:1804-1836
async function afterLogin(socket, user) {
    // 1. 绑定用户ID，加入房间
    socket.userID = user.id;
    socket.join(user.id);
    
    // 2. 推送监控列表
    let monitorList = await server.sendMonitorList(socket);
    
    // 3. 并行推送其他配置数据
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
    
    // 4. 推送状态页列表
    await StatusPage.sendStatusPageList(io, socket);
    
    // 5. 推送每个监控的心跳历史和统计数据
    const monitorPromises = [];
    for (let monitorID in monitorList) {
        monitorPromises.push(sendHeartbeatList(socket, monitorID));
        monitorPromises.push(Monitor.sendStats(io, monitorID, user.id));
    }
    await Promise.all(monitorPromises);
}
```

### 3.5 API 级别的权限检查

所有需要认证的 Socket.IO 事件都通过 `checkLogin` 中间件验证：

```javascript
// server/util-server.js:637-641
exports.checkLogin = (socket) => {
    if (!socket.userID) {
        throw new Error("You are not logged in.");
    }
};
```

**使用示例**：
```javascript
// server/server.js:725-730
socket.on("add", async (monitor, callback) => {
    try {
        checkLogin(socket);  // 检查登录状态
        // ... 处理添加监控
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 3.6 登出机制

```javascript
// server/server.js:512-524
socket.on("logout", async (callback) => {
    // 速率限制
    if (!(await loginRateLimiter.pass(callback))) {
        return;
    }
    
    // 离开房间，清除用户ID
    socket.leave(socket.userID);
    socket.userID = null;
    
    if (typeof callback === "function") {
        callback();
    }
});
```

---

## 4. 前端接收推送并更新状态

### 4.1 前端 Socket.IO 客户端架构

前端使用 Vue mixin 实现 Socket.IO 客户端，核心文件为 `src/mixins/socket.js`。

#### 4.1.1 连接初始化

```javascript
// src/mixins/socket.js:85-120
initSocketIO(bypass = false) {
    // 1. 检查是否已初始化
    if (this.socket.initedSocketIO) {
        return;
    }
    
    // 2. 某些页面不需要 Socket.IO（如状态页、首页）
    if (!bypass && location.pathname) {
        for (let page of noSocketIOPages) {
            if (location.pathname.match(page)) {
                return;
            }
        }
    }
    
    // 3. 创建 Socket.IO 连接
    let url;
    const env = process.env.NODE_ENV || "production";
    if (env === "development" && isDevContainer()) {
        url = protocol + getDevContainerServerHostname();
    } else if (env === "development" || localStorage.dev === "dev") {
        url = protocol + location.hostname + ":3001";
    } else {
        url = undefined;  // 连接当前 URL
    }
    
    socket = io(url);
    
    // 4. 注册事件监听器
    this.setupEventListeners();
}
```

### 4.2 响应式数据存储

前端使用 Vue 的响应式数据存储所有状态：

```javascript
// src/mixins/socket.js:30-72
data() {
    return {
        info: {},                    // 服务器信息
        socket: {
            token: null,
            firstConnect: true,
            connected: false,
            connectCount: 0,
            initedSocketIO: false,
        },
        username: null,              // 当前用户名
        loggedIn: false,              // 登录状态
        
        // 核心监控数据
        monitorList: {},              // 监控列表 { monitorID: monitorObj }
        heartbeatList: {},            // 心跳历史 { monitorID: [beat1, beat2, ...] }
        avgPingList: {},              // 平均延迟 { monitorID: ms }
        uptimeList: {},               // 可用性 { monitorID_type: value }
        tlsInfoList: {},              // TLS 证书信息 { monitorID: info }
        domainInfoList: {},           // 域名信息 { monitorID: info }
        
        // 其他配置数据
        maintenanceList: {},
        notificationList: [],
        apiKeyList: {},
        dockerHostList: [],
        remoteBrowserList: [],
        statusPageList: [],
        proxyList: [],
    };
}
```

### 4.3 事件监听与数据更新

#### 4.3.1 连接状态事件

```javascript
// src/mixins/socket.js:260-286
// 连接错误
socket.on("connect_error", (err) => {
    console.error(`Failed to connect to the backend. Socket.io connect_error: ${err.message}`);
    this.connectionErrorMsg = `${this.$t("Cannot connect to the socket server.")} [${err}] ${this.$t("Reconnecting...")}`;
    this.socket.connected = false;
});

// 断开连接
socket.on("disconnect", () => {
    this.connectionErrorMsg = `${this.$t("Lost connection to the socket server.")} ${this.$t("Reconnecting...")}`;
    this.socket.connected = false;
});

// 连接成功
socket.on("connect", () => {
    this.socket.connectCount++;
    this.socket.connected = true;
    this.showReverseProxyGuide = false;
    
    // 重连时清空心跳列表
    if (this.socket.connectCount >= 2) {
        this.clearData();
    }
});
```

#### 4.3.2 认证相关事件

```javascript
// src/mixins/socket.js:122-145
// 服务器信息
socket.on("info", (info) => {
    this.info = info;
});

// 需要登录
socket.on("loginRequired", () => {
    let token = this.storage().token;
    if (token && token !== "autoLogin") {
        this.loginByToken(token);  // 使用保存的 token 登录
    } else {
        this.$root.storage().removeItem("token");
        this.allowLoginDialog = true;  // 显示登录对话框
    }
});

// 自动登录（禁用认证模式）
socket.on("autoLogin", () => {
    this.loggedIn = true;
    this.storage().token = "autoLogin";
    this.socket.token = "autoLogin";
    this.allowLoginDialog = false;
});
```

#### 4.3.3 监控列表更新事件

```javascript
// src/mixins/socket.js:147-163
// 全量更新监控列表
socket.on("monitorList", (data) => {
    this.assignMonitorUrlParser(data);  // 添加 URL 解析方法
    this.monitorList = data;
});

// 增量更新监控
socket.on("updateMonitorIntoList", (data) => {
    this.assignMonitorUrlParser(data);
    Object.entries(data).forEach(([monitorID, updatedMonitor]) => {
        this.monitorList[monitorID] = updatedMonitor;
    });
});

// 删除监控
socket.on("deleteMonitorFromList", (monitorID) => {
    if (this.monitorList[monitorID]) {
        delete this.monitorList[monitorID];
    }
});
```

#### 4.3.4 心跳更新事件（核心状态更新）

```javascript
// src/mixins/socket.js:204-234
// 实时心跳推送
socket.on("heartbeat", (data) => {
    // 1. 确保该监控有心跳数组
    if (!(data.monitorID in this.heartbeatList)) {
        this.heartbeatList[data.monitorID] = [];
    }
    
    // 2. 添加新心跳
    this.heartbeatList[data.monitorID].push(data);
    
    // 3. 限制数组长度（最多 150 条）
    if (this.heartbeatList[data.monitorID].length >= 150) {
        this.heartbeatList[data.monitorID].shift();
    }
    
    // 4. 重要心跳（状态变更）处理
    if (data.important) {
        // 显示 Toast 通知
        if (this.monitorList[data.monitorID] !== undefined) {
            if (data.status === 0) {
                toast.error(`[${this.monitorList[data.monitorID].name}] [DOWN] ${data.msg}`, {
                    timeout: getToastErrorTimeout(),
                });
            } else if (data.status === 1) {
                toast.success(`[${this.monitorList[data.monitorID].name}] [Up] ${data.msg}`, {
                    timeout: getToastSuccessTimeout(),
                });
            }
        }
        // 触发事件总线
        this.emitter.emit("newImportantHeartbeat", data);
    }
});

// 心跳历史列表
socket.on("heartbeatList", (monitorID, data, overwrite = false) => {
    if (!(monitorID in this.heartbeatList) || overwrite) {
        this.heartbeatList[monitorID] = data;
    } else {
        // 合并历史数据
        this.heartbeatList[monitorID] = data.concat(this.heartbeatList[monitorID]);
    }
});
```

#### 4.3.5 统计数据更新事件

```javascript
// src/mixins/socket.js:244-258
// 平均延迟
socket.on("avgPing", (monitorID, data) => {
    this.avgPingList[monitorID] = data;
});

// 可用性
socket.on("uptime", (monitorID, type, data) => {
    this.uptimeList[`${monitorID}_${type}`] = data;
});

// TLS 证书信息
socket.on("certInfo", (monitorID, data) => {
    this.tlsInfoList[monitorID] = JSON.parse(data);
});

// 域名信息
socket.on("domainInfo", (monitorID, daysRemaining, expiresOn) => {
    this.domainInfoList[monitorID] = { daysRemaining: daysRemaining, expiresOn: expiresOn };
});
```

### 4.4 计算属性（状态派生）

前端使用 Vue 计算属性从原始数据派生出展示状态：

#### 4.4.1 最新心跳

```javascript
// src/mixins/socket.js:744-753
lastHeartbeatList() {
    let result = {};
    for (let monitorID in this.heartbeatList) {
        let index = this.heartbeatList[monitorID].length - 1;
        result[monitorID] = this.heartbeatList[monitorID][index];
    }
    return result;
}
```

#### 4.4.2 状态列表（用于 UI 展示）

```javascript
// src/mixins/socket.js:755-794
statusList() {
    let result = {};
    let unknown = {
        text: this.$t("Unknown"),
        color: "secondary",
    };
    
    for (let monitorID in this.lastHeartbeatList) {
        let lastHeartBeat = this.lastHeartbeatList[monitorID];
        
        if (!lastHeartBeat) {
            result[monitorID] = unknown;
        } else if (lastHeartBeat.status === UP) {
            result[monitorID] = {
                text: this.$t("Up"),
                color: "primary",
            };
        } else if (lastHeartBeat.status === DOWN) {
            result[monitorID] = {
                text: this.$t("Down"),
                color: "danger",
            };
        } else if (lastHeartBeat.status === PENDING) {
            result[monitorID] = {
                text: this.$t("Pending"),
                color: "warning",
            };
        } else if (lastHeartBeat.status === MAINTENANCE) {
            result[monitorID] = {
                text: this.$t("statusMaintenance"),
                color: "maintenance",
            };
        } else {
            result[monitorID] = unknown;
        }
    }
    return result;
}
```

#### 4.4.3 统计汇总

```javascript
// src/mixins/socket.js:796-832
stats() {
    let result = {
        active: 0,
        up: 0,
        down: 0,
        maintenance: 0,
        pending: 0,
        unknown: 0,
        pause: 0,
    };
    
    for (let monitorID in this.$root.monitorList) {
        let beat = this.$root.lastHeartbeatList[monitorID];
        let monitor = this.$root.monitorList[monitorID];
        
        if (monitor && !monitor.active) {
            result.pause++;
        } else if (beat) {
            result.active++;
            if (beat.status === UP) {
                result.up++;
            } else if (beat.status === DOWN) {
                result.down++;
            } else if (beat.status === PENDING) {
                result.pending++;
            } else if (beat.status === MAINTENANCE) {
                result.maintenance++;
            } else {
                result.unknown++;
            }
        } else {
            result.unknown++;
        }
    }
    return result;
}
```

### 4.5 响应式更新示例

#### 4.5.1 Favicon 角标更新

```javascript
// src/mixins/socket.js:857-868
watch: {
    // 当 down 数量变化时，更新 favicon 角标
    "stats.down"(to, from) {
        if (to !== from) {
            // 防抖处理
            if (this.faviconUpdateDebounce != null) {
                clearTimeout(this.faviconUpdateDebounce);
            }
            this.faviconUpdateDebounce = setTimeout(() => {
                favicon.badge(to);  // 更新角标数字
            }, 1000);
        }
    },
    
    // 服务器版本变化时刷新页面
    "info.version"(to, from) {
        if (from && from !== to) {
            window.location.reload();
        }
    },
}
```

#### 4.5.2 登录方法

```javascript
// src/mixins/socket.js:412-438
login(username, password, token, callback) {
    socket.emit(
        "login",
        { username, password, token },
        (res) => {
            if (res.tokenRequired) {
                callback(res);
            }
            if (res.ok) {
                // 保存 token
                this.storage().token = res.token;
                this.socket.token = res.token;
                this.loggedIn = true;
                this.username = this.getJWTPayload()?.username;
            }
            callback(res);
        }
    );
},

loginByToken(token) {
    socket.emit("loginByToken", token, (res) => {
        this.allowLoginDialog = true;
        if (!res.ok) {
            this.logout();
        } else {
            this.loggedIn = true;
            this.username = this.getJWTPayload()?.username;
        }
    });
},
```

---

## 5. 状态产生与广播机制

### 5.1 状态在哪里产生

状态产生于 `Monitor` 类的 `beat()` 函数中，这是监控检查的核心循环。

#### 5.1.1 状态定义

```javascript
// src/util.ts 中定义的状态常量
const UP = 1;           // 正常
const DOWN = 0;         // 故障
const PENDING = 2;      // 重试中
const MAINTENANCE = 3;  // 维护中
```

#### 5.1.2 状态产生流程

```javascript
// server/model/monitor.js:800-990
const beat = async () => {
    let startTime = dayjs().valueOf();
    let retries = 0;
    
    // 1. 创建心跳 Bean
    let bean = R.dispense("heartbeat");
    bean.monitor_id = this.id;
    bean.time = R.isoDateTime(dayjs.utc());
    bean.msg = "";
    bean.ping = 0;
    bean.status = UP;  // 默认 UP
    bean.important = false;
    bean.downCount = 0;
    
    try {
        // 2. 执行不同类型的监控检查
        if (this.type === "http" || this.type === "keyword" || this.type === "json-query") {
            // HTTP 类型监控
            // ... axios 请求，检查响应状态码、关键词等
        } else if (this.type === "port") {
            // TCP 端口监控
            // ... 建立 TCP 连接
        } else if (this.type === "ping") {
            // ICMP Ping 监控
            // ... 发送 ping 包
        } else if (this.type in UptimeKumaServer.monitorTypeList) {
            // 其他监控类型（DNS、MySQL、PostgreSQL 等）
            const monitorType = UptimeKumaServer.monitorTypeList[this.type];
            await monitorType.check(this, bean, UptimeKumaServer.getInstance());
        }
        
        // 3. 翻转模式（Upside Down）：将 UP 翻转为 DOWN
        if (this.isUpsideDown()) {
            bean.status = flipStatus(bean.status);
        }
        
        retries = 0;
    } catch (error) {
        // 4. 错误处理：设置状态为 PENDING 或 DOWN
        bean.msg = error.message;
        
        if (this.maxretries > 0 && retries < this.maxretries) {
            retries++;
            bean.status = PENDING;  // 重试中
        } else {
            bean.status = DOWN;      // 超过重试次数，标记为故障
        }
    }
    
    // 5. 检查是否为"重要心跳"（状态变更）
    let isImportant = Monitor.isImportantBeat(isFirstBeat, previousBeat?.status, bean.status);
    
    if (isImportant) {
        bean.important = true;
        // ... 发送通知等处理
    }
    
    // 6. 更新可用性计算
    let uptimeCalculator = await UptimeCalculator.getUptimeCalculator(this.id);
    await uptimeCalculator.update(bean.status, parseFloat(bean.ping));
    
    // 7. 广播到前端
    io.to(this.user_id).emit("heartbeat", bean.toJSON());  // 广播心跳
    Monitor.sendStats(io, this.id, this.user_id);          // 广播统计数据
    
    // 8. 存储到数据库
    await R.store(bean);
    
    // 9. 调度下一次检查
    if (!this.isStop) {
        let intervalRemainingMs = Math.max(1, beatInterval * 1000 - dayjs().diff(dayjs.utc(bean.time)));
        this.heartbeatInterval = setTimeout(safeBeat, intervalRemainingMs);
    }
};
```

#### 5.1.3 重要心跳判断逻辑

```javascript
// server/model/monitor.js:1419-1440
static isImportantBeat(isFirstBeat, previousBeatStatus, currentBeatStatus) {
    // 首次心跳 = 重要
    // UP -> PENDING = 不重要
    // UP -> DOWN = 重要
    // UP -> UP = 不重要
    // PENDING -> PENDING = 不重要
    // PENDING -> DOWN = 重要
    // PENDING -> UP = 不重要
    // DOWN -> DOWN = 不重要
    // DOWN -> UP = 重要
    // MAINTENANCE -> MAINTENANCE = 不重要
    // MAINTENANCE -> UP = 重要
    // MAINTENANCE -> DOWN = 重要
    // DOWN -> MAINTENANCE = 重要
    // UP -> MAINTENANCE = 重要
    
    return (
        isFirstBeat ||
        (previousBeatStatus === DOWN && currentBeatStatus === MAINTENANCE) ||
        (previousBeatStatus === UP && currentBeatStatus === MAINTENANCE) ||
        (previousBeatStatus === MAINTENANCE && currentBeatStatus === DOWN) ||
        (previousBeatStatus === MAINTENANCE && currentBeatStatus === UP) ||
        (previousBeatStatus === DOWN && currentBeatStatus === UP) ||
        (previousBeatStatus === PENDING && currentBeatStatus === DOWN) ||
        (previousBeatStatus === UP && currentBeatStatus === DOWN)
    );
}
```

### 5.2 如何触发广播

#### 5.2.1 心跳广播（每次检查后）

```javascript
// server/model/monitor.js:1092-1095
// 发送到前端
log.debug("monitor", `[${this.name}] Send to socket`);
io.to(this.user_id).emit("heartbeat", bean.toJSON());  // 广播实时心跳
Monitor.sendStats(io, this.id, this.user_id);          // 广播统计数据
```

#### 5.2.2 统计数据广播

```javascript
// server/model/monitor.js:1349-1377
static async sendStats(io, monitorID, userID) {
    // 1. 检查房间内是否有客户端
    const hasClients = getTotalClientInRoom(io, userID) > 0;
    let uptimeCalculator = await UptimeCalculator.getUptimeCalculator(monitorID);
    
    if (hasClients) {
        // 2. 发送 24 小时平均延迟
        let data24h = await uptimeCalculator.get24Hour();
        io.to(userID).emit("avgPing", monitorID, data24h.avgPing ? Number(data24h.avgPing.toFixed(2)) : null);
        
        // 3. 发送 24 小时可用性
        io.to(userID).emit("uptime", monitorID, 24, data24h.uptime);
        
        // 4. 发送 30 天可用性
        let data30d = await uptimeCalculator.get30Day();
        io.to(userID).emit("uptime", monitorID, 720, data30d.uptime);
        
        // 5. 发送 1 年可用性
        let data1y = await uptimeCalculator.get1Year();
        io.to(userID).emit("uptime", monitorID, "1y", data1y.uptime);
        
        // 6. 发送证书信息
        await Monitor.sendCertInfo(io, monitorID, userID);
        
        // 7. 发送域名信息
        await Monitor.sendDomainInfo(io, monitorID, userID);
    } else {
        log.debug("monitor", "No clients in the room, no need to send stats");
    }
}
```

#### 5.2.3 API 触发的心跳广播（Push 类型监控）

```javascript
// server/routers/api-router.js:127-129
// Push 类型监控通过 API 接收心跳
io.to(monitor.user_id).emit("heartbeat", bean.toJSON());
Monitor.sendStats(io, monitor.id, monitor.user_id);
```

#### 5.2.4 维护状态变更广播

```javascript
// server/model/monitor.js:1018-1020
// 重要心跳时触发维护列表广播
await UptimeKumaServer.getInstance().sendMaintenanceListByUserID(this.user_id);
```

### 5.3 广播触发汇总

| 触发场景 | 广播位置 | 广播事件 |
|---------|---------|---------|
| 每次监控检查完成 | `monitor.js:1094` | `heartbeat` |
| 每次监控检查完成 | `monitor.js:1095` | `avgPing`, `uptime`, `certInfo`, `domainInfo` |
| API Push 监控 | `api-router.js:127` | `heartbeat` |
| 监控增删改 | `server.js` | `updateMonitorIntoList`, `deleteMonitorFromList` |
| 重要状态变更 | `monitor.js:1020` | `maintenanceList` |

---

## 6. 断线重连与状态补齐

### 6.1 前端断线检测与自动重连

Socket.IO 客户端内置了自动重连机制，Uptime Kuma 在客户端添加了额外的状态管理。

#### 6.1.1 连接状态监听

```javascript
// src/mixins/socket.js:260-286
// 连接错误
socket.on("connect_error", (err) => {
    console.error(`Failed to connect to the backend. Socket.io connect_error: ${err.message}`);
    this.connectionErrorMsg = `${this.$t("Cannot connect to the socket server.")} [${err}] ${this.$t("Reconnecting...")}`;
    this.showReverseProxyGuide = true;
    this.socket.connected = false;
    this.socket.firstConnect = false;
});

// 断开连接
socket.on("disconnect", () => {
    console.log("disconnect");
    this.connectionErrorMsg = `${this.$t("Lost connection to the socket server.")} ${this.$t("Reconnecting...")}`;
    this.socket.connected = false;
});

// 连接成功
socket.on("connect", () => {
    console.log("Connected to the socket server");
    this.socket.connectCount++;
    this.socket.connected = true;
    this.showReverseProxyGuide = false;
    
    // 关键：重连时清空心跳列表
    if (this.socket.connectCount >= 2) {
        this.clearData();
    }
    
    this.socket.firstConnect = false;
});
```

#### 6.1.2 清空数据方法

```javascript
// src/mixins/socket.js:666-669
clearData() {
    console.log("reset heartbeat list");
    this.heartbeatList = {};
}
```

### 6.2 重连后的状态补齐流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        断线重连流程                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 连接断开                                                            │
│     ↓                                                                   │
│  2. Socket.IO 客户端自动重连（指数退避策略）                              │
│     ↓                                                                   │
│  3. 重连成功，connectCount >= 2                                          │
│     ↓                                                                   │
│  4. 前端执行 clearData()，清空心跳列表                                   │
│     ↓                                                                   │
│  5. 服务器发送 "loginRequired" 事件                                      │
│     ↓                                                                   │
│  6. 前端使用保存的 token 自动调用 loginByToken()                         │
│     ↓                                                                   │
│  7. 服务器验证 token，执行 afterLogin()                                   │
│     ↓                                                                   │
│  8. 服务器推送所有初始数据（状态补齐）                                      │
│     ↓                                                                   │
│  9. 前端响应式更新 UI                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 服务器端状态补齐（afterLogin）

重连后，服务器通过 `afterLogin` 函数推送完整的状态数据：

```javascript
// server/server.js:1804-1836
async function afterLogin(socket, user) {
    // 1. 绑定用户ID，加入房间
    socket.userID = user.id;
    socket.join(user.id);
    
    // 2. 推送监控列表
    let monitorList = await server.sendMonitorList(socket);  // 事件：monitorList
    
    // 3. 并行推送配置数据
    await Promise.allSettled([
        sendInfo(socket),                    // 事件：info
        server.sendMaintenanceList(socket),  // 事件：maintenanceList
        sendNotificationList(socket),         // 事件：notificationList
        sendProxyList(socket),                // 事件：proxyList
        sendDockerHostList(socket),           // 事件：dockerHostList
        sendAPIKeyList(socket),               // 事件：apiKeyList
        sendRemoteBrowserList(socket),        // 事件：remoteBrowserList
        sendMonitorTypeList(socket),          // 事件：monitorTypeList
    ]);
    
    // 4. 推送状态页列表
    await StatusPage.sendStatusPageList(io, socket);  // 事件：statusPageList
    
    // 5. 为每个监控推送心跳历史和统计数据
    const monitorPromises = [];
    for (let monitorID in monitorList) {
        // 推送最近 100 条心跳历史
        monitorPromises.push(sendHeartbeatList(socket, monitorID));  // 事件：heartbeatList
        
        // 推送统计数据（avgPing、uptime、certInfo、domainInfo）
        monitorPromises.push(Monitor.sendStats(io, monitorID, user.id));
    }
    await Promise.all(monitorPromises);
}
```

### 6.4 前端接收补齐数据

前端通过事件监听器接收服务器推送的补齐数据：

```javascript
// src/mixins/socket.js:147-258
// 监控列表
socket.on("monitorList", (data) => {
    this.assignMonitorUrlParser(data);
    this.monitorList = data;
});

// 心跳历史列表（重连时会覆盖）
socket.on("heartbeatList", (monitorID, data, overwrite = false) => {
    if (!(monitorID in this.heartbeatList) || overwrite) {
        this.heartbeatList[monitorID] = data;
    } else {
        this.heartbeatList[monitorID] = data.concat(this.heartbeatList[monitorID]);
    }
});

// 统计数据
socket.on("avgPing", (monitorID, data) => {
    this.avgPingList[monitorID] = data;
});

socket.on("uptime", (monitorID, type, data) => {
    this.uptimeList[`${monitorID}_${type}`] = data;
});

socket.on("certInfo", (monitorID, data) => {
    this.tlsInfoList[monitorID] = JSON.parse(data);
});

socket.on("domainInfo", (monitorID, daysRemaining, expiresOn) => {
    this.domainInfoList[monitorID] = { daysRemaining: daysRemaining, expiresOn: expiresOn };
});
```

### 6.5 重连期间可能丢失的数据

虽然服务器会推送完整的历史数据，但在断线期间产生的**实时心跳事件**可能会丢失。不过：

1. **心跳历史完整**：`sendHeartbeatList` 会推送数据库中最近 100 条心跳
2. **统计数据完整**：`sendStats` 会推送最新的计算好的统计数据
3. **状态最终一致**：前端通过 `lastHeartbeatList` 计算属性获取最新状态

### 6.6 主动刷新机制

服务器可以强制客户端刷新：

```javascript
// src/mixins/socket.js:299-301
socket.on("refresh", () => {
    location.reload();
});

// 应用场景：修改密码时，强制其他连接刷新
// server/server.js:548-557
disconnectAllSocketClients(userID, currentSocketID) {
    for (const socket of this.io.sockets.sockets.values()) {
        if (socket.userID === userID && socket.id !== currentSocketID) {
            try {
                socket.emit("refresh");  // 发送刷新事件
                socket.disconnect();
            } catch (e) {}
        }
    }
}
```

---

## 7. 完整状态链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              后端 (Node.js)                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────────────────┐  │
│  │   Monitor    │─────▶│  UptimeKuma  │─────▶│     Socket.IO (io)       │  │
│  │  (执行监控)   │      │   Server     │      │  (管理连接和房间)         │  │
│  └──────────────┘      └──────────────┘      └──────────────────────────┘  │
│         │                                       │                            │
│         │ 心跳/状态变更                          │ 推送消息                    │
│         ▼                                       ▼                            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      推送事件类型                                        │  │
│  │  monitorList / updateMonitorIntoList / deleteMonitorFromList         │  │
│  │  heartbeat / heartbeatList                                             │  │
│  │  avgPing / uptime / certInfo / domainInfo                             │  │
│  │  maintenanceList / notificationList / ...                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        鉴权流程                                         │  │
│  │  1. WebSocket 源检查 (allowRequest)                                    │  │
│  │  2. 登录验证 (login / loginByToken)                                     │  │
│  │  3. JWT 验证 + 密码变更检测                                             │  │
│  │  4. 设置 socket.userID + 加入房间                                       │  │
│  │  5. API 级别检查 (checkLogin)                                          │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ Socket.IO 事件
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端 (Vue.js)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      事件监听器 (socket.js)                            │  │
│  │  socket.on("monitorList", ...)                                        │  │
│  │  socket.on("heartbeat", ...)                                          │  │
│  │  socket.on("avgPing", ...)                                            │  │
│  │  ...                                                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      │ 更新响应式数据                         │
│                                      ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      响应式数据存储                                     │  │
│  │  monitorList: {}        // 监控列表                                    │  │
│  │  heartbeatList: {}      // 心跳历史                                    │  │
│  │  avgPingList: {}        // 平均延迟                                    │  │
│  │  uptimeList: {}         // 可用性                                      │  │
│  │  statusList: {}         // 状态列表（计算属性）                         │  │
│  │  stats: {}              // 统计汇总（计算属性）                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      │ 响应式更新 UI                          │
│                                      ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      UI 组件                                           │  │
│  │  Dashboard: 显示监控列表、状态卡片、统计数字                            │  │
│  │  Monitor Detail: 显示图表、心跳历史、详细信息                           │  │
│  │  Toast: 状态变更通知                                                   │  │
│  │  Favicon: 角标显示 down 数量                                           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键设计要点总结

### 6.1 推送策略

| 场景 | 推送方式 | 目标范围 |
|------|---------|---------|
| 登录初始化 | `io.to(userID).emit()` | 用户所有连接 |
| 实时心跳 | `io.to(userID).emit()` | 用户所有连接 |
| 增量更新 | `io.to(userID).emit()` | 用户所有连接 |
| 单次响应 | `socket.emit()` | 当前连接 |

### 6.2 安全机制

1. **WebSocket 源检查**：防止 CSRF 攻击
2. **JWT Token**：无状态认证，30 天过期
3. **密码变更检测**：JWT payload 包含密码哈希，密码变更后旧 token 失效
4. **2FA 支持**：可选的双因素认证
5. **速率限制**：登录尝试有速率限制

### 6.3 前端响应式设计

1. **事件驱动**：所有状态变更通过 Socket.IO 事件推送
2. **响应式数据**：使用 Vue 响应式系统自动更新 UI
3. **计算属性**：从原始数据派生出展示状态，避免重复计算
4. **防抖优化**：favicon 更新等操作使用防抖

### 6.4 房间机制的优势

1. **用户隔离**：每个用户有独立的房间，消息不会串扰
2. **多端同步**：同一用户的多个浏览器标签页实时同步
3. **高效推送**：`io.to(room).emit()` 一次调用推送给房间内所有连接

---

## 7. 参考文件

| 文件路径 | 说明 |
|---------|------|
| `server/uptime-kuma-server.js` | Socket.IO 服务初始化、房间管理、监控列表推送 |
| `server/server.js` | 连接处理、认证逻辑、登录流程、事件处理器注册 |
| `server/util-server.js` | `checkLogin` 权限检查、JWT 相关工具 |
| `server/client.js` | 各类数据推送函数（heartbeatList、notificationList 等） |
| `src/mixins/socket.js` | 前端 Socket.IO 客户端、事件监听、响应式数据 |
