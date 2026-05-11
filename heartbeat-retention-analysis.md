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

**时间周期选项：**
- `0`: 最近（实时心跳列表）
- `3h`, `6h`, `24h`: 小时级别
- `168h` (1周): 天级别

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

**特点：** 无降采样，直接使用原始心跳数据

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

## 6. 文件索引

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
