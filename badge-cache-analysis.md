# Uptime Kuma 状态徽章系统分析报告

## 1. 各类徽章的生成链路与 5 分钟缓存入口

### 1.1 徽章类型与路由概览

Uptime Kuma 支持两类徽章，所有徽章端点统一使用 5 分钟服务器端缓存：

#### 1.1.1 状态页面整体徽章
- **路由**: `/api/status-page/:slug/badge`
- **文件**: `server/routers/status-page-router.js:170`
- **缓存入口**: `cache("5 minutes")` - 位于路由定义的中间件位置

#### 1.1.2 单个监控器徽章
共有 6 种单个监控器徽章，全部定义在 `server/routers/api-router.js` 中：

| 徽章类型 | 路由 | 行号 | 缓存入口 |
|---------|------|------|---------|
| 状态徽章 | `/api/badge/:id/status` | 148 | `cache("5 minutes")` |
| 正常运行时间徽章 | `/api/badge/:id/uptime/:duration?` | 221 | `cache("5 minutes")` |
| Ping 时间徽章 | `/api/badge/:id/ping/:duration?` | 285 | `cache("5 minutes")` |
| 平均响应时间徽章 | `/api/badge/:id/avg-response/:duration?` | 351 | `cache("5 minutes")` |
| 证书过期徽章 | `/api/badge/:id/cert-exp` | 424 | `cache("5 minutes")` |
| 响应徽章 | `/api/badge/:id/response` | 507 | `cache("5 minutes")` |

### 1.2 缓存系统架构

所有徽章使用相同的缓存系统，基于自定义的 `apicache` 模块：

#### 1.2.1 缓存模块结构
- **主文件**: `server/modules/apicache/index.js:1-12`
- **核心实现**: `server/modules/apicache/apicache.js`
- **内存缓存**: `server/modules/apicache/memory-cache.js`

#### 1.2.2 缓存配置
在 `server/modules/apicache/index.js:3-10` 中定义了缓存选项：

```javascript
apicache.options({
    headerBlacklist: ["cache-control"],
    headers: {
        "cache-control": "no-cache",
    },
});
```

**关键配置**:
- 黑名单 `cache-control` 头
- 强制设置 `cache-control: no-cache` 响应头
- 这意味着浏览器不会缓存徽章，但服务器端会缓存 5 分钟

### 1.3 徽章生成链路

#### 1.3.1 状态页面徽章生成链路

**代码位置**: `server/routers/status-page-router.js:170-262`

**生成流程**:
1. **缓存检查**: `cache("5 minutes")` 中间件首先检查是否有缓存
2. **获取状态页 ID**: 通过 slug 调用 `StatusPage.slugToID(slug)` (第 174 行)
3. **查询公开监控器**: SQL 查询获取该状态页面下所有 `public = 1` 的监控器 ID (第 185-193 行)
4. **状态聚合**:
   - 遍历所有监控器，获取最新心跳记录 (第 199-209 行)
   - 检查每个监控器的状态：
     - 状态 3: 维护模式 (Maintenance)
     - 状态 2: 忽略 (Ignored)
     - 状态 1: 正常 (Up)
     - 状态 0: 故障 (Down)
5. **徽章状态决策** (第 229-252 行):
   - 所有正常且无维护: 显示 "Up" (第 239-242 行)
   - 有故障但有正常: 显示 "Degraded" (第 243-246 行)
   - 全部故障: 显示 "Down" (第 247-250 行)
   - 有维护: 显示 "Maintenance" (第 235-238 行)
   - 无公开监控器: 显示 "N/A" (第 229-233 行)
6. **生成 SVG**: 使用 `makeBadge(badgeValues)` 生成 SVG 图像 (第 255 行)
7. **缓存存储**: apicache 自动缓存响应 5 分钟

#### 1.3.2 单个监控器徽章生成链路（以状态徽章为例）

**代码位置**: `server/routers/api-router.js:148-219`

**生成流程**:
1. **缓存检查**: `cache("5 minutes")` 中间件首先检查是否有缓存
2. **参数解析**: 从查询字符串解析自定义参数（颜色、标签、样式等）(第 151-163 行)
3. **输入验证**: 监控器 ID 必须是有效整数 (第 166-169 行)
4. **获取状态**: 
   - 调用 `Monitor.getPreviousHeartbeat(requestedMonitorId)` 获取最新心跳记录 (第 180 行)
   - 支持通过 `value` 参数覆盖状态（仅用于演示）(第 170-171 行, 181 行)
5. **构建徽章值** (第 183-208 行):
   - 根据状态设置颜色和消息
   - 支持自定义标签
6. **生成 SVG**: 使用 `makeBadge(badgeValues)` 生成 SVG 图像 (第 212 行)
7. **缓存存储**: apicache 自动缓存响应 5 分钟

#### 1.3.3 其他单个监控器徽章的特殊处理

**正常运行时间徽章** (`api-router.js:221-283`):
- 使用 `UptimeCalculator` 计算指定时间段的正常运行时间百分比 (第 257-258 行)
- 精度控制: 限制显示 4 位有效数字 (第 261 行): `const cleanUptime = (uptime * 100).toPrecision(4)`

**Ping 时间徽章** (`api-router.js:285-349`):
- 使用 `UptimeCalculator` 计算平均延迟 (第 317-318 行)
- 取整为整数毫秒 (第 328 行): `const avgPingValue = parseInt(overrideValue ?? avgPing)`

**平均响应时间徽章** (`api-router.js:351-422`):
- 直接通过 SQL 查询计算平均值 (第 378-390 行)
- 查询条件明确包含 `public = 1` (第 385 行)

**证书过期徽章** (`api-router.js:424-505`):
- 查询 `monitor_tls_info` 表获取证书信息 (第 461 行)
- 根据剩余天数设置颜色 (第 477-483 行)

**响应徽章** (`api-router.js:507-565`):
- 从最新心跳获取 ping 值 (第 538 行, 546 行)

---

## 2. 状态页徽章与单监控徽章在公开访问条件和跨域处理上的差异

### 2.1 跨域处理差异

#### 2.1.1 状态页面徽章的跨域处理

**代码位置**: `server/routers/status-page-router.js:171`

```javascript
router.get("/api/status-page/:slug/badge", cache("5 minutes"), async (request, response) => {
    allowDevAllOrigin(response);
    // ...
});
```

**函数定义**: `server/util-server.js:614-618`

```javascript
exports.allowDevAllOrigin = (res) => {
    if (process.env.NODE_ENV === "development") {
        exports.allowAllOrigin(res);
    }
};
```

**行为分析**:
- **生产环境**: 不设置任何 CORS 头
- **开发环境**: 设置 `Access-Control-Allow-Origin: *` 等 CORS 头
- **实际效果**: 生产环境下，状态页面徽章只能在同源页面中嵌入使用

#### 2.1.2 单个监控器徽章的跨域处理

**代码位置**: `server/routers/api-router.js:149, 222, 286, 352, 425, 508`

所有 6 种单个监控器徽章都使用 `allowAllOrigin(response)`：

```javascript
router.get("/api/badge/:id/status", cache("5 minutes"), async (request, response) => {
    allowAllOrigin(response);
    // ...
});

router.get("/api/badge/:id/uptime/:duration?", cache("5 minutes"), async (request, response) => {
    allowAllOrigin(response);
    // ...
});

// ... 其他 4 个徽章端点同样使用 allowAllOrigin
```

**函数定义**: `server/util-server.js:625-629`

```javascript
exports.allowAllOrigin = (res) => {
    res.header("Access-Control-Allow-Origin", "*");
    res.header("Access-Control-Allow-Methods", "GET, PUT, POST, DELETE, OPTIONS");
    res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
};
```

**行为分析**:
- **所有环境**: 始终设置 `Access-Control-Allow-Origin: *`
- **实际效果**: 单个监控器徽章可以嵌入到任何网站中，不受同源策略限制

#### 2.1.3 差异总结

| 对比项 | 状态页面徽章 | 单个监控器徽章 |
|--------|-------------|---------------|
| 使用函数 | `allowDevAllOrigin` | `allowAllOrigin` |
| 生产环境 CORS | 无（受同源策略限制） | `*`（允许所有来源） |
| 开发环境 CORS | `*` | `*` |
| 可嵌入外部网站 | 否（生产环境） | 是 |

### 2.2 公开访问条件差异

#### 2.2.1 状态页面徽章的公开访问条件

**代码位置**: `server/routers/status-page-router.js:185-193`

```javascript
let monitorIDList = await R.getCol(
    `
    SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
    WHERE monitor_group.group_id = \`group\`.id
    AND public = 1
    AND \`group\`.status_page_id = ?
`,
    [statusPageID]
);
```

**访问条件分析**:
1. **依赖状态页 slug**: 通过 slug 查找状态页面
2. **自动过滤私有监控器**: SQL 查询明确包含 `AND public = 1` 条件
3. **无显式检查函数**: 不像单监控徽章那样调用 `isMonitorPublic()` 函数
4. **聚合计算**: 只计算公开监控器的状态，私有监控器完全不参与状态计算
5. **无公开监控器处理**: 如果没有公开监控器，返回 "N/A" 灰色徽章 (第 229-233 行)

#### 2.2.2 单个监控器徽章的公开访问条件

**代码位置**: `server/routers/api-router.js:626-637` - 检查函数定义

```javascript
async function isMonitorPublic(monitorID) {
    let publicMonitor = await R.getRow(
        `
            SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
            WHERE monitor_group.group_id = \`group\`.id
            AND monitor_group.monitor_id = ?
            AND public = 1
        `,
        [monitorID]
    );
    return !!publicMonitor;
}
```

**各个徽章端点的调用**:
- 状态徽章: `api-router.js:171` - `const publicMonitor = await isMonitorPublic(requestedMonitorId);`
- 正常运行时间徽章: `api-router.js:249` - 同上
- Ping 时间徽章: `api-router.js:315` - 同上
- 证书过期徽章: `api-router.js:452` - 同上
- 响应徽章: `api-router.js:529` - 同上

**访问条件分析**:
1. **显式调用检查函数**: 所有端点都调用 `isMonitorPublic()` 函数
2. **依赖组的公开属性**: 检查监控器是否属于某个 `public = 1` 的组
3. **非公开处理**: 非公开监控器统一返回 "N/A" 灰色徽章，不暴露任何状态信息
4. **平均响应时间徽章的特殊处理**: `api-router.js:378-390` 的 SQL 查询也包含 `AND public = 1` 条件

#### 2.2.3 差异总结

| 对比项 | 状态页面徽章 | 单个监控器徽章 |
|--------|-------------|---------------|
| 检查方式 | SQL 查询中内嵌 `public = 1` 条件 | 显式调用 `isMonitorPublic()` 函数 |
| 检查对象 | 该状态页下的所有组 | 指定监控器所在的组 |
| 非公开处理 | 私有监控器不参与计算，只计算公开的 | 整个徽章返回 "N/A" |
| 访问标识 | 通过 slug 访问 | 通过 monitor ID 访问 |

---

## 3. 公开展示时实际暴露/隐藏了哪些状态信息

### 3.1 状态页面徽章暴露的信息

**代码位置**: `server/routers/status-page-router.js:227-258`

#### 3.1.1 实际暴露的信息

1. **聚合状态** (5 种可能):
   - `"Up"` - 所有公开监控器正常
   - `"Degraded"` - 部分正常部分故障
   - `"Down"` - 所有公开监控器故障
   - `"Maintenance"` - 有监控器处于维护模式
   - `"N/A"` - 无公开监控器或状态页不存在

2. **可选自定义标签**: 通过 `label` 查询参数自定义（第 236、240、244、248 行）

3. **可配置颜色**:
   - `upColor` - 正常状态颜色，默认绿色 `#66c20a`
   - `downColor` - 故障状态颜色，默认红色 `#c2290a`
   - `partialColor` - 降级状态颜色，硬编码为 `#F6BE00`
   - `maintenanceColor` - 维护状态颜色，硬编码为 `#808080`

4. **样式**: 通过 `style` 查询参数自定义，默认 `"flat"`

#### 3.1.2 隐藏的信息

1. **具体监控器状态**:
   - 不显示具体哪个监控器故障
   - 不显示故障监控器的数量
   - 不显示正常监控器的数量

2. **私有监控器信息**:
   - 私有监控器完全不参与状态计算
   - 不暴露私有监控器的存在

3. **详细状态信息**:
   - 不显示具体心跳时间
   - 不显示具体响应时间
   - 不显示具体错误信息

4. **配置信息**:
   - 不显示状态页面的完整配置
   - 不显示监控器的任何配置

### 3.2 单个监控器徽章暴露的信息

#### 3.2.1 状态徽章 (`api-router.js:148-219`)

**暴露的信息**:
1. **当前状态** (4 种可能):
   - `"Up"` - 正常
   - `"Down"` - 故障
   - `"Pending"` - 待定（重试中）
   - `"Maintenance"` - 维护模式
   - `"N/A"` - 非公开或不存在

2. **可配置标签和颜色**:
   - 自定义状态文本: `upLabel`, `downLabel`, `pendingLabel`, `maintenanceLabel`
   - 自定义颜色: `upColor`, `downColor`, `pendingColor`, `maintenanceColor`
   - 自定义标签: `label`（默认 "Status"）

**隐藏的信息**:
- 不显示具体心跳时间
- 不显示响应时间
- 不显示错误原因
- 不显示监控器配置

#### 3.2.2 正常运行时间徽章 (`api-router.js:221-283`)

**暴露的信息**:
1. **正常运行时间百分比**: 限制 4 位有效数字 (第 261 行)
   - 例如: `99.99%`, `98.5%`

2. **时间段标识**: 标签中显示时间段
   - 默认格式: `Uptime (24h)`
   - 可通过 `labelPrefix`, `label`, `labelSuffix` 自定义

3. **动态颜色**:
   - 可自定义固定颜色
   - 默认根据百分比计算颜色 (第 264 行): `percentageToColor(uptime)`

**隐藏的信息**:
- 不显示具体故障时间点
- 不显示故障次数
- 不显示故障持续时间
- 不显示波动详情

#### 3.2.3 Ping 时间徽章 (`api-router.js:285-349`)

**暴露的信息**:
1. **平均延迟**: 整数毫秒 (第 328 行: `parseInt()`)
   - 例如: `45ms`

2. **时间段标识**: 标签中显示时间段
   - 默认格式: `Avg. Ping (24h)`

**隐藏的信息**:
- 不显示延迟波动
- 不显示最大/最小延迟
- 不显示丢包率

#### 3.2.4 平均响应时间徽章 (`api-router.js:351-422`)

**暴露的信息**:
1. **平均响应时间**: 整数毫秒
   - 例如: `120ms`

2. **时间段标识**: 标签中显示时间段
   - 默认格式: `Avg. Response (24h)`

**特殊处理**:
- SQL 查询本身就包含 `public = 1` 条件 (第 385 行)
- 如果结果为 0，返回 "N/A" (第 394-398 行)

**隐藏的信息**:
- 与 Ping 徽章相同

#### 3.2.5 证书过期徽章 (`api-router.js:424-505`)

**暴露的信息**:
1. **证书状态**:
   - `"No/Bad Cert"` - 无证书或证书无效
   - `"Bad Cert"` - 证书验证失败
   - 剩余天数或过期日期

2. **分级颜色**:
   - 剩余天数 > `warnDays`: 绿色 (`upColor`)
   - `downDays` < 剩余天数 <= `warnDays`: 黄色 (`warnColor`)
   - 剩余天数 <= `downDays`: 红色 (`downColor`)

**隐藏的信息**:
- 不显示证书详细信息（颁发者、序列号等）
- 不显示证书链信息
- 不显示完整的证书内容

#### 3.2.6 响应徽章 (`api-router.js:507-565`)

**暴露的信息**:
1. **当前响应时间**: 最新心跳的 ping 值，整数毫秒

**隐藏的信息**:
- 只显示最新值，不显示历史趋势
- 不显示状态（Up/Down）

### 3.3 统一的隐藏机制

#### 3.3.1 非公开监控器的信息隐藏

所有徽章端点对非公开监控器统一处理：

**代码模式**（以状态徽章为例，`api-router.js:174-178`）:

```javascript
if (!publicMonitor) {
    badgeValues.message = "N/A";
    badgeValues.color = badgeConstants.naColor;
}
```

**隐藏效果**:
- 返回灰色的 "N/A" 徽章
- 不暴露任何状态信息
- 不确认监控器是否存在
- 不确认监控器的任何属性

#### 3.3.2 数据精度控制

1. **正常运行时间**: 4 位有效数字 (`api-router.js:261`)
   ```javascript
   const cleanUptime = (uptime * 100).toPrecision(4);
   ```

2. **时间值**: 取整为整数 (`api-router.js:328, 475, 546`)
   ```javascript
   const avgPingValue = parseInt(overrideValue ?? avgPing);
   ```

#### 3.3.3 敏感配置隐藏

所有徽章都不暴露：
- 监控器 URL、端口、路径
- 认证信息（用户名、密码、Token）
- 监控间隔、超时设置
- 重试策略
- 通知配置
- 标签信息

### 3.4 信息暴露总结

| 徽章类型 | 暴露的核心信息 | 隐藏的关键信息 |
|---------|--------------|---------------|
| 状态页面徽章 | 聚合状态（Up/Down/Degraded/Maintenance/N/A） | 具体监控器状态、私有监控器信息、详细配置 |
| 状态徽章 | 当前状态（Up/Down/Pending/Maintenance） | 心跳时间、错误原因、监控配置 |
| 正常运行时间徽章 | 指定时间段的正常运行百分比（4位精度） | 故障时间点、故障次数、波动详情 |
| Ping 徽章 | 平均延迟（整数毫秒） | 最大/最小延迟、丢包率、波动详情 |
| 平均响应时间徽章 | 平均响应时间（整数毫秒） | 与 Ping 徽章相同 |
| 证书过期徽章 | 剩余天数/过期日期、状态分级 | 证书详细信息、证书链 |
| 响应徽章 | 最新响应时间 | 状态信息、历史趋势 |

---

## 结论

Uptime Kuma 的徽章系统在设计上充分考虑了安全性和隐私保护：

1. **跨域策略差异明显**: 状态页面徽章在生产环境不允许跨域（保护状态页），而单个监控器徽章始终允许跨域（便于嵌入）
2. **公开访问条件严格**: 所有徽章都基于组的 `public` 属性，非公开监控器不暴露任何信息
3. **信息最小化原则**: 只暴露必要的聚合数据，隐藏详细历史、配置信息和错误详情
4. **统一的 5 分钟缓存**: 所有徽章使用相同的缓存策略，平衡了实时性和性能
