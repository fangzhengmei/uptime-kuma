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
            checkLogin(socket);
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
                await doubleCheckPassword(socket, currentPassword);
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

## 六、相关文件索引

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

---

## 七、总结

Uptime Kuma 的 Status Page 通过多层次的设计实现了灵活的公开访问：

1. **数据渲染**：采用 SSR + API 轮询的混合模式，首次加载快速，后续数据实时更新
2. **访问方式**：支持 URL 路径路由、域名映射 (CNAME)、Cloudflare Tunnel 三种方式
3. **反向代理支持**：通过 `trustProxy` 设置正确处理 X-Forwarded-* 头部
4. **多服务协作**：各模块职责清晰，路由层、模型层、Socket 层协同工作

这种设计使得 Uptime Kuma 既可以直接暴露在公网，也可以灵活地部署在各种反向代理后面，满足不同规模和安全要求的部署场景。
