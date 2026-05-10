# Uptime-Kuma 维护窗口配置与调度机制分析

## 概述

Uptime-Kuma 的维护窗口（Maintenance Window）功能允许用户在指定时间内暂停特定监控任务的告警，避免计划内维护操作（如服务器升级、数据库迁移等）产生误报。本文档深入分析维护窗口的配置、调度、监控暂停机制以及前端展示。

---

## 一、维护窗口配置

### 1.1 数据模型

维护窗口的数据模型定义在 `server/model/maintenance.js` 中，核心字段包括：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | number | 维护窗口ID |
| `title` | string | 维护标题 |
| `description` | string | 维护描述（支持 Markdown） |
| `strategy` | string | 调度策略 |
| `active` | boolean | 是否启用 |
| `start_date` | datetime | 开始日期 |
| `end_date` | datetime | 结束日期 |
| `start_time` | string | 开始时间（HH:mm） |
| `end_time` | string | 结束时间（HH:mm） |
| `weekdays` | JSON | 适用的星期几（用于 recurring-weekday） |
| `days_of_month` | JSON | 适用的月中日期（用于 recurring-day-of-month） |
| `interval_day` | number | 间隔天数（用于 recurring-interval） |
| `cron` | string | Cron 表达式（用于 cron 策略） |
| `duration` | number | 持续时间（秒） |
| `timezone` | string | 时区 |
| `last_start_date` | datetime | 上次开始时间 |
| `user_id` | number | 创建者用户ID |

### 1.2 调度策略

维护窗口支持以下 6 种调度策略（`EditMaintenance.vue:110-123`）：

1. **manual** - 手动模式：持续处于维护状态，由用户手动暂停/恢复
2. **single** - 单次维护：在指定的开始和结束时间执行
3. **cron** - Cron 表达式：灵活的 Cron 调度
4. **recurring-interval** - 间隔重复：每隔 N 天在指定时间窗口执行
5. **recurring-weekday** - 星期重复：在指定的星期几的时间窗口执行
6. **recurring-day-of-month** - 月中日期重复：在每月指定日期的时间窗口执行

### 1.3 配置关系

维护窗口通过中间表与监控和状态页关联：

- **monitor_maintenance** 表：维护窗口与监控的多对多关系
- **maintenance_status_page** 表：维护窗口与状态页的多对多关系

---

## 二、调度机制

### 2.1 核心调度器

维护窗口使用 [Croner](https://github.com/hexagon/croner) 库进行调度。调度逻辑在 `Maintenance.run()` 方法中实现（`maintenance.js:212-340`）。

#### 2.1.1 不同策略的调度实现

**单次维护（single）策略：**
```javascript
// maintenance.js:233-238
if (this.strategy === "single") {
    this.beanMeta.job = new Cron(this.start_date, { timezone: await this.getTimezone() }, () => {
        log.info("maintenance", "Maintenance id: " + this.id + " is under maintenance now");
        UptimeKumaServer.getInstance().sendMaintenanceListByUserID(this.user_id);
        apicache.clear();
    });
}
```

**Cron 和循环策略：**
```javascript
// maintenance.js:244-265
this.beanMeta.status = "scheduled";

let startEvent = async (customDuration = 0) => {
    log.info("maintenance", "Maintenance id: " + this.id + " is under maintenance now");

    this.beanMeta.status = "under-maintenance";
    clearTimeout(this.beanMeta.durationTimeout);

    let duration = this.inferDuration(customDuration);

    UptimeKumaServer.getInstance().sendMaintenanceListByUserID(this.user_id);

    this.beanMeta.durationTimeout = setTimeout(() => {
        // End of maintenance for this timeslot
        this.beanMeta.status = "scheduled";
        UptimeKumaServer.getInstance().sendMaintenanceListByUserID(this.user_id);
    }, duration);

    // Set last start date to current time
    this.last_start_date = current.utc().format(SQL_DATETIME_FORMAT);
    await R.store(this);
};
```

### 2.2 状态转换

维护窗口的状态定义（`maintenance.js:421-449`）：

```javascript
async getStatus() {
    if (!this.active) {
        return "inactive";
    }

    if (this.strategy === "manual") {
        return "under-maintenance";
    }

    // Check if the maintenance is started
    if (this.start_date && dayjs().isBefore(dayjs.tz(this.start_date, await this.getTimezone()))) {
        return "scheduled";
    }

    // Check if the maintenance is ended
    if (this.end_date && dayjs().isAfter(dayjs.tz(this.end_date, await this.getTimezone()))) {
        return "ended";
    }

    if (this.strategy === "single") {
        return "under-maintenance";
    }

    if (!this.beanMeta.status) {
        return "unknown";
    }

    return this.beanMeta.status;
}
```

状态流转图：
```
inactive ── resume ──→ scheduled ── cron触发 ──→ under-maintenance ── duration超时 ──→ scheduled
    ↑                                                                │
    └──────────────────────── pause ─────────────────────────────────┘

scheduled ── 超过end_date ──→ ended
```

### 2.3 运行中检测

`getRunningTimeslot()` 方法用于检测当前是否正在维护窗口内（`maintenance.js:346-359`）：

```javascript
getRunningTimeslot() {
    let start = dayjs(this.beanMeta.job.nextRun(dayjs().add(-this.duration, "second").toDate()));
    let end = start.add(this.duration, "second");
    let current = dayjs();

    if (current.isAfter(start) && current.isBefore(end)) {
        return {
            startDate: start.toISOString(),
            endDate: end.toISOString(),
        };
    } else {
        return null;
    }
}
```

---

## 三、监控暂停机制

### 3.1 核心检查点

监控在执行心跳（beat）时，首先检查是否处于维护状态。检查点位于 `monitor.js:465-467`：

```javascript
try {
    if (await Monitor.isUnderMaintenance(this.id)) {
        bean.msg = "Monitor under maintenance";
        bean.status = MAINTENANCE;
    } else if (this.type === "http" || this.type === "keyword" || this.type === "json-query") {
        // 正常监控逻辑...
    } else if (this.type === "ping") {
        // Ping 探测...
    } else if (this.type === "push") {
        // Push 检查...
    } else if (this.type === "docker") {
        // Docker 容器检查...
    }
    // ... 其他监控类型
}
```

**关键点：**
- 使用 `if...else if` 结构，**维护期间监控探测被完全跳过**
- 当 `isUnderMaintenance()` 返回 `true` 时，所有 `else if` 分支（HTTP/Ping/Docker 等实际探测）都不会执行
- 心跳记录的 `status` 字段直接设置为 `MAINTENANCE`，不会执行任何网络请求或系统调用

### 3.1.1 代码结构分析

```
┌─────────────────────────────────────────────────────────────┐
│ try {                                                       │
│   if (isUnderMaintenance)    ──┬── true  ──▶ 跳过所有探测    │
│       bean.status = MAINTENANCE│                            │
│   else if (http/keyword/json)  │                            │
│       执行 HTTP 请求          ──┴── false ──▶ 执行对应探测    │
│   else if (ping)                                              │
│       执行 Ping 探测                                         │
│   else if (push)                                              │
│       检查 Push 心跳                                         │
│   else if (docker)                                            │
│       检查 Docker 容器                                       │
│   ...                                                         │
│ }                                                             │
└─────────────────────────────────────────────────────────────┘
```

**设计意图：** 在维护期间完全不消耗系统资源进行探测，因为已知服务可能处于不可用或不稳定状态。

### 3.2 维护状态判断逻辑

`Monitor.isUnderMaintenance()` 方法实现（`monitor.js:1630-1652`）：

```javascript
static async isUnderMaintenance(monitorID) {
    const maintenanceIDList = await R.getCol(
        `
        SELECT maintenance_id FROM monitor_maintenance
        WHERE monitor_id = ?
        `,
        [monitorID]
    );

    for (const maintenanceID of maintenanceIDList) {
        const maintenance = await UptimeKumaServer.getInstance().getMaintenance(maintenanceID);
        if (maintenance && (await maintenance.isUnderMaintenance())) {
            return true;
        }
    }

    // 检查父监控（支持监控组继承）
    const parent = await Monitor.getParent(monitorID);
    if (parent != null) {
        return await Monitor.isUnderMaintenance(parent.id);
    }

    return false;
}
```

**检查流程：**
1. 从 `monitor_maintenance` 表获取该监控关联的所有维护窗口
2. 遍历检查每个维护窗口是否处于 `under-maintenance` 状态
3. 支持监控组继承：如果父监控在维护中，子监控也视为在维护中
4. 只要有一个维护窗口处于活动状态，返回 `true`

### 3.3 维护状态的影响

当监控处于维护状态时：

1. **不发送告警通知**：在 `isImportantForNotification()` 中，维护状态变化不会触发通知
   ```javascript
   // monitor.js:1003-1011
   if (Monitor.isImportantForNotification(isFirstBeat, previousBeat?.status, bean.status)) {
       log.debug("monitor", `[${this.name}] sendNotification`);
       await Monitor.sendNotification(isFirstBeat, this, bean);
   } else {
       log.debug(
           "monitor",
           `[${this.name}] will not sendNotification because it is (or was) under maintenance`
       );
   }
   ```

2. **心跳记录标记维护**：心跳记录的 `status` 字段设置为 `MAINTENANCE`（值为 3）

3. **日志记录**：
   ```javascript
   // monitor.js:1078-1079
   } else if (bean.status === MAINTENANCE) {
       log.warn("monitor", `Monitor #${this.id} '${this.name}': Under Maintenance | Type: ${this.type}`);
   ```

---

## 四、状态计算与维护窗口

### 4.1 维护状态的扁平化处理

在 `uptime-calculator.js:545-555` 中，维护状态被视为"正常"状态：

```javascript
flatStatus(status) {
    switch (status) {
        case UP:
        case MAINTENANCE:
            return UP;  // 维护状态被扁平化为 UP
        case DOWN:
        case PENDING:
            return DOWN;
    }
    throw new Error("Invalid status");
}
```

**重要设计决策：** 维护期间不计入 downtime，从可用性计算角度视为"正常运行"。

### 4.2 维护计数统计

虽然维护状态不计入 downtime，但会单独统计维护次数（`uptime-calculator.js:231-234`）：

```javascript
if (status === MAINTENANCE) {
    minutelyData.maintenance = minutelyData.maintenance ? minutelyData.maintenance + 1 : 1;
    hourlyData.maintenance = hourlyData.maintenance ? hourlyData.maintenance + 1 : 1;
    dailyData.maintenance = dailyData.maintenance ? dailyData.maintenance + 1 : 1;
} else if (flatStatus === UP) {
    // UP 统计...
} else if (flatStatus === DOWN) {
    // DOWN 统计...
}
```

### 4.3 可用性计算

可用性计算在 `getData()` 方法中（`uptime-calculator.js:563-688`）：

```javascript
let uptimeData = new UptimeDataResult();

// ... 统计 total.up 和 total.down ...

if (total.up + total.down === 0) {
    uptimeData.uptime = 0;
} else {
    uptimeData.uptime = total.up / (total.up + total.down);
}
```

**关键点：** 可用性计算只考虑 `up` 和 `down`，`maintenance` 状态完全被排除在计算之外。

---

## 五、前端状态页展示

### 5.1 维护信息获取

状态页通过 API 获取当前活跃的维护窗口列表。后端逻辑在 `status_page.js:558-582`：

```javascript
static async getMaintenanceList(statusPageId) {
    try {
        const publicMaintenanceList = [];

        let maintenanceIDList = await R.getCol(
            `
            SELECT DISTINCT maintenance_id
            FROM maintenance_status_page
            WHERE status_page_id = ?
            `,
            [statusPageId]
        );

        for (const maintenanceID of maintenanceIDList) {
            let maintenance = UptimeKumaServer.getInstance().getMaintenance(maintenanceID);
            if (maintenance && (await maintenance.isUnderMaintenance())) {
                publicMaintenanceList.push(await maintenance.toPublicJSON());
            }
        }

        return publicMaintenanceList;
    } catch (error) {
        return [];
    }
}
```

**注意：** 只返回**当前正在进行**的维护窗口（`isUnderMaintenance()` 为 true）。

### 5.2 状态页展示逻辑

前端状态页在 `StatusPage.vue` 中展示维护状态。状态页的维护相关展示分为**两个独立机制**：

---

### 5.2.1 整体状态计算机制

整体状态由 `overallStatus()` 计算属性决定（`StatusPage.vue:793-818`）：

```javascript
overallStatus() {
    if (Object.keys(this.$root.publicLastHeartbeatList).length === 0) {
        return -1;
    }

    let status = STATUS_PAGE_ALL_UP;
    let hasUp = false;

    for (let id in this.$root.publicLastHeartbeatList) {
        let beat = this.$root.publicLastHeartbeatList[id];

        if (beat.status === MAINTENANCE) {
            return STATUS_PAGE_MAINTENANCE;  // ⚠️ 立即返回！
        } else if (beat.status === UP) {
            hasUp = true;
        } else {
            status = STATUS_PAGE_PARTIAL_DOWN;
        }
    }

    if (!hasUp) {
        status = STATUS_PAGE_ALL_DOWN;
    }

    return status;
}
```

#### 核心逻辑分析：优先级判定

| 优先级 | 状态常量（数值） | 触发条件 | 判定顺序 |
|--------|------------------|----------|----------|
| **1（最高）** | `STATUS_PAGE_MAINTENANCE` (3) | 任意监控的 `beat.status === MAINTENANCE` | 循环中**第一个条件**，一旦命中立即 `return` 退出循环 |
| 2 | `STATUS_PAGE_ALL_DOWN` (0) | 所有监控都是 DOWN/PENDING，**且循环中未命中 MAINTENANCE** | 循环结束后检查 `hasUp` |
| 3 | `STATUS_PAGE_PARTIAL_DOWN` (2) | 至少一个 UP，至少一个 DOWN/PENDING，**且循环中未命中 MAINTENANCE** | 循环中 `else` 分支累积 |
| 4（最低） | `STATUS_PAGE_ALL_UP` (1) | 所有监控都是 UP，**且循环中未命中 MAINTENANCE** | 默认值 |

#### 边界条件说明

**条件 1：MAINTENANCE 是短路返回**
- 代码结构：`for...in` 循环中 `if (MAINTENANCE) return`
- 行为：**只要有任何一个监控的心跳状态是 MAINTENANCE，整个循环立即终止**
- 后续监控的 UP/DOWN 状态**完全不会被检查**

**条件 2：模板中的展示顺序不等于优先级**

模板代码（`StatusPage.vue:385-409`）：
```vue
<div v-if="allUp">
    {{ $t("All Systems Operational") }}
</div>
<div v-else-if="partialDown">
    {{ $t("Partially Degraded Service") }}
</div>
<div v-else-if="allDown">
    {{ $t("Degraded Service") }}
</div>
<div v-else-if="isMaintenance">
    {{ $t("maintenanceStatus-under-maintenance") }}
</div>
```

**关键点：** 模板中的 `v-if/v-else-if` 顺序是 `allUp → partialDown → allDown → isMaintenance`，但这只是 UI 展示顺序。

**实际优先级由 `overallStatus()` 的返回值决定**：
- `isMaintenance()` 检查 `overallStatus === STATUS_PAGE_MAINTENANCE`
- 由于 `overallStatus()` 中 MAINTENANCE 是最高优先级，一旦命中就立即返回
- 因此**不会出现「维护中但显示故障」的情况**

**条件 3：维护期间无法看到实际故障**

由于维护中的监控心跳状态已是 `MAINTENANCE`（值为 3），而非实际的 `UP`（1）或 `DOWN`（0）：
- 状态页无法从心跳数据中知道这些监控在维护期间的实际状态
- 即使服务在维护期间故障，心跳记录也是 `MAINTENANCE`
- 因此 `overallStatus()` 只能看到 `MAINTENANCE`，无法看到隐藏的 `DOWN`

---

### 5.2.2 维护信息卡片机制

维护信息卡片由独立的 API 获取（`status_page.js:558-582`）：

```javascript
static async getMaintenanceList(statusPageId) {
    let maintenanceIDList = await R.getCol(
        `SELECT DISTINCT maintenance_id
         FROM maintenance_status_page
         WHERE status_page_id = ?`,
        [statusPageId]
    );

    for (const maintenanceID of maintenanceIDList) {
        let maintenance = UptimeKumaServer.getInstance().getMaintenance(maintenanceID);
        if (maintenance && (await maintenance.isUnderMaintenance())) {
            publicMaintenanceList.push(await maintenance.toPublicJSON());
        }
    }
    return publicMaintenanceList;
}
```

**触发条件：** `maintenanceList.length > 0`

**检查逻辑：**
1. 从 `maintenance_status_page` 表查询与该状态页关联的维护窗口
2. 检查每个维护窗口是否 `isUnderMaintenance()`
3. 只返回**当前正在进行**的维护窗口

---

### 5.2.3 两种机制的独立与联动

| 展示元素 | 触发条件 | 数据来源 | 与整体状态的关系 |
|----------|----------|----------|------------------|
| **整体状态徽章**（"维护中"） | `overallStatus()` 返回 `MAINTENANCE` | 所有监控的心跳状态（`publicLastHeartbeatList`） | 完全独立，不受维护窗口关联影响 |
| **维护信息卡片**（标题+描述） | `maintenanceList.length > 0` | `maintenance_status_page` 表关联的维护窗口 | 完全独立，不受监控心跳状态影响 |

#### 边界场景矩阵

| 场景 | 监控心跳状态 | 维护窗口关联状态页 | 整体状态徽章 | 维护信息卡片 |
|------|-------------|-------------------|--------------|--------------|
| 场景 1 | 有监控在维护 | 维护窗口关联了状态页 | 🔧 维护中 | ✅ 显示卡片 |
| 场景 2 | 有监控在维护 | 维护窗口**未关联**状态页 | 🔧 维护中 | ❌ 不显示卡片 |
| 场景 3 | 无监控在维护 | 维护窗口关联了状态页且正在进行 | ⚡ 根据实际状态 | ✅ 显示卡片 |
| 场景 4 | 无监控在维护 | 无活跃维护窗口 | ⚡ 根据实际状态 | ❌ 不显示卡片 |

**场景 2 说明：** 即使没有为状态页配置任何维护窗口，只要状态页下的某个监控被添加到维护窗口中，状态页就会显示"维护中"。

**场景 3 说明：** 可以创建"纯粹的公告式"维护信息卡片，不关联任何监控，只在状态页上显示维护通知，不影响任何监控的状态计算。

---

### 5.2.4 整体状态徽章展示

```vue
<!-- StatusPage.vue:401-404 -->
<div v-else-if="isMaintenance">
    <font-awesome-icon icon="wrench" class="status-maintenance" />
    {{ $t("maintenanceStatus-under-maintenance") }}
</div>
```

---

### 5.2.5 维护信息卡片展示

```vue
<!-- StatusPage.vue:412-425 -->
<template v-if="maintenanceList.length > 0">
    <div
        v-for="maintenance in maintenanceList"
        :key="maintenance.id"
        class="shadow-box alert mb-4 p-3 bg-maintenance mt-4 position-relative"
        role="alert"
    >
        <h4 class="alert-heading">{{ maintenance.title }}</h4>
        <div class="content" v-html="maintenanceHTML(maintenance.description)"></div>
        <MaintenanceTime :maintenance="maintenance" />
    </div>
</template>
```

---

## 六、关键数据流图

### 6.1 维护窗口创建与启动

```
┌─────────────────┐     ┌─────────────────────────┐     ┌──────────────────────┐
│  EditMaintenance │────▶│  maintenance-socket-   │────▶│  Maintenance.run()   │
│    (前端表单)     │     │  handler.js            │     │  创建 Cron 任务       │
└─────────────────┘     └─────────────────────────┘     └──────────────────────┘
                                                              │
                                                              ▼
                                                    ┌──────────────────────┐
                                                    │  更新 beanMeta.status │
                                                    │  = "scheduled"        │
                                                    └──────────────────────┘
```

### 6.2 监控心跳检查流程

```
┌──────────────────┐
│ Monitor.beat()   │
│  (心跳循环)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│ Monitor.isUnderMaintenance() │─────┬── true ──▶ 设为 MAINTENANCE 状态
│ 检查关联的维护窗口            │     │
└──────────────────────────────┘     │
         │                           │
         ▼ false                     │
┌──────────────────┐                 │
│ 执行实际监控检查  │◀────────────────┘
│ (HTTP/Ping等)    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ UptimeCalculator │
│ .update()        │
└────────┬─────────┘
         │
    ┌────┴────┐
    ▼         ▼
 MAINTENANCE   UP/DOWN
    │           │
    ▼           ▼
计入maintenance 计入up/down
  计数            可用性
```

### 6.3 状态页数据获取

```
┌──────────────────┐     ┌────────────────────────────┐     ┌───────────────────────┐
│ StatusPage.vue   │────▶│ API: /api/status-page/:slug│────▶│ StatusPage.           │
│ (前端状态页)      │     │                            │     │ getMaintenanceList()  │
└──────────────────┘     └────────────────────────────┘     └──────────┬────────────┘
                                                                       │
                                                    ┌──────────────────┴──────────────────┐
                                                    ▼                                     ▼
                                    ┌─────────────────────────┐              ┌─────────────────────────┐
                                    │ maintenance_status_page │              │ Maintenance.            │
                                    │ 表查询关联的维护窗口       │              │ isUnderMaintenance()    │
                                    └─────────────────────────┘              │ 检查是否正在维护         │
                                                                             └─────────────────────────┘
                                                                                       │
                                                                                       ▼
                                                                             只返回当前活跃的维护窗口
```

---

## 七、关键代码位置汇总

| 功能 | 文件位置 | 关键方法/行号 |
|------|----------|---------------|
| 维护窗口模型 | `server/model/maintenance.js` | 完整文件 |
| 维护调度启动 | `server/model/maintenance.js:212-340` | `run()` |
| 维护状态判断 | `server/model/maintenance.js:421-449` | `getStatus()` |
| 运行中检测 | `server/model/maintenance.js:346-359` | `getRunningTimeslot()` |
| 监控维护检查 | `server/model/monitor.js:1630-1652` | `isUnderMaintenance()` |
| 心跳维护处理 | `server/model/monitor.js:465-467` | `beat()` 中的检查 |
| 可用性计算 | `server/uptime-calculator.js` | 完整文件 |
| 维护状态扁平化 | `server/uptime-calculator.js:545-555` | `flatStatus()` |
| 维护计数 | `server/uptime-calculator.js:231-234` | `update()` |
| Socket处理器 | `server/socket-handlers/maintenance-socket-handler.js` | 完整文件 |
| 状态页维护API | `server/model/status_page.js:558-582` | `getMaintenanceList()` |
| 维护管理页面 | `src/pages/ManageMaintenance.vue` | 完整文件 |
| 维护编辑页面 | `src/pages/EditMaintenance.vue` | 完整文件 |
| 状态页展示 | `src/pages/StatusPage.vue:401-425` | 维护状态展示 |

---

## 八、设计要点总结

### 8.1 核心设计理念

1. **维护期间跳过探测**：使用 `if...else if` 结构，维护期间完全跳过所有实际的监控探测（HTTP/Ping/Docker 等），不消耗系统资源。

2. **维护不计入 downtime**：从可用性角度，维护状态被视为"正常"，不影响 SLA 计算。

3. **支持监控组继承**：父监控的维护状态会传递给子监控，方便对整个服务组进行维护。

4. **实时状态同步**：维护状态变化通过 Socket.io 实时推送到前端，包括管理界面和状态页。

5. **维护状态最高优先级**：在状态页的 `overallStatus()` 计算中，MAINTENANCE 状态优先级最高，一旦发现立即返回。

### 8.2 潜在注意事项

1. **维护期间无实际探测**：维护结束后无法从心跳数据判断服务在维护期间的实际状态。如果需要确认维护期间服务状态，需要额外的验证机制。

2. **状态页维护状态可能隐藏故障**：由于 MAINTENANCE 在 `overallStatus()` 中优先级最高，且维护中的监控心跳状态已是 MAINTENANCE，即使有监控实际故障，状态页也只会显示维护状态。

3. **状态页的两个独立触发条件**：
   - 顶部维护信息卡片：只显示与状态页关联的维护窗口
   - 整体状态徽章：只要状态页下的任何监控处于维护中就显示
   - 可能出现：没有维护卡片但状态显示"维护中"的情况

4. **手动模式特性**：手动模式的维护窗口只要处于 active 状态，就会一直视为"维护中"。

5. **时区处理**：维护窗口支持自定义时区，需要注意 Cron 表达式与时区的匹配。
