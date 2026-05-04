# Uptime Kuma 监控请求组装与代理优先级分析

## 一、HTTP 监控请求选项组装流程

### 1. URL 组装

URL 直接使用 Monitor 对象的 `url` 属性：

```javascript
const options = {
    url: this.url,
    // ...
};
```
`server/model/monitor.js:546`

### 2. HTTP Method 处理

Method 使用 `this.method`，默认为 "get"：

```javascript
method: (this.method || "get").toLowerCase(),
```
`server/model/monitor.js:547`

### 3. 超时配置

超时时间以秒为单位存储，实际请求时转换为毫秒：

```javascript
timeout: this.timeout * 1000,
```
`server/model/monitor.js:548`

此外，还设置了 axios 的 abort signal，比超时多 10 秒作为缓冲：

```javascript
signal: axiosAbortSignal((this.timeout + 10) * 1000),
```
`server/model/monitor.js:560`

### 4. Headers 组装

Headers 采用多层合并策略，按优先级从低到高排序：

```javascript
headers: {
    Accept: "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9",
    ...(contentType ? { "Content-Type": contentType } : {}),
    ...basicAuthHeader,
    ...oauth2AuthHeader,
    ...(this.headers ? JSON.parse(this.headers) : {}),
},
```
`server/model/monitor.js:549-555`

**Headers 优先级（高优先级覆盖低优先级）：**
1. 自定义 headers（用户在监控配置中填写的 JSON headers）
2. OAuth2 Authorization 头
3. Basic Auth Authorization 头
4. Content-Type（根据请求体编码格式决定）
5. 默认 Accept 头

#### 自定义 Headers 覆盖认证头的风险

**重要安全警告**：由于用户自定义 headers 优先级最高，如果用户在自定义 headers 中包含 `Authorization` 字段，会**静默覆盖** Basic Auth 或 OAuth2 自动生成的认证头。

**风险场景示例：**

| 场景 | auth_method | 配置的自定义 headers | 实际发送的 Authorization | 结果 |
|------|-------------|----------------------|-------------------------|------|
| 风险 1 | `basic` | `{"Authorization": "Bearer invalid"}` | `Bearer invalid` | 覆盖正确的 Basic Auth，认证失败 |
| 风险 2 | `oauth2-cc` | `{"Authorization": "Basic oldcred"}` | `Basic oldcred` | 覆盖有效的 OAuth2 token，认证失败 |
| 风险 3 | `basic` | `{"authorization": "Test"}`（小写 key） | `Basic xxx`（未覆盖） | 大小写敏感，不会覆盖 |

**技术原因分析：**

```javascript
// 代码中的展开顺序
headers: {
    Accept: "...",                              // 1
    ...(contentType ? { "Content-Type": ... }), // 2
    ...basicAuthHeader,                         // 3: { "Authorization": "Basic xxx" }
    ...oauth2AuthHeader,                        // 4: { "Authorization": "Bearer yyy" }
    ...(this.headers ? JSON.parse(this.headers) : {}),  // 5: 最高优先级
},
```
`server/model/monitor.js:549-555`

JavaScript 对象展开运算符的特性：**后面展开的属性会覆盖前面的同名属性**。

**后果：**
1. **静默失败**：没有任何错误提示，请求直接携带错误的认证信息
2. **难以排查**：用户可能忘记自己在自定义 headers 中配置了 Authorization，导致调试困难
3. **安全隐患**：如果自定义 headers 中的 Authorization 是敏感信息（如旧 token、硬编码凭证），可能意外暴露

**建议：**
- 如果使用 Basic Auth 或 OAuth2，避免在自定义 headers 中配置 `Authorization` 字段
- 如需调试认证问题，先检查是否在自定义 headers 中配置了同名字段

#### Content-Type 判定逻辑

根据 `httpBodyEncoding` 字段决定：

```javascript
if (this.body && typeof this.body === "string" && this.body.trim().length > 0) {
    if (!this.httpBodyEncoding || this.httpBodyEncoding === "json") {
        bodyValue = JSON.parse(this.body);
        contentType = "application/json";
    } else if (this.httpBodyEncoding === "form") {
        bodyValue = this.body;
        contentType = "application/x-www-form-urlencoded";
    } else if (this.httpBodyEncoding === "xml") {
        bodyValue = this.body;
        contentType = "text/xml; charset=utf-8";
    }
}
```
`server/model/monitor.js:527-542`

### 5. 认证方式处理

支持三种认证方式：Basic Auth、OAuth2 Client Credentials、mTLS。

#### 5.1 Basic Auth

当 `auth_method === "basic"` 时：

```javascript
let basicAuthHeader = {};
if (this.auth_method === "basic") {
    basicAuthHeader = {
        Authorization: "Basic " + encodeBase64(this.basic_auth_user, this.basic_auth_pass),
    };
}
```
`server/model/monitor.js:473-478`

#### 5.2 OAuth2 Client Credentials

当 `auth_method === "oauth2-cc"` 时，会先获取（或从缓存中获取）access token：

```javascript
let oauth2AuthHeader = {};
if (this.auth_method === "oauth2-cc") {
    try {
        if (
            this.oauthAccessToken === undefined ||
            new Date(this.oauthAccessToken.expires_at * 1000) <= new Date()
        ) {
            this.oauthAccessToken = await this.makeOidcTokenClientCredentialsRequest();
        }
        oauth2AuthHeader = {
            Authorization:
                this.oauthAccessToken.token_type + " " + this.oauthAccessToken.access_token,
        };
    } catch (e) {
        throw new Error("The oauth config is invalid. " + e.message);
    }
}
```
`server/model/monitor.js:482-498`

**特点：**
- Token 有缓存机制，`expires_at` 过期后才重新获取
- Token 格式为：`{token_type} {access_token}`（通常是 `Bearer {token}`）

#### 5.3 mTLS 双向认证

当 `auth_method === "mtls"` 时，直接在 HTTPS Agent 上设置证书：

```javascript
if (this.auth_method === "mtls") {
    if (this.tlsCert !== null && this.tlsCert !== "") {
        options.httpsAgent.options.cert = Buffer.from(this.tlsCert);
    }
    if (this.tlsCa !== null && this.tlsCa !== "") {
        options.httpsAgent.options.ca = Buffer.from(this.tlsCa);
    }
    if (this.tlsKey !== null && this.tlsKey !== "") {
        options.httpsAgent.options.key = Buffer.from(this.tlsKey);
    }
}
```
`server/model/monitor.js:603-613`

**支持的证书配置：**
- `tlsCert`: 客户端证书
- `tlsCa`: CA 证书
- `tlsKey`: 客户端私钥

### 6. 证书校验（TLS/SSL）

证书校验由 `ignoreTls` 配置控制：

```javascript
const httpsAgentOptions = {
    maxCachedSessions: 0,
    rejectUnauthorized: !this.getIgnoreTls(),  // 关键配置
    secureOptions: crypto.constants.SSL_OP_LEGACY_SERVER_CONNECT,
    autoSelectFamily: true,
    ...(agentFamily ? { family: agentFamily } : {}),
};
```
`server/model/monitor.js:508-514`

**逻辑：**
- `ignoreTls = false`（默认）：`rejectUnauthorized = true`，严格校验证书
- `ignoreTls = true`：`rejectUnauthorized = false`，忽略证书错误（不推荐用于生产）

### 7. IP 地址族选择

支持 IPv4 或 IPv6 强制指定：

```javascript
let agentFamily = undefined;
if (this.ipFamily === "ipv4") {
    agentFamily = 4;
}
if (this.ipFamily === "ipv6") {
    agentFamily = 6;
}
// 然后合并到 agent options
...(agentFamily ? { family: agentFamily } : {}),
```
`server/model/monitor.js:500-513`

---

## 二、代理配置处理与优先级

### 1. 代理配置存储

代理配置存储在 `proxy` 表中，关键字段：

| 字段 | 说明 |
|------|------|
| `protocol` | 代理协议：http, https, socks, socks5, socks5h, socks4 |
| `host` | 代理主机 |
| `port` | 代理端口 |
| `auth` | 是否启用认证 |
| `username` | 代理用户名 |
| `password` | 代理密码 |
| `active` | 是否启用 |
| `default` | 是否为默认代理 |

`server/proxy.js:46-53`

### 2. 支持的代理协议

```javascript
static SUPPORTED_PROXY_PROTOCOLS = ["http", "https", "socks", "socks5", "socks5h", "socks4"];
```
`server/proxy.js:11`

### 3. 请求时的代理处理流程

请求执行时，检查监控是否配置了 `proxy_id`：

```javascript
if (this.proxy_id) {
    const proxy = await R.load("proxy", this.proxy_id);

    if (proxy && proxy.active) {
        const { httpAgent, httpsAgent } = Proxy.createAgents(proxy, {
            httpsAgentOptions: httpsAgentOptions,
            httpAgentOptions: httpAgentOptions,
        });

        options.proxy = false;  // 禁用 axios 内置代理机制，使用自定义 agent
        options.httpAgent = httpAgent;
        options.httpsAgent = httpsAgent;
    }
}
```
`server/model/monitor.js:575-588`

**关键逻辑：**
1. 只有当 `this.proxy_id` 有值时才会尝试使用代理
2. 代理必须满足 `proxy.active === true` 才会生效
3. **只有当使用代理时**，才设置 `options.proxy = false` 并设置自定义 agent

### 4. 三种场景下的字段落值对比

根据代码逻辑分析，三种场景的最终 `options` 字段值如下：

| 场景 | proxy_id | proxy.active | options.proxy | options.httpAgent | options.httpsAgent | 实际请求方式 |
|------|----------|--------------|---------------|-------------------|--------------------|-------------|
| **场景 A：有代理且启用** | 有值 | `true` | `false` | 代理 Agent | 代理 Agent | 走代理 |
| **场景 B：无代理** | `null/undefined` | - | `undefined`（未设置） | 默认 `http.Agent` | 默认 `HttpsCookieAgent` | 直连 |
| **场景 C：有代理但禁用** | 有值 | `false` | `undefined`（未设置） | 默认 `http.Agent` | 默认 `HttpsCookieAgent` | 直连 |

**代码执行路径详解：**

```javascript
// 初始 options 不包含 proxy、httpAgent、httpsAgent
const options = {
    url: this.url,
    method: ...,
    timeout: ...,
    headers: ...,
    // 注意：没有 proxy, httpAgent, httpsAgent
};

// ========== 代理判断逻辑 ==========
if (this.proxy_id) {
    const proxy = await R.load("proxy", this.proxy_id);
    
    if (proxy && proxy.active) {
        // ========== 场景 A：有代理且启用 ==========
        options.proxy = false;           // 只有这里设置 proxy: false
        options.httpAgent = httpAgent;   // 代理 Agent
        options.httpsAgent = httpsAgent; // 代理 Agent
    }
    // else: proxy 不存在或 active=false，不做任何设置
    // 进入场景 C
}
// else: proxy_id 无值，不做任何设置
// 进入场景 B

// ========== 默认 Agent 创建 ==========
if (!options.httpAgent) {
    // 场景 B 和 C 会进入这里
    options.httpAgent = new http.Agent(httpAgentOptions);
}

if (!options.httpsAgent) {
    // 场景 B 和 C 会进入这里
    options.httpsAgent = new HttpsCookieAgent(httpsCookieAgentOptions);
}
```
`server/model/monitor.js:544-601`

**重要结论：**
1. **`options.proxy = false` 仅在"有代理且启用"时设置**，目的是禁用 axios 的内置代理机制（避免同时使用两套代理配置）
2. **"无代理"和"代理被禁用"两种场景最终选项完全相同**：都使用默认 Agent 直连目标服务器
3. **场景 B 和 C 的 `options.proxy` 是 `undefined`**，不是 `false`

### 5. 无代理时的默认 Agent

如果没有配置代理（或代理被禁用），会创建默认的 HTTP/HTTPS Agent：

```javascript
if (!options.httpAgent) {
    options.httpAgent = new http.Agent(httpAgentOptions);
}

if (!options.httpsAgent) {
    let jar = new CookieJar();
    let httpsCookieAgentOptions = {
        ...httpsAgentOptions,
        cookies: { jar },
    };
    options.httpsAgent = new HttpsCookieAgent(httpsCookieAgentOptions);
}
```
`server/model/monitor.js:590-601`

### 6. 代理 Agent 创建细节

`Proxy.createAgents` 方法负责创建不同协议的代理 Agent：

`server/proxy.js:91-158`

#### 6.1 代理 URL 组装（含认证）

```javascript
const proxyUrl = new URL(`${proxy.protocol}://${proxy.host}:${proxy.port}`);

if (proxy.auth) {
    proxyUrl.username = proxy.username;
    proxyUrl.password = proxy.password;
}
```
`server/proxy.js:103-108`

**代理认证信息会编码到 URL 中**，格式为：
`protocol://username:password@host:port`

#### 6.2 HTTP/HTTPS 协议代理

使用 `http-proxy-agent` 和 `https-proxy-agent`：

```javascript
case "http":
case "https":
    const HttpCookieProxyAgent = createCookieAgent(HttpProxyAgent);
    const HttpsCookieProxyAgent = createCookieAgent(HttpsProxyAgent);

    httpAgent = new HttpCookieProxyAgent(proxyUrl.toString(), {
        ...(httpAgentOptions || {}),
        ...proxyOptions,  // 包含 cookie jar
    });
    httpsAgent = new HttpsCookieProxyAgent(proxyUrl.toString(), {
        ...(httpsAgentOptions || {}),
        ...proxyOptions,
    });
    break;
```
`server/proxy.js:114-131`

#### 6.3 SOCKS 协议代理

使用 `socks-proxy-agent`，**HTTP 和 HTTPS 共用同一个 Agent**：

```javascript
case "socks":
case "socks5":
case "socks5h":
case "socks4":
    const SocksCookieProxyAgent = createCookieAgent(SocksProxyAgent);
    agent = new SocksCookieProxyAgent(proxyUrl.toString(), {
        ...httpAgentOptions,
        ...httpsAgentOptions,
        tls: {
            rejectUnauthorized: httpsAgentOptions.rejectUnauthorized,
        },
    });

    httpAgent = agent;
    httpsAgent = agent;
    break;
```
`server/proxy.js:132-148`

---

## 三、请求级配置 vs 全局默认代理：优先级分析

### 1. 前端创建监控时的默认代理逻辑

**只有在新建监控时**，如果用户没有手动选择代理，前端会自动应用默认代理：

```javascript
watch: {
    "$root.proxyList"() {
        if (this.isAdd) {  // 只有新增模式才生效
            if (this.$root.proxyList && !this.monitor.proxyId) {
                const proxy = this.$root.proxyList.find((proxy) => proxy.default);

                if (proxy) {
                    this.monitor.proxyId = proxy.id;
                }
            }
        }
    },
    // ...
}
```
`src/pages/EditMonitor.vue:3463-3473`

**触发条件：**
1. 处于新增监控模式（`isAdd === true`）
2. 代理列表已加载
3. 当前监控尚未配置 `proxyId`
4. 存在标记为 `default: true` 的代理

### 2. 后端请求执行时的逻辑

**后端只关心 `proxy_id` 字段是否有值**，不关心该值是用户手动选择的还是默认代理自动填充的：

```javascript
if (this.proxy_id) {
    const proxy = await R.load("proxy", this.proxy_id);
    if (proxy && proxy.active) {
        // 使用代理
    }
}
// 没有其他关于 "default" 代理的检查
```
`server/model/monitor.js:575-588`

### 3. 优先级总结

| 场景 | 行为 | proxy_id 值 | 是否使用代理 |
|------|------|-------------|--------------|
| 新建监控，用户手动选择代理 A | 使用用户选择的 | 代理 A 的 ID | 是 |
| 新建监控，用户选择"无代理" | 不使用代理 | null | 否 |
| 新建监控，用户不做选择，存在默认代理 | 自动应用默认代理 | 默认代理的 ID | 是 |
| 新建监控，用户不做选择，无默认代理 | 不使用代理 | null | 否 |
| 编辑现有监控，修改代理选择 | 使用新选择的 | 新代理 ID 或 null | 依新值而定 |
| 运行时执行请求 | 只看 proxy_id 字段 | 有值且 active | 是 |
| 运行时执行请求 | 只看 proxy_id 字段 | 无值或 inactive | 否 |

### 4. 默认代理的"默认"含义

**默认代理仅影响新建监控的初始值**，具体表现为：

1. **创建时**：新建监控如果不手动选择，会自动选中默认代理
2. **保存后**：一旦保存，`proxy_id` 就固化了，后续与"默认代理"概念解绑
3. **修改默认代理**：不会影响已创建的监控（除非手动修改它们）
4. **运行时**：后端完全不知道"默认代理"这个概念

### 5. 代理变更的影响

#### 场景 A：修改某个代理的 `default` 标记

- **影响**：只影响**未来**新建的监控
- **已存在的监控**：不受影响，它们的 `proxy_id` 保持不变

#### 场景 B：将某个代理设置为 `active: false`（禁用）

- **已配置该代理的监控**：请求时检查 `proxy.active` 为 false，会**绕过代理**直接请求
- 代码逻辑：
  ```javascript
  if (proxy && proxy.active) {  // active 必须为 true
      // 使用代理
  }
  // 不满足则跳过，走默认 agent（无代理）
  ```

#### 场景 C：删除一个代理

```javascript
static async delete(proxyID, userID) {
    // ...
    // Delete removed proxy from monitors if exists
    await R.exec("UPDATE monitor SET proxy_id = null WHERE proxy_id = ?", [proxyID]);
    // ...
}
```
`server/proxy.js:70-82`

- **已配置该代理的监控**：`proxy_id` 被置为 `null`，变为"无代理"
- **后续请求**：不再使用任何代理

---

## 四、完整请求选项组装流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     HTTP 监控请求组装流程                         │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 基础配置                                                       │
│     - url: this.url                                              │
│     - method: (this.method || "get").toLowerCase()              │
│     - timeout: this.timeout * 1000 (毫秒)                        │
│     - maxRedirects: this.maxredirects                            │
│     注意：初始 options 不含 proxy、httpAgent、httpsAgent          │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 认证处理（按优先级可能覆盖 headers）                           │
│     ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│     │ Basic Auth   │  │ OAuth2 CC    │  │ mTLS             │  │
│     │ auth_method  │  │ auth_method  │  │ auth_method      │  │
│     │ === "basic"  │  │ === "oauth2" │  │ === "mtls"       │  │
│     └──────────────┘  └──────────────┘  └──────────────────┘  │
│            │                   │                    │            │
│            ▼                   ▼                    ▼            │
│     Authorization:       Authorization:        设置 Agent 的    │
│     "Basic ..."          "Bearer token"        cert/ca/key      │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Headers 组装（展开优先级）                                     │
│     {                                                             │
│       Accept: "...",                          // 最低优先级       │
│       "Content-Type": contentType,            // 可被覆盖         │
│       ...basicAuthHeader,                      // Authorization   │
│       ...oauth2AuthHeader,                     // 可覆盖上面      │
│       ...JSON.parse(this.headers)              // 最高优先级       │
│     }                                                           │
│                                                                   │
│     ⚠️  风险警告：如果自定义 headers 包含 Authorization，         │
│         会静默覆盖 Basic Auth/OAuth2 生成的认证头！               │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. Agent 选项准备（证书校验 & IP 族）                            │
│     httpsAgentOptions: {                                          │
│       rejectUnauthorized: !this.getIgnoreTls(),  // 证书校验     │
│       family: agentFamily,                        // IPv4/IPv6   │
│       autoSelectFamily: true                      // 自动选择     │
│     }                                                             │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 代理处理分支（三种场景的选项不同）                             │
│                                                                  │
│     ┌──────────────────────────────────────────────────────┐    │
│     │ this.proxy_id 有值？                                  │    │
│     └──────────────────────────────────────────────────────┘    │
│              │                          │                         │
│              │ Yes                      │ No                      │
│              ▼                          ▼                         │
│     ┌─────────────────┐        ┌─────────────────────────┐     │
│     │ 加载 proxy 配置  │        │ 场景 B：无代理           │     │
│     │ 检查 active?     │        │                         │     │
│     └─────────────────┘        │ options.proxy: undefined │     │
│              │                  │ httpAgent: 默认          │     │
│              │                  │ httpsAgent: 默认         │     │
│              ▼                  └─────────────────────────┘     │
│     ┌─────────────────┐                                          │
│     │ proxy.active    │                                          │
│     │ === true ?      │                                          │
│     └─────────────────┘                                          │
│         │         │                                               │
│         │ Yes     │ No                                            │
│         ▼         ▼                                               │
│ ┌─────────────┐ ┌─────────────────────────┐                      │
│ │场景 A：     │ │ 场景 C：代理被禁用       │                      │
│ │有代理且启用 │ │                         │                      │
│ │             │ │ options.proxy: undefined│                      │
│ │options.proxy│ │ httpAgent: 默认         │                      │
│ │  = false    │ │ httpsAgent: 默认        │                      │
│ │httpAgent:   │ └─────────────────────────┘                      │
│ │  代理 Agent  │                                                    │
│ │httpsAgent:  │                                                    │
│ │  代理 Agent  │                                                    │
│ └─────────────┘                                                    │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 默认 Agent 兜底（场景 B 和 C 进入）                           │
│     if (!options.httpAgent) {                                    │
│         options.httpAgent = new http.Agent(...);                │
│     }                                                             │
│     if (!options.httpsAgent) {                                   │
│         options.httpsAgent = new HttpsCookieAgent(...);         │
│     }                                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、关键配置项对照表

| 配置项 | 存储字段 | 影响选项 | 默认值 | 说明 |
|--------|----------|----------|--------|------|
| URL | `url` | `options.url` | - | 目标地址 |
| HTTP 方法 | `method` | `options.method` | `"get"` | GET/POST/PUT 等 |
| 超时 | `timeout` | `options.timeout` (ms) | 动态计算 | 秒为单位 |
| 最大重定向 | `maxredirects` | `options.maxRedirects` | - | |
| 自定义 Headers | `headers` (JSON 字符串) | `options.headers` 展开 | `{}` | **最高优先级，可覆盖认证头** |
| 认证方式 | `auth_method` | 多种 | - | `basic`/`oauth2-cc`/`mtls` |
| Basic Auth 用户 | `basic_auth_user` | Authorization 头 | - | |
| Basic Auth 密码 | `basic_auth_pass` | Authorization 头 | - | |
| OAuth2 配置 | 多个字段 | 获取 token | - | Client Credentials |
| mTLS 证书 | `tlsCert` | Agent 选项 | - | Buffer 格式 |
| mTLS CA | `tlsCa` | Agent 选项 | - | |
| mTLS 私钥 | `tlsKey` | Agent 选项 | - | |
| 忽略 TLS 错误 | `ignoreTls` 相关 | `rejectUnauthorized` | `false` | 即默认校验证书 |
| IP 协议 | `ipFamily` | Agent `family` 选项 | 自动 | `ipv4`/`ipv6`/空 |
| 代理 | `proxy_id` | 创建代理 Agent | `null` | 关联 proxy 表 |
| 请求体 | `body` | `options.data` | - | |
| 请求体编码 | `httpBodyEncoding` | Content-Type | `"json"` | `json`/`form`/`xml` |
| 缓存破坏 | `cacheBust` | `options.params` | `false` | 添加随机 query 参数 |

---

## 六、代理配置优先级决策树

```
                    ┌─────────────────────────┐
                    │   开始：是否使用代理？   │
                    └─────────────────────────┘
                                │
                                ▼
              ┌─────────────────────────────────┐
              │   阶段 1: 监控创建时（前端）    │
              │   isAdd === true ?              │
              └─────────────────────────────────┘
                        │              │
                       Yes             No
                        │              │
                        ▼              ▼
              ┌──────────────┐   ┌──────────────┐
              │ proxyId 为空? │   │  使用已有    │
              └──────────────┘   │  proxy_id    │
                 │       │       └──────────────┘
                Yes      No
                 │       │
                 ▼       ▼
         ┌──────────┐  ┌──────────┐
         │ 有默认   │  │ 使用用户 │
         │ 代理吗？ │  │ 选择的   │
         └──────────┘  │ proxyId  │
            │    │     └──────────┘
           Yes   No
            │    │
            ▼    ▼
    ┌──────────┐ ┌──────────┐
    │ 设置为  │ │ 保持为空 │
    │ 默认代理 │ │ (无代理) │
    └──────────┘ └──────────┘
            │
            ▼
    ┌─────────────────────────────────────┐
    │   保存到数据库 monitor.proxy_id      │
    │   (此时与"默认代理"概念完全解绑)      │
    └─────────────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────┐
    │      阶段 2: 运行时请求（后端）           │
    │      每次 beat() 执行时检查              │
    └─────────────────────────────────────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │ this.proxy_id 有值吗？ │
         └────────────────────────┘
               │            │
              Yes          No
               │            │
               ▼            ▼
    ┌──────────────────┐  ┌────────────────┐
    │ 加载 proxy 配置   │  │ 场景 B：无代理  │
    │ FROM proxy 表    │  │ options.proxy: │
    └──────────────────┘  │ undefined      │
               │           │ httpAgent:    │
               ▼           │ 默认           │
    ┌────────────────────┐ │ httpsAgent:   │
    │ proxy.active ===   │ │ 默认           │
    │ true ?             │ └────────────────┘
    └────────────────────┘
           │            │
          Yes          No
           │            │
           ▼            ▼
    ┌────────────────┐ ┌──────────────────┐
    │ 场景 A：       │ │ 场景 C：          │
    │ 有代理且启用   │ │ 代理被禁用        │
    │                │ │                   │
    │ options.proxy: │ │ options.proxy:    │
    │ false          │ │ undefined         │
    │ httpAgent:     │ │ httpAgent:        │
    │ 代理 Agent     │ │ 默认              │
    │ httpsAgent:    │ │ httpsAgent:       │
    │ 代理 Agent     │ │ 默认              │
    └────────────────┘ └──────────────────┘
```

---

## 七、重要代码位置索引

| 功能 | 文件位置 | 行号范围 |
|------|----------|----------|
| HTTP 请求选项组装 | `server/model/monitor.js` | 544-565 |
| Headers 合并逻辑 | `server/model/monitor.js` | 549-555 |
| Basic Auth 处理 | `server/model/monitor.js` | 473-478 |
| OAuth2 处理 | `server/model/monitor.js` | 482-498 |
| mTLS 处理 | `server/model/monitor.js` | 603-613 |
| 证书校验配置 | `server/model/monitor.js` | 508-514 |
| 请求时代理判断 | `server/model/monitor.js` | 575-588 |
| 默认 Agent 创建 | `server/model/monitor.js` | 590-601 |
| 代理 Agent 工厂 | `server/proxy.js` | 91-158 |
| 代理 URL+认证组装 | `server/proxy.js` | 103-108 |
| 删除代理时级联 | `server/proxy.js` | 77-78 |
| 新建监控默认代理 | `src/pages/EditMonitor.vue` | 3463-3473 |

---

## 八、关键结论

### 关于请求组装

1. **Headers 采用展开覆盖**：用户自定义 headers 优先级最高，**可以静默覆盖** Basic Auth 或 OAuth2 生成的 `Authorization` 头，存在安全和调试风险
2. **多种认证方式**：Basic Auth、OAuth2、mTLS 三种认证是互斥的（通过 `auth_method` 选择）
3. **超时双重保险**：axios timeout + abort signal，后者比前者多 10 秒缓冲
4. **证书校验默认开启**：`ignoreTls` 默认为 false，即 `rejectUnauthorized = true`

### 关于代理优先级

1. **默认代理只影响新建**："默认代理"的概念**仅存在于前端创建监控的 UI 逻辑中**
2. **运行时不认识"默认"**：后端执行请求时，只看 `monitor.proxy_id` 是否有值，完全不关心 `proxy.default` 标记
3. **保存即固化**：一旦监控保存，`proxy_id` 就固定了，之后修改"默认代理"不会影响已存在的监控
4. **三种场景的 `options.proxy` 值不同**：
   - 有代理且启用：`options.proxy = false`
   - 无代理 / 代理被禁用：`options.proxy = undefined`（未设置）
5. **代理禁用 = 无代理**：如果监控配置了代理但该代理被标记为 `active: false`，请求会绕过代理直接发送（与"无代理"场景完全相同）
6. **删除代理 = 无代理**：删除代理时，所有使用该代理的监控的 `proxy_id` 会被置为 `null`

### 关于代理认证

1. **代理认证信息在 URL 中**：`protocol://username:password@host:port`
2. **与目标服务认证分开**：代理认证（proxy.username/password）和目标服务认证（Basic Auth/OAuth2/mTLS）是两个独立的概念，互不干扰

### 修正说明

本文档修正了原结论中关于 `options.proxy` 字段的不准确描述：

- ❌ 原错误描述：`options.proxy` 总是 `false`
- ✅ 正确描述：`options.proxy = false` **仅在"有代理且启用"时设置**，目的是禁用 axios 内置代理机制；"无代理"和"代理被禁用"场景下 `options.proxy` 为 `undefined`
