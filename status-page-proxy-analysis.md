# Uptime Kuma Status Page 反向代理与公开访问分析报告

## 一、概述

Uptime Kuma 的 Status Page（状态页）是一个用于向公众展示服务监控状态的功能模块。本报告分析了状态页如何通过 Cloudflared（Cloudflare Tunnel）或传统反向代理（如 Nginx、Apache、Traefik）对外公开访问，包括数据渲染机制、公开访问路径、多服务协作关系等核心内容。

---

## 二、数据渲染机制

### 2.1 服务器端预渲染 (SSR)

状态页采用混合渲染策略，首次加载时使用服务器端预渲染，后续通过 API 轮询更新数据。

#### 核心渲染流程

**路由入口** (`server/routers/status-page-router.js:16-70`)：

```javascript
// 主要状态页路由 - 带缓存
router.get("/status/:slug", cache("5 minutes"), async (request, response) => {
    let slug = request.params.slug;
    slug = slug.toLowerCase();
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});

// 默认状态页路由
router.get("/status", cache("5 minutes"), async (request, response) => {
    let slug = "default";
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});

router.get("/status-page", cache("5 minutes"), async (request, response) => {
    let slug = "default";
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});
```

**渲染处理** (`server/model/status_page.js:57-71, 149-201`)：

```javascript
static async handleStatusPageResponse(response, indexHTML, slug) {
    let statusPage = await R.findOne("status_page", " slug = ? ", [slug]);
    
    if (statusPage) {
        response.send(await StatusPage.renderHTML(indexHTML, statusPage));
    } else {
        response.status(404).send(UptimeKumaServer.getInstance().indexHTML);
    }
}

static async renderHTML(indexHTML, statusPage) {
    const $ = cheerio.load(indexHTML);
    
    // 1. 设置页面标题和描述
    $("title").text(statusPage.title);
    $("meta[name=description]").attr("content", description155);
    
    // 2. 注入分析脚本（如果配置了）
    if (analytics.isValidAnalyticsConfig(statusPage)) {
        let escapedAnalyticsScript = analytics.getAnalyticsScript(statusPage);
        head.append($(escapedAnalyticsScript));
    }
    
    // 3. 预加载数据 - 关键步骤
    const escapedJSONObject = jsesc(await StatusPage.getStatusPageData(statusPage), {
        isScriptContext: true,
    });
    
    const script = $(`
        <script id="preload-data" data-json="{}">
            window.preloadData = ${escapedJSONObject};
        </script>
    `);
    head.append(script);
    
    // 4. 设置 manifest.json
    $("link[rel=manifest]").attr("href", `/api/status-page/${statusPage.slug}/manifest.json`);
    
    return $.root().html();
}
```

### 2.2 数据预加载内容

`getStatusPageData` 方法 (`server/model/status_page.js:309-340`) 收集以下数据：

```javascript
static async getStatusPageData(statusPage) {
    const config = await statusPage.toPublicJSON();
    
    // 1. 活动事件 (Active Incidents)
    let incidents = await R.find(
        "incident",
        " pin = 1 AND active = 1 AND status_page_id = ? ORDER BY created_date DESC",
        [statusPage.id]
    );
    incidents = incidents.map((i) => i.toPublicJSON());
    
    // 2. 维护计划列表
    let maintenanceList = await StatusPage.getMaintenanceList(statusPage.id);
    
    // 3. 公开监控组列表
    const publicGroupList = [];
    const showTags = !!statusPage.show_tags;
    const list = await R.find("group", " public = 1 AND status_page_id = ? ORDER BY weight ", [statusPage.id]);
    
    for (let groupBean of list) {
        let monitorGroup = await groupBean.toPublicJSON(showTags, config?.showCertificateExpiry);
        publicGroupList.push(monitorGroup);
    }
    
    return {
        config,
        incidents,
        publicGroupList,
        maintenanceList,
    };
}
```

### 2.3 前端数据获取与刷新

**前端组件** (`src/pages/StatusPage.vue`)：

```javascript
// 数据获取方法 - 优先使用预加载数据
getData: function () {
    if (window.preloadData) {
        return new Promise((resolve) =>
            resolve({
                data: window.preloadData,
            })
        );
    } else {
        return axios.get("/api/status-page/" + this.slug);
    }
}

// 心跳数据定时刷新
updateHeartbeatList() {
    if (!this.editMode) {
        axios.get("/api/status-page/heartbeat/" + this.slug).then((res) => {
            const { heartbeatList, uptimeList } = res.data;
            this.$root.heartbeatList = heartbeatList;
            this.$root.uptimeList = uptimeList;
            // ... 更新 favicon 徽章
        });
    }
}

// 自动刷新定时器配置 (mounted 中)
feedInterval = setInterval(
    () => {
        this.updateHeartbeatList();
    },
    Math.max(5, this.config.autoRefreshInterval) * 1000
);
```

### 2.4 API 端点与缓存策略

| API 端点 | 缓存时间 | 用途 | 文件位置 |
|---------|---------|------|---------|
| `GET /api/status-page/:slug` | 5 分钟 | 获取状态页配置、事件、监控组数据 | `status-page-router.js:39-60` |
| `GET /api/status-page/heartbeat/:slug` | 1 分钟 | 获取心跳数据和可用性统计 | `status-page-router.js:64-110` |
| `GET /api/status-page/:slug/incident-history` | 5 分钟 | 获取事件历史（游标分页） | `status-page-router.js:145-167` |
| `GET /api/status-page/:slug/badge` | 5 分钟 | 生成 SVG 状态徽章 | `status-page-router.js:170-262` |
| `GET /api/status-page/:slug/manifest.json` | 1440 分钟 (24h) | PWA 应用清单 | `status-page-router.js:113-143` |

**心跳数据 API 实现** (`status-page-router.js:64-110`)：

```javascript
router.get("/api/status-page/heartbeat/:slug", cache("1 minutes"), async (request, response) => {
    let heartbeatList = {};
    let uptimeList = {};
    
    let slug = request.params.slug;
    slug = slug.toLowerCase();
    let statusPageID = await StatusPage.slugToID(slug);
    
    // 获取该状态页下所有公开的监控 ID
    let monitorIDList = await R.getCol(
        `
        SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
        WHERE monitor_group.group_id = \`group\`.id
        AND public = 1
        AND \`group\`.status_page_id = ?
        `,
        [statusPageID]
    );
    
    // 为每个监控获取最近 100 条心跳数据
    for (let monitorID of monitorIDList) {
        let list = await R.getAll(
            `
                SELECT * FROM heartbeat
                WHERE monitor_id = ?
                ORDER BY time DESC
                LIMIT 100
            `,
            [monitorID]
        );
        
        list = R.convertToBeans("heartbeat", list);
        heartbeatList[monitorID] = list.reverse().map((row) => row.toPublicJSON());
        
        // 计算 24 小时可用性
        const uptimeCalculator = await UptimeCalculator.getUptimeCalculator(monitorID);
        uptimeList[`${monitorID}_24`] = uptimeCalculator.get24Hour().uptime;
    }
    
    response.json({ heartbeatList, uptimeList });
});
```

---

## 三、公开访问路径

Uptime Kuma 提供了三种主要的公开访问方式：

1. **URL 路径路由** (Path-based Routing)
2. **域名映射** (Domain/CNAME Mapping)
3. **Cloudflare Tunnel (cloudflared)**

### 3.1 URL 路径路由

这是最基础的访问方式，通过 URL 路径中的 slug 来区分不同的状态页。

**前端路由定义** (`src/router.js:177-187`)：

```javascript
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
}
```

**后端路由处理** (`server/routers/status-page-router.js:16-36`)：

```javascript
// 带 slug 的状态页
router.get("/status/:slug", cache("5 minutes"), async (request, response) => {
    let slug = request.params.slug;
    slug = slug.toLowerCase();
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});

// RSS 订阅
router.get("/status/:slug/rss", cache("5 minutes"), async (request, response) => {
    let slug = request.params.slug;
    slug = slug.toLowerCase();
    await StatusPage.handleStatusPageRSSResponse(response, slug, request);
});

// 默认状态页 (slug = "default")
router.get("/status", cache("5 minutes"), async (request, response) => {
    let slug = "default";
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});

router.get("/status-page", cache("5 minutes"), async (request, response) => {
    let slug = "default";
    await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
});
```

### 3.2 域名映射 (CNAME)

Uptime Kuma 支持将自定义域名直接映射到特定的状态页，无需通过 URL 路径。

#### 数据存储结构

域名映射存储在 `status_page_cname` 表中，与 `status_page` 表关联。

**加载映射列表** (`server/model/status_page.js:347-353`)：

```javascript
static async loadDomainMappingList() {
    StatusPage.domainMappingList = await R.getAssoc(`
        SELECT domain, slug
        FROM status_page, status_page_cname
        WHERE status_page.id = status_page_cname.status_page_id
    `);
}
```

**更新域名列表** (`server/model/status_page.js:379-411`)：

```javascript
async updateDomainNameList(domainNameList) {
    if (!Array.isArray(domainNameList)) {
        throw new Error("Invalid array");
    }
    
    let trx = await R.begin();
    
    // 先删除旧的映射
    await trx.exec("DELETE FROM status_page_cname WHERE status_page_id = ?", [this.id]);
    
    try {
        for (let domain of domainNameList) {
            // 验证域名格式
            if (typeof domain !== "string") {
                throw new Error("Invalid domain");
            }
            if (domain.trim() === "") {
                continue;
            }
            
            // 确保域名不被其他状态页使用
            await trx.exec("DELETE FROM status_page_cname WHERE domain = ?", [domain]);
            
            // 创建新映射
            let mapping = trx.dispense("status_page_cname");
            mapping.status_page_id = this.id;
            mapping.domain = domain;
            await trx.store(mapping);
        }
        await trx.commit();
    } catch (error) {
        await trx.rollback();
        throw error;
    }
}
```

#### 路由解析逻辑

**入口页面路由** (`server/server.js:246-268`)：

```javascript
app.get("/", async (request, response) => {
    let hostname = request.hostname;
    
    // 如果启用了 trustProxy，使用 X-Forwarded-Host
    if (await setting("trustProxy")) {
        const proxy = request.headers["x-forwarded-host"];
        if (proxy) {
            hostname = proxy;
        }
    }
    
    log.debug("entry", `Request Domain: ${hostname}`);
    
    const uptimeKumaEntryPage = server.entryPage;
    
    // 检查是否为状态页域名
    if (hostname in StatusPage.domainMappingList) {
        log.debug("entry", "This is a status page domain");
        
        let slug = StatusPage.domainMappingList[hostname];
        await StatusPage.handleStatusPageResponse(response, server.indexHTML, slug);
    } 
    // 检查是否配置了默认入口页为状态页
    else if (uptimeKumaEntryPage && uptimeKumaEntryPage.startsWith("statusPage-")) {
        response.redirect("/status/" + uptimeKumaEntryPage.replace("statusPage-", ""));
    } 
    // 否则重定向到仪表板
    else {
        response.redirect("/dashboard");
    }
});
```

**API 入口检测** (`server/routers/api-router.js:28-45`)：

```javascript
router.get("/api/entry-page", async (request, response) => {
    let result = {};
    let hostname = request.hostname;
    
    if ((await Settings.get("trustProxy")) && request.headers["x-forwarded-host"]) {
        hostname = request.headers["x-forwarded-host"];
    }
    
    if (hostname in StatusPage.domainMappingList) {
        result.type = "statusPageMatchedDomain";
        result.statusPageSlug = StatusPage.domainMappingList[hostname];
    } else {
        result.type = "entryPage";
        result.entryPage = server.entryPage;
    }
    response.json(result);
});
```

#### 加载时机

域名映射列表在以下时机被加载/刷新：

1. **服务器启动时** (`server/server.js:234`)：
   ```javascript
   await StatusPage.loadDomainMappingList();
   ```

2. **保存状态页配置时** (`server/socket-handlers/status-page-socket-handler.js:351`)：
   ```javascript
   await statusPage.updateDomainNameList(config.domainNameList);
   await StatusPage.loadDomainMappingList();
   ```

### 3.3 Cloudflare Tunnel (cloudflared)

Uptime Kuma 内置了对 Cloudflare Tunnel 的支持，通过 `node-cloudflared-tunnel` 库实现。

#### 端到端链路详解

##### 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              Cloudflare 全球边缘网络                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                        │
│  │   用户浏览器   │────▶│  Cloudflare  │────▶│  Tunnel 边缘  │                        │
│  │              │     │   CDN/Edge   │     │    节点       │                        │
│  └──────────────┘     └──────────────┘     └───────┬──────┘                        │
└───────────────────────────────────────────────────────┼──────────────────────────────┘
                                                        │
                                                        │  出站连接 (从 Uptime Kuma 主动发起)
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           用户本地网络 / 私有网络                                      │
│                                                                                        │
│   ┌──────────────────────────────────────────────────────────────────────────────┐  │
│   │                        cloudflared 进程 (独立子进程)                           │  │
│   │                                                                                  │  │
│   │  ┌────────────────────────────────────────────────────────────────────────┐  │  │
│   │  │  功能:                                                                  │  │  │
│   │  │  1. 与 Cloudflare 边缘节点建立持久的 WebSocket/QUIC 出站连接           │  │  │
│   │  │  2. 接收来自 Cloudflare 边缘的 HTTP/WebSocket 请求                     │  │  │
│   │  │  3. 反向代理到本地 Uptime Kuma 服务 (localhost:3001)                  │  │  │
│   │  │  4. 将响应返回给 Cloudflare 边缘                                        │  │  │
│   │  └────────────────────────────────────────────────────────────────────────┘  │  │
│   │                                      │                                         │  │
│   │                                      ▼                                         │  │
│   │  ┌────────────────────────────────────────────────────────────────────────┐  │  │
│   │  │                    Uptime Kuma Node.js 服务                              │  │  │
│   │  │                    监听: localhost:3001 (默认)                           │  │  │
│   │  │                                                                          │  │  │
│   │  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │  │  │
│   │  │  │  Express HTTP   │  │  Socket.IO      │  │  数据库操作      │         │  │  │
│   │  │  │  Server         │  │  Server         │  │                 │         │  │  │
│   │  │  └─────────────────┘  └─────────────────┘  └─────────────────┘         │  │  │
│   │  └────────────────────────────────────────────────────────────────────────┘  │  │
│   └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                        │
│  关键特性:                                                                             │
│  - 无需开放任何入站端口                                                                │
│  - 所有连接都是从 cloudflared 主动向 Cloudflare 发起的出站连接                       │
│  - 本地 Uptime Kuma 只监听 localhost，不暴露到公网                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

##### 完整请求链路（以状态页访问为例）

```
步骤 1: 用户发起请求
┌─────────────┐
│  用户浏览器  │──────▶ GET https://status.example.com/status/my-page
└─────────────┘
         │
         ▼
步骤 2: Cloudflare DNS 解析
- CNAME 记录指向: <tunnel-id>.cfargotunnel.com
- Cloudflare 边缘识别这是一个 Tunnel 域名

         │
         ▼
步骤 3: Cloudflare 边缘节点处理
┌─────────────────────┐
│  Cloudflare 边缘节点  │
│                     │
│  1. 接收用户请求      │
│  2. 查找对应的 Tunnel │
│  3. 通过已建立的出站  │
│     连接发送请求到    │
│     cloudflared      │
└──────────┬──────────┘
           │
           ▼  已建立的出站 WebSocket/QUIC 连接
           │  (从 cloudflared 主动发起，保持持久连接)
           │
步骤 4: cloudflared 反向代理
┌───────────────────────────────────┐
│        cloudflared 进程            │
│                                   │
│  1. 从 Cloudflare 边缘接收请求      │
│  2. 解析 HTTP 请求                  │
│  3. 转发到本地 Uptime Kuma:         │
│     localhost:3001/status/my-page  │
│  4. 接收响应并返回给 Cloudflare     │
└──────────────┬────────────────────┘
               │
               ▼  HTTP 请求到 localhost:3001
               │
步骤 5: Uptime Kuma 处理
┌─────────────────────────────────────────────────────────┐
│              Uptime Kuma Node.js Server                 │
│                                                          │
│  1. Express 路由匹配:                                    │
│     GET /status/:slug                                    │
│                                                          │
│  2. server.js 入口路由检测 (如果是根路径 /):             │
│     - 获取 hostname (通过 X-Forwarded-Host)            │
│     - 检查是否在 domainMappingList 中                    │
│                                                          │
│  3. status-page-router.js 处理:                         │
│     - 应用 5 分钟缓存                                    │
│     - 调用 StatusPage.handleStatusPageResponse()         │
│                                                          │
│  4. SSR 渲染:                                            │
│     - 从数据库获取 status_page 记录                      │
│     - 调用 StatusPage.getStatusPageData()               │
│     - 收集公开监控组、事件、维护计划                      │
│     - 注入 window.preloadData                            │
│     - 返回完整的 HTML                                    │
└─────────────────────────────────────────────────────────┘
               │
               ▼  HTTP 响应
               │
步骤 6: 响应返回
cloudflared ──▶ Cloudflare 边缘 ──▶ 用户浏览器
```

##### Socket.IO 连接通过 Tunnel 的处理

状态页在**公开访问模式**下**不使用** Socket.IO，只使用 HTTP API 轮询。

**关键代码** (`src/mixins/socket.js:19-23`)：

```javascript
const noSocketIOPages = [
    /^\/status-page$/, //  /status-page
    /^\/status/, // /status**
    /^\/$/, //  /
];
```

**初始化逻辑** (`src/mixins/socket.js:85-98`)：

```javascript
initSocketIO(bypass = false) {
    // 已经初始化过，不需要重新连接
    if (this.socket.initedSocketIO) {
        return;
    }

    // 状态页不需要连接 Socket.IO
    if (!bypass && location.pathname) {
        for (let page of noSocketIOPages) {
            if (location.pathname.match(page)) {
                return;  // 直接返回，不建立 Socket.IO 连接
            }
        }
    }
    // ... 建立 Socket.IO 连接
}
```

**这意味着**：
- **公开状态页** (`/status/*`)：只使用 HTTP API + 轮询，不建立 WebSocket 连接
- **后台管理** (`/dashboard/*`)：使用 Socket.IO 进行实时通信
- **编辑模式** (`/status/:slug?edit`)：需要登录并建立 Socket.IO 连接

#### 核心实现

**Socket 处理器** (`server/socket-handlers/cloudflared-socket-handler.js`)：

```javascript
const { CloudflaredTunnel } = require("node-cloudflared-tunnel");
const cloudflared = new CloudflaredTunnel();

const prefix = "cloudflared_";

// 状态变化回调
cloudflared.change = (running, message) => {
    io.to("cloudflared").emit(prefix + "running", running);
    io.to("cloudflared").emit(prefix + "message", message);
};

cloudflared.error = (errorMessage) => {
    io.to("cloudflared").emit(prefix + "errorMessage", errorMessage);
};

module.exports.cloudflaredSocketHandler = (socket) => {
    // 加入 cloudflared 房间
    socket.on(prefix + "join", async () => {
        try {
            checkLogin(socket);  // 需要登录才能管理 cloudflared
            socket.join("cloudflared");
            io.to(socket.userID).emit(prefix + "installed", cloudflared.checkInstalled());
            io.to(socket.userID).emit(prefix + "running", cloudflared.running);
            io.to(socket.userID).emit(prefix + "token", await setting("cloudflaredTunnelToken"));
        } catch (error) {
            log.error("cloudflared", "Error in join handler: " + error.message);
        }
    });
    
    // 启动隧道
    socket.on(prefix + "start", async (token) => {
        try {
            checkLogin(socket);
            if (token && typeof token === "string") {
                await setSetting("cloudflaredTunnelToken", token);
                cloudflared.token = token;
            } else {
                cloudflared.token = null;
            }
            cloudflared.start();
        } catch (error) {
            log.error("cloudflared", "Error in start handler: " + error.message);
        }
    });
    
    // 停止隧道
    socket.on(prefix + "stop", async (currentPassword, callback) => {
        try {
            checkLogin(socket);
            const disabledAuth = await setting("disableAuth");
            if (!disabledAuth) {
                await doubleCheckPassword(socket, currentPassword);  // 需要密码确认
            }
            cloudflared.stop();
        } catch (error) {
            callback({ ok: false, msg: error.message });
        }
    });
};
```

**自动启动逻辑** (`cloudflared-socket-handler.js:103-117`)：

```javascript
module.exports.autoStart = async (token) => {
    if (!token) {
        token = await setting("cloudflaredTunnelToken");
    } else {
        // 通过命令行参数或环境变量覆盖 token
        await setSetting("cloudflaredTunnelToken", token);
        log.info("cloudflare", "Use cloudflared token from args or env var");
    }
    
    if (token) {
        log.info("cloudflare", "Start cloudflared");
        cloudflared.token = token;
        cloudflared.start();
    }
};
```

#### 前端配置界面

**反向代理设置组件** (`src/components/settings/ReverseProxy.vue`)：

```vue
<template>
    <div>
        <!-- Cloudflare Tunnel 配置 -->
        <h4 class="mt-4">Cloudflare Tunnel</h4>
        
        <div class="my-3">
            <div>
                cloudflared:
                <span v-if="installed === true" class="text-primary">{{ $t("Installed") }}</span>
                <span v-else-if="installed === false" class="text-danger">{{ $t("Not installed") }}</span>
            </div>
            <div>
                {{ $t("Status") }}:
                <span v-if="running" class="text-primary">{{ $t("Running") }}</span>
                <span v-else-if="!running" class="text-danger">{{ $t("Not running") }}</span>
            </div>
        </div>
        
        <!-- Token 输入 -->
        <div v-if="installed" class="mb-2">
            <div class="mb-4">
                <label class="form-label" for="cloudflareTunnelToken">Cloudflare Tunnel {{ $t("Token") }}</label>
                <HiddenInput
                    id="cloudflareTunnelToken"
                    v-model="cloudflareTunnelToken"
                    autocomplete="new-password"
                    :readonly="running"
                />
                <div class="form-text">
                    {{ $t("Don't know how to get the token? Please read the guide:") }}
                    <br />
                    <a href="https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy-with-Cloudflare-Tunnel" target="_blank">
                        https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy-with-Cloudflare-Tunnel
                    </a>
                </div>
            </div>
            
            <div>
                <button v-if="!running" class="btn btn-primary" type="submit" @click="start">
                    {{ $t("Start") }} cloudflared
                </button>
                <button v-if="running" class="btn btn-danger" type="submit" @click="$refs.confirmStop.show()">
                    {{ $t("Stop") }} cloudflared
                </button>
            </div>
        </div>
    </div>
</template>
```

---

## 四、多服务协作关系

### 4.1 Trust Proxy 设置

当 Uptime Kuma 运行在反向代理（如 Nginx、Apache、Traefik）后面时，需要启用 `trustProxy` 设置来正确获取客户端信息。

#### 配置位置

**前端设置** (`src/components/settings/ReverseProxy.vue:111-147`)：

```vue
<div class="my-3">
    <label class="form-label">
        {{ $t("Trust Proxy") }}
    </label>
    <div class="form-check">
        <input
            id="trustProxyYes"
            v-model="settings.trustProxy"
            class="form-check-input"
            type="radio"
            name="trustProxyYes"
            :value="true"
            required
        />
        <label class="form-check-label" for="trustProxyYes">
            {{ $t("Yes") }}
        </label>
    </div>
    <div class="form-check">
        <input
            id="trustProxyNo"
            v-model="settings.trustProxy"
            class="form-check-input"
            type="radio"
            name="flexRadioDefault"
            :value="false"
            required
        />
        <label class="form-check-label" for="trustProxyNo">
            {{ $t("No") }}
        </label>
    </div>
    <div class="form-text">
        {{ $t("trustProxyDescription") }}
    </div>
</div>
```

**多语言描述** (`src/lang/zh-CN.json:599`)：
```json
"trustProxyDescription": "信任 'X-Forwarded-*' 头。如果您的 Uptime Kuma 是通过 Nginx 或 Apache 等反代服务对外提供访问的话，则您应当启用本功能以获取正确的客户端 IP。"
```

#### 使用场景

**1. 域名映射检测** (`server/server.js:246-253`)：
```javascript
let hostname = request.hostname;
if (await setting("trustProxy")) {
    const proxy = request.headers["x-forwarded-host"];
    if (proxy) {
        hostname = proxy;
    }
}
```

**2. RSS URL 构建** (`server/model/status_page.js:117-141`)：
```javascript
static async buildRSSUrl(slug, request) {
    if (request) {
        const trustProxy = await setting("trustProxy");
        
        // 确定协议
        let proto = request.protocol;
        if (trustProxy && request.headers["x-forwarded-proto"]) {
            proto = request.headers["x-forwarded-proto"].split(",")[0].trim();
        }
        
        // 确定主机名
        let host = request.get("host");
        if (trustProxy && request.headers["x-forwarded-host"]) {
            host = request.headers["x-forwarded-host"];
        }
        
        return `${proto}://${host}/status/${slug}`;
    }
    // ...
}
```

**3. WebSocket 源验证** (`server/uptime-kuma-server.js:170-195`)：
```javascript
let xForwardedFor;
if (await Settings.get("trustProxy")) {
    xForwardedFor = req.headers["x-forwarded-for"];
}

if (host !== originURL.host && xForwardedFor !== originURL.host) {
    callback(null, false);
    log.error("auth", `Origin (${origin}) does not match host (${host}), IP: ${clientIP}`);
} else {
    callback(null, true);
}
```

**4. API 入口页检测** (`server/routers/api-router.js:33-35`)：
```javascript
let hostname = request.hostname;
if ((await Settings.get("trustProxy")) && request.headers["x-forwarded-host"]) {
    hostname = request.headers["x-forwarded-host"];
}
```

### 4.2 模块协作架构

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                        外部访问层                              │
                    │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
                    │  │  Nginx/Apache │  │  Cloudflare  │  │  Direct Access   │  │
                    │  │  (反向代理)    │  │    Tunnel    │  │   (直接访问)      │  │
                    │  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
                    └─────────┼─────────────────┼───────────────────┼──────────────┘
                              │                 │                   │
                              ▼                 ▼                   ▼
                    ┌─────────────────────────────────────────────────────────────┐
                    │                      Uptime Kuma Server                       │
                    │  ┌─────────────────────────────────────────────────────────┐ │
                    │  │                    server.js (主入口)                      │ │
                    │  │  - 加载 domainMappingList                                │ │
                    │  │  - 处理入口路由 / 的域名映射检测                          │ │
                    │  │  - 注册 status-page-router                               │ │
                    │  └─────────────────────────────────────────────────────────┘ │
                    │                              │                                │
                    │                              ▼                                │
                    │  ┌─────────────────────────────────────────────────────────┐ │
                    │  │              status-page-router.js (路由层)              │ │
                    │  │  Routes:                                                 │ │
                    │  │  - GET /status/:slug            (状态页 HTML)           │ │
                    │  │  - GET /status/:slug/rss        (RSS 订阅)              │ │
                    │  │  - GET /api/status-page/:slug   (配置数据 API)          │ │
                    │  │  - GET /api/status-page/heartbeat/:slug (心跳数据)      │ │
                    │  └─────────────────────────────────────────────────────────┘ │
                    │                              │                                │
                    │                              ▼                                │
                    │  ┌─────────────────────────────────────────────────────────┐ │
                    │  │               status_page.js (模型层)                     │ │
                    │  │  - StatusPage 数据模型                                   │ │
                    │  │  - domainMappingList (域名 → slug 映射)                 │ │
                    │  │  - renderHTML() SSR 渲染                                 │ │
                    │  │  - getStatusPageData() 数据收集                          │ │
                    │  │  - buildRSSUrl() 处理 X-Forwarded-* 头                 │ │
                    │  └─────────────────────────────────────────────────────────┘ │
                    │                              │                                │
                    │              ┌───────────────┼───────────────┐              │
                    │              ▼               ▼               ▼              │
                    │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  │
                    │  │ Socket 处理器  │  │  数据库层     │  │  缓存层        │  │
                    │  │  - status-page│  │  - status_page│  │  - apicache    │  │
                    │  │    -handler   │  │  - status_page│  │  (5min/1min)  │  │
                    │  │  - cloudflared│  │    _cname     │  │               │  │
                    │  │    -handler   │  │  - heartbeat  │  │               │  │
                    │  └───────────────┘  └───────────────┘  └───────────────┘  │
                    └─────────────────────────────────────────────────────────────┘
```

### 4.3 关键数据流程

#### 流程 1：通过域名访问状态页

```
1. 用户请求: http://status.example.com/
                              │
                              ▼
2. server.js 入口路由处理
   - 获取 hostname (考虑 trustProxy 和 X-Forwarded-Host)
   - 检查 hostname in StatusPage.domainMappingList
                              │
                              ▼
3. 找到匹配的 slug (例如 "my-status-page")
                              │
                              ▼
4. 调用 StatusPage.handleStatusPageResponse()
   - 从数据库获取 status_page 记录
   - 调用 StatusPage.renderHTML() 进行 SSR
                              │
                              ▼
5. renderHTML() 注入预加载数据
   - 调用 StatusPage.getStatusPageData()
   - 收集 config, incidents, publicGroupList, maintenanceList
   - 注入 window.preloadData
                              │
                              ▼
6. 返回渲染后的 HTML
```

#### 流程 2：通过 URL 路径访问状态页

```
1. 用户请求: http://kuma.example.com/status/my-status-page
                              │
                              ▼
2. status-page-router.js 路由匹配
   - 提取 slug: "my-status-page"
   - 应用 5 分钟缓存
                              │
                              ▼
3. 调用 StatusPage.handleStatusPageResponse()
   (后续步骤同上)
```

#### 流程 3：前端数据刷新

```
1. StatusPage.vue 组件 mounted
                              │
                              ▼
2. 调用 getData()
   - 优先使用 window.preloadData (SSR 预加载)
   - 否则请求 /api/status-page/:slug
                              │
                              ▼
3. 设置自动刷新定时器
   - 周期: config.autoRefreshInterval 秒 (最小 5 秒)
                              │
                              ▼
4. 定时调用 updateHeartbeatList()
   - 请求 /api/status-page/heartbeat/:slug (1 分钟缓存)
   - 更新 $root.heartbeatList 和 $root.uptimeList
   - 更新 favicon 徽章显示
```

---

## 五、配置建议与最佳实践

### 5.1 反向代理配置示例

#### Nginx 配置

```nginx
server {
    listen 80;
    server_name status.example.com kuma.example.com;
    
    # 代理 Uptime Kuma
    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_read_timeout 86400;
    }
}
```

#### Uptime Kuma 内部设置

1. **启用 Trust Proxy**：
   - 进入 Settings → Reverse Proxy
   - 将 "Trust Proxy" 设置为 "Yes"

2. **配置域名映射**：
   - 进入 Status Page 编辑界面
   - 在 "Domain Names" 部分添加自定义域名
   - 例如：`status.example.com`

### 5.2 Cloudflare Tunnel 配置

1. **在 Cloudflare Zero Trust 中创建 Tunnel**：
   - 登录 Cloudflare Zero Trust Dashboard
   - 进入 Networks → Tunnels
   - 创建新 Tunnel，获取 Token

2. **在 Uptime Kuma 中配置**：
   - 进入 Settings → Reverse Proxy
   - 在 "Cloudflare Tunnel Token" 中输入 Token
   - 点击 "Start cloudflared"

3. **配置 DNS**：
   - 在 Cloudflare DNS 中添加 CNAME 记录指向 Tunnel

### 5.3 性能优化建议

1. **缓存策略**：
   - 状态页配置缓存 5 分钟，心跳数据缓存 1 分钟
   - 可通过 CDN 进一步缓存静态资源

2. **自动刷新间隔**：
   - 根据实际需求设置 `autoRefreshInterval`
   - 公开状态页建议 30-60 秒，内部监控可更短

3. **数据库优化**：
   - 心跳数据量会随时间增长，定期清理旧数据
   - 考虑使用 PostgreSQL 替代 SQLite 应对高并发

---

## 七、公开接口与后台管理接口的数据裁剪和鉴权边界对照

Uptime Kuma 的 Status Page 系统通过明确的接口边界设计，实现了公开访问与后台管理的安全隔离。以下是按访问入口逐项分析的详细对照。

---

### 7.1 访问入口分类总览

| 访问类型 | 入口路径 | 连接方式 | 鉴权要求 |
|---------|---------|---------|---------|
| **公开状态页** | `/status/:slug`, `/status` | HTTP 仅 | 无 |
| **公开 API** | `/api/status-page/:slug/*` | HTTP 仅 | 无 |
| **后台管理** | `/dashboard/*`, `/settings/*` | HTTP + Socket.IO | 需要登录 |
| **编辑模式** | `/status/:slug?edit` | HTTP + Socket.IO | 需要登录 |

**连接层边界规则** (`src/mixins/socket.js:19-23`)：

```javascript
const noSocketIOPages = [
    /^\/status-page$/, //  /status-page
    /^\/status/,       // /status** (所有状态页路径)
    /^\/$/,            //  / (根路径)
];
```

**关键设计**：公开状态页路径在匹配 `noSocketIOPages` 时，`initSocketIO()` 直接返回，不建立 WebSocket 连接。只有当 `bypass=true`（编辑模式）时才会强制建立连接。

---

### 7.2 公开接口详细分析

#### 7.2.1 状态页 HTML 渲染入口

| 项目 | 详情 |
|-----|------|
| **路由路径** | `GET /status/:slug`, `GET /status`, `GET /status-page` |
| **文件位置** | `server/routers/status-page-router.js:16-36` |
| **缓存策略** | 5 分钟 (`cache("5 minutes")`) |
| **鉴权规则** | 无 - 完全公开 |
| **连接方式** | HTTP 仅，不建立 Socket.IO |

**数据裁剪**：
- 通过 `StatusPage.renderHTML()` 进行 SSR 渲染
- 预加载数据使用 `StatusPage.getStatusPageData()` 收集
- 所有数据通过 `toPublicJSON()` 序列化

**可见数据**：
- ✅ 状态页配置（`slug`, `title`, `description`, `theme`, `autoRefreshInterval` 等）
- ✅ 公开监控组列表（仅包含 `public = 1` 的组）
- ✅ 活动事件（`pin = 1 AND active = 1`）
- ✅ 维护计划列表

**被隐藏数据**：
- ❌ 状态页 `id`
- ❌ 域名映射列表 (`domainNameList`)
- ❌ 非公开的监控组和监控项

**控制代码** (`server/model/status_page.js:323-328`)：

```javascript
const list = await R.find(
    "group", 
    " public = 1 AND status_page_id = ? ORDER BY weight ", 
    [statusPage.id]
);
```

---

#### 7.2.2 状态页配置数据 API

| 项目 | 详情 |
|-----|------|
| **路由路径** | `GET /api/status-page/:slug` |
| **文件位置** | `server/routers/status-page-router.js:39-60` |
| **缓存策略** | 5 分钟 |
| **鉴权规则** | 无 - 完全公开 |

**数据收集逻辑** (`server/model/status_page.js:309-340`)：

```javascript
static async getStatusPageData(statusPage) {
    const config = await statusPage.toPublicJSON();  // 使用公开版本
    
    // 1. 活动事件
    let incidents = await R.find(
        "incident",
        " pin = 1 AND active = 1 AND status_page_id = ? ORDER BY created_date DESC",
        [statusPage.id]
    );
    incidents = incidents.map((i) => i.toPublicJSON());
    
    // 2. 维护计划
    let maintenanceList = await StatusPage.getMaintenanceList(statusPage.id);
    
    // 3. 公开监控组（SQL 过滤 public = 1）
    const list = await R.find("group", " public = 1 AND status_page_id = ? ORDER BY weight ", [statusPage.id]);
    
    for (let groupBean of list) {
        let monitorGroup = await groupBean.toPublicJSON(showTags, config?.showCertificateExpiry);
        publicGroupList.push(monitorGroup);
    }
    
    return { config, incidents, publicGroupList, maintenanceList };
}
```

**返回数据结构对照**：

| 数据项 | 公开 API (`toPublicJSON`) | 后台管理 (`toJSON`) |
|-------|--------------------------|---------------------|
| `id` | ❌ 隐藏 | ✅ 可见 |
| `slug` | ✅ 可见 | ✅ 可见 |
| `title` | ✅ 可见 | ✅ 可见 |
| `description` | ✅ 可见 | ✅ 可见 |
| `theme` | ✅ 可见 | ✅ 可见 |
| `autoRefreshInterval` | ✅ 可见 | ✅ 可见 |
| `published` | ✅ 可见 | ✅ 可见 |
| `showTags` | ✅ 可见 | ✅ 可见 |
| `domainNameList` | ❌ 隐藏 | ✅ 可见（敏感配置） |
| `customCSS` | ✅ 可见 | ✅ 可见 |
| `footerText` | ✅ 可见 | ✅ 可见 |

---

#### 7.2.3 心跳数据 API（最敏感的裁剪点）

| 项目 | 详情 |
|-----|------|
| **路由路径** | `GET /api/status-page/heartbeat/:slug` |
| **文件位置** | `server/routers/status-page-router.js:64-110` |
| **缓存策略** | 1 分钟 |
| **鉴权规则** | 无 - 完全公开 |
| **关键裁剪** | `toPublicJSON()` 将 `msg` 设为空字符串 |

**数据获取逻辑** (`server/routers/status-page-router.js:75-97`)：

```javascript
// 只获取公开组的监控 ID
let monitorIDList = await R.getCol(
    `
    SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
    WHERE monitor_group.group_id = \`group\`.id
    AND public = 1
    AND \`group\`.status_page_id = ?
    `,
    [statusPageID]
);

// 获取心跳数据并使用 toPublicJSON() 裁剪
for (let monitorID of monitorIDList) {
    let list = await R.getAll(
        `SELECT * FROM heartbeat WHERE monitor_id = ? ORDER BY time DESC LIMIT 100`,
        [monitorID]
    );
    list = R.convertToBeans("heartbeat", list);
    heartbeatList[monitorID] = list.reverse().map((row) => row.toPublicJSON());
}
```

**心跳数据裁剪对照**：

| 字段 | 公开 API (`toPublicJSON`) | 后台管理 (`toJSON`/`toJSONAsync`) |
|-----|--------------------------|---------------------|
| `status` | ✅ 可见 (0=DOWN, 1=UP, 2=PENDING, 3=MAINTENANCE) | ✅ 可见 |
| `time` | ✅ 可见 | ✅ 可见 |
| `msg` | ❌ **设为空字符串**（最关键的安全措施） | ✅ 完整可见（错误详情） |
| `ping` | ✅ 可见 | ✅ 可见 |
| `monitorID` | ❌ 隐藏 | ✅ 可见 |
| `important` | ❌ 隐藏 | ✅ 可见 |
| `duration` | ❌ 隐藏 | ✅ 可见 |
| `retries` | ❌ 隐藏 | ✅ 可见 |
| `response` | ❌ 隐藏 | ✅ 完整响应（可能包含敏感内容） |

**关键代码** (`server/model/heartbeat.js:19-26`)：

```javascript
toPublicJSON() {
    return {
        status: this.status,
        time: this.time,
        msg: "",  // 强制设为空字符串！防止泄露错误详情
        ping: this.ping,
    };
}
```

**安全意义**：
- `msg` 字段可能包含内部 URL、认证错误、数据库连接信息等敏感内容
- `response` 字段包含完整的 HTTP 响应体，可能泄露内部系统信息
- 公开访问时这些字段被完全隐藏，只保留最基本的状态信息

---

#### 7.2.4 事件历史 API

| 项目 | 详情 |
|-----|------|
| **路由路径** | `GET /api/status-page/:slug/incident-history` |
| **文件位置** | `server/routers/status-page-router.js:145-167` |
| **缓存策略** | 5 分钟 |
| **鉴权规则** | 无 - 完全公开 |
| **游标分页** | 使用 `cursor` 参数实现游标分页 |

**获取逻辑** (`server/model/status_page.js:512-528`)：

```javascript
static async getIncidentHistory(statusPageId, cursor = null, isPublic = true) {
    let incidents;
    
    if (cursor) {
        incidents = await R.find(
            "incident",
            " status_page_id = ? AND created_date < ? ORDER BY created_date DESC LIMIT ? ",
            [statusPageId, cursor, INCIDENT_PAGE_SIZE]
        );
    } else {
        incidents = await R.find(
            "incident", 
            " status_page_id = ? ORDER BY created_date DESC LIMIT ? ",
            [statusPageId, INCIDENT_PAGE_SIZE]
        );
    }
    
    // isPublic=true 时使用 toPublicJSON()
    const incidentsJSON = incidents.map((i) => i.toPublicJSON());
}
```

**事件数据对照**：

| 字段 | 公开 API (`toPublicJSON`) | 后台管理 |
|-----|--------------------------|---------------------|
| `id` | ✅ 可见 | ✅ 可见 |
| `style` | ✅ 可见 | ✅ 可见 |
| `title` | ✅ 可见 | ✅ 可见 |
| `content` | ✅ 可见 | ✅ 可见 |
| `pin` | ✅ 可见 | ✅ 可见 |
| `active` | ✅ 可见 | ✅ 可见 |
| `createdDate` | ✅ 可见 | ✅ 可见 |
| `lastUpdatedDate` | ✅ 可见 | ✅ 可见 |
| `status_page_id` | ✅ 可见 | ✅ 可见 |

**注意**：Incident 模型没有单独的 `toJSON()` 方法，因为事件本身就是设计为公开显示的，所有字段都是公开的。

---

#### 7.2.5 状态徽章 API

| 项目 | 详情 |
|-----|------|
| **路由路径** | `GET /api/status-page/:slug/badge` |
| **文件位置** | `server/routers/status-page-router.js:170-262` |
| **缓存策略** | 5 分钟 |
| **返回格式** | SVG 图像 |
| **鉴权规则** | 无 - 完全公开 |

**状态判断逻辑** (`server/routers/status-page-router.js:195-251`)：

```javascript
// 只检查公开组的监控
let monitorIDList = await R.getCol(
    `
    SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
    WHERE monitor_group.group_id = \`group\`.id
    AND public = 1
    AND \`group\`.status_page_id = ?
    `,
    [statusPageID]
);

// 检查每个监控的最新心跳状态
for (let monitorID of monitorIDList) {
    let beat = await R.getAll(
        `SELECT * FROM heartbeat WHERE monitor_id = ? ORDER BY time DESC LIMIT 1`,
        [monitorID]
    );
    
    if (beat.length === 0) continue;
    
    if (beat[0].status === 3) {
        hasMaintenance = true;
    } else if (beat[0].status === 2) {
        // ignored (PENDING)
    } else if (beat[0].status === 1) {
        hasUp = true;
    } else {
        hasDown = true;
    }
}
```

**可能的状态值**：

| 状态 | 消息 | 颜色 | 条件 |
|-----|------|------|------|
| **Up** | "Up" | 绿色 (upColor) | 所有公开监控正常 |
| **Degraded** | "Degraded" | 黄色 (#F6BE00) | 部分正常，部分故障 |
| **Down** | "Down" | 红色 (downColor) | 所有公开监控故障 |
| **Maintenance** | "Maintenance" | 灰色 (#808080) | 存在维护状态 |
| **N/A** | "N/A" | 灰色 | 无公开监控 |

---

#### 7.2.6 RSS 订阅接口

| 项目 | 详情 |
|-----|------|
| **路由路径** | `GET /status/:slug/rss` |
| **文件位置** | `server/routers/status-page-router.js:22-26` |
| **缓存策略** | 5 分钟 |
| **返回格式** | RSS XML |
| **鉴权规则** | 无 - 完全公开 |

**URL 构建逻辑** (`server/model/status_page.js:117-141`)：

```javascript
static async buildRSSUrl(slug, request) {
    if (request) {
        const trustProxy = await setting("trustProxy");
        
        // 考虑 X-Forwarded-* 头
        let proto = request.protocol;
        if (trustProxy && request.headers["x-forwarded-proto"]) {
            proto = request.headers["x-forwarded-proto"].split(",")[0].trim();
        }
        
        let host = request.get("host");
        if (trustProxy && request.headers["x-forwarded-host"]) {
            host = request.headers["x-forwarded-host"];
        }
        
        return `${proto}://${host}/status/${slug}`;
    }
}
```

---

### 7.3 后台管理接口详细分析

后台管理接口全部通过 **Socket.IO** 实现，并且所有操作都需要 `checkLogin(socket)` 验证。

#### 7.3.1 获取状态页完整配置

| 项目 | 详情 |
|-----|------|
| **Socket 事件** | `getStatusPage` |
| **文件位置** | `server/socket-handlers/status-page-socket-handler.js:268-288` |
| **连接方式** | Socket.IO |
| **鉴权规则** | `checkLogin(socket)` - 必须登录 |
| **数据序列化** | 使用 `toJSON()` 而非 `toPublicJSON()` |

**关键代码**：

```javascript
socket.on("getStatusPage", async (slug, callback) => {
    try {
        checkLogin(socket);  // 必须登录

        let statusPage = await R.findOne("status_page", " slug = ? ", [slug]);

        if (!statusPage) {
            throw new Error("No slug?");
        }

        callback({
            ok: true,
            config: await statusPage.toJSON(),  // 使用完整版本！
        });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

**与公开 API 的差异**：

| 数据项 | 后台管理 (`toJSON`) | 公开 API (`toPublicJSON`) |
|-------|---------------------|--------------------------|
| `id` | ✅ 可见 | ❌ 隐藏 |
| `domainNameList` | ✅ 可见（敏感配置） | ❌ 隐藏 |
| 所有其他字段 | ✅ 完整可见 | ✅ 可见 |

---

#### 7.3.2 保存状态页配置

| 项目 | 详情 |
|-----|------|
| **Socket 事件** | `saveStatusPage` |
| **文件位置** | `server/socket-handlers/status-page-socket-handler.js:292-433` |
| **连接方式** | Socket.IO |
| **鉴权规则** | `checkLogin(socket)` - 必须登录 |

**可修改的字段**（全部需要登录）：

| 字段 | 说明 |
|-----|------|
| `slug` | 状态页 URL 路径 |
| `title` | 标题 |
| `description` | 描述 |
| `logo/icon` | Logo 图标 |
| `autoRefreshInterval` | 自动刷新间隔 |
| `theme` | 主题 (auto/light/dark) |
| `showTags` | 是否显示标签 |
| `domainNameList` | 域名映射列表（敏感配置） |
| `customCSS` | 自定义 CSS |
| `footerText` | 页脚文字 |
| `showPoweredBy` | 是否显示 Powered by |
| `analyticsId` | 分析配置 |
| `showOnlyLastHeartbeat` | 是否只显示最后心跳 |
| `showCertificateExpiry` | 是否显示证书过期 |
| `publicGroupList` | 公开监控组列表 |

**关键代码片段**：

```javascript
socket.on("saveStatusPage", async (slug, config, imgDataUrl, publicGroupList, callback) => {
    try {
        checkLogin(socket);  // 必须登录

        // ... 保存配置 ...
        
        // 更新域名映射
        await statusPage.updateDomainNameList(config.domainNameList);
        await StatusPage.loadDomainMappingList();  // 重新加载映射

        // 保存公开监控组
        // ...
        
        apicache.clear();  // 清除缓存

        callback({ ok: true, publicGroupList });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

---

#### 7.3.3 事件管理操作

| Socket 事件 | 功能 | 鉴权 |
|------------|------|------|
| `postIncident` | 创建/编辑事件 | `checkLogin(socket)` |
| `editIncident` | 编辑事件 | `checkLogin(socket)` |
| `resolveIncident` | 解决事件 | `checkLogin(socket)` |
| `deleteIncident` | 删除事件 | `checkLogin(socket)` |
| `unpinIncident` | 取消置顶事件 | `checkLogin(socket)` |
| `getIncidentHistory` | 获取事件历史 | **无需登录**（但有 `isPublic` 参数） |

**注意**：`getIncidentHistory` 是唯一无需登录的事件操作，但它有 `isPublic` 参数控制返回数据：

```javascript
socket.on("getIncidentHistory", async (slug, cursor, callback) => {
    try {
        let statusPageID = await StatusPage.slugToID(slug);
        
        const isPublic = !socket.userID;  // 未登录用户视为公开访问
        const result = await StatusPage.getIncidentHistory(statusPageID, cursor, isPublic);
        callback({ ok: true, ...result });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

**设计意图**：
- 未登录用户（`!socket.userID`）`isPublic=true`，使用 `toPublicJSON()`
- 已登录用户 `isPublic=false`，可能使用更多字段（但当前实现中 Incident 没有额外字段）

---

#### 7.3.4 状态页管理操作

| Socket 事件 | 功能 | 鉴权 |
|------------|------|------|
| `addStatusPage` | 创建新状态页 | `checkLogin(socket)` |
| `deleteStatusPage` | 删除状态页 | `checkLogin(socket)` |

**删除操作的连锁反应** (`server/socket-handlers/status-page-socket-handler.js:482-523`)：

```javascript
socket.on("deleteStatusPage", async (slug, callback) => {
    try {
        checkLogin(socket);

        let statusPageID = await StatusPage.slugToID(slug);

        if (statusPageID) {
            // 重置入口页配置
            if (server.entryPage === "statusPage-" + slug) {
                server.entryPage = "dashboard";
                await Settings.set("entryPage", server.entryPage, "general");
            }

            // 手动删除关联数据（没有级联外键）
            await R.exec("DELETE FROM incident WHERE status_page_id = ? ", [statusPageID]);
            await R.exec("DELETE FROM `group` WHERE status_page_id = ? ", [statusPageID]);
            await R.exec("DELETE FROM status_page WHERE id = ? ", [statusPageID]);

            apicache.clear();
        } else {
            throw new Error("Status Page is not found");
        }

        callback({ ok: true });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

---

#### 7.3.5 Cloudflared 管理操作

| Socket 事件 | 功能 | 鉴权 |
|------------|------|------|
| `cloudflared_join` | 加入 cloudflared 房间 | `checkLogin(socket)` |
| `cloudflared_start` | 启动 Cloudflare Tunnel | `checkLogin(socket)` |
| `cloudflared_stop` | 停止 Cloudflare Tunnel | `checkLogin(socket)` + `doubleCheckPassword` |

**停止操作需要双重密码验证** (`server/socket-handlers/cloudflared-socket-handler.js:71-80`)：

```javascript
socket.on(prefix + "stop", async (currentPassword, callback) => {
    try {
        checkLogin(socket);
        const disabledAuth = await setting("disableAuth");
        if (!disabledAuth) {
            await doubleCheckPassword(socket, currentPassword);  // 需要再次输入密码
        }
        cloudflared.stop();
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

---

### 7.4 监控项数据的详细裁剪

#### 7.4.1 Monitor 模型数据对照

| 字段 | 公开访问 (`toPublicJSON`) | 后台管理 (`toJSON`) | 裁剪规则 |
|-----|--------------------------|---------------------|---------|
| `id` | ✅ 可见 | ✅ 可见 | 始终可见 |
| `name` | ✅ 可见 | ✅ 可见 | 始终可见 |
| `type` | ✅ 可见 | ✅ 可见 | 始终可见 |
| `sendUrl` | ✅ 可见 | ✅ 可见 | 控制 URL 显示 |
| `url` | ⚠️ 条件可见 | ✅ 可见 | 仅当 `sendUrl=true` 时 |
| `customUrl` | ⚠️ 条件可见 | ✅ 可见 | 仅当 `sendUrl=true` 时 |
| `tags` | ⚠️ 条件可见 | ✅ 可见 | 由状态页 `showTags` 控制 |
| `certExpiryDaysRemaining` | ⚠️ 条件可见 | ❌ 无此字段 | 由状态页 `showCertificateExpiry` 控制 |
| `validCert` | ⚠️ 条件可见 | ❌ 无此字段 | 由状态页 `showCertificateExpiry` 控制 |
| `description` | ❌ 隐藏 | ✅ 可见 | 后台管理可见 |
| `interval` | ❌ 隐藏 | ✅ 可见 | 监控间隔 |
| `timeout` | ❌ 隐藏 | ✅ 可见 | 超时时间 |
| `retryInterval` | ❌ 隐藏 | ✅ 可见 | 重试间隔 |
| `maxretries` | ❌ 隐藏 | ✅ 可见 | 最大重试次数 |
| `accepted_statuscodes` | ❌ 隐藏 | ✅ 可见 | 接受的状态码 |
| `keyword` | ❌ 隐藏 | ✅ 可见 | 关键字检查 |
| `method` | ❌ 隐藏 | ✅ 可见 | HTTP 方法 |
| `proxyId` | ❌ 隐藏 | ✅ 可见 | 代理配置 ID |
| `notificationIDList` | ❌ 隐藏 | ✅ 可见 | 通知配置列表 |

**关键代码** (`server/model/monitor.js:85-108`)：

```javascript
async toPublicJSON(showTags = false, certExpiry = false) {
    let obj = {
        id: this.id,
        name: this.name,
        sendUrl: this.sendUrl,
        type: this.type,
    };

    // URL 显示受 sendUrl 控制
    if (this.sendUrl) {
        obj.url = this.customUrl ?? this.url;
    }

    // 标签显示受状态页配置控制
    if (showTags) {
        obj.tags = await this.getTags();
    }

    // 证书过期信息受状态页配置控制
    if (certExpiry) {
        const { certExpiryDaysRemaining, validCert } = await this.getCertExpiry(this.id);
        obj.certExpiryDaysRemaining = certExpiryDaysRemaining;
        obj.validCert = validCert;
    }

    return obj;
}
```

---

#### 7.4.2 Group 模型数据对照

| 字段 | 公开访问 (`toPublicJSON`) | 说明 |
|-----|--------------------------|------|
| `id` | ✅ 可见 | 组 ID |
| `name` | ✅ 可见 | 组名称 |
| `weight` | ✅ 可见 | 排序权重 |
| `monitorList` | ✅ 可见 | 监控列表（使用 `toPublicJSON()`） |

**关键代码** (`server/model/group.js:13-27`)：

```javascript
async toPublicJSON(showTags = false, certExpiry = false) {
    let monitorBeanList = await this.getMonitorList();
    let monitorList = [];

    for (let bean of monitorBeanList) {
        monitorList.push(await bean.toPublicJSON(showTags, certExpiry));  // 级联裁剪
    }

    return {
        id: this.id,
        name: this.name,
        weight: this.weight,
        monitorList,
    };
}
```

---

### 7.5 鉴权边界总览

#### 7.5.1 三层鉴权架构

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              鉴权边界架构                                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  第一层：连接层边界 (Connection Layer)                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                              │   │
│  │  noSocketIOPages 白名单                                                       │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                      │   │
│  │  │ /status/*   │    │ /status-page│    │      /      │                      │   │
│  │  │             │    │             │    │             │                      │   │
│  │  │ 不建立       │    │ 不建立       │    │ 不建立       │                      │   │
│  │  │ Socket.IO   │    │ Socket.IO   │    │ Socket.IO   │                      │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘                      │   │
│  │                                                                              │   │
│  │  例外：?edit 参数 bypass=true，强制建立 Socket.IO                            │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                               │
│                                      ▼                                               │
│  第二层：认证层边界 (Authentication Layer)                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                              │   │
│  │  HTTP API (公开接口)                    Socket.IO (管理接口)                   │   │
│  │  ┌─────────────────────┐              ┌─────────────────────┐               │   │
│  │  │ GET /api/status-page│              │ getStatusPage       │               │   │
│  │  │ GET /api/status-page│              │ saveStatusPage      │               │   │
│  │  │ /heartbeat          │              │ postIncident        │               │   │
│  │  │ GET /api/status-page│              │ deleteIncident      │               │   │
│  │  │ /incident-history   │              │ addStatusPage       │               │   │
│  │  │ GET /api/status-page│              │ deleteStatusPage    │               │   │
│  │  │ /badge              │              │ cloudflared_*       │               │   │
│  │  │                     │              │                     │               │   │
│  │  │ 鉴权：无            │              │ 鉴权：checkLogin()  │               │   │
│  │  │ 任何人可访问         │              │ 必须登录            │               │   │
│  │  └─────────────────────┘              └─────────────────────┘               │   │
│  │                                                                              │   │
│  │  特殊情况：cloudflared_stop 需要 doubleCheckPassword() 二次密码验证          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                               │
│                                      ▼                                               │
│  第三层：数据层边界 (Data Layer)                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                              │   │
│  │  toJSON() (完整数据)                    toPublicJSON() (裁剪数据)              │   │
│  │  ┌─────────────────────┐              ┌─────────────────────┐               │   │
│  │  │ StatusPage:         │              │ StatusPage:         │               │   │
│  │  │ - id                │              │ - 无 id             │               │   │
│  │  │ - domainNameList    │              │ - 无 domainNameList │               │   │
│  │  │ - 所有其他字段       │              │ - 其他字段保留       │               │   │
│  │  │                     │              │                     │               │   │
│  │  │ Monitor:            │              │ Monitor:            │               │   │
│  │  │ - 所有配置字段       │              │ - 仅 id, name, type │               │   │
│  │  │ - url, interval,    │              │ - url 受 sendUrl    │               │   │
│  │  │   timeout, etc.     │              │   控制              │               │   │
│  │  │                     │              │                     │               │   │
│  │  │ Heartbeat:          │              │ Heartbeat:          │               │   │
│  │  │ - msg (完整错误消息) │              │ - msg = "" (空！)   │               │   │
│  │  │ - response          │              │ - 无 response       │               │   │
│  │  │ - important,        │              │ - 无 important 等   │               │   │
│  │  │   duration, retries │              │                     │               │   │
│  │  └─────────────────────┘              └─────────────────────┘               │   │
│  │                                                                              │   │
│  │  使用场景：                                                                  │   │
│  │  - 后台管理 Socket.IO 操作 → toJSON()                                     │   │
│  │  - 公开 HTTP API → toPublicJSON()                                         │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 7.5.2 鉴权规则速查表

| 操作类型 | 入口方式 | 连接层 | 认证层 | 数据层 |
|---------|---------|--------|--------|--------|
| 查看公开状态页 | HTTP GET /status/:slug | ❌ 无 Socket.IO | ❌ 无需登录 | `toPublicJSON()` |
| 获取状态页配置 | HTTP GET /api/status-page/:slug | ❌ 无 Socket.IO | ❌ 无需登录 | `toPublicJSON()` |
| 获取心跳数据 | HTTP GET /api/status-page/heartbeat/:slug | ❌ 无 Socket.IO | ❌ 无需登录 | `toPublicJSON()` (msg="") |
| 查看事件历史 | HTTP GET /api/status-page/:slug/incident-history | ❌ 无 Socket.IO | ❌ 无需登录 | `toPublicJSON()` |
| 获取徽章 | HTTP GET /api/status-page/:slug/badge | ❌ 无 Socket.IO | ❌ 无需登录 | SVG 生成 |
| RSS 订阅 | HTTP GET /status/:slug/rss | ❌ 无 Socket.IO | ❌ 无需登录 | RSS 生成 |
| **获取完整配置** | Socket.IO `getStatusPage` | ✅ 需 Socket.IO | ✅ `checkLogin()` | `toJSON()` (含敏感字段) |
| **保存配置** | Socket.IO `saveStatusPage` | ✅ 需 Socket.IO | ✅ `checkLogin()` | 写操作 |
| **创建事件** | Socket.IO `postIncident` | ✅ 需 Socket.IO | ✅ `checkLogin()` | 写操作 |
| **删除状态页** | Socket.IO `deleteStatusPage` | ✅ 需 Socket.IO | ✅ `checkLogin()` | 写操作 |
| **启动 Tunnel** | Socket.IO `cloudflared_start` | ✅ 需 Socket.IO | ✅ `checkLogin()` | 写操作 |
| **停止 Tunnel** | Socket.IO `cloudflared_stop` | ✅ 需 Socket.IO | ✅ `checkLogin()` + `doubleCheckPassword()` | 写操作 |

---

## 八、相关文件索引

| 文件路径 | 主要职责 |
|---------|---------|
| `server/routers/status-page-router.js` | 状态页路由定义、API 端点 |
| `server/model/status_page.js` | 状态页数据模型、SSR 渲染、域名映射 |
| `server/socket-handlers/status-page-socket-handler.js` | 状态页 WebSocket 处理 |
| `server/socket-handlers/cloudflared-socket-handler.js` | Cloudflare Tunnel 管理 |
| `server/server.js` | 服务器入口、域名路由检测 |
| `src/pages/StatusPage.vue` | 前端状态页组件 |
| `src/components/settings/ReverseProxy.vue` | 反向代理设置界面 |
| `src/router.js` | 前端路由配置 |
| `src/mixins/socket.js` | Socket.IO 连接初始化、白名单规则 |
| `server/util-server.js` | `checkLogin()`, `doubleCheckPassword()` 鉴权函数 |
| `server/model/heartbeat.js` | 心跳数据模型、`toPublicJSON()` 裁剪逻辑 |
| `server/model/monitor.js` | 监控数据模型、URL 显示控制 |
| `server/model/incident.js` | 事件数据模型 |
| `server/model/group.js` | 监控组数据模型 |

---

## 九、总结

Uptime Kuma 的 Status Page 通过多层次的安全设计和灵活的部署选项，实现了安全、可靠的公开访问。以下是核心要点的总结：

### 8.1 端到端链路：Cloudflared 的零信任访问

**Cloudflare Tunnel 的核心优势**：

1. **零入站端口**：cloudflared 进程主动向 Cloudflare 边缘发起出站 WebSocket/QUIC 连接，无需在防火墙开放任何入站端口
2. **本地监听**：Uptime Kuma 只监听 `localhost:3001`，不直接暴露到公网
3. **流量加密**：所有流量通过 Cloudflare 的全球网络加密传输
4. **自动重连**：隧道连接是持久的，断开后自动重连

**与传统反向代理的对比**：

| 特性 | Cloudflare Tunnel | Nginx/Apache 反向代理 |
|-----|------------------|----------------------|
| 开放端口 | 无需开放 | 需要开放 80/443 |
| 公网暴露 | 服务器不直接暴露 | 服务器直接暴露 |
| SSL 证书 | Cloudflare 自动管理 | 需自行申请和配置 |
| DDoS 防护 | Cloudflare 提供 | 需自行配置 |
| 配置复杂度 | 低（输入 Token 即可） | 高（需配置 Nginx、SSL 等） |

### 8.2 权限边界：连接层与数据层的双重防护

**连接层边界**：

1. **Socket.IO 白名单机制**：
   - 状态页路径 (`/status/*`, `/`, `/status-page`) 不建立 Socket.IO 连接
   - 只有后台管理路径 (`/dashboard/*`) 才建立 WebSocket 连接
   - 编辑模式 (`?edit`) 通过 `bypass=true` 参数强制建立连接

2. **登录检查**：
   - 所有 Socket.IO 操作（保存状态页、创建事件、管理 cloudflared 等）都需要 `checkLogin(socket)` 验证
   - `checkLogin()` 检查 `socket.userID` 是否存在，不存在则抛出错误

**数据层边界**：

1. **公开 API 与内部 API 分离**：
   - 公开访问：HTTP API + 缓存（状态页配置 5 分钟，心跳数据 1 分钟）
   - 内部管理：Socket.IO + 实时推送

2. **URL 路由隔离**：
   - 公开路径：`/status/*`, `/api/status-page/*`
   - 管理路径：`/dashboard/*`, `/settings/*`

### 8.3 数据裁剪：`toPublicJSON()` 作为安全边界

**核心裁剪策略**：

1. **StatusPage 模型**：
   - 不返回 `id` 和 `domainNameList`（敏感配置）
   - `toJSON()` 返回完整数据（包含域名映射）
   - `toPublicJSON()` 返回裁剪后的数据

2. **Monitor 模型**：
   - 仅返回 `id`, `name`, `type` 基础信息
   - URL 显示受 `sendUrl` 控制（状态页配置）
   - 标签和证书过期信息是可选的

3. **Heartbeat 模型（最重要的安全措施）**：
   - `toPublicJSON()` 将 `msg` 字段设为空字符串！
   - 这防止了公开访问时泄露错误详情（可能包含内部 URL、认证错误、敏感路径等）
   - `toJSON()` 返回完整的 `msg`, `important`, `duration`, `retries`, `response` 等调试信息

**安全设计意图**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           数据流向与安全边界                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  数据库层                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  status_page, monitor, heartbeat, incident, maintenance 等表        │   │
│  │  包含所有字段：敏感配置、错误消息、内部 URL 等                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  模型层 (BeanModel)                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  toJSON() ←───────────────────── toPublicJSON()                     │   │
│  │  │                                    │                               │   │
│  │  ▼                                    ▼                               │   │
│  │  ┌──────────────┐              ┌──────────────┐                      │   │
│  │  │  完整数据     │              │  裁剪数据     │                      │   │
│  │  │  (含敏感字段) │              │  (仅公开字段) │                      │   │
│  │  └──────────────┘              └──────────────┘                      │   │
│  │         │                              │                               │   │
│  │         ▼                              ▼                               │   │
│  │  Socket.IO 操作                    HTTP API                            │   │
│  │  (需要登录)                        (无身份认证)                         │   │
│  │  后台管理界面                      公开状态页                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.4 最终总结

Uptime Kuma 的 Status Page 设计体现了以下核心原则：

1. **零信任网络**：通过 Cloudflare Tunnel 实现零入站端口的访问模式，服务器无需直接暴露到公网

2. **深度防御**：
   - 连接层：Socket.IO 白名单，状态页不建立 WebSocket 连接
   - 认证层：所有写操作和敏感读操作需要登录
   - 数据层：`toPublicJSON()` 裁剪敏感字段，特别是心跳消息隐藏

3. **灵活部署**：
   - 支持 URL 路径路由、域名映射、Cloudflare Tunnel 三种访问方式
   - `trustProxy` 设置确保在反向代理后面正确获取客户端信息
   - 域名映射允许将自定义域名直接绑定到状态页

4. **性能优化**：
   - SSR 预加载 + API 轮询的混合渲染模式
   - 分层缓存策略（状态页配置 5 分钟，心跳数据 1 分钟）
   - 公开页不建立 Socket.IO 连接，减少服务器负载

这种设计使得 Uptime Kuma 既可以作为小型团队的内部监控工具，也可以作为大型企业的公开状态页系统，满足不同规模和安全要求的部署场景。
