# Uptime Kuma 心跳历史保留与清理机制分析报告

## 1. 数据存储架构

### 1.1 数据模型概览

Uptime Kuma 采用了多层数据聚合策略来管理心跳历史数据，主要涉及以下四张数据表：

#### 1.1.1 `heartbeat` 表（原始心跳数据）

定义位置：`server/model/heartbeat.js`

**核心字段：**
- `status`: 状态码（0=DOWN, 1=UP, 2=PENDING, 3=MAINTENANCE）
- `time`: 心跳时间戳
- `ping`: 响应时间（毫秒）
- `msg`: 状态消息
- `important`: 是否为重要心跳（状态变更时标记）
- `duration`: 持续时间
- `retries`: 重试次数
- `response`: 响应体（Brotli 压缩存储）

**数据来源：**
- 每次监控检查执行后生成，在 `server/model/monitor.js:1099` 通过 `R.store(bean)` 持久化
- 状态变更时自动标记为 `important=true`

#### 1.1.2 聚合统计表（三层架构）

定义位置：`server/uptime-calculator.js`

| 表名 | 聚合粒度 | 保留周期 | 数据容量 |
|------|---------|---------|---------|
| `stat_minutely` | 分钟级 | 24小时 | 1440 个数据点/监控 |
| `stat_hourly` | 小时级 | 30天 | 720 个数据点/监控 |
| `stat_daily` | 天级 | 用户配置（默认365天） | 365 个数据点/监控 |

**聚合字段：**
- `up`: 该时间段内 UP 状态的心跳次数
- `down`: 该时间段内 DOWN 状态的心跳次数
- `ping`: 平均响应时间
- `pingMin`: 最小响应时间
- `pingMax`: 最大响应时间
- `extras`: 扩展数据（JSON 格式）
- `maintenance`: 维护模式计数（运行时数据）

### 1.2 数据聚合流程

**核心类：** `UptimeCalculator`（`server/uptime-calculator.js`）

#### 初始化过程（`init` 方法，行 123-203）：
1. 从数据库加载最近 24 小时的分钟级统计
2. 加载最近 30 天的小时级统计
3. 加载最近 365 天的天级统计
4. 存储到内存中的 `LimitQueue` 结构

#### 数据更新流程（`update` 方法，行 212-374）：

每次心跳到达时：

1. **时间分区键计算**
   - `getMinutelyKey()`: 截断到分钟（秒级时间戳）
   - `getHourlyKey()`: 截断到小时
   - `getDailyKey()`: 截断到天（UTC 时区）

2. **状态计数更新**
   ```javascript
   // UP 状态：增加 up 计数，更新 ping 统计
   // DOWN 状态：增加 down 计数
   // MAINTENANCE 状态：增加 maintenance 计数
   ```

3. **Ping 统计计算**
   - 首次心跳：直接赋值
   - 后续心跳：增量平均算法
   ```javascript
   avgPing = (prevAvg * (up-1) + currentPing) / up
   ```

4. **持久化到数据库**
   - 天级数据：始终存储
   - 小时级数据：仅保留最近 30 天内的数据
   - 分钟级数据：仅保留最近 24 小时内的数据

---

## 2. 数据保留与清理机制

### 2.1 清理任务调度

**调度配置：** `server/jobs.js`

```javascript
{
    name: "clear-old-data",
    interval: "14 03 * * *",  // 每天凌晨 03:14 执行
    jobFunc: clearOldData,
}
```

使用 `croner` 库进行调度，时区从服务器配置获取。

### 2.2 清理策略详解

**主清理函数：** `server/jobs/clear-old-data.js:clearOldData()`

#### 2.2.1 保留周期配置

- **配置键：** `keepDataPeriodDays`
- **默认值：** 365 天
- **用户可配置：** 设置页面（`src/components/settings/MonitorHistory.vue`）
- **最小值：** 0（表示禁用清理）

#### 2.2.2 清理执行流程

**第一步：清理非重要心跳**（`Database.clearHeartbeatData()`，`server/database.js:953-980`）

```sql
DELETE FROM heartbeat
WHERE monitor_id = ?
  AND important = 0
  AND time < [24小时前]
  AND id NOT IN (最近100条心跳)
```

**策略说明：**
- 仅删除 `important=0` 的常规心跳
- 始终保留最近 24 小时的所有心跳
- 始终保留每个监控的最近 100 条心跳（无论是否重要）
- `important=true` 的心跳（状态变更）永久保留（除非被第二步清理）

**第二步：清理过期原始心跳**（`clear-old-data.js:44`）

```sql
DELETE FROM heartbeat WHERE time < [keepDataPeriodDays 天前]
```

**第三步：清理过期天级统计**（`clear-old-data.js:49`）

```sql
DELETE FROM stat_daily WHERE timestamp < [keepDataPeriodDays 天前]
```

**第四步：SQLite 优化**（`clear-old-data.js:51-53`）

```sql
PRAGMA optimize;  -- 仅 SQLite 数据库
```

### 2.3 聚合数据的实时清理

除了定时任务，`UptimeCalculator` 在每次更新时也会进行增量清理（`uptime-calculator.js:358-371`）：

```javascript
// 清理超过 24 小时的分钟级统计
DELETE FROM stat_minutely WHERE monitor_id = ? AND timestamp < ?

// 清理超过 30 天的小时级统计  
DELETE FROM stat_hourly WHERE monitor_id = ? AND timestamp < ?
```

**注意：** `stat_daily` 的清理仅在定时任务中执行。

---

## 3. 图表展示与数据降采样

### 3.1 数据源选择

**图表组件：** `src/components/PingChart.vue`

**时间周期选项（`src/components/PingChart.vue:79-84`）：**
```javascript
chartPeriodOptions: {
    0: this.$t("recent"),
    3: "3h",
    6: "6h",
    24: "24h",
    168: "1w",
}
```

**周期到统计层级的映射（`server/socket-handlers/chart-socket-handler.js:19-25`）：**

| 周期选项 | 小时数 | 判断条件 | 实际统计层级 | 数据来源 |
|---------|-------|---------|-------------|---------|
| recent | 0 | 特殊处理 | 原始心跳 | `heartbeatList` |
| 3h | 3 | `period <= 24` | 分钟级 | `stat_minutely` |
| 6h | 6 | `period <= 24` | 分钟级 | `stat_minutely` |
| 24h | 24 | `period <= 24` | 分钟级 | `stat_minutely` |
| 1w | 168 | `period <= 720` (30天) | **小时级** | `stat_hourly` |
| >720h (>30天) | >720 | `period > 720` | 天级 | `stat_daily` |

**关键发现：1w (168h) 周期实际使用小时级数据，而非天级。**

原因：`168 <= 720` 为 true，所以命中 `period <= 720` 分支，使用小时级统计数据。

#### 数据获取逻辑（`chartData` 计算属性，行 217-223）：

```javascript
if (chartPeriodHrs === "0") {
    // 使用实时心跳列表（heartbeatList）
    return this.getChartDatapointsFromHeartbeatList();
} else {
    // 使用聚合统计数据
    return this.getChartDatapointsFromStats();
}
```

### 3.2 服务端数据接口

**Socket Handler：** `server/socket-handlers/chart-socket-handler.js`

```javascript
if (period <= 24) {
    // 24小时内：返回分钟级数据
    data = uptimeCalculator.getDataArray(period * 60, "minute");
} else if (period <= 720) {
    // 24-720小时（30天）：返回小时级数据
    data = uptimeCalculator.getDataArray(period, "hour");
} else {
    // 超过30天：返回天级数据
    data = uptimeCalculator.getDataArray(period / 24, "day");
}
```

**数据方法：** `UptimeCalculator.getDataArray(num, type)`（行 697-767）

- 从对应的 `LimitQueue` 中提取指定数量的数据点
- 返回包含 `up`, `down`, `avgPing`, `minPing`, `maxPing`, `timestamp` 的数组

### 3.3 前端降采样策略

#### 3.3.1 实时模式（`getChartDatapointsFromHeartbeatList`，行 352-440）

**数据来源：** `this.$root.heartbeatList[monitorId]`

**特点：** 名义上"无降采样"，但受双重上限约束，实际上是有限数据点的直接展示。

##### 双重数据上限机制

**第一重：服务端查询限制**（`server/client.js:sendHeartbeatList()`，行 46-64）

```sql
SELECT * FROM heartbeat
WHERE monitor_id = ?
ORDER BY time DESC
LIMIT 100
```

- 初始加载时，服务端从数据库查询最近 **100 条** 心跳
- `ORDER BY time DESC LIMIT 100`: 按时间倒序取最新的 100 条
- `list.reverse()`: 查询结果反转，形成从旧到新的正序列表
- 实际发送条数：**恰好 100 条**（如果数据库中有足够数据）

**第二重：客户端内存限制**（`src/mixins/socket.js`，行 204-213）

```javascript
socket.on("heartbeat", (data) => {
    if (!(data.monitorID in this.heartbeatList)) {
        this.heartbeatList[data.monitorID] = [];
    }

    this.heartbeatList[data.monitorID].push(data);

    if (this.heartbeatList[data.monitorID].length >= 150) {
        this.heartbeatList[data.monitorID].shift();
    }
});
```

**代码逻辑精确分析：**

1. `push(data)`: 每次新心跳到达时追加到列表尾部
2. `length >= 150`: 判断条件是"大于等于"而非"大于"
3. `shift()`: 移除列表的第一个元素（最早的一条）

**达到上限时的实际保留数量：**

| 操作步骤 | 列表长度 | 说明 |
|---------|---------|------|
| 初始状态（刚加载） | 100 | 来自 `sendHeartbeatList` 的 100 条 |
| 第 1 条新心跳 push | 101 | `101 >= 150` 为 false，不移除 |
| 第 50 条新心跳 push | 150 | `150 >= 150` 为 true，执行 shift |
| shift 后 | **149** | 移除最早的 1 条，实际保留 149 条 |
| 第 51 条新心跳 push | 150 | `150 >= 150` 为 true，执行 shift |
| shift 后 | **149** | 保持 149 条 |

**关键结论：达到上限时实际保留 149 条，而非 150 条。**

原因：`push()` 先将长度增加到 150，然后 `shift()` 立即移除 1 条，最终长度为 149。

##### 实际数据量完整分析

| 阶段 | 数据来源 | 列表长度 | 说明 |
|------|---------|---------|------|
| 页面刚加载 | `sendHeartbeatList` | 100 | 数据库中最近 100 条 |
| 新增 1-49 条心跳 | 实时推送 | 101-149 | 未达到 150 阈值，不触发移除 |
| 新增第 50 条心跳 | push 后 length=150 | **149** | 触发 shift，实际保留 149 条 |
| 长期运行 | 滑动窗口 | **149** | 每次 push 后立即 shift，始终保持 149 条 |

**特殊情况：**
- 如果数据库中心跳不足 100 条，初始加载条数 = 实际心跳数
- 如果监控刚创建且没有历史心跳，初始列表为空
- 页面刷新后重新从数据库加载，重置为 100 条（或实际条数）

##### 对"无降采样"判断的影响

**代码层面的判断：**

在 `PingChart.vue:217-223`：
```javascript
chartData() {
    if (this.chartPeriodHrs === "0") {
        return this.getChartDatapointsFromHeartbeatList();
    } else {
        return this.getChartDatapointsFromStats();
    }
}
```

`getChartDatapointsFromHeartbeatList()` 方法内部：
```javascript
getChartDatapointsFromHeartbeatList() {
    let heartbeatList =
        (this.monitorId in this.$root.heartbeatList && this.$root.heartbeatList[this.monitorId]) || [];

    for (const beat of heartbeatList) {
        // 直接遍历每一条心跳，没有任何聚合逻辑
        // 逐条 push 到 pingData 和 downData
    }
}
```

**实际含义分析：**

1. **算法层面**：确实"无降采样"
   - 没有滑动窗口聚合
   - 没有平均值计算
   - 没有数据点合并
   - 每条心跳在 `heartbeatList` 中存在就会在图表中显示

2. **数据层面**：实际上是"截断采样"
   - 触发条件是 `length >= 150`，但实际稳态保留 **149 条**（push 后立即 shift）
   - 这不是智能降采样（保留特征点），而是简单的尾部截断
   - 对于高频监控（如 30 秒间隔），149 条仅覆盖约 74.5 分钟

3. **与统计模式的本质区别：**

| 对比项 | Recent 模式 | 统计模式（3h/6h/24h） |
|-------|------------|---------------------|
| 数据粒度 | 原始心跳 | 分钟/小时聚合 |
| 时间覆盖 | 取决于监控频率和 149 条稳态上限 | 固定时间窗口 |
| 聚合算法 | 无（直接展示） | 滑动窗口 + 平均值 |
| DOWN 状态 | 每条单独显示 | 可能与相邻 UP 聚合 |
| Ping 统计 | 单次 ping 值 | avg/min/max |

4. **潜在的信息丢失：**

例如，监控间隔 20 秒，运行 2 小时后：
- 产生心跳数：3600/20 × 2 = 360 条
- 实际保留：**149 条**（最近约 49.7 分钟）
- 丢失：前约 70.3 分钟的 211 条心跳
- 图表显示范围：约 50 分钟，而非"最近"的完整 2 小时

**处理逻辑：**
- 遍历 `heartbeatList`（Socket 实时推送的最近心跳）
- 检测时间间隙（> 10倍监控间隔），插入空数据点分隔
- 状态映射：
  - UP: 显示 ping 值，状态栏为透明
  - DOWN/PENDING/MAINTENANCE: ping 为 null，状态栏着色

#### 3.3.2 统计模式（`getChartDatapointsFromStats`，行 441-590）

**第一级降采样：服务端聚合**

已根据时间周期选择不同粒度的数据：
- ≤24h: 分钟级（1440 点/天）
- 24h-30d: 小时级（720 点/30天）
- >30d: 天级（365 点/年）

**第二级降采样：前端滑动窗口聚合**（行 453-520）

**聚合窗口大小：**
```javascript
let aggregatePoints = period > 6 ? 12 : 4;
// period <= 6小时: 4个点聚合为1个
// period > 6小时: 12个点聚合为1个
```

**触发条件：**
```javascript
if (datapoint.up > 0 && this.chartRawData.length > aggregatePoints * 2)
```
- 仅对 UP 状态的数据进行聚合
- 数据量不足时跳过聚合

**滑动窗口算法：**
1. 累积 `aggregatePoints` 个数据点到缓冲区
2. 计算平均值（`getAverage` 方法，行 331-351）
3. 推入图表数据
4. 移除缓冲区前半部分（滑动步长 = window/2，重叠平滑）
5. 重复直到处理完所有数据

**聚合计算方法：**
```javascript
// 状态计数：简单求和
totalUp = sum(datapoints.up)
totalDown = sum(datapoints.down)
totalMaintenance = sum(datapoints.maintenance)

// Ping 统计：加权平均 + 极值
avgPing = sum(avgPing * up) / totalUp
minPing = min(datapoints.minPing)
maxPing = max(datapoints.maxPing)

// 时间戳：取中间点
timestamp = datapoints[midpoint].timestamp
```

#### 3.3.3 间隙检测与处理

**间隙判断条件：**
```javascript
// 短周期（≤24h）：间隙 > 10分钟 或 > 10倍监控间隔
// 长周期（>24h）：间隙 > 10小时 或 > 10倍监控间隔
```

**处理方式：**
- 先清空并输出当前聚合缓冲区
- 插入两个空数据点（y=null）作为视觉分隔
- 继续处理后续数据

#### 3.3.4 特殊状态处理

**完全 DOWN 的数据点：**
- 不参与聚合
- 立即输出当前缓冲区
- 单独输出该 DOWN 数据点
- 保持 DOWN 状态的视觉显著性

**空数据点：**
```javascript
if (datapoint.up === 0 && datapoint.down === 0 && datapoint.maintenance === 0) {
    continue;  // 完全跳过
}
```

### 3.4 图表渲染

**数据集：**
1. **Min Ping Line**: 深绿色 (`#126331`)
2. **Avg Ping Line**: 浅绿色 (`#5CDD8B`) 
3. **Max Ping Line**: 绿色 (`#21b55a`)
4. **Status Bar**: 背景色表示状态
   - 正常 UP: 透明 (`#000`)
   - 完全 DOWN: 红色 (`rgba(220, 53, 69, 0.41)`)
   - 混合状态: 黄色 (`rgba(245, 182, 23, 0.41)`)
   - 维护模式: 蓝色 (`rgba(23,71,245,0.41)`)

**Chart.js 配置：**
- 双 Y 轴：左侧为响应时间(ms)，右侧隐藏（用于柱状图）
- 柱状图 `barPercentage=1` 实现无缝背景
- 点默认隐藏（`radius: 0`），鼠标悬停时通过 `hitRadius` 交互

---

## 4. 数据迁移与初始化

### 4.1 v1 到 v2 的数据迁移

**迁移函数：** `Database.migrateAggregateTable()`（`server/database.js:819-946`）

**迁移流程：**
1. 获取所有唯一的 monitor_id
2. 检查 `stat_*` 表是否为空（防止重复迁移）
3. 按日期分批处理每个监控的心跳数据
4. 使用 `UptimeCalculator`（migrationMode=true）重新聚合
5. 清理非重要心跳（保留状态变更记录）
6. 标记迁移状态为 `migrated`

**迁移状态设置：**
- `migrating`: 迁移中（中断后需手动重置）
- `migrated`: 已完成（跳过后续迁移）

**重置命令：**
```bash
npm run reset-migrate-aggregate-table-state
```

### 4.2 数据清理工具

**手动清理所有统计：** `UptimeCalculator.clearStatistics()`
```javascript
DELETE FROM heartbeat WHERE monitor_id = ?
DELETE FROM stat_minutely WHERE monitor_id = ?
DELETE FROM stat_hourly WHERE monitor_id = ?
DELETE FROM stat_daily WHERE monitor_id = ?
```

**SQLite 数据库收缩：** `Database.shrink()`
```sql
VACUUM;  -- 重建数据库文件，回收未使用空间
```

---

## 5. 关键设计决策分析

### 5.1 三层聚合架构的优势

1. **查询性能**：图表查询直接从聚合表读取，无需扫描大量原始心跳
2. **存储效率**：分钟级数据仅保留24小时，大幅减少存储需求
3. **实时性平衡**：最近24小时可查看分钟级细节，历史数据保持趋势可见

### 5.2 重要心跳保护机制

`important` 标记确保：
- 状态变更事件永久保留（不受24小时限制）
- 故障排查时可追溯完整的状态切换历史
- 每个监控至少保留100条心跳作为安全缓冲

### 5.3 前端降采样策略

滑动窗口 + 50% 重叠：
- 减少数据点数量（约 2-6 倍压缩）
- 保持峰值和趋势的可见性
- 平滑过渡避免边界突变
- DOWN 状态不聚合，确保故障点醒目

### 5.4 时区处理

- 天级统计使用 UTC 时间开始边界（避免用户时区切换影响计算）
- 定时任务使用服务器配置的时区

---

## 6. 监控长期停更时的表现

### 6.1 停更场景定义

**监控停更**指监控暂停或长时间不产生新心跳的情况，包括：
- 用户手动暂停监控（`active=false`）
- Push 类型监控长时间未推送数据
- 监控配置错误导致无法执行
- 服务器异常导致监控任务中断

### 6.2 对数据清理的影响

#### 6.2.1 聚合统计数据

**清理延迟问题：**

如 2.3.4 节所述，`stat_minutely` 和 `stat_hourly` 的清理仅在 `update()` 调用时触发。

**时间线示例（监控间隔 60 秒，3月1日开始停更）：**

| 日期 | 事件 | `stat_minutely` 数据状态 | `stat_hourly` 数据状态 |
|------|------|------------------------|-----------------------|
| 3月1日 | 监控最后一次心跳 | 保留最近 24 小时数据 | 保留最近 30 天数据 |
| 3月2日 | 停更第 1 天 | 24小时前的数据**未清理** | 正常 |
| 3月31日 | 停更第 30 天 | 全部数据**未清理** | 30天前的数据**未清理** |
| 4月1日 | 监控恢复运行，第一次心跳 | **立即清理全部过期数据** | **立即清理全部过期数据** |

**与定时任务的对比：**

`stat_daily` 和 `heartbeat` 表的清理由定时任务（每天 03:14）执行，不受监控是否运行影响。

#### 6.2.2 原始心跳数据

**受定时任务保护：**

即使监控停更，定时任务仍会按 `keepDataPeriodDays` 配置清理过期心跳。

```
keepDataPeriodDays = 365（默认）
监控于 2025-01-01 停更
2026-01-02 的定时任务会清理 2025-01-01 及之前的所有心跳
```

### 6.3 对图表展示的影响

#### 6.3.1 Recent 模式（`chartPeriodHrs = "0"`）

**数据来源：** `heartbeatList`（客户端内存中的最近心跳）

**停更后的表现：**

1. **Socket 重连时的数据丢失：**
   - 重连时调用 `sendHeartbeatList()` 从数据库查询最近 100 条
   - 如果监控已停更超过 24 小时且 `important=0` 的心跳已被清理
   - 可能返回空列表或仅包含少量 `important=true` 的心跳

2. **客户端内存限制：**
   - 页面未刷新时：`heartbeatList` 滑动窗口，达到 150 时触发 shift，实际保留 **149 条**（详见 3.3.1 节精确分析）
   - 页面刷新或重连后：从数据库重新加载 100 条（或实际可用条数），可能丢失更多数据

3. **图表表现：**
   - 图表显示最后一次心跳的时间点
   - 之后没有新数据点，图表"冻结"在停更时刻
   - 不会显示"监控已停止"的明确提示

#### 6.3.2 统计模式（3h/6h/24h/1w）

**数据来源：** `UptimeCalculator.getDataArray()`

**UptimeCalculator 初始化行为：** `server/uptime-calculator.js:72-82`

```javascript
static async getUptimeCalculator(monitorID) {
    if (!monitorID) {
        throw new Error("Monitor ID is required");
    }

    if (!UptimeCalculator.list[monitorID]) {
        UptimeCalculator.list[monitorID] = new UptimeCalculator();
        await UptimeCalculator.list[monitorID].init(monitorID);
    }
    return UptimeCalculator.list[monitorID];
}
```

**`init()` 方法的数据加载：** `server/uptime-calculator.js:123-203`

```javascript
async init(monitorID) {
    let now = this.getCurrentDate();

    // 加载最近 24 小时的分钟级统计
    let minutelyStatBeans = await R.find("stat_minutely", 
        " monitor_id = ? AND timestamp > ? ORDER BY timestamp", [
        monitorID,
        this.getMinutelyKey(now.subtract(24, "hour")),
    ]);

    // 加载最近 30 天的小时级统计
    let hourlyStatBeans = await R.find("stat_hourly", 
        " monitor_id = ? AND timestamp > ? ORDER BY timestamp", [
        monitorID,
        this.getHourlyKey(now.subtract(30, "day")),
    ]);

    // 加载最近 365 天的天级统计
    let dailyStatBeans = await R.find("stat_daily", 
        " monitor_id = ? AND timestamp > ? ORDER BY timestamp", [
        monitorID,
        this.getDailyKey(now.subtract(365, "day")),
    ]);
}
```

**停更后的数据加载：**

| 图表周期 | 查询条件 | 停更 25 小时后 | 停更 31 天后 |
|---------|---------|--------------|-------------|
| 3h/6h/24h | `timestamp > now - 24h` | `stat_minutely` 中无符合条件的数据 | 空数据 |
| 1w (168h) | `timestamp > now - 30d` | 有数据（小时级） | `stat_hourly` 中无符合条件的数据 |
| 更长周期 | 天级统计 | 有数据 | 有数据（直到 `keepDataPeriodDays`） |

**图表表现：**

1. **24 小时内的周期（3h/6h/24h）：**
   - 停更超过 24 小时后，`stat_minutely` 中没有 `timestamp > now - 24h` 的数据
   - 图表可能显示为空或只有很少的数据点
   - 即使数据库中还有旧的 `stat_minutely` 数据（因清理延迟未删除），也不会被查询到

2. **1 周周期：**
   - 停更 30 天内仍可看到小时级聚合数据
   - 超过 30 天后，`stat_hourly` 查询结果为空
   - 图表显示为空白

3. **数据"老化"现象：**
   - `getDataArray()` 从"当前时间"向前追溯
   - 监控停更后，数据点的时间戳越来越"旧"
   - 最终超出查询时间窗口，图表变为空白

**实际示例：**

```
监控最后心跳：2026-05-01 12:00:00
监控间隔：60 秒

2026-05-01 13:00（停更 1 小时）：
  - 3h 图表：显示 10:00-13:00 的数据，最后 1 小时无新数据
  - 24h 图表：显示完整 24 小时数据

2026-05-02 13:00（停更 25 小时）：
  - 3h/6h/24h 图表：stat_minutely 查询 timestamp > now-24h
  - 最后一条 stat_minutely 数据在 2026-05-01 12:00
  - now-24h = 2026-05-01 13:00
  - 查询结果为空，图表显示空白

2026-05-02 13:00 查看 1w 图表：
  - 使用 stat_hourly，查询 timestamp > now-30d
  - 仍有数据，图表正常显示

2026-06-01 13:00（停更 31 天）查看 1w 图表：
  - stat_hourly 查询 timestamp > now-30d
  - 最后一条 stat_hourly 数据在 2026-05-01 12:00
  - 查询结果为空，图表显示空白
```

### 6.4 对可用性计算的影响

**`UptimeCalculator.getData()` 方法：** `server/uptime-calculator.js:563-688`

```javascript
getData(num, type = "day") {
    // 从当前时间向前追溯
    let key = this.getKey(this.getCurrentDate(), type);
    
    // 计算结束时间戳
    switch (type) {
        case "day":
            endTimestamp = key - 86400 * (num - 1);
            break;
        case "hour":
            endTimestamp = key - 3600 * (num - 1);
            break;
        case "minute":
            endTimestamp = key - 60 * (num - 1);
            break;
    }

    // 遍历时间窗口内的数据
    while (key >= endTimestamp) {
        // 从对应队列获取数据
        let data = this.dailyUptimeDataList[key];  // 或 hourly/minutely
        
        if (data) {
            total.up += data.up;
            total.down += data.down;
        }
        key -= 时间间隔;
    }

    // 无数据时的回退逻辑
    if (total.up === 0 && total.down === 0) {
        // 尝试使用 lastDailyUptimeData / lastHourlyUptimeData / lastUptimeData
        if (this.lastDailyUptimeData) {
            total = this.lastDailyUptimeData;
        }
    }
}
```

**停更后的可用性计算：**

1. **时间窗口内无数据：**
   - `total.up = 0` 且 `total.down = 0`
   - 触发回退逻辑，使用 `last*UptimeData`
   - 如果 `last*UptimeData` 也不存在，返回空结果

2. **`lastUptimeData` 的更新：** `server/uptime-calculator.js:284-294`
   ```javascript
   if (minutelyData !== this.lastUptimeData) {
       this.lastUptimeData = minutelyData;
   }
   ```
   - 仅在数据实际变化时更新
   - 监控停更后保持最后一次的值

3. **实际表现：**
   - 短时间停更：可用性计算可能使用最后已知数据
   - 长时间停更：所有数据超出时间窗口，可用性可能显示为 0 或异常值

### 6.5 完整停更时间线示例

**场景：** 监控间隔 60 秒，2026-05-01 12:00 开始停更，`keepDataPeriodDays = 365`

| 时间点 | Recent 模式 | 24h 图表 | 1w 图表 | 数据清理状态 |
|-------|------------|---------|---------|-------------|
| 5月1日 13:00 | 显示最近心跳，停更后无新数据 | 正常（含停更前数据） | 正常 | 无清理触发 |
| 5月2日 13:00 | 页面刷新后可能数据减少 | **空白**（24h 窗口内无数据） | 正常 | `stat_minutely` 旧数据仍在数据库（未触发清理） |
| 5月3日 03:14 | 定时任务清理 24h 前的非重要心跳 | 空白 | 正常 | 非重要心跳被清理 |
| 6月1日 12:00 | 数据更少（重要心跳 + 最近 100 条） | 空白 | **空白**（30d 窗口内无数据） | `stat_hourly` 旧数据仍在数据库 |
| 2027年5月2日 03:14 | 数据库中心跳已全部清理 | 空白 | 空白 | 定时任务清理所有过期数据 |

---

## 7. 文件索引

| 功能模块 | 主要文件 | 关键方法/行号 |
|---------|---------|--------------|
| 心跳模型 | `server/model/heartbeat.js` | 完整文件 |
| 聚合计算 | `server/uptime-calculator.js` | `update()`:212, `getDataArray()`:697 |
| 监控心跳 | `server/model/monitor.js` | `beat()`:421, 存储:1099 |
| 数据清理任务 | `server/jobs/clear-old-data.js` | `clearOldData()`:13 |
| 任务调度 | `server/jobs.js` | 完整文件 |
| 数据库清理 | `server/database.js` | `clearHeartbeatData()`:953 |
| 图表组件 | `src/components/PingChart.vue` | `getChartDatapointsFromStats()`:441 |
| 图表Socket | `server/socket-handlers/chart-socket-handler.js` | 完整文件 |
| 设置界面 | `src/components/settings/MonitorHistory.vue` | 完整文件 |
