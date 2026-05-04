# 监控任务调度与 Heartbeat 状态处理机制

## 目录

1. [监控任务调度机制](#1-监控任务调度机制)
2. [Heartbeat 状态更新与判定](#2-heartbeat-状态更新与判定)
3. [重要心跳与通知机制](#3-重要心跳与通知机制)
4. [Uptime 计算与统计](#4-uptime-计算与统计)

---

## 1. 监控任务调度机制

### 1.1 核心调度器

监控任务的调度核心位于 `server/model/monitor.js` 中的 `start()` 方法。Uptime Kuma 采用**递归 setTimeout 而非 setInterval 进行调度，以确保每次监控检查完成后再计算下一次执行时间。

```javascript
// server/model/monitor.js:409-1149
```

### 1.2 调度流程

#### 1.2.1 启动流程

1. **初始化阶段**：
   - 加载上一次心跳记录（`previousBeat`）
   - 初始化重试计数器（`retries`）
   - 初始化 Prometheus 监控（如果启用）

2. **核心调度函数 `beat()`**：
   - 执行实际的监控检查
   - 更新状态
   - 存储心跳记录
   - 计算并调度下一次检查

3. **安全包装 `safeBeat()`**：
   - 捕获 `beat()` 中的异常
   - 异常时自动重启监控

#### 1.2.2 调度间隔计算

下一次调度时间通过以下公式计算：

```javascript
// server/model/monitor.js:1112
let intervalRemainingMs = Math.max(1, beatInterval * 1000 - dayjs().diff(dayjs.utc(bean.time)));
```

**调度间隔规则**：

| 状态 | 间隔来源 | 说明 |
|------|----------|------|
| UP/MAINTENANCE | `this.interval` | 正常监控间隔 |
| PENDING | `this.retryInterval`（如果 > 0） | 重试间隔，优先使用 `retryInterval` |
| PENDING | `this.interval` | 重试间隔，未设置 `retryInterval` 时使用默认间隔 |

### 1.3 特殊监控类型调度

#### 1.3.1 Push 类型监控

Push 类型监控有特殊的调度逻辑：

```javascript
// server/model/monitor.js:1141-1148
// Delay Push Type
if (this.type === "push") {
    setTimeout(() => {
        safeBeat();
    }, this.interval * 1000);
} else {
    safeBeat();
}
```

**Push 类型特点**：
- 启动时延迟 `interval * 1000` 毫秒后开始第一次检查
- 检查逻辑：判断是否在时间窗口内收到心跳
- 时间窗口计算：`beatInterval * 1000 + bufferTime`（bufferTime = 1000ms）

**Push 类型心跳检查逻辑**：

```javascript
// server/model/monitor.js:726-763
} else if (this.type === "push") {
    const bufferTime = 1000;
    
    if (previousBeat) {
        const msSinceLastBeat = dayjs.utc().valueOf() - dayjs.utc(previousBeat.time).valueOf();
        
        // 如果上一次心跳状态不是 UP（考虑 upsideDown 模式）
        // 或者超过时间窗口
        if (
            previousBeat.status !== (this.isUpsideDown() ? DOWN : UP) ||
            msSinceLastBeat > beatInterval * 1000 + bufferTime
        ) {
            bean.duration = Math.round(msSinceLastBeat / 1000);
            throw new Error("No heartbeat in the time window");
        } else {
            // 在时间窗口内，调整下一次超时时间
            let timeout = beatInterval * 1000 - msSinceLastBeat;
            if (timeout < 0) {
                timeout = bufferTime;
            } else {
                timeout += bufferTime;
            }
            // 不需要插入成功的心跳记录
            retries = 0;
            this.heartbeatInterval = setTimeout(safeBeat, timeout);
            return;
        }
    } else {
        // 第一次检查，没有历史心跳
        bean.duration = beatInterval;
        throw new Error("No heartbeat in the time window");
    }
}
```

#### 1.3.2 其他监控类型

其他类型（HTTP、Ping、Docker 等）立即开始第一次检查，然后根据检查结果决定下一次调度时间。

### 1.4 停止监控

```javascript
// server/model/monitor.js:1245-1250
async stop() {
    clearTimeout(this.heartbeatInterval);
    this.isStop = true;
    this.prometheus?.remove();
}
```

---

## 2. Heartbeat 状态更新与判定

### 2.1 状态常量定义

状态常量定义在 `src/util.js` 中：

```javascript
// src/util.js:21-24
DOWN = 0;       // 服务不可用
UP = 1;           // 服务正常
PENDING = 2;      // 重试中/待确认
MAINTENANCE = 3;  // 维护模式
```

### 2.2 状态更新流程

#### 2.2.1 初始状态设置

每次心跳开始时，创建一个新的 heartbeat bean，默认状态为 `DOWN`：

```javascript
// server/model/monitor.js:448-456
let bean = R.dispense("heartbeat");
bean.monitor_id = this.id;
bean.time = R.isoDateTimeMillis(dayjs.utc());
bean.status = DOWN;  // 默认状态
bean.downCount = previousBeat?.downCount || 0;

if (this.isUpsideDown()) {
    bean.status = flipStatus(bean.status);  // 翻转状态
}
```

#### 2.2.2 维护状态检测

在执行任何检查之前，先检测是否处于维护模式：

```javascript
// server/model/monitor.js:465-467
if (await Monitor.isUnderMaintenance(this.id)) {
    bean.msg = "Monitor under maintenance";
    bean.status = MAINTENANCE;
}
```

**维护状态检测逻辑**（`server/model/monitor.js:1630-1652`）：
- 检查当前监控是否关联了维护计划
- 检查父级监控是否处于维护模式（继承维护状态）

#### 2.2.3 监控类型检查

根据监控类型执行不同的检查逻辑：

| 监控类型 | 检查方式 | 状态设置 |
|----------|----------|--------|
| http | HTTP 请求 | 状态码验证通过 → UP |
| keyword | HTTP 请求 + 关键字匹配 | 关键字匹配 → UP |
| json-query | HTTP 请求 + JSONPath 查询 | 查询条件满足 → UP |
| ping | ICMP Ping | Ping 成功 → UP |
| push | 时间窗口检查 | 收到心跳 → UP |
| docker | Docker API | 容器运行中 → UP |
| steam | Steam API | 服务器在线 → UP |
| radius | Radius 认证 | 认证成功 → UP |
| 其他插件类型 | 插件自定义检查 | 插件返回 UP |

#### 2.2.4 Upside Down 模式

Upside Down 模式会翻转状态判断：

```javascript
// server/model/monitor.js:940-946
if (this.isUpsideDown()) {
    bean.status = flipStatus(bean.status);

    if (bean.status === DOWN) {
        throw new Error("Flip UP to DOWN");
    }
}
```

**翻转逻辑**（`src/util.js:116-124`）：
- UP → DOWN
- DOWN → UP
- 其他状态保持不变

### 2.3 重试机制

当监控检查失败时，会进入重试逻辑：

```javascript
// server/model/monitor.js:980-990
} else {
    // 通用重试逻辑适用于所有其他监控类型
    if (this.maxretries > 0 && retries < this.maxretries) {
        retries++;
        bean.status = PENDING;
    } else {
        // 继续计数重试次数（即使处于 DOWN 状态）
        retries++;
    }
}
```

#### 2.3.1 重试规则

| 条件 | 结果 |
|------|------|
| `maxretries > 0` 且 `retries < maxretries` | 状态设为 PENDING，继续重试 |
| `maxretries = 0` | 直接标记为 DOWN，不重试 |
| `retries >= maxretries` | 标记为 DOWN，停止重试 |

#### 2.3.2 JSON Query 特殊处理

JSON Query 类型有特殊的重试选项：

```javascript
// server/model/monitor.js:964-980
} else if (this.type === "json-query" && this.retry_only_on_status_code_failure) {
    // 对于启用了 retry_only_on_status_code_failure 的 json-query 监控
    // 仅在错误不是来自 JSON 查询评估时重试
    // JSON 查询错误的消息包含 "JSON query does not pass..."
    const isJsonQueryError =
        typeof error.message === "string" && error.message.includes("JSON query does not pass");

    if (isJsonQueryError) {
        // JSON 查询失败不重试，立即标记为 DOWN
        retries = 0;
    } else if (this.maxretries > 0 && retries < this.maxretries) {
        retries++;
        bean.status = PENDING;
    } else {
        retries++;
    }
}
```

**规则**：
- 如果 `retry_only_on_status_code_failure = true`：
  - 网络错误（状态码错误）：正常重试
  - JSON 查询错误：不重试，直接标记为 DOWN

---

## 3. 重要心跳与通知机制

### 3.1 重要心跳判定

重要心跳（Important Beat）是指状态发生变化的心跳，会触发特殊处理。

#### 3.1.1 `isImportantBeat()` 方法

```javascript
// server/model/monitor.js:1419-1445
static isImportantBeat(isFirstBeat, previousBeatStatus, currentBeatStatus) {
    return (
        isFirstBeat ||
        (previousBeatStatus === DOWN && currentBeatStatus === MAINTENANCE) ||
        (previousBeatStatus === UP && currentBeatStatus === MAINTENANCE) ||
        (previousBeatStatus === MAINTENANCE && currentBeatStatus === DOWN) ||
        (previousBeatStatus === MAINTENANCE && currentBeatStatus === UP) ||
        (previousBeatStatus === UP && currentBeatStatus === DOWN) ||
        (previousBeatStatus === DOWN && currentBeatStatus === UP) ||
        (previousBeatStatus === PENDING && currentBeatStatus === DOWN)
    );
}
```

#### 3.1.2 重要心跳状态转换表

| 前状态 | 后状态 | 是否重要 | 说明 |
|--------|--------|---------|------|
| 无（首次） | 任意 | ✅ 是 | 第一次心跳总是重要 |
| UP | PENDING | ❌ 否 | 正常重试中 |
| UP | DOWN | ✅ 是 | 服务从正常变为不可用 |
| UP | UP | ❌ 否 | 状态未变 |
| UP | MAINTENANCE | ✅ 是 | 进入维护模式 |
| PENDING | PENDING | ❌ 否 | 重试中 |
| PENDING | DOWN | ✅ 是 | 重试失败，确认 DOWN |
| PENDING | UP | ❌ 否 | 重试成功恢复 |
| DOWN | PENDING | 不存在 | DOWN 后不会变为 PENDING |
| DOWN | DOWN | ❌ 否 | 持续 DOWN |
| DOWN | UP | ✅ 是 | 服务恢复 |
| DOWN | MAINTENANCE | ✅ 是 | 进入维护模式 |
| MAINTENANCE | MAINTENANCE | ❌ 否 | 持续维护 |
| MAINTENANCE | UP | ✅ 是 | 维护结束恢复 |
| MAINTENANCE | DOWN | ✅ 是 | 维护结束但服务 DOWN |

### 3.2 通知触发判定

通知触发判定比重要心跳判定更严格：

```javascript
// server/model/monitor.js:1454-1477
static isImportantForNotification(isFirstBeat, previousBeatStatus, currentBeatStatus) {
    return (
        isFirstBeat ||
        (previousBeatStatus === MAINTENANCE && currentBeatStatus === DOWN) ||
        (previousBeatStatus === UP && currentBeatStatus === DOWN) ||
        (previousBeatStatus === DOWN && currentBeatStatus === UP) ||
        (previousBeatStatus === PENDING && currentBeatStatus === DOWN)
    );
}
```

#### 3.2.1 通知触发状态转换表

| 前状态 | 后状态 | 是否触发通知 | 说明 |
|--------|--------|-------------|------|
| 无（首次） | 任意 | ✅ 是 | 第一次心跳总是通知 |
| UP | DOWN | ✅ 是 | 服务故障 |
| PENDING | DOWN | ✅ 是 | 重试确认故障 |
| DOWN | UP | ✅ 是 | 服务恢复 |
| MAINTENANCE | DOWN | ✅ 是 | 维护结束但服务故障 |
| 其他转换 | 任意 | ❌ 否 | 不触发通知 |

**关键差异**：
- 进入维护模式（任意状态 → MAINTENANCE）**不触发通知**
- 从维护模式恢复到 UP（MAINTENANCE → UP）**不触发通知**

### 3.3 持续 DOWN 重发通知

当服务持续 DOWN 时，可以设置重发通知：

```javascript
// server/model/monitor.js:1024-1037
if (bean.status === DOWN && this.resendInterval > 0) {
    ++bean.downCount;
    if (bean.downCount >= this.resendInterval) {
        // 仍然 DOWN，再次发送通知
        log.debug(
            "monitor",
            `[${this.name}] sendNotification again: Down Count: ${bean.downCount} | Resend Interval: ${this.resendInterval}`
        );
        await Monitor.sendNotification(isFirstBeat, this, bean);

        // 重置 downCount
        bean.downCount = 0;
    }
}
```

**规则**：
- `resendInterval > 0` 时启用重发
- 每 `resendInterval` 次心跳后重发一次通知
- 重发后重置 `downCount`

---

## 4. Uptime 计算与统计

### 4.1 UptimeCalculator 类

Uptime 计算由 `server/uptime-calculator.js` 中的 `UptimeCalculator` 类负责。

#### 4.1.1 数据存储结构

| 统计类型 | 时间粒度 | 保留时间 | 表名 |
|---------|---------|---------|------|
| 分钟级 | 1 分钟 | 24 小时 | `stat_minutely` |
| 小时级 | 1 小时 | 30 天 | `stat_hourly` |
| 天级 | 1 天 | 365 天 | `stat_daily` |

#### 4.1.2 状态扁平化

在计算 Uptime 时，状态会被扁平化：

```javascript
// server/uptime-calculator.js:545-555
flatStatus(status) {
    switch (status) {
        case UP:
        case MAINTENANCE:
            return UP;      // 维护模式视为 UP
        case DOWN:
        case PENDING:
            return DOWN;     // PENDING 视为 DOWN
    }
    throw new Error("Invalid status");
}
```

### 4.2 心跳更新流程

每次心跳后调用 `UptimeCalculator.update()`：

```javascript
// server/model/monitor.js:1088-1090
let uptimeCalculator = await UptimeCalculator.getUptimeCalculator(this.id);
let endTimeDayjs = await uptimeCalculator.update(bean.status, parseFloat(bean.ping));
bean.end_time = R.isoDateTimeMillis(endTimeDayjs);
```

#### 4.2.1 更新逻辑

```javascript
// server/uptime-calculator.js:212-282
async update(status, ping = 0, date) {
    let flatStatus = this.flatStatus(status);

    let divisionKey = this.getMinutelyKey(date);  // 分钟级 key
    let hourlyKey = this.getHourlyKey(date);        // 小时级 key
    let dailyKey = this.getDailyKey(date);          // 天级 key

    if (status === MAINTENANCE) {
        // 维护状态单独计数
        minutelyData.maintenance = minutelyData.maintenance ? minutelyData.maintenance + 1 : 1;
        hourlyData.maintenance = hourlyData.maintenance ? hourlyData.maintenance + 1 : 1;
        dailyData.maintenance = dailyData.maintenance ? dailyData.maintenance + 1 : 1;
    } else if (flatStatus === UP) {
        // UP 状态：增加 up 计数，更新 ping 统计
        minutelyData.up += 1;
        // ... 更新 avgPing, minPing, maxPing 计算
    } else if (flatStatus === DOWN) {
        // DOWN 状态：增加 down 计数
        minutelyData.down += 1;
    }

    // 存储到数据库
    // ...
}
```

### 4.3 Uptime 计算公式

```javascript
// server/uptime-calculator.js:681-685
if (total.up + total.down === 0) {
    uptimeData.uptime = 0;
} else {
    uptimeData.uptime = total.up / (total.up + total.down);
}
```

**公式**：`Uptime = UP 次数 / (UP 次数 + DOWN 次数)`

**注意**：
- MAINTENANCE 状态在扁平化时视为 UP，但在存储时单独记录 `maintenance` 计数
- PENDING 状态视为 DOWN

### 4.4 常用统计方法

| 方法 | 说明 | 时间范围 |
|------|------|---------|
| `get24Hour()` | 24 小时 uptime | 最近 1440 分钟 |
| `get7Day()` | 7 天 uptime | 最近 168 小时 |
| `get30Day()` | 30 天 uptime | 最近 30 天 |
| `get1Year()` | 1 年 uptime | 最近 365 天 |

---

## 附录：关键代码位置速查表

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|----------|
| 监控启动 | `server/model/monitor.js` | 409-1149 |
| 心跳执行 | `server/model/monitor.js` | 421-1120 |
| 重要心跳判定 | `server/model/monitor.js` | 1419-1445 |
| 通知触发判定 | `server/model/monitor.js` | 1454-1477 |
| 发送通知 | `server/model/monitor.js` | 1486-1553 |
| 维护状态检测 | `server/model/monitor.js` | 1630-1652 |
| 状态常量 | `src/util.js` | 21-24 |
| Uptime 计算 | `server/uptime-calculator.js` | 全文 |
| 后台任务 | `server/jobs.js` | 全文 |
