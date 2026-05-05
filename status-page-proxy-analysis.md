# Uptime Kuma 状态页代理暴露安全分析报告

## 概述
**分析日期**: 2026-05-05
**分析对象**: Uptime Kuma v1.x 状态页通过 Cloudflared 或反向代理对外暴露时的安全机制

---

## 一、路由分离机制

### 1.1 后端路由架构

Uptime Kuma 采用 Express.js 作为后端框架，路由采用模块化设计：

#### 状态页面路由（公开访问）
文件位置: `server/routers/status-page-router.js`

```javascript
// 状态页面路由（公开访问，无认证）
router.get("/status/:slug", cache("5 minutes"), async (request, response) => {...});
router.get("/status", cache("5 minutes"), async (request, response) => {...});
router.get("/status-page", cache("5 minutes"), async (request, response) => {...});

// 状态页面 API（公开访问，无认证）
router.get("/api/status-page/:slug", cache("5 minutes"), async (request, response) => {...});
router.get("/api/status-page/heartbeat/:slug", cache("1 minutes"), async (request, response) => {...});
router.get("/api/status-page/:slug/manifest.json", cache("1440 minutes"), async (request, response) => {...});
router.get("/api/status-page/:slug/incident-history", cache("5 minutes"), async (request, response) => {...});
router.get("/api/status-page/:slug/badge", cache("5 minutes"), async (request, response) => {...});
```

**关键特征**:
- 所有状态页相关路由**没有任何认证中间件**
- 使用 `apicache` 缓存机制（5分钟/1分钟不等）
- 完全公开访问

#### API 路由（混合访问）
文件位置: `server/routers/api-router.js`

```javascript
// 公开 API（无认证）
router.get("/api/entry-page", async (request, response) => {...});
router.all("/api/push/:pushToken", async (request, response) => {...});
router.get("/api/badge/:id/status", cache("5 minutes"), async (request, response) => {...});
router.get("/api/badge/:id/uptime/:duration?", cache("5 minutes"), async (request, response) => {...});
// ... 其他 badge API
```

#### 主服务器路由（入口控制）
文件位置: `server/server.js`

```javascript
// 入口页面路由
app.get("/", async (request, response) => {
    let hostname = request.hostname;
    if (await setting("trustProxy")) {
        const proxy = request.headers["x-forwarded-host"];
        if (proxy) {
            hostname = proxy;
        }
    }

    // 域名映射检查
    if (hostname in StatusPage.domainMappingList) {
        // 匹配到状态页域名，显示状态页
        let slug = StatusPage.domainMappingList[hostname];
        await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
    } else if (uptimeKumaEntryPage && uptimeKumaEntryPage.startsWith("statusPage-")) {
        // 入口页配置为状态页
        response.redirect("/status/" + uptimeKumaEntryPage.replace("statusPage-", ""));
    } else {
        // 默认重定向到管理后台
        response.redirect("/dashboard");
    }
});
```

### 1.2 前端路由架构

文件位置: `src/router.js`

```javascript
const routes = [
    // 入口页
    {
        path: "/",
        component: Entry,
    },
    
    // 管理后台路由（需要认证）
    {
        path: "/empty",
        component: Layout,
        children: [
            {
                path: "/dashboard",
                component: DashboardHome,
                // ... 其他管理页面
            },
            {
                path: "/settings",
                component: Settings,
                // ... 设置页面
            },
            {
                path: "/manage-status-page",
                component: ManageStatusPage,
            },
            // ... 更多管理路由
        ],
    },
    
    // 状态页面路由（公开访问）
    {
        path: "/status-page",
        component: StatusPage,
    },
    {
        path: "/status",
        component: StatusPage,
    },
    {
        path: "/status/:slug",
        component: StatusPage,
    },
];
```

### 1.3 路由分离总结

| 路由类型 | 路径模式 | 访问权限 | 认证机制 |
|---------|---------|---------|----------|
| **状态页面** | `/status/*`, `/status-page/*` | **公开** | 无 |
| **状态页 API** | `/api/status-page/*` | **公开** | 无 |
| **徽章 API** | `/api/badge/*` | **公开** | 无 |
| **Push API** | `/api/push/:pushToken` | 半公开 | Token 验证 |
| **管理后台** | `/dashboard/*`, `/settings/*` | **私有** | Socket.io JWT |
| **Prometheus** | `/metrics` | **私有** | Basic Auth / API Key |

---

## 二、状态页与管理后台的前后端分流机制

### 2.1 为什么状态页不会走进登录后台链路

Uptime Kuma 采用了**多层分离机制**，确保状态页访问不会触发管理后台的认证链路。

#### 第一层：前端 Socket.io 连接排除机制

文件位置: `src/mixins/socket.js:19-98`

```javascript
// 明确排除状态页路径，不建立 Socket.io 连接
const noSocketIOPages = [
    /^\/status-page$/, //  /status-page
    /^\/status/,       // /status**
    /^\/$/,            //  /
];

methods: {
    initSocketIO(bypass = false) {
        // 已初始化则跳过
        if (this.socket.initedSocketIO) {
            return;
        }

        // 关键：状态页路径不建立 Socket.io 连接
        if (!bypass && location.pathname) {
            for (let page of noSocketIOPages) {
                if (location.pathname.match(page)) {
                    return;  // 直接返回，不建立连接
                }
            }
        }

        // 也不需要为数据库设置页面建立连接
        if (location.pathname === "/setup-database") {
            return;
        }

        // 只有管理后台页面才会执行到这里，建立 Socket.io 连接
        this.socket.initedSocketIO = true;
        socket = io(url);
        // ... 后续 Socket.io 事件监听
    },
}
```

**工作原理**：
1. `noSocketIOPages` 数组定义了需要排除的路径模式
2. `initSocketIO()` 函数在组件创建时被调用
3. 如果当前路径匹配状态页模式，函数直接 `return`，**不会建立 Socket.io 连接**
4. 没有 Socket.io 连接，就不会触发后续的 `loginRequired` 事件和登录流程

#### 第二层：路由切换时的动态连接控制

文件位置: `src/mixins/socket.js:877-893`

```javascript
watch: {
    // 从状态页切换到管理后台时，动态建立 Socket.io 连接
    "$route.fullPath"(newValue, oldValue) {
        if (newValue) {
            for (let page of noSocketIOPages) {
                if (newValue.match(page)) {
                    return;  // 状态页路径，不建立连接
                }
            }
        }

        // 非状态页路径，确保 Socket.io 已初始化
        this.initSocketIO();
    },
},
```

#### 第三层：状态页数据获取方式（HTTP API，非 Socket.io）

文件位置: `src/pages/StatusPage.vue`

状态页组件不依赖 Socket.io 获取数据，而是通过**公开的 HTTP API**：

```javascript
// 状态页使用 axios 调用公开 API，不使用 Socket.io
created() {
    // 前端本地检查是否有 token（仅用于显示编辑按钮）
    this.hasToken = "token" in this.$root.storage();
    
    // 数据通过公开 HTTP API 获取
    this.loadData();
},

methods: {
    async loadData() {
        // 调用公开的状态页 API，无认证要求
        const res = await axios.get(`/api/status-page/${this.slug}`);
        this.config = res.data.config;
        // ...
    },
    
    // 编辑模式才需要 Socket.io
    edit() {
        if (this.hasToken) {
            // 强制建立 Socket.io 连接（bypass=true）
            this.$root.initSocketIO(true);
            this.enableEditMode = true;
            // ...
        }
    },
}
```

**关键区别**：

| 场景 | 数据获取方式 | 是否建立 Socket.io | 是否触发认证 |
|-----|------------|-------------------|-------------|
| **状态页浏览** | HTTP API (`/api/status-page/*`) | ❌ 否 | ❌ 否 |
| **状态页编辑模式** | Socket.io | ✅ 是（`bypass=true`） | ✅ 是 |
| **管理后台** | Socket.io | ✅ 是 | ✅ 是 |

#### 第四层：后端状态页路由无认证中间件

文件位置: `server/routers/status-page-router.js`

```javascript
let router = express.Router();

// 状态页路由 - 完全没有认证中间件
router.get("/status/:slug", cache("5 minutes"), async (request, response) => {
    let slug = request.params.slug;
    slug = slug.toLowerCase();
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});

// 状态页 API - 也没有认证中间件
router.get("/api/status-page/:slug", cache("5 minutes"), async (request, response) => {
    allowDevAllOrigin(response);
    let slug = request.params.slug;
    slug = slug.toLowerCase();

    try {
        let statusPage = await R.findOne("status_page", " slug = ? ", [slug]);
        // ... 直接返回数据，无认证检查
    } catch (error) {
        sendHttpError(response, error.message);
    }
});

module.exports = router;
```

对比管理操作的认证要求（`server/server.js`）：

```javascript
// 管理操作必须通过 checkLogin 检查
socket.on("add", async (monitor, callback) => {
    try {
        checkLogin(socket);  // 强制认证
        // ... 后续操作
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 2.2 前端 `hasToken` 的作用与局限

文件位置: `src/pages/StatusPage.vue:690, 964, 1133-1142`

```javascript
data() {
    return {
        hasToken: false,  // 只是前端状态，不影响后端认证
        // ...
    };
},

async created() {
    // 前端本地检查 localStorage/sessionStorage 中是否有 token
    this.hasToken = "token" in this.$root.storage();
    // 注意：这只是前端判断，不影响后端认证逻辑
},

methods: {
    edit() {
        if (this.hasToken) {
            // 只有前端判断有 token 时才尝试进入编辑模式
            this.$root.initSocketIO(true);  // 强制建立 Socket.io 连接
            this.enableEditMode = true;
            // 但真正的认证还是在后端进行
        }
    },
},
```

**关键理解**：
- `hasToken` 只是**前端本地状态**，用于控制 UI 显示（如编辑按钮）
- 即使前端绕过这个检查，后端 `checkLogin` 仍然会验证 `socket.userID`
- 真正的认证发生在后端的 `loginByToken` 和 `checkLogin` 环节

---

## 二、认证机制详解

### 2.1 Socket.io 认证机制（管理后台）

文件位置: `server/server.js`

#### 登录流程

```javascript
// 通过用户名密码登录
socket.on("login", async (data, callback) => {
    // 登录速率限制
    if (!(await loginRateLimiter.pass(callback))) {
        return;
    }
    
    // 验证用户名密码
    let user = await login(data.username, data.password);
    
    if (user) {
        // 2FA 检查
        if (user.twofa_status === 0) {
            await afterLogin(socket, user);
            callback({
                ok: true,
                token: User.createJWT(user, server.jwtSecret),
            });
        }
        // ... 2FA 处理
    }
});

// 通过 JWT Token 登录
socket.on("loginByToken", async (token, callback) => {
    try {
        let decoded = jwt.verify(token, server.jwtSecret);
        
        let user = await R.findOne("user", " username = ? AND active = 1 ", [decoded.username]);
        
        if (user) {
            // 检查密码是否变更（token 失效）
            if (decoded.h !== shake256(user.password, SHAKE256_LENGTH)) {
                throw new Error("The token is invalid due to password change or old token");
            }
            
            await afterLogin(socket, user);
            callback({ ok: true });
        }
    } catch (error) {
        callback({ ok: false, msg: "authInvalidToken" });
    }
});
```

#### 认证检查中间件

文件位置: `server/util-server.js`

```javascript
/**
 * 检查 Socket 是否已登录
 * @param {Socket} socket Socket.io 连接对象
 * @throws {Error} 如果未登录
 */
function checkLogin(socket) {
    if (!socket.userID) {
        throw new Error("Unauthorized");
    }
}
```

**使用示例（管理操作）:
```javascript
// 所有管理操作都需要先调用 checkLogin
socket.on("add", async (monitor, callback) => {
    try {
        checkLogin(socket);  // 强制认证检查
        // ... 后续操作
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 2.2 HTTP Basic Auth 机制

文件位置: `server/auth.js`

```javascript
/**
 * 用户认证（用户名密码认证
 */
exports.basicAuth = async function (req, res, next) {
    const middleware = basicAuth({
        authorizer: userAuthorizer,
        authorizeAsync: true,
        challenge: true,
    });

    const disabledAuth = await Settings.get("disableAuth");
    
    if (!disabledAuth) {
        middleware(req, res, next);
    } else {
        next();
    }
};

/**
 * API Key 认证
 */
exports.apiAuth = async function (req, res, next) {
    if (!(await Settings.get("disableAuth"))) {
        let usingAPIKeys = await Settings.get("apiKeysEnabled");
        let middleware;
        if (usingAPIKeys) {
            middleware = basicAuth({
                authorizer: apiAuthorizer,  // 使用 API Key 验证
                authorizeAsync: true,
                challenge: true,
            });
        } else {
            middleware = basicAuth({
                authorizer: userAuthorizer,  // 使用用户名密码验证
                authorizeAsync: true,
                challenge: true,
            });
        }
        middleware(req, res, next);
    } else {
        next();
    }
};
```

### 2.3 API Key 验证机制

文件位置: `server/auth.js`

```javascript
/**
 * 验证 API Key
 * 格式: uk{keyID}_{keyContent}
 */
async function verifyAPIKey(key) {
    if (typeof key !== "string") {
        return false;
    }

    // 解析 Key 格式
    let index = key.substring(2, key.indexOf("_"));  // Key ID
    let clear = key.substring(key.indexOf("_") + 1, key.length);  // Key 内容

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

    // 验证哈希
    return hash && passwordHash.verify(clear, hash.key);
}
```

---

## 三、代理 Header 处理机制

### 3.1 信任代理设置 (`trustProxy`)

`trustProxy` 是 Uptime Kuma 中最关键的代理相关设置，控制是否信任反向代理传递的 `X-Forwarded-*` 系列 Header。

#### 设置位置
- 前端设置页面: `src/components/settings/ReverseProxy.vue`
- 数据库存储: `setting` 表，key 为 `trustProxy`

#### 使用场景汇总

| 场景 | 文件位置 | 相关 Header | 代码引用 |
|-----|---------|------------|---------|
| **入口页面域名检测** | `server/server.js:248` | `X-Forwarded-Host` | 用于判断域名映射 |
| **客户端 IP 获取** | `server/uptime-kuma-server.js:384` | `X-Forwarded-For`, `X-Real-IP` | 日志记录、速率限制 |
| **Socket.io Origin 检查** | `server/uptime-kuma-server.js:177` | `X-Forwarded-For` | WebSocket 安全验证 |
| **状态页 URL 构建** | `server/model/status_page.js:119` | `X-Forwarded-Proto`, `X-Forwarded-Host` | RSS feed URL 生成 |
| **API 入口检测** | `server/routers/api-router.js:33` | `X-Forwarded-Host` | 入口页 API |

### 3.2 入口页面域名检测

文件位置: `server/server.js:245-268`

```javascript
app.get("/", async (request, response) => {
    let hostname = request.hostname;
    
    // 关键：信任代理时使用 X-Forwarded-Host
    if (await setting("trustProxy")) {
        const proxy = request.headers["x-forwarded-host"];
        if (proxy) {
            hostname = proxy;
        }
    }

    // 检查是否匹配状态页自定义域名
    if (hostname in StatusPage.domainMappingList) {
        // 显示对应状态页
        let slug = StatusPage.domainMappingList[hostname];
        await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
    } else if (uptimeKumaEntryPage && uptimeKumaEntryPage.startsWith("statusPage-")) {
        // 入口页配置为状态页
        response.redirect("/status/" + uptimeKumaEntryPage.replace("statusPage-", ""));
    } else {
        // 默认重定向到管理后台
        response.redirect("/dashboard");
    }
});
```

**安全影响**:
- 如果 `trustProxy` 启用，攻击者可能通过伪造 `X-Forwarded-Host` 来绕过域名映射检查
- 但实际上这通常只影响入口页重定向逻辑，不会直接导致安全漏洞

### 3.3 客户端 IP 获取

文件位置: `server/uptime-kuma-server.js:379-395`

```javascript
async getClientIPwithProxy(clientIP, headers) {
    if (clientIP === undefined) {
        clientIP = "";
    }

    if (await Settings.get("trustProxy")) {
        const forwardedFor = headers["x-forwarded-for"];

        return (
            // 优先使用 X-Forwarded-For 的第一个 IP
            (typeof forwardedFor === "string" ? forwardedFor.split(",")[0].trim() : null) ||
            // 其次使用 X-Real-IP
            headers["x-real-ip"] ||
            // 最后使用直接连接的 IP
            clientIP.replace(/^::ffff:/, "")
        );
    } else {
        // 不信任代理，直接使用连接 IP
        return clientIP.replace(/^::ffff:/, "");
    }
}
```

**安全影响**:
- 用于日志记录、速率限制、审计追踪
- 如果 `trustProxy` 启用但代理配置不当，攻击者可伪造 IP
- 影响登录速率限制 (`loginRateLimiter`) 的有效性

### 3.4 Socket.io Origin 检查（关键安全机制）

文件位置: `server/uptime-kuma-server.js:144-196`

```javascript
this.io = new Server(this.httpServer, {
    cors,
    allowRequest: async (req, callback) => {
        let transport = req._query?.transport || "polling";
        const clientIP = await this.getClientIPwithProxy(req.connection.remoteAddress, req.headers);

        // Polling 连接由 CORS 保护
        if (transport === "polling") {
            callback(null, true);
        } else if (transport === "websocket") {
            // WebSocket 需要额外的 Origin 检查
            const bypass = process.env.UPTIME_KUMA_WS_ORIGIN_CHECK === "bypass";
            
            if (bypass) {
                // 绕过检查（危险！）
                log.info("auth", "WebSocket origin check is bypassed");
                callback(null, true);
            } else if (!req.headers.origin) {
                // 无 Origin 头（非浏览器请求）
                log.info("auth", "WebSocket with no origin is allowed");
                callback(null, true);
            } else {
                let host = req.headers.host;
                let origin = req.headers.origin;

                try {
                    let originURL = new URL(origin);
                    let xForwardedFor;
                    
                    // 关键：信任代理时使用 X-Forwarded-For
                    if (await Settings.get("trustProxy")) {
                        xForwardedFor = req.headers["x-forwarded-for"];
                    }

                    // 检查 Origin 是否匹配 Host 或 X-Forwarded-For
                    if (host !== originURL.host && xForwardedFor !== originURL.host) {
                        callback(null, false);
                        log.error("auth", `Origin (${origin}) does not match host (${host}), IP: ${clientIP}`);
                    } else {
                        callback(null, true);
                    }
                } catch (e) {
                    // 无效的 Origin URL
                    callback(null, false);
                    log.error("auth", `Invalid origin url (${origin}), IP: ${clientIP}`);
                }
            }
        }
    },
});
```

**关键安全逻辑：

1. **WebSocket vs Polling**:
   - Polling 传输由 CORS 保护
   - WebSocket 传输需要额外的 Origin 检查

2. **Origin 检查条件**:
   - `host === originURL.host` → 允许
   - `xForwardedFor === originURL.host` → 允许（仅当 trustProxy 启用）
   - 否则 → 拒绝

3. **绕过机制**:
   - 环境变量 `UPTIME_KUMA_WS_ORIGIN_CHECK=bypass` 可完全绕过检查
   - 无 Origin 头的请求被允许（非浏览器环境）

**安全风险**:
- 如果 `trustProxy` 启用且代理未正确配置，攻击者可能通过伪造 `X-Forwarded-For` 来绕过 Origin 检查
- `UPTIME_KUMA_WS_ORIGIN_CHECK=bypass` 会完全禁用 WebSocket 安全检查

### 3.5 状态页 URL 构建

文件位置: `server/model/status_page.js:117-141`

```javascript
static async buildRSSUrl(slug, request) {
    if (request) {
        const trustProxy = await setting("trustProxy");

        // 确定协议（检查 X-Forwarded-Proto）
        let proto = request.protocol;
        if (trustProxy && request.headers["x-forwarded-proto"]) {
            proto = request.headers["x-forwarded-proto"].split(",")[0].trim();
        }

        // 确定主机名（检查 X-Forwarded-Host）
        let host = request.get("host");
        if (trustProxy && request.headers["x-forwarded-host"]) {
            host = request.headers["x-forwarded-host"];
        }

        return `${proto}://${host}/status/${slug}`;
    }

    // Fallback to config values
    const proto = config.isSSL ? "https" : "http";
    const host = config.hostname || "localhost";
    const port = config.port;
    return `${proto}://${host}:${port}/status/${slug}`;
}
```

**用途**:
- 生成 RSS feed 中的链接
- 影响状态页中生成的绝对 URL

---

## 四、Cloudflared/反向代理场景分析

### 4.1 Cloudflared Tunnel 工作原理

Cloudflared Tunnel 通过以下方式工作：
1. Cloudflared 客户端在 Uptime Kuma 服务器上运行
2. 建立到 Cloudflare 边缘网络的出站连接
3. 外部用户通过 Cloudflare 边缘访问
4. 请求通过 Tunnel 转发到 Uptime Kuma

**关键 Header**:
- Cloudflare 会添加 `X-Forwarded-*` 系列 Header
- 包括 `CF-Connecting-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Host`

### 4.2 常见反向代理配置示例

#### Nginx 反向代理配置

```nginx
server {
    listen 443 ssl http2;
    server_name status.example.com;
    
    # SSL 配置
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    location / {
        proxy_pass http://localhost:3001;
        
        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # 代理 Header（关键配置）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
        
        # 超时设置
        proxy_read_timeout 86400;
    }
}
```

#### Cloudflared 配置 (config.yml)

```yaml
tunnel: your-tunnel-uuid
credentials-file: /root/.cloudflared/your-tunnel-uuid.json

ingress:
  # 状态页域名
  - hostname: status.example.com
    service: http://localhost:3001
  
  # 管理后台域名（可选，建议使用不同域名）
  - hostname: admin.example.com
    service: http://localhost:3001
  
  - service: http_status:404
```

### 4.3 安全配置建议

#### 推荐配置：状态页与管理后台分离

**方案 A：不同域名 + Nginx 路径限制**

```nginx
# 状态页域名（公开访问）
server {
    listen 443 ssl;
    server_name status.example.com;
    
    location / {
        # 只允许访问状态页相关路径
        if ($request_uri !~* "^/(status|api/status-page|api/badge|robots\.txt|\.well-known)") {
            return 403;
        }
        
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
    }
}

# 管理后台域名（内部/限制访问）
server {
    listen 443 ssl;
    server_name admin.example.com;
    
    # IP 白名单或 VPN 限制
    allow 192.168.1.0/24;
    deny all;
    
    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
    }
}
```

**方案 B：Cloudflared + 访问策略**

在 Cloudflare Zero Trust 控制台配置：
1. 创建两个应用：
   - `status.example.com`：公开访问，无需认证
   - `admin.example.com`：需要 Cloudflare Access 认证（如 OAuth、2FA）

2. Cloudflared 配置：
```yaml
ingress:
  - hostname: status.example.com
    service: http://localhost:3001
    originRequest:
      httpHostHeader: status.example.com
  - hostname: admin.example.com
    service: http://localhost:3001
    originRequest:
      httpHostHeader: admin.example.com
```

### 4.4 风险场景分析

#### 场景 1：状态页与管理后台使用同一域名

**风险等级**: 中

**问题**:
- 攻击者可访问 `/dashboard`, `/settings` 等管理路径
- 虽然需要认证，但暴露了攻击面
- 管理后台的 Socket.io 连接也暴露

**缓解措施**:
- 确保启用 `trustProxy` 配置正确
- 不要设置 `UPTIME_KUMA_WS_ORIGIN_CHECK=bypass`
- 使用强密码和 2FA

#### 场景 2：`trustProxy` 启用但代理配置不当

**风险等级**: 高

**问题**:
- Uptime Kuma 信任 `X-Forwarded-*` Header
- 但代理未正确设置或清理这些 Header
- 攻击者可伪造 Header

**示例攻击**:
```
# 攻击者直接连接 Uptime Kuma（不通过代理）
# 伪造 X-Forwarded-For 来绕过 Origin 检查
curl -H "X-Forwarded-For: localhost" \
     -H "Origin: http://malicious-site.com" \
     ws://uptime-kuma:3001/socket.io/
```

**缓解措施**:
- 只在确实需要时启用 `trustProxy`
- 确保代理正确设置正确的 Header 并清理用户传入的 Header
- Nginx 配置示例：
  ```nginx
  # 清理用户传入的代理 Header
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  # 不要使用 $http_x_forwarded_for
  ```

#### 场景 3：`UPTIME_KUMA_WS_ORIGIN_CHECK=bypass`

**风险等级**: 严重

**问题**:
- 完全禁用 WebSocket Origin 检查
- 允许任意网站建立 WebSocket 连接
- 可能导致 CSRF 类攻击

**攻击场景**:
1. 攻击者创建恶意网站
2. 用户已登录 Uptime Kuma
3. 恶意网站通过 JavaScript 建立 WebSocket 连接
4. 执行管理操作

**缓解措施**:
- **永远不要**在生产环境设置此环境变量
- 仅在开发环境调试时临时使用

#### 场景 4：状态页自定义域名映射

**风险等级**: 低

**机制**:
- `StatusPage.domainMappingList` 存储域名到 slug 的映射
- 入口页根据 `hostname` 检查映射
- `trustProxy` 启用时使用 `X-Forwarded-Host`

**潜在问题**:
- 理论上攻击者可伪造 `X-Forwarded-Host` 访问其他状态页
- 但状态页本身是公开的，这不是严重安全问题

---

## 五、安全建议和最佳实践

### 5.1 反向代理配置检查清单

- [ ] **启用 `trustProxy` 当且仅当确实位于反向代理之后
- [ ] 确保代理正确设置以下 Header：
  - `X-Forwarded-For`: 客户端真实 IP
  - `X-Forwarded-Proto`: 协议（http/https）
  - `X-Forwarded-Host`: 原始请求主机名
- [ ] 代理清理/覆盖用户传入的 `X-Forwarded-*` Header
- [ ] WebSocket 支持正确配置（Upgrade 和 Connection 头）
- [ ] 超时设置足够长（WebSocket 长连接）

### 5.2 域名分离最佳实践

1. **使用不同域名**:
   - 状态页: `status.yourdomain.com`
   - 管理后台: `admin.yourdomain.com`（内部或 VPN 访问）

2. **路径级访问控制**:
   - 状态页域名只允许访问:
     - `/status/*`
     - `/api/status-page/*`
     - `/api/badge/*`
     - `/robots.txt`
     - `/.well-known/*`
   - 管理后台域名限制 IP 或使用额外认证层

3. **Cloudflare Access 策略:
   - 管理后台应用启用 Cloudflare Access
   - 要求 OAuth/2FA 认证

### 5.3 认证增强建议

1. **强制 2FA:
   - 为所有管理员账户启用双因素认证
   - Uptime Kuma 支持 TOTP 标准

2. **强密码策略**:
   - 使用复杂密码
   - 定期更换

3. **API Key 管理**:
   - 仅在需要时创建 API Key
   - 设置合理的过期时间
   - 定期轮换

4. **监控和审计**:
   - 监控登录日志
   - 关注失败的登录尝试
   - 设置登录速率限制已启用（默认启用）

### 5.4 环境变量安全配置

```bash
# 安全配置（推荐）
UPTIME_KUMA_WS_ORIGIN_CHECK=cors-like  # 默认值，启用 Origin 检查

# 危险配置（避免）
UPTIME_KUMA_WS_ORIGIN_CHECK=bypass  # 禁用 Origin 检查，仅开发用

# 其他安全相关
UPTIME_KUMA_DISABLE_FRAME_SAMEORIGIN=false  # 默认，启用 X-Frame-Options
```

---

## 六、代码引用索引

### 6.1 关键文件列表

| 文件路径 | 功能描述 |
|---------|---------|
| `server/routers/status-page-router.js` | 状态页面路由（公开） |
| `server/routers/api-router.js` | API 路由（混合） |
| `server/server.js` | 主服务器、入口路由、Socket.io 认证 |
| `server/auth.js` | 认证中间件实现 |
| `server/uptime-kuma-server.js` | Socket.io 服务器、Origin 检查 |
| `server/model/status_page.js` | 状态页模型、域名映射 |
| `src/router.js` | 前端路由配置 |
| `src/components/settings/ReverseProxy.vue` | 反向代理设置 UI |

### 6.2 关键代码位置

#### 路由分离
- 状态页公开路由: `server/routers/status-page-router.js:1-262`
- 入口页路由逻辑: `server/server.js:245-268`
- 前端路由配置: `src/router.js:35-192`

#### 认证机制
- Socket.io 登录: `server/server.js:383-510`
- JWT 验证: `server/server.js:383-430`
- Basic Auth 中间件: `server/auth.js:125-176`
- API Key 验证: `server/auth.js:41-63`

#### 代理 Header 处理
- `trustProxy` 设置使用:
  - 入口页域名: `server/server.js:248`
  - 客户端 IP: `server/uptime-kuma-server.js:384`
  - Origin 检查: `server/uptime-kuma-server.js:177`
  - URL 构建: `server/model/status_page.js:119`
  - API 入口: `server/routers/api-router.js:33`

#### WebSocket 安全
- Origin 检查逻辑: `server/uptime-kuma-server.js:144-196`
- 环境变量 bypass: `server/uptime-kuma-server.js:163-166`

---

## 七、总结

### 7.1 核心结论

1. **路由分离机制**:
   - Uptime Kuma 采用清晰的路由分离设计
   - 状态页相关路径完全公开，无认证
   - 管理后台通过 Socket.io JWT 认证保护

2. **代理 Header 影响**:
   - `trustProxy` 设置是关键开关
   - 启用后信任 `X-Forwarded-*` 系列 Header
   - 影响域名检测、IP 获取、Origin 检查、URL 构建

3. **WebSocket 安全**:
   - WebSocket 有额外的 Origin 检查
   - `trustProxy` 启用时检查 `X-Forwarded-For`
   - `UPTIME_KUMA_WS_ORIGIN_CHECK=bypass` 可绕过（危险）

4. **最佳实践**:
   - 状态页与管理后台使用不同域名
   - 合理配置 `trustProxy`
   - 不要绕过 WebSocket Origin 检查
   - 启用 2FA 和强密码

### 7.2 安全配置速查表

| 配置项 | 推荐值 | 风险说明 |
|-------|--------|---------|
| `trustProxy` | 仅在代理后启用 | 错误配置可导致 Header 伪造 |
| `UPTIME_KUMA_WS_ORIGIN_CHECK` | `cors-like`（默认） | `bypass` 禁用安全检查 |
| 域名分离 | 推荐使用不同域名 | 同一域名暴露攻击面 |
| 2FA | 强制启用 | 增强账户安全 |
| API Key | 按需创建，定期轮换 | 长期有效 Key 风险高 |

---

**报告完成日期**: 2026-05-05
**分析版本**: Uptime Kuma v1.x（基于代码库分析）
