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

## 5. 关键代码位置汇总

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 获取状态页公开数据 | `server/model/status_page.js` | 304-340 |
| 状态页配置公开 JSON | `server/model/status_page.js` | 462-482 |
| 监控公开 JSON | `server/model/monitor.js` | 85-108 |
| 心跳公开 JSON | `server/model/heartbeat.js` | 19-26 |
| 分组公开 JSON | `server/model/group.js` | 13-27 |
| 心跳数据 API | `server/routers/status-page-router.js` | 64-110 |
| 状态页数据 API | `server/routers/status-page-router.js` | 39-60 |
| 保存状态页 | `server/socket-handlers/status-page-socket-handler.js` | 292-433 |
| 数据清理任务 | `server/jobs/clear-old-data.js` | 13-60 |
| 可用率计算器 | `server/uptime-calculator.js` | 1-150 |
| 前端公开数据混入 | `src/mixins/public.js` | 1-55 |
| 公开分组列表组件 | `src/components/PublicGroupList.vue` | 1-408 |
| 状态页前端 | `src/pages/StatusPage.vue` | 1-1100+ |

---

## 6. 总结

Uptime Kuma 的公开状态页设计采用了**多层裁剪和隔离机制**：

1. **分组隔离**：通过公开分组控制哪些监控可以被公开
2. **数据裁剪**：所有公开数据都通过 `toPublicJSON()` 方法进行字段级过滤
3. **数量限制**：心跳数据限制为最近 100 条
4. **历史清理**：定时任务自动清理过期数据
5. **认证分离**：公开 API 和私有 API 完全分离
6. **缓存保护**：公开数据使用缓存，减少数据库压力

这些机制确保了未登录用户只能看到必要的信息，同时保护了系统的敏感数据和用户隐私。
