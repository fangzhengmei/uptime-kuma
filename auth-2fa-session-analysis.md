# Uptime Kuma 认证、会话、2FA 和权限深度分析

## 1. 核心概念澄清

### 1.1 "Session" 在 Uptime Kuma 中的真实含义

Uptime Kuma **不使用传统的服务器端 Session**（如 express-session），而是采用 **JWT (JSON Web Token) 无状态机制**。

| 传统 Session 机制 | Uptime Kuma JWT 机制 |
|-----------------|---------------------|
| 服务器存储 session 数据 | 无状态，Token 自包含信息 |
| Session ID 存在 Cookie | JWT 存在 localStorage/sessionStorage |
| 每次请求查 Session 存储 | 每次请求验证 JWT 签名 |
| 服务器可主动使 Session 失效 | 依赖密码修改或 Secret 重置来失效 |

### 1.2 入口类型与鉴权模型总览

Uptime Kuma 有**6 种入口类型**，每种入口有**不同的鉴权模型**，`disableAuth` 对它们的影响也完全不同：

| 入口类型 | 典型端点 | 鉴权模型 | 是否受 disableAuth 影响 |
|---------|---------|---------|------------------------|
| **Socket 管理入口** | `socket.emit("add", ...)`, `socket.emit("editMonitor", ...)` | JWT + `socket.userID` | **是** |
| **HTTP API (basicAuth)** | 部分受保护的 HTTP 端点 | Basic Auth (用户名密码) | **是** |
| **HTTP API (apiAuth)** | API 管理端点 | Basic Auth (API Key 或用户名密码) | **是** |
| **公开状态页** | `/status/:slug`, `/api/status-page/:slug` | 无鉴权（设计上公开） | **否** |
| **Push API** | `/api/push/:pushToken` | pushToken 独立鉴权 | **否** |
| **Badge API** | `/api/badge/:id/status`, `/api/badge/:id/uptime` | monitor 公开分组检查 | **否** |

**关键区分**：
- **受 disableAuth 影响**：需要用户身份认证的管理操作
- **不受 disableAuth 影响**：设计上就是公开的，或有自己独立的鉴权机制

---

## 2. 各入口类型的鉴权模型详解

### 2.1 Socket 管理入口

#### 核心机制

**文件位置**: `server/server.js:1726-1734`

```javascript
// Socket 连接建立后的鉴权逻辑
io.on("connection", async (socket) => {
    // ... 绑定所有 socket handlers ...
    
    log.debug("auth", "check auto login");
    if (await setting("disableAuth")) {
        // disableAuth = true: 自动登录到 admin
        log.info("auth", "Disabled Auth: auto login to admin");
        await afterLogin(socket, await R.findOne("user"));
        socket.emit("autoLogin");
    } else {
        // disableAuth = false: 要求登录
        socket.emit("loginRequired");
        log.debug("auth", "need auth");
    }
});
```

#### afterLogin 的作用

**文件位置**: `server/server.js:1804-1818`

```javascript
async function afterLogin(socket, user) {
    socket.userID = user.id;           // 核心：设置 socket 级别的用户标识
    socket.join(user.id);              // 加入用户房间（用于广播）
    
    // 发送初始化数据：监控列表、状态页、通知等
    let monitorList = await server.sendMonitorList(socket);
    // ...
}
```

#### 权限检查：checkLogin

**文件位置**: `server/util-server.js:637-641`

```javascript
exports.checkLogin = (socket) => {
    if (!socket.userID) {
        throw new Error("You are not logged in.");
    }
};
```

#### disableAuth 对 Socket 入口的影响

| disableAuth 值 | 行为 |
|----------------|------|
| `false` (默认) | 发送 `loginRequired` 事件，前端需要调用 `login()` 或 `loginByToken()` |
| `true` | 自动执行 `afterLogin(socket, firstUser)`，设置 `socket.userID`，所有 `checkLogin` 检查通过 |

**影响范围**：
- 监控的增删改查
- 状态页的创建/编辑/删除
- 系统设置的读取/修改
- 2FA 的启用/禁用
- 所有需要 `checkLogin` 的 Socket 操作

---

### 2.2 HTTP API (basicAuth)

#### 核心机制

**文件位置**: `server/auth.js:132-146`

```javascript
exports.basicAuth = async function (req, res, next) {
    const middleware = basicAuth({
        authorizer: userAuthorizer,    // 用户名密码验证
        authorizeAsync: true,
        challenge: true,                // 触发浏览器 Basic Auth 弹窗
    });

    const disabledAuth = await Settings.get("disableAuth");

    if (!disabledAuth) {
        // 认证启用：执行 Basic Auth
        middleware(req, res, next);
    } else {
        // 认证禁用：直接跳过
        next();
    }
};
```

#### userAuthorizer 实现

**文件位置**: `server/auth.js:106-123`

```javascript
function userAuthorizer(username, password, callback) {
    // 登录速率限制
    loginRateLimiter.pass(null, 0).then((pass) => {
        if (pass) {
            // 验证用户名密码
            exports.login(username, password).then((user) => {
                callback(null, user != null);
                
                if (user == null) {
                    // 登录失败消耗 token
                    loginRateLimiter.removeTokens(1);
                }
            });
        } else {
            // 速率限制触发
            callback(null, false);
        }
    });
}
```

#### disableAuth 对 basicAuth 的影响

| disableAuth 值 | 行为 |
|----------------|------|
| `false` | 执行 `express-basic-auth` 中间件，要求用户名密码 |
| `true` | 直接 `next()`，跳过所有认证检查 |

---

### 2.3 HTTP API (apiAuth)

#### 核心机制

**文件位置**: `server/auth.js:155-176`

```javascript
exports.apiAuth = async function (req, res, next) {
    if (!(await Settings.get("disableAuth"))) {
        // 认证启用
        let usingAPIKeys = await Settings.get("apiKeysEnabled");
        let middleware;
        
        if (usingAPIKeys) {
            // API Key 模式：使用 apiAuthorizer
            middleware = basicAuth({
                authorizer: apiAuthorizer,
                authorizeAsync: true,
                challenge: true,
            });
        } else {
            // 普通模式：使用 userAuthorizer（用户名密码）
            middleware = basicAuth({
                authorizer: userAuthorizer,
                authorizeAsync: true,
                challenge: true,
            });
        }
        middleware(req, res, next);
    } else {
        // 认证禁用：直接跳过
        next();
    }
};
```

#### API Key 验证逻辑

**文件位置**: `server/auth.js:41-63`

```javascript
async function verifyAPIKey(key) {
    if (typeof key !== "string") {
        return false;
    }

    // API Key 格式：uk{ID}_{key}
    let index = key.substring(2, key.indexOf("_"));  // 提取 ID
    let clear = key.substring(key.indexOf("_") + 1, key.length);  // 提取实际 key

    let hash = await R.findOne("api_key", " id=? ", [index]);

    if (hash === null) {
        return false;
    }

    // 检查过期时间和是否激活
    let current = dayjs();
    let expiry = dayjs(hash.expires);
    if (expiry.diff(current) < 0 || !hash.active) {
        return false;
    }

    // 验证 key 哈希
    return hash && passwordHash.verify(clear, hash.key);
}
```

#### disableAuth 对 apiAuth 的影响

| disableAuth 值 | 行为 |
|----------------|------|
| `false` | 根据 `apiKeysEnabled` 设置，使用 API Key 或用户名密码验证 |
| `true` | 直接 `next()`，跳过所有认证检查 |

---

### 2.4 公开状态页入口

#### 核心机制

**文件位置**: `server/routers/status-page-router.js`

公开状态页的所有端点**完全不包含认证检查**，设计上就是公开的：

```javascript
// 状态页展示页面
router.get("/status/:slug", cache("5 minutes"), async (request, response) => {
    let slug = request.params.slug;
    slug = slug.toLowerCase();
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});

// 状态页配置 API
router.get("/api/status-page/:slug", cache("5 minutes"), async (request, response) => {
    allowDevAllOrigin(response);
    let slug = request.params.slug;
    slug = slug.toLowerCase();

    try {
        let statusPage = await R.findOne("status_page", " slug = ? ", [slug]);
        if (!statusPage) {
            sendHttpError(response, "Status Page Not Found");
            return null;
        }
        let statusPageData = await StatusPage.getStatusPageData(statusPage);
        response.json(statusPageData);
    } catch (error) {
        sendHttpError(response, error.message);
    }
});

// 状态页心跳数据
router.get("/api/status-page/heartbeat/:slug", cache("1 minutes"), async (request, response) => {
    allowDevAllOrigin(response);
    // ... 直接返回公开分组的 monitor 数据，无认证检查
});
```

#### 公开状态页的端点列表

| 端点 | 说明 | 是否受 disableAuth 影响 |
|-----|------|------------------------|
| `GET /status/:slug` | 状态页展示页面 | **否** |
| `GET /status` | 默认状态页 | **否** |
| `GET /status-page` | 默认状态页别名 | **否** |
| `GET /api/status-page/:slug` | 状态页配置 API | **否** |
| `GET /api/status-page/heartbeat/:slug` | 状态页心跳数据 | **否** |
| `GET /api/status-page/:slug/rss` | 状态页 RSS 订阅 | **否** |
| `GET /api/status-page/:slug/manifest.json` | PWA manifest | **否** |
| `GET /api/status-page/:slug/badge` | 状态页整体状态徽章 | **否** |
| `GET /api/status-page/:slug/incident-history` | 事件历史 | **否** |

#### 公开状态页的数据过滤机制

虽然不需要认证，但状态页有自己的**数据过滤机制**：

```javascript
// server/routers/status-page-router.js:75-94
let monitorIDList = await R.getCol(
    `
    SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
    WHERE monitor_group.group_id = \`group\`.id
    AND public = 1          -- 关键：只返回公开分组的 monitor
    AND \`group\`.status_page_id = ?
`,
    [statusPageID]
);
```

**说明**：
- 状态页只展示加入了**公开分组** (`public = 1`) 的 monitor
- 非公开分组的 monitor 不会出现在状态页上
- 这与 `disableAuth` 无关，是状态页本身的设计

---

### 2.5 Push API 入口

#### 核心机制

**文件位置**: `server/routers/api-router.js:47-146`

Push API 使用 **pushToken 独立鉴权**，完全不依赖 `disableAuth` 设置：

```javascript
router.all("/api/push/:pushToken", async (request, response) => {
    try {
        let pushToken = request.params.pushToken;
        let msg = request.query.msg || "OK";
        let ping = parseFloat(request.query.ping) || null;
        let statusString = request.query.status || "up";

        // 核心：通过 pushToken 查找 monitor
        let monitor = await R.findOne(
            "monitor", 
            " push_token = ? AND active = 1 ", 
            [pushToken]
        );

        if (!monitor) {
            throw new Error("Monitor not found or not active.");
        }

        // 处理心跳数据...
        // ...
    } catch (e) {
        response.status(404).json({ ok: false, msg: e.message });
    }
});
```

#### Push API 的设计特点

| 特性 | 说明 |
|-----|------|
| 认证方式 | `pushToken`（URL 路径参数） |
| Token 存储位置 | `monitor.push_token` 字段 |
| 有效条件 | `monitor.active = 1` |
| HTTP 方法 | `GET` 和 `POST` 都支持 |
| 是否受 disableAuth 影响 | **否** |

#### 使用示例

```bash
# 推送心跳（up 状态）
curl "http://localhost:3001/api/push/abc123def456?status=up&msg=OK&ping=23"

# 推送心跳（down 状态）
curl "http://localhost:3001/api/push/abc123def456?status=down&msg=Connection%20failed"
```

#### pushToken 的生成

pushToken 在创建/编辑 monitor 时生成：

```javascript
// 通常在创建 monitor 时设置 push_token
// 用户可以在 UI 中重新生成 push_token
```

**安全提示**：
- pushToken 是敏感信息，泄露后任何人都可以推送心跳
- 应该定期重新生成 pushToken
- `disableAuth` 不影响 Push API 的安全性，因为它有自己的 token 机制

---

### 2.6 Badge API 入口

#### 核心机制

**文件位置**: `server/routers/api-router.js:148-565`

Badge API 使用 **monitor 公开分组检查**，完全不依赖 `disableAuth` 设置：

```javascript
router.get("/api/badge/:id/status", cache("5 minutes"), async (request, response) => {
    allowAllOrigin(response);  // 允许跨域

    try {
        const requestedMonitorId = parseInt(request.params.id, 10);
        
        // 关键：检查 monitor 是否公开
        const publicMonitor = await isMonitorPublic(requestedMonitorId);
        const badgeValues = { style };

        if (!publicMonitor) {
            // 非公开 monitor：返回灰色的 "N/A" 徽章
            badgeValues.message = "N/A";
            badgeValues.color = badgeConstants.naColor;
        } else {
            // 公开 monitor：返回实际状态
            const heartbeat = await Monitor.getPreviousHeartbeat(requestedMonitorId);
            // ... 根据状态设置徽章颜色和文字
        }

        const svg = makeBadge(badgeValues);
        response.type("image/svg+xml");
        response.send(svg);
    } catch (error) {
        sendHttpError(response, error.message);
    }
});
```

#### isMonitorPublic 函数

**文件位置**: `server/routers/api-router.js:626-637`

```javascript
async function isMonitorPublic(monitorID) {
    let publicMonitor = await R.getRow(
        `
            SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
            WHERE monitor_group.group_id = \`group\`.id
            AND monitor_group.monitor_id = ?
            AND public = 1          -- 关键：检查是否加入公开分组
        `,
        [monitorID]
    );
    return !!publicMonitor;
}
```

#### Badge API 端点列表

| 端点 | 说明 | 是否受 disableAuth 影响 |
|-----|------|------------------------|
| `GET /api/badge/:id/status` | 监控状态徽章（Up/Down） | **否** |
| `GET /api/badge/:id/uptime/:duration?` | 在线率徽章（如 99.9%） | **否** |
| `GET /api/badge/:id/ping/:duration?` | 平均延迟徽章 | **否** |
| `GET /api/badge/:id/avg-response/:duration?` | 平均响应时间徽章 | **否** |
| `GET /api/badge/:id/cert-exp` | 证书过期天数徽章 | **否** |
| `GET /api/badge/:id/response` | 当前响应时间徽章 | **否** |

#### Badge API 的设计特点

| 特性 | 说明 |
|-----|------|
| 认证方式 | monitor 是否加入公开分组 (`public = 1`) |
| 非公开 monitor 行为 | 返回灰色的 "N/A" 徽章 |
| 缓存策略 | 5 分钟缓存（减少数据库压力） |
| 跨域策略 | `allowAllOrigin`，允许任何网站嵌入 |
| 是否受 disableAuth 影响 | **否** |

#### 使用示例

```html
<!-- 在 README 或网站中嵌入状态徽章 -->
![Status](http://localhost:3001/api/badge/1/status)
![Uptime](http://localhost:3001/api/badge/1/uptime/24h)
![Ping](http://localhost:3001/api/badge/1/ping/24h)
```

---

## 3. 按入口类型划分的权限边界对照表

### 3.1 完整对照表

| 入口类型 | 端点示例 | 鉴权模型 | disableAuth=false 时的行为 | disableAuth=true 时的行为 | 核心差异点 |
|---------|---------|---------|---------------------------|--------------------------|-----------|
| **Socket 管理入口** | `add`, `editMonitor`, `deleteMonitor`, `getSettings`, `postIncident` | JWT + `socket.userID` + `checkLogin` | 发送 `loginRequired`，需要 `login()` 或 `loginByToken()`，所有操作需 `checkLogin` 通过 | 自动执行 `afterLogin()`，设置 `socket.userID`，所有 `checkLogin` 自动通过 | 所有管理操作的权限由 `disableAuth` 开关控制 |
| **HTTP basicAuth** | 受保护的 HTTP 端点 | `express-basic-auth` (用户名密码) | 触发浏览器 Basic Auth 弹窗，验证用户名密码 | 直接跳过认证，`next()` | HTTP 层面的基础认证 |
| **HTTP apiAuth** | API 管理端点 | `express-basic-auth` (API Key 或用户名密码) | 根据 `apiKeysEnabled`，使用 API Key 或用户名密码验证 | 直接跳过认证，`next()` | 支持 API Key 替代密码认证 |
| **公开状态页** | `/status/:slug`, `/api/status-page/:slug` | 无鉴权（设计上公开） | 始终公开，无认证要求 | 始终公开，无认证要求 | **完全不受 disableAuth 影响**，有自己的公开分组过滤 |
| **Push API** | `/api/push/:pushToken` | `pushToken` 独立鉴权 | 通过 `push_token` 查找 monitor，不检查 `disableAuth` | 通过 `push_token` 查找 monitor，不检查 `disableAuth` | **完全不受 disableAuth 影响**，有自己独立的 token 机制 |
| **Badge API** | `/api/badge/:id/status`, `/api/badge/:id/uptime` | monitor 公开分组检查 | 检查 `isMonitorPublic()`，非公开返回 "N/A" | 检查 `isMonitorPublic()`，非公开返回 "N/A" | **完全不受 disableAuth 影响**，依赖公开分组设置 |

### 3.2 disableAuth 影响范围可视化

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    disableAuth 影响范围示意图                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      受 disableAuth 影响的入口                        │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │                                                             │   │  │
│  │  │   Socket 管理入口                                            │   │  │
│  │  │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │   │  │
│  │  │   │监控管理  │ │状态页管理│ │系统设置  │ │  2FA    │        │   │  │
│  │  │   │add/edit │ │postInc-  │ │get/set- │ │enable/  │        │   │  │
│  │  │   │/delete  │ │ident    │ │Settings │ │disable  │        │   │  │
│  │  │   └─────────┘ └─────────┘ └─────────┘ └─────────┘        │   │  │
│  │  │                                                             │   │  │
│  │  │   HTTP API (basicAuth / apiAuth)                           │   │  │
│  │  │   ┌─────────────────────────────────────────────────┐     │   │  │
│  │  │   │ Basic Auth 弹窗 / API Key 验证                   │     │   │  │
│  │  │   └─────────────────────────────────────────────────┘     │   │  │
│  │  │                                                             │   │  │
│  │  │   影响：                                                    │   │  │
│  │  │   disableAuth=true 时，所有认证检查被跳过                  │   │  │
│  │  │   任何人都可以进行管理操作                                  │   │  │
│  │  │                                                             │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                   不受 disableAuth 影响的入口                         │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │                                                             │   │  │
│  │  │   公开状态页                                                 │   │  │
│  │  │   ┌─────────────────────────────────────────────────┐     │   │  │
│  │  │   │ /status/:slug, /api/status-page/:slug, RSS     │     │   │  │
│  │  │   │ 始终公开，依赖状态页的 published 状态             │     │   │  │
│  │  │   └─────────────────────────────────────────────────┘     │   │  │
│  │  │                                                             │   │  │
│  │  │   Push API                                                  │   │  │
│  │  │   ┌─────────────────────────────────────────────────┐     │   │  │
│  │  │   │ /api/push/:pushToken                             │     │   │  │
│  │  │   │ 独立鉴权：通过 push_token 查找 monitor           │     │   │  │
│  │  │   │ pushToken 泄露 = monitor 可被任意推送心跳        │     │   │  │
│  │  │   └─────────────────────────────────────────────────┘     │   │  │
│  │  │                                                             │   │  │
│  │  │   Badge API                                                 │   │  │
│  │  │   ┌─────────────────────────────────────────────────┐     │   │  │
│  │  │   │ /api/badge/:id/status, /api/badge/:id/uptime   │     │   │  │
│  │  │   │ 依赖 isMonitorPublic() 检查                      │     │   │  │
│  │  │   │ monitor 必须加入公开分组才能显示数据              │     │   │  │
│  │  │   │ 非公开 monitor 返回 "N/A" 灰色徽章               │     │   │  │
│  │  │   └─────────────────────────────────────────────────┘     │   │  │
│  │  │                                                             │   │  │
│  │  │   共同点：                                                  │   │  │
│  │  │   - 设计上就是公开的，或有自己独立的鉴权机制              │   │  │
│  │  │   - disableAuth 的开关状态不影响它们的行为                │   │  │
│  │  │                                                             │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 系统设置入口的二次验密条件

### 4.1 唯一需要双重验密的场景

**文件位置**: `server/server.js:1475-1535`

```javascript
socket.on("setSettings", async (data, currentPassword, callback) => {
    try {
        checkLogin(socket);

        // 注释非常重要：
        // Disabled Auth + Want to Disable Auth => No Check
        // Disabled Auth + Want to Enable Auth => No Check
        // Enabled Auth + Want to Disable Auth => Check!!  ← 唯一需要双重验密的场景
        // Enabled Auth + Want to Enable Auth => No Check
        
        const currentDisabledAuth = await setting("disableAuth");
        if (!currentDisabledAuth && data.disableAuth) {
            // 条件：
            // 1. 当前认证已启用 (!currentDisabledAuth)
            // 2. 用户想要禁用认证 (data.disableAuth)
            await doubleCheckPassword(socket, currentPassword);
        }

        // 其他设置修改不需要双重验密
        await setSettings("general", data);
        
        // 重要：启用认证时，断开所有其他客户端连接
        if (currentDisabledAuth && !data.disableAuth) {
            server.disconnectAllSocketClients(socket.userID, socket.id);
        }
        
        callback({ ok: true });
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 4.2 场景决策表

| 当前 disableAuth 值 | 目标 disableAuth 值 | 操作含义 | 是否需要 doubleCheckPassword | 原因 |
|---------------------|---------------------|---------|------------------------------|------|
| `true` (已禁用) | `true` (保持禁用) | 无安全状态变化 | 否 | 无风险 |
| `true` (已禁用) | `false` (启用认证) | **安全升级** | 否 | 提升安全性，不需要额外验证 |
| `false` (已启用) | `true` (禁用认证) | **安全降级** | **是** | 高危操作，必须验证身份 |
| `false` (已启用) | `false` (保持启用) | 无安全状态变化 | 否 | 无风险 |

### 4.3 为什么禁用认证需要双重验密？

禁用认证 (`disableAuth = true`) 会导致：

1. **Socket 管理入口完全开放**
   - 所有 `checkLogin` 检查自动通过
   - 任何人可以添加/修改/删除监控
   - 任何人可以修改系统设置
   - 任何人可以禁用/启用 2FA

2. **HTTP API 完全开放**
   - `basicAuth` 和 `apiAuth` 中间件直接跳过
   - 无需任何认证即可访问受保护的 HTTP 端点

3. **但公开状态页、Push API、Badge API 不受影响**
   - 它们本来就是公开的或有自己的独立鉴权机制

**这是一个不可逆的安全降级操作**，因此要求用户再次确认密码，防止：
- CSRF 攻击
- 会话劫持后的恶意操作
- 误操作

### 4.4 系统设置操作权限汇总

| 操作 | 权限检查 | 是否需要双重验密 | 触发条件 |
|-----|---------|-----------------|---------|
| `getSettings` | `checkLogin` | 否 | - |
| `setSettings` (修改时区) | `checkLogin` | 否 | - |
| `setSettings` (修改入口页) | `checkLogin` | 否 | - |
| `setSettings` (修改 Chrome 路径) | `checkLogin` | 否 | - |
| `setSettings` (修改 API Keys 设置) | `checkLogin` | 否 | - |
| `setSettings` (**禁用认证**) | `checkLogin` | **是** | 仅当 `disableAuth` 从 `false` 变为 `true` |
| `setSettings` (**启用认证**) | `checkLogin` | 否 | 安全升级操作 |

---

## 5. 监控操作：仅登录校验 vs 资源归属校验

### 5.1 发现的权限检查不一致

监控相关操作的权限检查**不一致**，存在两种模式：

#### 模式 A：资源归属校验（相对安全）

| 操作 | 检查方式 | 代码位置 | 检查逻辑 |
|-----|---------|---------|---------|
| `getMonitor` | SQL 条件隐式检查 | server.js:993 | `WHERE id = ? AND user_id = ?` |
| `editMonitor` | 显式比较 | server.js:807-809 | `if (bean.user_id !== socket.userID)` |
| `deleteMonitor` | SQL 条件隐式检查 | server.js:1116 | `WHERE id = ? AND user_id = ?` |
| `resumeMonitor` | `checkOwner` 函数 | server.js:1067-1068 | 内部调用 `checkOwner` |
| `pauseMonitor` | `checkOwner` 函数 | server.js:1085-1086 | 内部调用 `checkOwner` |

#### editMonitor 的显式检查示例

```javascript
// server/server.js:800-809
socket.on("editMonitor", async (monitor, callback) => {
    try {
        checkLogin(socket);
        
        let bean = await R.findOne("monitor", " id = ? ", [monitor.id]);
        // 显式检查所有权
        if (bean.user_id !== socket.userID) {
            throw new Error("Permission denied.");
        }
        // ... 修改逻辑
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

---

#### 模式 B：仅登录校验（潜在隐患）

| 操作 | 权限检查方式 | 代码位置 | 是否检查资源归属 | 风险 |
|-----|-------------|---------|-----------------|------|
| `add` (新建监控) | `checkLogin` only | server.js:727 | 否 | 新建操作，合理 |
| `getMonitorList` | `checkLogin` only | server.js:973 | 否 | SQL 过滤返回，合理 |
| `getMonitorBeats` | `checkLogin` only | server.js:1031 | **否** | 多用户场景下可查看任意 monitor 的心跳历史 |
| `addMonitorTag` | `checkLogin` only | server.js:1284 | **否** | 多用户场景下可为任意 monitor 添加标签 |
| `deleteMonitorTag` | `checkLogin` only | server.js:1334 | **否** | 多用户场景下可删除任意 monitor 的标签 |

#### getMonitorBeats 的隐患示例

```javascript
// server/server.js:1029-1062
socket.on("getMonitorBeats", async (monitorID, period, callback) => {
    try {
        checkLogin(socket);  // 只检查是否登录
        
        // ⚠️ 没有检查 monitorID 是否属于 socket.userID！
        // SQL 中只过滤了 monitor_id，没有 user_id
        let list = await R.getAll(
            `SELECT * FROM heartbeat 
             WHERE monitor_id = ? AND time > ${sqlHourOffset}
             ORDER BY time ASC`,
            [monitorID, -period]
        );
        
        callback({ ok: true, data: list });
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 5.2 标签操作的特殊性

标签系统分为两层：

| 层级 | 操作 | 权限检查 | 设计意图 |
|-----|------|---------|---------|
| 标签定义 | `addTag`, `editTag`, `deleteTag` | `checkLogin` only | 标签是系统级资源，不属于特定用户 |
| 标签关联 | `addMonitorTag`, `deleteMonitorTag` | `checkLogin` only | **应该检查 monitor 归属，但实际没有** |

### 5.3 当前架构下的实际影响

**重要说明**：当前 Uptime Kuma 设计上是**单用户系统**，所以这些"隐患"不会实际触发风险。

```
单用户场景下：
- 所有 monitor 的 user_id 都相同（第一个用户的 ID）
- checkOwner 检查永远通过
- getMonitorBeats 等操作只能访问自己的 monitor
- 没有"其他用户"的概念

多用户场景下（未来可能的扩展）：
- checkOwner 会真正起作用
- getMonitorBeats 等操作需要添加归属检查
- 当前实现存在越权风险
```

### 5.4 监控操作权限检查差异图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          监控操作权限检查差异                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                     资源归属校验（相对安全）                           │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │  │
│  │  │ getMonitor│ │editMonitor│ │deleteMonitor│ │resumeMonitor│ │pauseMonitor│ │  │
│  │  │          │ │          │ │          │ │          │ │          │ │  │
│  │  │ SQL隐式  │ │ 显式比较  │ │ SQL隐式  │ │checkOwner│ │checkOwner│ │  │
│  │  │ AND user_id│ │bean.user_id│ │ AND user_id│ │  函数   │ │  函数   │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                     仅登录校验（潜在隐患）                             │  │
│  │  ┌──────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │  │
│  │  │   add    │ │getMonitorList│ │getMonitorBeats│ │addMonitorTag │ │  │
│  │  │ (新建)   │ │ (SQL过滤返回) │ │   ⚠️ 隐患    │ │  ⚠️ 隐患    │ │  │
│  │  └──────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │  │
│  │                                                      ┌──────────────┐│  │
│  │                                                      │deleteMonitorTag││  │
│  │                                                      │  ⚠️ 隐患    ││  │
│  │                                                      └──────────────┘│  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  说明：                                                                      │
│  - "新建"操作不需要归属检查是合理的                                          │
│  - "getMonitorList" 通过 SQL user_id 条件过滤返回结果，也是安全的           │
│  - 但 "getMonitorBeats"、"addMonitorTag"、"deleteMonitorTag" 没有检查      │
│    monitor 是否属于当前用户，在多用户场景下存在越权风险                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 状态页管理的保护边界

### 6.1 状态页的两类操作

状态页分为**两类完全不同的操作**，有不同的保护边界：

#### 类型 1：公开读取操作（HTTP 端点）

| 操作 | 入口类型 | 鉴权模型 | 说明 |
|-----|---------|---------|------|
| 查看状态页 | 公开状态页入口 | 无鉴权 | 始终公开，不依赖 `disableAuth` |
| 获取状态页配置 API | 公开状态页入口 | 无鉴权 | 始终公开 |
| 获取心跳数据 | 公开状态页入口 | 无鉴权 | 仅返回公开分组的 monitor |
| RSS 订阅 | 公开状态页入口 | 无鉴权 | 始终公开 |
| 事件历史 | 公开状态页入口 | 无鉴权 | 仅返回已发布的事件 |

#### 类型 2：管理操作（Socket 端点）

| 操作 | 入口类型 | 鉴权模型 | 是否受 disableAuth 影响 |
|-----|---------|---------|------------------------|
| 创建状态页 (`addStatusPage`) | Socket 管理入口 | `checkLogin` | **是** |
| 编辑状态页 (`saveStatusPage`) | Socket 管理入口 | `checkLogin` | **是** |
| 删除状态页 (`deleteStatusPage`) | Socket 管理入口 | `checkLogin` | **是** |
| 发布事件 (`postIncident`) | Socket 管理入口 | `checkLogin` | **是** |
| 编辑事件 (`editIncident`) | Socket 管理入口 | `checkLogin` | **是** |
| 删除事件 (`deleteIncident`) | Socket 管理入口 | `checkLogin` | **是** |
| 解决事件 (`resolveIncident`) | Socket 管理入口 | `checkLogin` | **是** |
| 获取管理配置 (`getStatusPage`) | Socket 管理入口 | `checkLogin` | **是** |

### 6.2 状态页管理的保护边界分析

#### 保护边界 1：公开读取 vs 管理操作

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    状态页访问边界                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  外部用户 (未登录)                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  公开读取操作（HTTP 端点）                                             │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │ ✓ GET /status/{slug} - 状态页展示                            │   │  │
│  │  │ ✓ GET /api/status-page/{slug} - 状态页配置                   │   │  │
│  │  │ ✓ GET /api/status-page/heartbeat/{slug} - 心跳数据          │   │  │
│  │  │ ✓ GET /api/status-page/{slug}/rss - RSS 订阅                 │   │  │
│  │  │ ✓ GET /api/status-page/{slug}/badge - 状态徽章               │   │  │
│  │  │                                                              │   │  │
│  │  │ 限制：                                                        │   │  │