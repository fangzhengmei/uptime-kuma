# Uptime Kuma Prometheus Metrics 导出分析

## 1. 整体架构

### 1.1 技术栈

- **Metrics 客户端库**: `prom-client` ~13.2.0
- **HTTP 端点中间件**: `prometheus-api-metrics` ~3.2.2
- **数据聚合引擎**: `UptimeCalculator` (自定义滑动窗口计算器)

### 1.2 初始化流程

```
server.js:237
    └── Prometheus.init()
            ├── 从数据库读取所有 tag 名称作为潜在标签
            └── 初始化所有 Gauge 类型的 metrics

server.js:337
    └── app.get("/metrics", apiAuth, prometheusAPIMetrics())
            └── 注册 /metrics 端点，使用第一个用户的 Basic Auth
```

## 2. Metrics 定义

所有 metrics 均为 **Gauge** 类型，定义在 `server/prometheus.js:62-96`。

| Metric 名称 | 类型 | 说明 | 标签 |
|------------|------|------|------|
| `monitor_cert_days_remaining` | Gauge | 证书剩余天数 | 通用标签 |
| `monitor_cert_is_valid` | Gauge | 证书是否有效 (1=有效, 0=无效) | 通用标签 |
| `monitor_uptime_ratio` | Gauge | 可用率比率 (0.0-1.0) | 通用标签 + `window` |
| `monitor_response_time_seconds` | Gauge | 平均响应时间 (秒) | 通用标签 + `window` |
| `monitor_response_time` | Gauge | 最新响应时间 (毫秒) | 通用标签 |
| `monitor_status` | Gauge | 监控状态 | 通用标签 |

### 2.1 监控状态值

`monitor_status` 的值定义:
- **1** = UP (正常)
- **0** = DOWN (故障)
- **2** = PENDING (重试中)
- **3** = MAINTENANCE (维护中)

### 2.2 滑动窗口标签

`window` 标签用于区分不同时间粒度的聚合数据:
- `1d` - 最近 24 小时
- `30d` - 最近 30 天
- `365d` - 最近 365 天

### 2.3 通用标签

每个 monitor 实例的标签包含:
```javascript
{
    monitor_id: monitor.id,           // 数字 ID
    monitor_name: monitor.name,        // 监控名称
    monitor_type: monitor.type,        // 类型: http/ping/docker 等
    monitor_url: monitor.url,          // URL (如适用)
    monitor_hostname: monitor.hostname,// 主机名 (如适用)
    monitor_port: monitor.port,        // 端口 (如适用)
    // + 自定义 tag 标签
}
```

## 3. 数据更新流程

### 3.1 整体数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                        Monitor.start()                            │
│  (monitor.js:409)                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  this.prometheus = new Prometheus(this, await this.getTags())  │
│  (monitor.js:416)                                                │
│  - 为每个 monitor 创建独立的 Prometheus 实例                      │
│  - 绑定 monitor 的标签信息                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (每次 heartbeat 循环)
┌─────────────────────────────────────────────────────────────────┐
│                         beat() 循环                               │
│  (monitor.js:421)                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │ 执行监控检查  │    │ TLS 证书检查 │    │ 状态判断    │
   │ (http/ping/ │    │             │    │ (UP/DOWN/   │
   │ docker...)  │    │             │    │ PENDING/    │
   └─────────────┘    └─────────────┘    │ MAINTENANCE)│
          │                   │            └─────────────┘
          ▼                   ▼                   │
   ┌────────────────────────────────────────────────────┐
   │              创建 Heartbeat Bean                     │
   │  bean = { monitor_id, time, status, ping, msg... } │
   └────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  UptimeCalculator 更新 (聚合历史数据)                             │
│  (monitor.js:1088-1090)                                          │
│  uptimeCalculator.update(bean.status, parseFloat(bean.ping))    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  存储到数据库 + 发送到前端                                        │
│  (monitor.js:1094-1099)                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Prometheus Metrics 更新                                          │
│  (monitor.js:1101-1105)                                          │
│  this.prometheus?.update(bean, tlsInfo, { data24h, data30d, data1y }) │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Prometheus.update() 详细逻辑

`server/prometheus.js:153-238`

```javascript
update(heartbeat, tlsInfo, uptime) {
    // 1. TLS 证书相关 metrics (实时)
    if (tlsInfo) {
        monitorCertIsValid.set(labels, tlsInfo.valid ? 1 : 0);
        monitorCertDaysRemaining.set(labels, tlsInfo.certInfo.daysRemaining);
    }
    
    // 2. 历史聚合 metrics (基于滑动窗口)
    if (uptime) {
        // 24 小时窗口
        monitorAverageResponseTimeSeconds.set({...labels, window: "1d"}, uptime.data24h.avgPing / 1000);
        monitorUptimeRatio.set({...labels, window: "1d"}, uptime.data24h.uptime);
        
        // 30 天窗口
        monitorAverageResponseTimeSeconds.set({...labels, window: "30d"}, uptime.data30d.avgPing / 1000);
        monitorUptimeRatio.set({...labels, window: "30d"}, uptime.data30d.uptime);
        
        // 365 天窗口
        monitorAverageResponseTimeSeconds.set({...labels, window: "365d"}, uptime.data1y.avgPing / 1000);
        monitorUptimeRatio.set({...labels, window: "365d"}, uptime.data1y.uptime);
    }
    
    // 3. 实时状态 metrics (基于最新 heartbeat)
    if (heartbeat) {
        monitorStatus.set(labels, heartbeat.status);
        monitorResponseTime.set(labels, heartbeat.ping ?? -1);
    }
}
```

## 4. 实时状态 vs 历史聚合的边界

### 4.1 核心分界点

| 维度 | 实时状态 | 历史聚合 |
|------|----------|----------|
| **数据来源** | 最新的单条 heartbeat 记录 | UptimeCalculator 滑动窗口聚合 |
| **时间粒度** | 瞬时值 (当前心跳) | 多时间窗口 (24h/30d/365d) |
| **更新时机** | 每次 heartbeat 后立即更新 | 每次 heartbeat 后累加聚合 |
| **Prometheus Metrics** | `monitor_status`, `monitor_response_time` | `monitor_uptime_ratio`, `monitor_response_time_seconds` |

### 4.2 实时状态数据流程

**数据路径**: `Heartbeat Bean` → `Prometheus.update()` → `prom-client` Gauge

```
┌────────────────────────────────────────────────────────────────┐
│                     单次 Heartbeat 数据                          │
│  {                                                               │
│    monitor_id: 1,                                                │
│    time: "2026-05-04T10:30:00Z",                               │
│    status: 1,     // UP                                         │
│    ping: 45,      // 45ms                                       │
│    msg: "200 - OK"                                               │
│  }                                                               │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼ 直接映射
┌────────────────────────────────────────────────────────────────┐
│                    Prometheus Gauge 瞬时值                       │
│                                                                  │
│  monitor_status{monitor_id="1",...}          = 1  (UP)        │
│  monitor_response_time{monitor_id="1",...}   = 45 (毫秒)      │
│                                                                  │
│  ⚠️ 这些值会被下一次 heartbeat 完全覆盖                          │
└────────────────────────────────────────────────────────────────┘
```

**关键代码位置**:
- `monitor.js:1102-1105`: 调用 `prometheus.update()` 时传入 `bean` (最新 heartbeat)
- `prometheus.js:218-237`: 直接使用 `heartbeat.status` 和 `heartbeat.ping`

### 4.3 历史聚合数据流程

**数据路径**: `Heartbeat Bean` → `UptimeCalculator.update()` → 三级时间桶 → `getData()` → `Prometheus.update()`

```
┌────────────────────────────────────────────────────────────────┐
│                     单次 Heartbeat 数据                          │
│  { status: 1, ping: 45, time: "2026-05-04T10:30:00Z" }       │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              UptimeCalculator.update() 聚合逻辑                  │
│  (uptime-calculator.js:212-374)                                 │
│                                                                  │
│  1. 计算时间桶 Key:                                              │
│     - minutelyKey = 按分钟对齐 (2026-05-04T10:30:00)           │
│     - hourlyKey   = 按小时对齐 (2026-05-04T10:00:00)           │
│     - dailyKey    = 按天对齐   (2026-05-04T00:00:00)           │
│                                                                  │
│  2. 更新对应时间桶的计数器:                                       │
│     if (status === UP) {                                         │
│         minutelyData.up += 1;                                    │
│         minutelyData.avgPing = 滑动平均计算;                     │
│         hourlyData.up += 1;                                      │
│         dailyData.up += 1;                                       │
│     } else if (status === DOWN) {                                │
│         minutelyData.down += 1;                                  │
│         hourlyData.down += 1;                                    │
│         dailyData.down += 1;                                     │
│     }                                                             │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                    三级时间桶数据结构                              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ minutelyUptimeDataList (LimitQueue, 容量: 1440 = 24×60) │  │
│  │ 每个元素: { up, down, avgPing, minPing, maxPing }          │  │
│  │ 保留时间: 最近 24 小时                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ hourlyUptimeDataList (LimitQueue, 容量: 720 = 30×24)    │  │
│  │ 每个元素: { up, down, avgPing, minPing, maxPing }          │  │
│  │ 保留时间: 最近 30 天                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ dailyUptimeDataList (LimitQueue, 容量: 365)              │  │
│  │ 每个元素: { up, down, avgPing, minPing, maxPing }          │  │
│  │ 保留时间: 最近 365 天                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                      持久化到数据库                               │
│                                                                  │
│  - stat_minutely 表: 保存 minutely 数据 (保留 24 小时)          │
│  - stat_hourly 表:  保存 hourly 数据  (保留 30 天)              │
│  - stat_daily 表:   保存 daily 数据   (保留 365 天)             │
│                                                                  │
│  ⚠️ 数据清理: 超过保留期的数据会被自动删除                        │
│     (uptime-calculator.js:361-370)                              │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              计算滑动窗口统计值 (getData 系列方法)                 │
│  (uptime-calculator.js:563-829)                                 │
│                                                                  │
│  get24Hour()  → getData(1440, "minute")                        │
│    - 遍历最近 1440 个分钟桶                                      │
│    - 累加 up + down 计数                                         │
│    - 计算: uptime = up / (up + down)                            │
│    - 计算: avgPing = (sum avgPing * up) / total up             │
│                                                                  │
│  get30Day()   → getData(30, "day")                              │
│  get1Year()   → getData(365, "day")                             │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                    输出到 Prometheus                              │
│                                                                  │
│  monitor_uptime_ratio{window="1d"}    = 0.995                 │
│  monitor_uptime_ratio{window="30d"}   = 0.998                 │
│  monitor_uptime_ratio{window="365d"}  = 0.999                 │
│                                                                  │
│  monitor_response_time_seconds{window="1d"}  = 0.045           │
│  monitor_response_time_seconds{window="30d"} = 0.042           │
└────────────────────────────────────────────────────────────────┘
```

### 4.4 边界示意图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              Prometheus Metrics                                │
├──────────────────────────────────┬───────────────────────────────────────────┤
│         实时状态 (Realtime)       │           历史聚合 (Aggregated)           │
├──────────────────────────────────┼───────────────────────────────────────────┤
│                                  │                                           │
│   monitor_status                 │   monitor_uptime_ratio{window="1d"}      │
│   monitor_response_time          │   monitor_uptime_ratio{window="30d"}     │
│   monitor_cert_days_remaining    │   monitor_uptime_ratio{window="365d"}    │
│   monitor_cert_is_valid          │                                           │
│                                  │   monitor_response_time_seconds{window="1d"}│
│                                  │   monitor_response_time_seconds{window="30d"}│
│                                  │   monitor_response_time_seconds{window="365d"}│
│                                  │                                           │
├──────────────────────────────────┼───────────────────────────────────────────┤
│       数据来源: 最新 heartbeat    │       数据来源: UptimeCalculator 滑动窗口   │
│                                  │                                           │
│   ┌──────────────┐              │   ┌─────────────────────────────────┐    │
│   │ Heartbeat    │              │   │   [分钟桶] [分钟桶] ... [分钟桶] │    │
│   │ (单条记录)   │              │   │   最近 1440 个 = 24 小时        │    │
│   │              │              │   └─────────────────────────────────┘    │
│   │ status: 1    │              │   ┌─────────────────────────────────┐    │
│   │ ping: 45ms   │              │   │   [小时桶] [小时桶] ... [小时桶] │    │
│   └──────────────┘              │   │   最近 720 个 = 30 天           │    │
│                                  │   └─────────────────────────────────┘    │
│                                  │   ┌─────────────────────────────────┐    │
│                                  │   │   [天桶] [天桶] ... [天桶]       │    │
│                                  │   │   最近 365 个 = 1 年             │    │
│                                  │   └─────────────────────────────────┘    │
│                                  │                                           │
├──────────────────────────────────┼───────────────────────────────────────────┤
│   语义: 当前监控的瞬时状态         │   语义: 过去一段时间的统计趋势            │
│                                  │                                           │
│   用途: 告警触发 (DOWN/PENDING)  │   用途: SLA 计算、长期趋势分析             │
│                                  │                                           │
└──────────────────────────────────┴───────────────────────────────────────────┘
```

## 5. UptimeCalculator 深入分析

### 5.1 初始化时的数据加载

`uptime-calculator.js:123-203`

```javascript
async init(monitorID) {
    // 1. 从 stat_minutely 加载最近 24 小时数据
    let minutelyStatBeans = await R.find(
        "stat_minutely", 
        "monitor_id = ? AND timestamp > ? ORDER BY timestamp",
        [monitorID, getMinutelyKey(now.subtract(24, "hour"))]
    );
    
    // 2. 从 stat_hourly 加载最近 30 天数据
    let hourlyStatBeans = await R.find(
        "stat_hourly", 
        "monitor_id = ? AND timestamp > ? ORDER BY timestamp",
        [monitorID, getHourlyKey(now.subtract(30, "day"))]
    );
    
    // 3. 从 stat_daily 加载最近 365 天数据
    let dailyStatBeans = await R.find(
        "stat_daily", 
        "monitor_id = ? AND timestamp > ? ORDER BY timestamp",
        [monitorID, getDailyKey(now.subtract(365, "day"))]
    );
}
```

**设计意图**: 服务重启后能快速恢复历史聚合状态，无需从头遍历所有 heartbeat 记录。

### 5.2 时间桶对齐策略

| 粒度 | 对齐方式 | Key 计算 | 示例 |
|------|----------|----------|------|
| 分钟级 | 开始 of minute | `date.startOf("minute").unix()` | 2026-05-04T10:30:00 |
| 小时级 | 开始 of hour | `date.startOf("hour").unix()` | 2026-05-04T10:00:00 |
| 天级 | 开始 of day (UTC) | `date.utc().startOf("day").unix()` | 2026-05-04T00:00:00 |

**注意**: 天级使用 UTC 时间，避免时区切换导致统计异常。

### 5.3 滑动平均计算逻辑

`uptime-calculator.js:240-277`

```javascript
// 只有 UP 状态才更新 ping 统计
if (flatStatus === UP && !isNaN(ping)) {
    // 分钟级平均
    if (minutelyData.up === 1) {
        // 该分钟的第一个 UP 心跳
        minutelyData.avgPing = ping;
        minutelyData.minPing = ping;
        minutelyData.maxPing = ping;
    } else {
        // 后续心跳: 增量平均
        minutelyData.avgPing = (minutelyData.avgPing * (minutelyData.up - 1) + ping) 
                               / minutelyData.up;
        minutelyData.minPing = Math.min(minutelyData.minPing, ping);
        minutelyData.maxPing = Math.max(minutelyData.maxPing, ping);
    }
    
    // 小时级、天级同理...
}
```

**特点**: 使用增量平均算法，无需保存所有历史 ping 值，节省内存。

## 6. 关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Prometheus 初始化 | `server/server.js` | 237 |
| /metrics 路由注册 | `server/server.js` | 337 |
| Metrics 定义 | `server/prometheus.js` | 62-96 |
| Metrics 更新逻辑 | `server/prometheus.js` | 153-238 |
| Monitor 中创建 Prometheus 实例 | `server/model/monitor.js` | 416 |
| Monitor 中调用 prometheus.update | `server/model/monitor.js` | 1101-1105 |
| UptimeCalculator 初始化 | `server/uptime-calculator.js` | 123-203 |
| UptimeCalculator 更新聚合 | `server/uptime-calculator.js` | 212-374 |
| 24 小时统计获取 | `server/uptime-calculator.js` | 806-808 |
| 30 天统计获取 | `server/uptime-calculator.js` | 820-822 |
| 365 天统计获取 | `server/uptime-calculator.js` | 827-829 |

## 7. 总结

### 7.1 核心设计原则

1. **分离关注点**:
   - 实时状态: 直接反映最新心跳结果，用于快速告警
   - 历史聚合: 基于滑动窗口的统计值，用于 SLA 和趋势分析

2. **三级时间粒度**:
   - 分钟级 (24h): 细粒度短期趋势
   - 小时级 (30d): 中期统计
   - 天级 (365d): 长期 SLA 计算

3. **内存高效**:
   - 使用 LimitQueue 固定容量队列
   - 增量平均算法，不存储全量历史
   - 数据持久化到数据库，支持服务重启恢复

### 7.2 边界清晰的好处

1. **告警准确性**: 实时状态直接触发告警，避免聚合延迟
2. **统计稳定性**: 历史聚合使用滑动窗口，抗单点波动
3. **资源隔离**: 实时路径和聚合路径独立，互不影响
4. **Prometheus 查询友好**: 不同用途的 metrics 有明确的命名和标签区分
