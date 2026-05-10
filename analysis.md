# Uptime Kuma 公开状态页分析报告

## 1. 私有监控如何被选择到公开状态页

### 1.1 核心选择机制

Uptime Kuma 采用"分组（Group）"作为监控从私有到公开的桥梁。监控本身没有直接的"公开/私有"属性，而是通过**分组的公开性**来控制：

- `group` 表中的 `public` 字段（BOOLEAN，默认值 0）决定该分组是否公开
- 监控通过 `monitor_group` 关联表被分配到分组中
- 只有属于 `public = 1` 分组的监控才会出现在公开状态页

### 1.2 数据模型关系

```
status_page (状态页)
    ↓ 1:N
group (分组)
    ↓ 1:N (通过 monitor_group 关联表)
monitor (监控)
```

关键表结构：

**status_page 表**：
- `id`, `slug`, `title`, `description`, `published`（是否发布）等

**group 表**（`db/old_migrations/patch-group-table.sql`）：
- `id`: 主键
- `name`: 分组名称
- `public`: 是否公开（BOOLEAN，默认 0）
- `status_page_id`: 关联的状态页 ID
- `weight`: 排序权重

**monitor_group 关联表**：
- `monitor_id`: 监控 ID
- `group_id`: 分组 ID
- `weight`: 监控在分组内的排序权重
- `send_url`: 是否在公开页显示监控 URL
- `custom_url`: 自定义公开 URL（覆盖监控原始 URL）

### 1.3 选择逻辑

在 `server/model/status_page.js:326` 中获取公开分组：
```javascript
const list = await R.find("group", " public = 1 AND status_page_id = ? ORDER BY weight ", [statusPage.id]);
```

在 `server/model/group.js:33-46` 中获取分组内的监控：
```javascript
async getMonitorList() {
    return R.convertToBeans(
        "monitor",
        await R.getAll(
            `
            SELECT monitor.*, monitor_group.send_url, monitor_group.custom_url FROM monitor, monitor_group
            WHERE monitor.id = monitor_group.monitor_id
            AND group_id = ?
            ORDER BY monitor_group.weight
        `,
            [this.id]
        )
    );
}
```

### 1.4 管理流程（前端编辑模式）

在 `src/pages/StatusPage.vue` 中：
1. 登录用户进入编辑模式（通过 `?edit` 查询参数或点击编辑按钮）
2. 从 `sortedMonitorList` 计算属性选择未在公开列表中的监控
3. 将监控添加到指定分组
4. 保存时通过 socket 事件 `saveStatusPage` 提交到后端

在 `server/socket-handlers/status-page-socket-handler.js:292-433` 中处理保存：
- 更新状态页配置
- 保存公开分组列表
- 建立/更新 `monitor_group` 关联关系
- 删除不在列表中的分组

---

## 2. 监控分组与排序机制

### 2.1 分组机制

**分组层级**：
- 每个状态页可以包含多个公开分组
- 每个分组可以包含多个监控
- 分组可以被折叠/展开（通过 URL 查询参数 `collapse`）

**分组创建/编辑**：
- 前端：`src/components/PublicGroupList.vue` 中的 `addGroup` 方法
- 后端：`server/socket-handlers/status-page-socket-handler.js` 中的 `saveStatusPage` 事件处理

### 2.2 排序机制

**分组排序**：
- 依据 `group.weight` 字段排序
- 在获取公开分组时按 `ORDER BY weight` 排序
- 代码位置：`server/model/status_page.js:326`

**监控在分组内的排序**：
- 依据 `monitor_group.weight` 字段排序
- 在获取分组内监控时按 `ORDER BY monitor_group.weight` 排序
- 代码位置：`server/model/group.js:41`

**排序权重管理**：
- 保存时后端自动分配递增的权重值
- 分组权重：`server/socket-handlers/status-page-socket-handler.js:355-371` 中的 `groupOrder`
- 监控权重：`server/socket-handlers/status-page-socket-handler.js:377-394` 中的 `monitorOrder`

**前端拖拽排序**：
- 使用 `vuedraggable` 组件实现拖拽
- 拖拽后顺序反映在数据中，保存时后端重新分配权重

---

## 3. 历史状态的裁剪机制

### 3.1 公开数据裁剪（API 层面）

**心跳数据裁剪**：
- **限制数量**：最近 100 条心跳记录
- 代码位置：`server/routers/status-page-router.js:86-94`
```javascript
let list = await R.getAll(
    `
        SELECT * FROM heartbeat
        WHERE monitor_id = ?
        ORDER BY time DESC
        LIMIT 100
    `,
    [monitorID]
);
```

**心跳数据字段裁剪**：
- 公开心跳数据只保留必要字段
- 代码位置：`server/model/heartbeat.js:19-26`
```javascript
toPublicJSON() {
    return {
        status: this.status,
        time: this.time,
        msg: "", // 隐藏错误消息！
        ping: this.ping,
    };
}
```

**监控数据裁剪**：
- 公开监控数据只保留必要字段
- 代码位置：`server/model/monitor.js:85-108`
```javascript
async toPublicJSON(showTags = false, certExpiry = false) {
    let obj = {
        id: this.id,
        name: this.name,
        sendUrl: this.sendUrl,
        type: this.type,
    };

    if (this.sendUrl) {
        obj.url = this.customUrl ?? this.url;
    }

    if (showTags) {
        obj.tags = await this.getTags();
    }

    if (certExpiry) {
        // 证书到期信息（可选）
    }

    return obj;
}
```

### 3.2 历史数据自动清理（数据库层面）

**数据保留策略**：
- 默认保留 365 天数据
- 可通过设置 `keepDataPeriodDays` 配置
- 代码位置：`server/jobs/clear-old-data.js:7-21`

**清理任务**：
- 定时清理 `heartbeat` 表中的旧数据
- 清理 `stat_daily` 表中的旧数据
- 代码位置：`server/jobs/clear-old-data.js:38-57`

**Uptime 统计数据保留**：
- 分钟级数据：保留 24 小时
- 小时级数据：保留 30 天
- 天级数据：保留 365 天
- 代码位置：`server/uptime-calculator.js:33-47, 63-64`

### 3.3 展示层面的裁剪

**只显示最后心跳**：
- 状态页可配置 `showOnlyLastHeartbeat` 选项
- 启用后只显示监控当前状态，不显示完整心跳条
- 代码位置：
  - 后端保存：`server/socket-handlers/status-page-socket-handler.js:337`
  - 后端公开 JSON：`server/model/status_page.js:452, 479`
  - 前端展示：`src/components/PublicGroupList.vue:74-77`

**事件历史分页**：
- 事件历史采用游标分页
- 每页 `INCIDENT_PAGE_SIZE` 条记录
- 代码位置：`server/model/status_page.js:512-551`

---

## 4. 未登录用户可见的数据边界

### 4.1 公开 API 端点

所有公开数据通过以下 API 端点获取，这些端点**不需要认证**：

| 端点 | 缓存 | 数据范围 |
|------|------|---------|
| `GET /api/status-page/:slug` | 5分钟 | 状态页配置、公开分组列表、活跃事件、维护计划 |
| `GET /api/status-page/heartbeat/:slug` | 1分钟 | 公开监控的心跳数据（最近100条）、24小时可用率 |
| `GET /api/status-page/:slug/incident-history` | 5分钟 | 事件历史（分页） |
| `GET /api/status-page/:slug/badge` | 5分钟 | 整体状态徽章（SVG） |
| `GET /api/status-page/:slug/manifest.json` | 1440分钟 | PWA manifest |

### 4.2 可见数据字段详解

**状态页配置** (`server/model/status_page.js:462-482`)：
```javascript
{
    slug,
    title,
    description,
    icon,
    autoRefreshInterval,
    theme,
    published,
    showTags,
    customCSS,
    footerText,
    showPoweredBy,
    analyticsId,
    analyticsScriptUrl,
    analyticsType,
    showCertificateExpiry,  // 是否显示证书到期
    showOnlyLastHeartbeat,   // 是否只显示最后心跳
    rssTitle,
}
```

**分组数据** (`server/model/group.js:13-27`)：
```javascript
{
    id: this.id,
    name: this.name,
    weight: this.weight,
    monitorList: [...],  // 分组内监控列表
}
```

**监控数据** (`server/model/monitor.js:85-108`)：
- 基础信息：`id`, `name`, `sendUrl`, `type`
- URL（可选）：仅当 `sendUrl = true` 时显示，可使用 `customUrl` 覆盖
- 标签（可选）：仅当状态页 `showTags = true` 时显示
- 证书到期（可选）：仅当状态页 `showCertificateExpiry = true` 时显示

**心跳数据** (`server/model/heartbeat.js:19-26`)：
- `status`: 状态（0=DOWN, 1=UP, 2=PENDING, 3=MAINTENANCE）
- `time`: 时间戳
- `msg`: **空字符串**（隐藏错误消息细节）
- `ping`: 响应时间（毫秒）

**事件数据** (`server/model/incident.js:21-33`)：
```javascript
{
    id,
    style,
    title,
    content,
    pin: !!this.pin,
    active: !!this.active,
    createdDate,
    lastUpdatedDate,
    status_page_id,
}
```

**维护计划数据** (`server/model/maintenance.js:15-94`)：
- 仅返回**当前进行中的**维护计划
- 包含：`id`, `title`, `description`, `strategy`, `dateRange`, `timeRange`, `timeslotList`, `timezone`, `status` 等

**可用率数据**：
- 仅返回最近 **24小时** 的可用率百分比
- 代码位置：`server/routers/status-page-router.js:99-100`

### 4.3 不可见的数据（安全边界）

未登录用户**无法访问**以下数据：

**监控敏感信息**：
- 监控描述 (`description`)
- 监控详细配置（HTTP 方法、请求头、认证信息等）
- 通知配置
- 代理配置
- 敏感字段如 `hostname`, `port`, `interval`, `timeout`, `keyword`, `dns_resolve_server` 等

**心跳详细信息**：
- 错误消息（`msg` 字段被置空）
- 重试次数
- 响应内容
- 重要性标记
- 持续时间

**用户/系统信息**：
- 其他用户的监控（除非添加到公开分组）
- 系统设置
- 日志信息
- API 密钥
- 2FA 相关信息

### 4.4 缓存机制

公开 API 使用 `apicache` 中间件缓存：
- 状态页数据：5 分钟缓存
- 心跳数据：1 分钟缓存
- 徽章：5 分钟缓存
- Manifest：1440 分钟（1 天）缓存

缓存清除时机：
- 保存状态页配置时
- 代码位置：`server/socket-handlers/status-page-socket-handler.js:419`

---

## 5. 发布态（published=0）对未登录用户的影响分析

### 5.1 状态页发布态的数据库模型

`status_page` 表包含 `published` 字段（`db/old_migrations/patch-status-page.sql:11`）：
```sql
[published] BOOLEAN NOT NULL DEFAULT 1
```

- 默认值为 `1`（已发布）
- 该字段在数据库初始化时从设置 `statusPagePublished` 继承（`server/database.js:632`）
- 但在状态页保存时，该字段**不会被更新**（`server/socket-handlers/status-page-socket-handler.js:329` 被注释掉）

### 5.2 各公开端点的发布态校验分析

以下是所有公开 API 端点的读取路径和发布态校验情况：

#### 5.2.1 状态页详情页（HTML 渲染）

**读取路径**：
1. `GET /status/:slug` → `server/routers/status-page-router.js:16-20`
2. 调用 `StatusPage.handleStatusPageResponse()` → `server/model/status_page.js:57-71`
3. 查询条件：`R.findOne("status_page", " slug = ? ", [slug])`

**校验逻辑**：
- ❌ **无发布态校验**，仅按 `slug` 查询
- 如果状态页存在，直接返回渲染后的 HTML
- 如果状态页不存在，返回 404

#### 5.2.2 状态页数据 API

**读取路径**：
1. `GET /api/status-page/:slug` → `server/routers/status-page-router.js:39-60`
2. 查询条件：`R.findOne("status_page", " slug = ? ", [slug])`
3. 调用 `StatusPage.getStatusPageData(statusPage)` → `server/model/status_page.js:309-340`

**校验逻辑**：
- ❌ **无发布态校验**
- 只要状态页存在，返回完整的公开数据：
  - 状态页配置（包含 `published: false` 字段）
  - 公开分组列表
  - 活跃事件列表
  - 进行中的维护计划

#### 5.2.3 心跳数据 API

**读取路径**：
1. `GET /api/status-page/heartbeat/:slug` → `server/routers/status-page-router.js:64-110`
2. 注释声称：`// Can fetch only if published`（第 63 行）
3. 实际代码：
   - `StatusPage.slugToID(slug)` → 仅通过 slug 获取 ID
   - 通过 `status_page_id` 查询公开分组和监控

**校验逻辑**：
- ❌ **无发布态校验**（与注释矛盾！）
- 只要知道 slug，就可以获取：
  - 最近 100 条心跳记录
  - 24 小时可用率百分比

#### 5.2.4 事件历史 API

**读取路径**：
1. `GET /api/status-page/:slug/incident-history` → `server/routers/status-page-router.js:145-167`
2. `StatusPage.slugToID(slug)` → 仅通过 slug 获取 ID
3. `StatusPage.getIncidentHistory(statusPageID, cursor, true)`

**校验逻辑**：
- ❌ **无发布态校验**
- 只要知道 slug，就可以获取完整的事件历史（分页）

#### 5.2.5 状态徽章 API

**读取路径**：
1. `GET /api/status-page/:slug/badge` → `server/routers/status-page-router.js:170-262`
2. `StatusPage.slugToID(slug)` → 仅通过 slug 获取 ID
3. 查询该状态页下的公开分组和监控状态

**校验逻辑**：
- ❌ **无发布态校验**
- 只要知道 slug，就可以获取：
  - 整体状态（Up/Down/Degraded/Maintenance/N/A）
  - SVG 格式的徽章图片

#### 5.2.6 RSS 订阅

**读取路径**：
1. `GET /status/:slug/rss` → `server/routers/status-page-router.js:22-26`
2. `StatusPage.handleStatusPageRSSResponse()` → `server/model/status_page.js:38-48`
3. 查询条件：`R.findOne("status_page", " slug = ? ", [slug])`
4. 调用 `StatusPage.renderRSS()` → `server/model/status_page.js:79-109`
5. 调用 `StatusPage.getRSSPageData()` → `server/model/status_page.js:266-302`

**校验逻辑**：
- ❌ **无发布态校验**
- 只要知道 slug，就可以获取：
  - 当前状态描述
  - 所有 DOWN 状态的监控列表（包含时间戳）

#### 5.2.7 PWA Manifest

**读取路径**：
1. `GET /api/status-page/:slug/manifest.json` → `server/routers/status-page-router.js:113-143`
2. 查询条件：`R.findOne("status_page", " slug = ? ", [slug])`

**校验逻辑**：
- ❌ **无发布态校验**
- 返回 PWA 应用清单（标题、图标等）

### 5.3 发布态校验汇总表

| 端点 | 发布态校验 | 仅按 slug 查询 | 未登录用户可访问 |
|------|-----------|---------------|-----------------|
| `GET /status/:slug`（HTML） | ❌ 无 | ✅ 是 | ✅ 可访问完整页面 |
| `GET /api/status-page/:slug` | ❌ 无 | ✅ 是 | ✅ 配置、分组、事件、维护 |
| `GET /api/status-page/heartbeat/:slug` | ❌ 无 | ✅ 是 | ✅ 心跳、24h 可用率 |
| `GET /api/status-page/:slug/incident-history` | ❌ 无 | ✅ 是 | ✅ 事件历史 |
| `GET /api/status-page/:slug/badge` | ❌ 无 | ✅ 是 | ✅ 状态徽章 |
| `GET /status/:slug/rss` | ❌ 无 | ✅ 是 | ✅ RSS 订阅 |
| `GET /api/status-page/:slug/manifest.json` | ❌ 无 | ✅ 是 | ✅ PWA 清单 |

### 5.4 对公开数据边界的影响

#### 5.4.1 发布态字段形同虚设

由于所有公开端点都没有校验 `published` 字段，导致：

1. **未发布的状态页仍然完全可访问**：只要知道 slug，未登录用户可以访问所有公开数据
2. **前端 `isPublished` 计算属性未使用**：`src/pages/StatusPage.vue:772-774` 定义了 `isPublished`，但代码中没有任何地方实际使用它
3. **数据泄露风险**：如果管理员设置 `published=0` 希望隐藏状态页，实际上**完全没有效果**

#### 5.4.2 与公开分组机制的关系

发布态校验缺失与公开分组机制形成对比：

| 机制 | 实际生效 | 作用 |
|------|---------|------|
| `group.public = 0/1` | ✅ 生效 | 控制哪些监控出现在状态页 |
| `status_page.published = 0/1` | ❌ 不生效 | 仅作为数据字段存储，不影响访问 |

这意味着：
- 即使状态页设置为 `published=0`，公开分组中的监控数据仍然完全暴露
- 唯一真正有效的隐私控制是：不将监控添加到公开分组

#### 5.4.3 安全隐患

1. **隐藏状态页可被枚举**：如果攻击者猜测或获取到 slug，即使状态页设置为"未发布"，数据仍然可访问
2. **注释与代码不一致**：`status-page-router.js:63` 注释声称 "Can fetch only if published"，但实际代码没有实现，可能误导开发者
3. **数据范围超出预期**：管理员可能认为设置 `published=0` 可以隐藏数据，但实际上：
   - 状态页 HTML 完全可访问
   - 所有公开 API 端点都可调用
   - RSS 订阅正常工作
   - 状态徽章可嵌入外部网站

### 5.5 `published=0` 在真实系统中的触发路径

经过代码分析，`published=0` 状态在真实系统中可能通过以下路径出现：

#### 5.5.1 路径 1：数据库迁移（从旧版本升级）

**触发条件**：从 Uptime Kuma 1.13.0 之前的版本升级

**代码路径**：`server/database.js:610-666` 的 `migrateNewStatusPage()` 方法

```javascript
statusPage.published = !!(await setting("statusPagePublished"));
```

**关键点**：
- 旧版本使用 `setting` 表存储状态页配置，包含 `statusPagePublished` 设置
- 迁移时从旧设置继承 `published` 字段值
- 如果旧版本中 `statusPagePublished = false`，迁移后 `status_page.published = 0`
- 迁移只执行一次（检查 `slug = 'default'` 是否已存在）

#### 5.5.2 路径 2：手工修改数据库

**触发条件**：管理员直接操作数据库

**常见操作**：
```sql
UPDATE status_page SET published = 0 WHERE slug = 'my-page';
```

**关键点**：
- 数据库有 `published` 字段，允许直接修改
- 没有 UI 界面可以修改此字段（保存时被注释掉）
- 管理员可能通过数据库管理工具（如 SQLite Browser、phpMyAdmin）修改

#### 5.5.3 路径 3：数据库导入/恢复

**触发条件**：从包含 `published=0` 的数据库备份恢复

**关键点**：
- 如果备份文件中的状态页记录有 `published=0`
- 恢复后该值会被保留
- 没有导入后的校验或修正逻辑

#### 5.5.4 路径 4：第三方脚本/API 操作

**触发条件**：通过自定义脚本直接写入数据库

**关键点**：
- 可能使用 ORM（RedBean）直接操作 `status_page` bean
- 可能使用原始 SQL 执行 UPDATE
- 绕过了正常的保存流程（`saveStatusPage` 事件）

### 5.6 后台保存对 `published` 字段的影响

#### 5.6.1 状态页保存流程

**代码路径**：`server/socket-handlers/status-page-socket-handler.js:292-433`

```javascript
socket.on("saveStatusPage", async (slug, config, publicGroupList, callback) => {
    // ...
    // statusPage.published = !!config.published;        // 第 329 行：被注释掉！
    // statusPage.search_engine_index = !!config.search_engine_index;  // 第 330 行：被注释掉！
    // statusPage.password = config.password;           // 第 331 行：被注释掉！
    // ...
    await R.store(statusPage);
});
```

#### 5.6.2 保存行为分析

| 字段 | 保存时是否更新 | 注释状态 |
|------|--------------|---------|
| `title` | ✅ 更新 | 正常代码 |
| `description` | ✅ 更新 | 正常代码 |
| `theme` | ✅ 更新 | 正常代码 |
| `published` | ❌ 不更新 | 第 329 行被注释 |
| `search_engine_index` | ❌ 不更新 | 第 330 行被注释 |
| `password` | ❌ 不更新 | 第 331 行被注释 |

**关键发现**：
- 通过后台 UI 保存状态页时，`published` 字段**不会被修改**
- 如果状态页已经是 `published=0`，保存后**仍然是** `published=0`
- 后台保存不会"修复"或"重置"这个字段
- 这意味着 `published=0` 一旦设置，会**永久保留**，除非直接操作数据库

#### 5.6.3 新增状态页的默认值

**代码路径**：`server/socket-handlers/status-page-socket-handler.js:436-464`

```javascript
socket.on("addStatusPage", async (title, slug, callback) => {
    let statusPage = R.dispense("status_page");
    statusPage.slug = slug;
    statusPage.title = title;
    statusPage.theme = "auto";
    statusPage.icon = "";
    statusPage.autoRefreshInterval = 300;
    // 没有显式设置 published，使用数据库默认值
    await R.store(statusPage);
});
```

**关键点**：
- 新增状态页时**没有显式设置** `published` 字段
- 依赖数据库表定义的默认值：`DEFAULT 1`
- 所以新增状态页默认是 `published=1`

### 5.7 未登录可见边界的实际风险评估

#### 5.7.1 风险等级评估框架

我们从以下维度评估风险：

| 维度 | 说明 |
|------|------|
| **触发概率** | `published=0` 在真实环境中出现的可能性 |
| **数据暴露范围** | 未登录用户可以访问的数据量 |
| **攻击前置条件** | 攻击者需要了解或猜测什么信息 |
| **利用难度** | 攻击的技术门槛 |

#### 5.7.2 各触发路径的风险评估

##### 路径 1：数据库迁移

**风险分析**：
- **触发概率**：低
  - 只有从 1.13.0 之前版本升级的用户会遇到
  - 且旧版本中 `statusPagePublished` 需要是 `false`
  - 大多数用户使用默认设置（`true`）
- **数据暴露范围**：高
  - 所有公开分组的监控数据完全暴露
  - 心跳数据、事件历史、RSS、徽章都可访问
- **攻击前置条件**：
  1. 必须知道或猜测状态页的 `slug`
  2. 默认状态页的 slug 是 `default`（容易猜测）
  3. 其他状态页的 slug 需要枚举或获取
- **利用难度**：极低
  - 只需发送标准 HTTP 请求
  - 不需要任何认证或特殊工具

**风险等级**：⭐⭐⭐⭐（中高风险）

##### 路径 2：手工修改数据库

**风险分析**：
- **触发概率**：中
  - 管理员可能为了"隐藏"状态页而手工修改数据库
  - 误以为设置 `published=0` 可以保护数据
- **数据暴露范围**：高（与路径 1 相同）
- **攻击前置条件**：
  1. 必须知道 slug
  2. 如果管理员为了隐藏而设置了特殊 slug，可能降低风险
- **利用难度**：极低

**关键矛盾**：
- 管理员**意图**：隐藏状态页
- **实际效果**：数据仍然完全可访问
- **风险放大**：管理员可能产生虚假安全感，反而将敏感监控添加到公开分组

**风险等级**：⭐⭐⭐⭐⭐（高风险）

##### 路径 3：数据库导入/恢复

**风险分析**：
- **触发概率**：低到中
  - 取决于备份文件的来源
  - 如果备份来自旧版本或被篡改的数据库，风险增加
- **数据暴露范围**：高
- **攻击前置条件**：知道 slug
- **利用难度**：极低

**风险等级**：⭐⭐⭐（中风险）

##### 路径 4：第三方脚本/API

**风险分析**：
- **触发概率**：低
  - 只有自定义部署才会使用
  - 需要刻意操作
- **数据暴露范围**：高
- **攻击前置条件**：知道 slug
- **利用难度**：极低

**风险等级**：⭐⭐⭐（中风险）

#### 5.7.3 综合风险评估

**整体风险等级**：⭐⭐⭐⭐（中高风险）

**核心问题**：

1. **虚假安全感**：
   - 数据库有 `published` 字段，暗示存在访问控制
   - 代码注释声称心跳 API "Can fetch only if published"
   - 前端有 `isPublished` 计算属性
   - 但实际**没有任何校验**

2. **管理员预期与实际行为不一致**：
   - 预期：`published=0` 应该隐藏状态页
   - 实际：数据完全可访问
   - 后果：管理员可能将敏感监控添加到公开分组

3. **默认状态页 slug 是 `default`**：
   - 非常容易猜测
   - 攻击者可以尝试 `/status/default`
   - 如果 `published=0`，所有数据立即暴露

#### 5.7.4 风险缓解建议

**立即缓解**：
1. **不要依赖 `published` 字段**：这是当前最有效的缓解方式
2. **通过分组控制**：不希望公开的监控，不要添加到公开分组
3. **检查数据库**：执行 `SELECT slug, published FROM status_page;` 确认所有状态页的 `published` 状态

**长期修复（需要代码修改）**：
1. 在所有公开端点添加 `published=1` 校验
2. 或者完全移除 `published` 字段，避免混淆
3. 实现真正的密码保护功能

### 5.8 未登录用户获取 slug 的真实渠道分析

#### 5.8.1 术语定义

在深入分析之前，需要明确两个关键概念：

| 概念 | 定义 | 风险等级 |
|------|------|---------|
| **可猜测** | 攻击者需要通过猜测、枚举或社会工程获取 slug | 中 |
| **可枚举** | 存在公开接口或机制可以直接获取所有 slug 列表 | 高 |

#### 5.8.2 公开端点分析

经过代码审查，以下是所有可能暴露 slug 的公开端点：

##### 端点 1：`GET /api/entry-page`

**代码路径**：`server/routers/api-router.js:28-45`

```javascript
router.get("/api/entry-page", async (request, response) => {
    let result = {};
    let hostname = request.hostname;
    // ...
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

**访问权限**：✅ 未登录用户可访问

**暴露内容**：
- 如果访问域名匹配 `domainMappingList`：返回对应的 `statusPageSlug`
- 如果入口页面是状态页（`entryPage` 以 `statusPage-` 开头）：返回完整的 `entryPage` 字符串，包含 slug

**示例响应**：
```json
// 域名匹配情况
{"type": "statusPageMatchedDomain", "statusPageSlug": "my-status-page"}

// 入口页面是状态页
{"type": "entryPage", "entryPage": "statusPage-default"}
```

**评估结论**：
- ❌ **可枚举单条**：针对特定域名
- ❌ **不可枚举全部**：无法获取所有状态页列表
- ⚠️ **可猜测辅助**：如果入口页面是状态页，可以直接获取其 slug

##### 端点 2：`GET /status/:slug` 及相关公开 API

**代码路径**：
- HTML：`server/routers/status-page-router.js:16-20`
- 数据 API：`server/routers/status-page-router.js:39-60`
- 心跳 API：`server/routers/status-page-router.js:64-110`
- 事件历史 API：`server/routers/status-page-router.js:145-167`
- 徽章 API：`server/routers/status-page-router.js:170-262`
- RSS：`server/routers/status-page-router.js:22-26`
- Manifest：`server/routers/status-page-router.js:113-143`

**访问权限**：✅ 未登录用户可访问

**404 行为分析**（`server/util-server.js:710-727`）：

```javascript
module.exports.sendHttpError = (res, msg = "") => {
    // ...
    } else if (msg.toLowerCase().includes("not found")) {
        res.status(404).json({
            status: "fail",
            msg: msg,
        });
    } else {
        res.status(403).json({
            // ...
        });
    }
};
```

**状态码差异**：
| 情况 | HTTP 状态码 | 响应内容 |
|------|------------|---------|
| slug 存在 | 200 | 正常响应（HTML/JSON/XML） |
| slug 不存在 | 404 | `{"status": "fail", "msg": "... Not Found"}` |

**评估结论**：
- ❌ **不可枚举**：没有直接返回列表的接口
- ⚠️ **可蛮力枚举**：通过 404 vs 200 的差异可以暴力枚举可能的 slug
- 📊 **枚举效率**：取决于 slug 命名模式和长度

#### 5.8.3 默认 slug 分析

**默认状态页**：
- 数据库迁移时自动创建，slug 为 `"default"`（`server/database.js:627`）
- 迁移代码：`statusPage.slug = "default";`

**入口页面配置**：
- 入口页面如果是状态页，格式为 `"statusPage-{slug}"`（`server/server.js:263`）
- 示例：`"statusPage-default"`、`"statusPage-my-custom-page"`

**评估结论**：
- ⚠️ **`default` 是高度可猜测的**：几乎所有从旧版本升级的实例都有这个 slug
- ⚠️ **入口页面泄漏**：如果入口页面是状态页，`/api/entry-page` 直接返回完整 slug

#### 5.8.4 域名映射机制

**代码路径**：
- 域名列表维护：`server/model/status_page.js:376-420`
- 入口检测：`server/server.js:258-262`、`server/routers/api-router.js:37-43`

**工作原理**：
1. 管理员可以为状态页配置域名列表（`domainNameList`）
2. 系统在启动和保存时加载 `domainMappingList`
3. 当请求的 `hostname` 匹配映射表中的域名时：
   - 直接渲染对应状态页
   - `/api/entry-page` 返回 `statusPageSlug`

**示例**：
```javascript
// 如果管理员配置了
// 状态页 "my-status-page" 绑定域名 "status.example.com"

// 访问 http://status.example.com/api/entry-page
// 返回：{"type": "statusPageMatchedDomain", "statusPageSlug": "my-status-page"}
```

**评估结论**：
- ⚠️ **域名即泄露**：攻击者只要知道绑定的域名，就可以获取 slug
- ⚠️ **多域名绑定**：一个状态页可以绑定多个域名，增加了泄露渠道

#### 5.8.5 可枚举 vs 可猜测的区分

让我创建一个完整的矩阵来区分这两种情况：

| 攻击类型 | 描述 | 前置条件 | 成功率 | 风险等级 |
|---------|------|---------|--------|---------|
| **直接可枚举** | 有公开 API 返回完整 slug 列表 | 无 | 100% | ⭐⭐⭐⭐⭐ |
| **部分可枚举** | 有公开 API 返回部分 slug（如域名映射） | 知道域名 | 100% | ⭐⭐⭐⭐ |
| **可猜测（已知）** | 通过 `/api/entry-page` 获取入口页面 slug | 无 | 100% | ⭐⭐⭐⭐ |
| **可猜测（默认）** | 尝试 `default`、`status`、`main` 等常见 slug | 无 | 取决于配置 | ⭐⭐⭐ |
| **可猜测（命名模式）** | 管理员使用公司名、产品名作为 slug | 了解目标组织 | 高 | ⭐⭐⭐ |
| **蛮力枚举** | 通过 404 vs 200 差异枚举所有可能 | 时间和计算资源 | 取决于复杂度 | ⭐⭐⭐ |
| **域名枚举** | 通过 DNS 枚举发现绑定的状态页域名 | 知道主域名 | 中等 | ⭐⭐⭐ |
| **社会工程** | 通过文档、截图、分享链接获取 slug | 内部信息 | 高 | ⭐⭐⭐⭐ |

**关键发现**：
- ❌ **不存在直接可枚举所有 slug 的公开接口**
- ⚠️ **`/api/entry-page` 可枚举单条**（域名匹配或入口页面）
- ⚠️ **`default` 是高度可猜测的**
- ⚠️ **404 行为允许蛮力枚举**

#### 5.8.6 404 行为的可观测差异

让我详细分析 404 响应是否存在可用于枚举的特征：

**公开端点的 404 响应**：

| 端点 | slug 存在 | slug 不存在 | 可观测差异 |
|------|----------|------------|-----------|
| `GET /status/:slug`（HTML） | 200 + Vue 应用 | 404 + `{"status":"fail","msg":"Status Page Not Found"}` | ✅ 明显差异 |
| `GET /api/status-page/:slug` | 200 + JSON 数据 | 404 + `{"status":"fail","msg":"Status Page Not Found"}` | ✅ 明显差异 |
| `GET /api/status-page/heartbeat/:slug` | 200 + JSON 数组 | 404 + `{"status":"fail","msg":"Not Found"}` | ✅ 明显差异 |
| `GET /api/status-page/:slug/incident-history` | 200 + JSON 分页 | 404 + `{"status":"fail","msg":"Status Page Not Found"}` | ✅ 明显差异 |
| `GET /api/status-page/:slug/badge` | 200 + SVG 图片 | 404 + `{"status":"fail","msg":"Not Found"}` | ✅ 明显差异 |
| `GET /status/:slug/rss` | 200 + XML | 404 + `{"status":"fail","msg":"Status Page Not Found"}` | ✅ 明显差异 |
| `GET /api/status-page/:slug/manifest.json` | 200 + JSON | 404 + `{"status":"fail","msg":"Not Found"}` | ✅ 明显差异 |

**响应时间分析**：
- 存在的 slug：需要数据库查询 → 可能略慢
- 不存在的 slug：数据库查询返回空 → 可能略快
- 差异可能很小，但理论上存在时序攻击空间

**缓存影响**：
- 公开 API 使用 `apicache` 中间件
- 存在的 slug 会被缓存
- 不存在的 slug 可能不会被缓存
- 这可能导致响应时间差异更加明显

**评估结论**：
- ⚠️ **完全可蛮力枚举**：所有公开端点都有明显的 404 vs 200 差异
- ⚠️ **缓存机制可能放大差异**
- 但枚举所有可能的 slug 仍然需要时间和计算资源

#### 5.8.7 状态页列表的获取渠道

**`statusPageList` 的发送机制**（`server/model/status_page.js:353-371`）：

```javascript
static async sendStatusPageList(socket) {
    checkLogin(socket);  // 🔒 需要登录！
    // ...
    const list = await R.find("status_page", " ORDER BY id ");
    // ...
    io.to(socket.userID).emit("statusPageList", result);  // 🔒 只发送给特定用户
}
```

**关键发现**：
- ✅ **需要登录**：`checkLogin(socket)` 确保只有登录用户可以获取
- ✅ **定向发送**：`io.to(socket.userID).emit(...)` 只发送给特定用户
- ❌ **未登录用户无法获取**完整状态页列表

**评估结论**：
- ✅ **不存在公开的状态页列表接口**
- 完整的 slug 枚举只能通过蛮力攻击或信息收集

### 5.9 重算风险等级与前置条件

基于以上分析，让我重新评估风险等级：

#### 5.9.1 新的风险评估框架

**新增维度**：

| 维度 | 说明 | 权重 |
|------|------|------|
| **触发概率** | `published=0` 在真实环境中出现的可能性 | 25% |
| **slug 获取难度** | 攻击者获取 slug 的难易程度 | 35% |
| **数据暴露范围** | 未登录用户可以访问的数据量 | 25% |
| **利用难度** | 攻击的技术门槛 | 15% |

#### 5.9.2 slug 获取难度的细分评估

| 场景 | slug 获取难度 | 评分（1-10） |
|------|--------------|-------------|
| 入口页面是状态页（通过 `/api/entry-page`） | 直接获取 | 1 |
| 域名映射到状态页（通过 `/api/entry-page`） | 直接获取（需要知道域名） | 2 |
| 默认 slug `default` | 高度可猜测 | 2 |
| 管理员使用公司名/产品名作为 slug | 中等可猜测 | 4 |
| 管理员使用随机/复杂 slug | 需要蛮力枚举 | 7 |
| 完全随机的长 slug | 蛮力枚举不现实 | 9 |

#### 5.9.3 各触发路径的重算风险等级

##### 路径 1：数据库迁移（旧版本升级）

**触发条件**：从 1.13.0 之前版本升级，且旧设置 `statusPagePublished=false`

**风险重算**：
- **触发概率**：低（只有特定升级路径）
- **slug 获取难度**：
  - 如果入口页面是状态页 → 直接获取（评分 1）
  - 如果不是 → 需要猜测 `default`（评分 2）
- **数据暴露范围**：高（所有公开分组数据）
- **利用难度**：极低

**综合风险等级**：
- 如果入口页面是状态页：⭐⭐⭐⭐⭐（高风险）
- 如果不是：⭐⭐⭐⭐（中高风险）

##### 路径 2：手工修改数据库

**触发条件**：管理员通过数据库工具直接修改 `published=0`

**风险重算**：
- **触发概率**：中
- **slug 获取难度**：
  - 如果入口页面是状态页 → 直接获取（评分 1）
  - 如果有域名映射 → 直接获取（需要知道域名，评分 2）
  - 如果是默认状态页 → 猜测 `default`（评分 2）
  - 如果是自定义 slug → 取决于命名（评分 2-7）
- **数据暴露范围**：高
- **利用难度**：极低
- **风险放大**：管理员可能产生虚假安全感

**综合风险等级**：
- 如果入口页面是状态页：⭐⭐⭐⭐⭐（高风险）
- 如果有域名映射：⭐⭐⭐⭐⭐（高风险）
- 如果是默认状态页：⭐⭐⭐⭐⭐（高风险）
- 如果是复杂自定义 slug：⭐⭐⭐（中风险）

##### 路径 3：数据库导入/恢复

**触发条件**：从包含 `published=0` 的备份恢复

**风险重算**：
- **触发概率**：低到中
- **slug 获取难度**：取决于备份来源
- **数据暴露范围**：高
- **利用难度**：极低

**综合风险等级**：⭐⭐⭐（中风险）

##### 路径 4：第三方脚本

**触发条件**：自定义脚本直接操作数据库

**风险重算**：
- **触发概率**：低
- **slug 获取难度**：取决于脚本用途
- **数据暴露范围**：高
- **利用难度**：极低

**综合风险等级**：⭐⭐⭐（中风险）

#### 5.9.4 综合风险矩阵

让我创建一个综合风险评估矩阵：

| 场景 | 触发概率 | slug 获取难度 | 数据暴露 | 利用难度 | 综合风险 |
|------|---------|--------------|---------|---------|---------|
| 入口页面 = 状态页 + `published=0` | 低-中 | **直接获取** | 高 | 极低 | ⭐⭐⭐⭐⭐ |
| 域名映射 + `published=0` | 低-中 | **直接获取**（需知域名） | 高 | 极低 | ⭐⭐⭐⭐⭐ |
| 默认状态页 `default` + `published=0` | 中 | **高度可猜测** | 高 | 极低 | ⭐⭐⭐⭐⭐ |
| 自定义简单 slug + `published=0` | 中 | **中等可猜测** | 高 | 低 | ⭐⭐⭐⭐ |
| 自定义复杂 slug + `published=0` | 中 | **需蛮力枚举** | 高 | 低-中 | ⭐⭐⭐ |
| 无公开分组 + `published=0` | 中 | 同上 | **无数据** | 极低 | ⭐（低风险） |

#### 5.9.5 关键前置条件

要成功利用 `published=0` 的漏洞，攻击者需要满足以下前置条件：

**必要条件**（AND 关系）：
1. ✅ 状态页的 `published` 字段确实是 `0`
2. ✅ 攻击者能够获取到状态页的 `slug`
3. ✅ 状态页至少有一个公开分组（`group.public = 1`）
4. ✅ 公开分组中至少有一个监控

**可选条件**（OR 关系，用于获取 slug）：
- 🔍 入口页面是状态页（`/api/entry-page` 返回 `statusPage-{slug}`）
- 🔍 知道绑定的域名（`/api/entry-page` 返回 `statusPageSlug`）
- 🔍 猜测到常见 slug（如 `default`、`status`、`main`）
- 🔍 能够通过社会工程获取 slug
- 🔍 能够实施蛮力枚举攻击

**最危险的组合**：
```
published=0 
AND (entryPage=statusPage-* OR 有域名映射 OR slug=default)
AND 有公开分组
AND 分组中有监控
```

这种组合的风险等级：⭐⭐⭐⭐⭐（极高风险）

#### 5.9.6 风险缓解的优先级

基于新的分析，重新排序风险缓解措施的优先级：

| 优先级 | 措施 | 说明 | 实施难度 |
|-------|------|------|---------|
| **P0（紧急）** | 检查并确保所有状态页 `published=1` | 直接消除触发条件 | 低 |
| **P0（紧急）** | 不将敏感监控添加到公开分组 | 减少数据暴露范围 | 低 |
| **P1（高）** | 避免使用简单/可猜测的 slug | 增加获取难度 | 低 |
| **P1（高）** | 避免将状态页设为入口页面 | 关闭直接获取渠道 | 低 |
| **P1（高）** | 避免使用域名映射 | 关闭直接获取渠道 | 中 |
| **P2（中）** | 修改代码添加 `published=1` 校验 | 长期修复 | 中 |
| **P3（低）** | 实现真正的密码保护 | 长期增强 | 高 |

### 5.10 相关未实现功能

除了 `published` 字段外，`status_page` 表还有其他字段也未完全实现：

1. **`search_engine_index`**：数据库字段存在，但没有搜索引擎索引控制逻辑
2. **`password`**：数据库字段存在，但没有密码保护的访问控制逻辑

这些字段在 `server/socket-handlers/status-page-socket-handler.js:329-332` 中都被注释掉，不会在保存状态页时被更新。

---

## 6. 关键代码位置汇总

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 获取状态页公开数据 | `server/model/status_page.js` | 304-340 |
| 状态页配置公开 JSON | `server/model/status_page.js` | 462-482 |
| 监控公开 JSON | `server/model/monitor.js` | 85-108 |
| 心跳公开 JSON | `server/model/heartbeat.js` | 19-26 |
| 分组公开 JSON | `server/model/group.js` | 13-27 |
| 状态页路由（所有公开端点） | `server/routers/status-page-router.js` | 1-264 |
| 心跳数据 API | `server/routers/status-page-router.js` | 64-110 |
| 状态页数据 API | `server/routers/status-page-router.js` | 39-60 |
| 保存状态页（published 被注释） | `server/socket-handlers/status-page-socket-handler.js` | 292-433 |
| 数据清理任务 | `server/jobs/clear-old-data.js` | 13-60 |
| 可用率计算器 | `server/uptime-calculator.js` | 1-150 |
| 前端公开数据混入 | `src/mixins/public.js` | 1-55 |
| 公开分组列表组件 | `src/components/PublicGroupList.vue` | 1-408 |
| 状态页前端（isPublished 未使用） | `src/pages/StatusPage.vue` | 772-774 |

---

## 7. 总结

### 7.1 已实现的安全机制

Uptime Kuma 的公开状态页设计采用了**多层裁剪和隔离机制**：

1. **分组隔离**：通过公开分组控制哪些监控可以被公开
2. **数据裁剪**：所有公开数据都通过 `toPublicJSON()` 方法进行字段级过滤
3. **数量限制**：心跳数据限制为最近 100 条
4. **历史清理**：定时任务自动清理过期数据
5. **认证分离**：公开 API 和私有 API 完全分离
6. **缓存保护**：公开数据使用缓存，减少数据库压力

### 7.2 发现的问题

经过深入分析，发现了以下重要问题：

#### 7.2.1 `published` 字段形同虚设

- 所有公开 API 端点都没有校验 `published` 字段
- 仅按 `slug` 查询，未发布的状态页仍然完全可访问
- 注释与代码不一致（`status-page-router.js:63` 声称 "Can fetch only if published"）

#### 7.2.2 `published=0` 在真实系统中的触发路径

| 路径 | 触发条件 | 概率 | 风险等级 |
|------|---------|------|---------|
| 数据库迁移 | 从 1.13.0 之前版本升级，且旧设置 `statusPagePublished=false` | 低 | ⭐⭐⭐⭐ |
| 手工修改数据库 | 管理员通过数据库工具直接修改 `published=0` | 中 | ⭐⭐⭐⭐⭐ |
| 数据库导入/恢复 | 从包含 `published=0` 的备份恢复 | 低-中 | ⭐⭐⭐ |
| 第三方脚本 | 自定义脚本直接操作数据库 | 低 | ⭐⭐⭐ |

#### 7.2.3 后台保存不会覆盖 `published` 字段

**关键发现**：
- 通过后台 UI 保存状态页时，`published` 字段**不会被修改**
- 保存代码中相关行被注释掉（`server/socket-handlers/status-page-socket-handler.js:329-331`）
- 如果状态页已经是 `published=0`，保存后**仍然是** `published=0`
- 这意味着 `published=0` 一旦设置，会**永久保留**，除非直接操作数据库

#### 7.2.4 相关功能未实现

- `search_engine_index`：数据库字段存在，但无实际逻辑
- `password`：数据库字段存在，但无密码保护逻辑
- `isPublished`：前端计算属性定义了，但未被使用

#### 7.2.5 对公开数据边界的影响

- 管理员设置 `published=0` 无法隐藏状态页
- 唯一真正有效的隐私控制是：不将监控添加到公开分组
- 存在数据泄露风险：隐藏状态页可通过 slug 枚举访问
- 最大风险是**虚假安全感**：管理员可能误以为 `published=0` 可以保护数据

### 7.3 风险评估总结

#### 7.3.1 整体风险等级

**⭐⭐⭐⭐（中高风险）**

#### 7.3.2 风险矩阵

| 维度 | 评估 |
|------|------|
| 触发概率 | 低到中（取决于升级路径和管理员操作） |
| 数据暴露范围 | 高（所有公开分组数据完全可访问） |
| 攻击前置条件 | 知道或猜测 slug（`default` 很容易猜测） |
| 利用难度 | 极低（标准 HTTP 请求即可） |

#### 7.3.3 风险缓解建议

**立即缓解**：
1. **不要依赖 `published` 字段**：这是当前最有效的缓解方式
2. **通过分组控制**：不希望公开的监控，不要添加到公开分组
3. **检查数据库**：执行 `SELECT slug, published FROM status_page;` 确认所有状态页的 `published` 状态

**长期修复（需要代码修改）**：
1. 在所有公开端点添加 `published=1` 校验
2. 或者完全移除 `published` 字段，避免混淆
3. 实现真正的密码保护功能

### 7.4 最终结论

Uptime Kuma 的公开数据保护主要依赖于**分组的公开性**（`group.public` 字段），而不是状态页的发布态（`status_page.published`）。这意味着：

- 如果不希望数据被公开，**不要将监控添加到公开分组**
- 仅设置 `published=0` **无法阻止未登录用户访问数据**
- 这是一个需要注意的设计特性（或潜在 bug）
- **管理员不应依赖 `published` 字段作为安全控制**
