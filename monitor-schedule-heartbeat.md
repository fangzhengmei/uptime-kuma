# 监控任务调度与 Heartbeat 状态处理机制

## 目录

1. [监控任务调度机制](#1-监控任务调度机制)
2. [外部心跳上报接口链路](#2-外部心跳上报接口链路)
3. [请求参数解析与校验](#3-请求参数解析与校验)
4. [Heartbeat 状态更新与判定](#4-heartbeat-状态更新与判定)
5. [重要心跳与通知机制](#5-重要心跳与通知机制)
6. [Uptime 计算与统计写入](#6-uptime-计算与统计写入)
7. [两种模式执行顺序对比](#7-两种模式执行顺序对比)
8. [完整时序图](#8-完整时序图)
9. [附录：关键代码位置速查表](#9-附录关键代码位置速查表)

---

## 1. 监控任务调度机制

### 1.1 核心调度器

监控任务的调度核心位于 `server/model/monitor.js` 中的 `start()` 方法。Uptime Kuma 采用**递归 setTimeout** 而非 setInterval 进行调度，以确保每次监控检查完成后再计算下一次执行时间。

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

### 1.3 Push 类型监控

Push 类型有特殊的调度逻辑：
- 启动时延迟 `interval * 1000` 毫秒后开始第一次检查
- 检查逻辑：判断是否在时间窗口内收到心跳
- 时间窗口计算：`beatInterval * 1000 + bufferTime`（bufferTime = 1000ms）

---

## 2. 请求参数解析与校验

### 2.1 HTTP 类型参数解析

HTTP 类型（包括 http、keyword、json-query）的参数解析在 `server/model/monitor.js` 的 `beat()` 函数中进行。

#### 2.1.1 参数解析流程

1. **基础超时处理**：
   - 如果 `timeout <= 0`，设置为 `interval * 1000 * 0.8`

2. **认证方式处理**：
   - **Basic Auth**：构建 `Authorization: Basic ...` 头
   - **OAuth2 Client Credentials**：自动获取/刷新 token，构建 `Authorization: Bearer ...` 头

3. **IP 协议族选择**：
   - `ipFamily = "ipv4"` → `agentFamily = 4`
   - `ipFamily = "ipv6"` → `agentFamily = 6`

4. **请求 Body 解析与校验**：
   - **JSON 格式（默认）**：尝试 `JSON.parse()`，失败抛出错误
   - **Form 格式**：直接使用原始字符串，`Content-Type: application/x-www-form-urlencoded`
   - **XML 格式**：直接使用原始字符串，`Content-Type: text/xml; charset=utf-8`

5. **Axios 选项构建**：
   - `validateStatus` 自定义函数，使用 `checkStatusCode()` 校验

6. **代理配置**：
   - 加载 proxy 配置，创建自定义 HTTP/HTTPS Agent
   - 禁用 Axios 内置代理

7. **mTLS 配置**：
   - 将 `tlsCert`、`tlsCa`、`tlsKey` 转为 Buffer 后设置到 Agent

### 2.2 状态码校验机制

#### 2.2.1 `checkStatusCode` 函数

```javascript
// server/util-server.js:552-585
exports.checkStatusCode = function (status, acceptedCodes) {
    if (acceptedCodes == null || acceptedCodes.length === 0) {
        return false;
    }

    for (const codeRange of acceptedCodes) {
        const codeRangeSplit = codeRange.split("-").map((string) => parseInt(string));
        
        // 单个状态码，如 "200"
        if (codeRangeSplit.length === 1) {
            if (status === codeRangeSplit[0]) {
                return true;
            }
        } 
        // 状态码范围，如 "200-299"
        else {
            if (status >= codeRangeSplit[0] && status <= codeRangeSplit[1]) {
                return true;
            }
        }
    }
    
    return false;
};
```

#### 2.2.2 状态码校验规则

| 配置格式 | 示例 | 说明 |
|---------|------|------|
| 单个状态码 | `"200"` | 精确匹配 |
| 状态码范围 | `"200-299"` | 区间匹配（包含边界） |
| 多个规则 | `["200", "400-404"]` | 任一匹配即通过 |

**默认配置**：通常为 `["200-299"]`，表示所有 2xx 状态码都视为成功。

#### 2.2.3 在 Axios 中的使用

```javascript
// server/model/monitor.js:557-559
validateStatus: (status) => {
    return checkStatusCode(status, this.getAcceptedStatuscodes());
},
```

**工作原理**：
- Axios 默认将 2xx 状态码视为成功，其他状态码抛出错误
- 通过 `validateStatus` 自定义成功判定逻辑
- 如果 `checkStatusCode` 返回 `true`，Axios 认为请求成功
- 如果返回 `false`，Axios 抛出错误，进入 catch 块

---

## 3. Heartbeat 状态更新与判定

### 3.1 状态常量定义

```javascript
// src/util.js:21-24
DOWN = 0;       // 服务不可用
UP = 1;         // 服务正常
PENDING = 2;    // 重试中/待确认
MAINTENANCE = 3; // 维护模式
```

### 3.2 完整状态判定流程

#### 3.2.1 状态判定六阶段

**阶段 1：初始化状态（初始翻转）**

```javascript
bean.status = DOWN;  // 默认状态

if (this.isUpsideDown()) {
    bean.status = flipStatus(bean.status);  // DOWN → UP
}
```

注意：此处翻转是为了后续逻辑的一致性，真正的状态判定在检查完成后进行。

**阶段 2：维护模式检测**

```javascript
if (await Monitor.isUnderMaintenance(this.id)) {
    bean.msg = "Monitor under maintenance";
    bean.status = MAINTENANCE;  // 直接设为维护状态
    // 跳过实际监控检查
}
```

维护状态检测逻辑：
1. 检查当前监控是否关联了维护计划
2. 检查父级监控是否处于维护模式（继承维护状态）

**阶段 3：执行监控检查**

根据监控类型执行不同检查：
- **HTTP 类型**：状态码校验 → 关键字检查 → JSON查询 → 设为 UP
- **Ping 类型**：ICMP Ping → 计算延迟 → 设为 UP
- **插件类型**：调用插件 → 获取状态 → 设为 UP

成功：设置 `bean.status = UP`
失败：抛出错误 → 进入 catch 块

**阶段 4：检查后翻转（Upside Down 模式）**

```javascript
if (this.isUpsideDown()) {
    bean.status = flipStatus(bean.status);

    if (bean.status === DOWN) {
        // 翻转后变为 DOWN，抛出错误
        // 这样会进入 catch 块，触发重试逻辑
        throw new Error("Flip UP to DOWN");
    }
}
```

**翻转逻辑（`flipStatus`）**：
- UP ↔ DOWN
- PENDING、MAINTENANCE 保持不变

**目的**：
- 正常检查成功（UP）→ 翻转后变为 DOWN → 抛出错误 → 视为"故障"
- 正常检查失败（DOWN）→ 翻转后变为 UP → 视为"正常"
- 适用于监控"故障场景"，例如监控某个服务是否已停止

**阶段 5：异常处理与重试机制（catch 块）**

```javascript
catch (error) {
    // 1. 设置错误消息
    if (error?.name === "CanceledError") {
        bean.msg = `timeout by AbortSignal (${this.timeout}s)`;
    } else {
        bean.msg = error.message;
    }

    // 2. 保存错误响应（如果启用）
    if (this.getSaveErrorResponse() && error?.response?.data !== undefined) {
        await this.saveResponseData(bean, error.response.data);
    }

    // 3. 状态判定
    
    // 情况 A: Upside Down 模式且当前状态是 UP
    // 这意味着原本是 DOWN，翻转后变为 UP，不需要重试
    if (this.isUpsideDown() && bean.status === UP) {
        retries = 0;  // 重置重试计数
    }
    
    // 情况 B: JSON Query 类型的特殊处理
    else if (this.type === "json-query" && this.retry_only_on_status_code_failure) {
        const isJsonQueryError =
            typeof error.message === "string" && 
            error.message.includes("JSON query does not pass");

        if (isJsonQueryError) {
            // JSON 查询失败不重试，立即标记为 DOWN
            retries = 0;
        } else if (this.maxretries > 0 && retries < this.maxretries) {
            // 网络错误（状态码错误）：正常重试
            retries++;
            bean.status = PENDING;
        } else {
            // 超过最大重试次数：保持 DOWN
            retries++;
        }
    }
    
    // 情况 C: 通用重试逻辑（所有其他类型）
    else {
        if (this.maxretries > 0 && retries < this.maxretries) {
            retries++;
            bean.status = PENDING;
        } else {
            // 超过最大重试次数：保持 DOWN
            retries++;
        }
    }
}
```

**阶段 6：最终状态确定**

```javascript
bean.retries = retries;  // 保存当前重试次数
```

此时 `bean.status` 可能的值：
- **UP**：检查成功，或 Upside Down 模式下检查失败
- **DOWN**：检查失败且无需重试，或超过最大重试次数
- **PENDING**：检查失败但可重试
- **MAINTENANCE**：处于维护模式

#### 3.2.2 状态判定决策表

**正常模式（非 Upside Down）**：

| 检查结果 | maxretries | retries 状态 | 最终状态 | 说明 |
|---------|------------|-------------|---------|------|
| 成功 | 任意 | 任意 | UP | 直接成功 |
| 失败 | 0 | 0 | DOWN | 不重试，直接失败 |
| 失败 | 3 | 0 | PENDING | 第1次重试 |
| 失败 | 3 | 1 | PENDING | 第2次重试 |
| 失败 | 3 | 2 | PENDING | 第3次重试 |
| 失败 | 3 | 3 | DOWN | 超过重试次数，确认失败 |

**Upside Down 模式**：

| 原始检查结果 | 翻转后状态 | 行为 | 说明 |
|-------------|-----------|------|------|
| 成功 (UP) | DOWN | 抛出错误，进入 catch | 视为"失败" |
| 失败 (DOWN) | UP | 正常继续，不重试 | 视为"成功" |

**注意**：Upside Down 模式下：
- `maxretries` 和重试机制仍然有效
- 如果原始检查成功（翻转后为 DOWN），会触发重试逻辑
- 如果原始检查失败（翻转后为 UP），不会触发重试

### 3.3 重试机制详解

#### 3.3.1 重试计数器生命周期

1. **初始化阶段**：
   ```javascript
   let retries = 0;
   // 如果有历史心跳，恢复重试计数
   if (previousBeat) {
       retries = previousBeat.retries;  // 恢复
   }
   ```

2. **成功时重置**：
   ```javascript
   // try 块中，检查成功后
   retries = 0;  // 重置重试计数
   ```

3. **失败时递增**：
   ```javascript
   // catch 块中
   if (this.maxretries > 0 && retries < this.maxretries) {
       retries++;                   // 递增
       bean.status = PENDING;        // 设为 PENDING
   } else {
       retries++;                   // 继续递增（即使 DOWN）
   }
   
   // 保存到 heartbeat bean
   bean.retries = retries;
   ```

4. **持久化存储**：
   ```javascript
   // 心跳记录存储到数据库
   await R.store(bean);  // retries 字段被持久化
   
   // 下次启动时恢复
   previousBeat = await R.findOne(...);
   retries = previousBeat.retries;
   ```

#### 3.3.2 PENDING 状态的特殊性质

**PENDING 状态与 UP/DOWN 的区别**：

| 特性 | PENDING | UP | DOWN |
|-----|---------|-----|------|
| 是否触发通知 | 否 | 状态变化时 | 状态变化时 |
| 是否计入 Uptime | 视为 DOWN | 视为 UP | 视为 DOWN |
| 是否影响重试间隔 | 是（使用 retryInterval） | 否 | 否 |
| 是否显示为"重要"心跳 | 否（除非 PENDING→DOWN） | 状态变化时是 | 状态变化时是 |

**PENDING 状态的调度间隔**：

```javascript
// server/model/monitor.js:1070-1073
} else if (bean.status === PENDING) {
    if (this.retryInterval > 0) {
        beatInterval = this.retryInterval;  // 使用重试间隔
    }
}
```

**PENDING → DOWN 的转换条件**：

```javascript
// 条件：retries >= maxretries
if (this.maxretries > 0 && retries < this.maxretries) {
    // 继续重试，保持 PENDING
} else {
    // 超过重试次数，变为 DOWN
    // retries 继续递增，但状态不再改变
}
```

---

## 4. 重要心跳与通知机制

### 4.1 重要心跳判定

重要心跳（Important Beat）是指状态发生变化的心跳，会触发特殊处理。

#### 4.1.1 `isImportantBeat()` 方法

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

#### 4.1.2 重要心跳状态转换表

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
| DOWN | DOWN | ❌ 否 | 持续 DOWN |
| DOWN | UP | ✅ 是 | 服务恢复 |
| DOWN | MAINTENANCE | ✅ 是 | 进入维护模式 |
| MAINTENANCE | MAINTENANCE | ❌ 否 | 持续维护 |
| MAINTENANCE | UP | ✅ 是 | 维护结束恢复 |
| MAINTENANCE | DOWN | ✅ 是 | 维护结束但服务 DOWN |

### 4.2 通知触发判定

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

#### 4.2.1 通知触发状态转换表

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

### 4.3 通知触发流程

#### 4.3.1 通知触发时序

```
阶段 1: 重要心跳判定
  let isImportant = Monitor.isImportantBeat(isFirstBeat, previousBeat?.status, bean.status);

阶段 2: 分支处理

  分支 A: isImportant = true
    2a. 标记 important
        bean.important = true;
    
    2b. 通知判定
        if (Monitor.isImportantForNotification(...)) {
            2c. 发送通知
                await Monitor.sendNotification(isFirstBeat, this, bean);
        }
    
    2d. 重置 downCount
        bean.downCount = 0;
    
    2e. 清除缓存
        apicache.clear();

  分支 B: isImportant = false
    2f. 检查持续 DOWN 重发通知
        if (bean.status === DOWN && this.resendInterval > 0) {
            ++bean.downCount;
            if (bean.downCount >= this.resendInterval) {
                // 重发通知
                await Monitor.sendNotification(isFirstBeat, this, bean);
                bean.downCount = 0;
            }
        }
```

#### 4.3.2 持续 DOWN 重发通知

```javascript
// server/model/monitor.js:1024-1037
if (bean.status === DOWN && this.resendInterval > 0) {
    ++bean.downCount;
    if (bean.downCount >= this.resendInterval) {
        // 仍然 DOWN，再次发送通知
        await Monitor.sendNotification(isFirstBeat, this, bean);
        
        // 重置 downCount
        bean.downCount = 0;
    }
}
```

**重发规则**：
- `resendInterval > 0` 时启用重发
- 每 `resendInterval` 次心跳后重发一次通知
- 重发后重置 `downCount`

#### 4.3.3 `sendNotification` 方法详情

```javascript
// server/model/monitor.js:1486-1553
static async sendNotification(isFirstBeat, monitor, bean) {
    // 首次心跳且状态不是 DOWN 时不发送
    if (!isFirstBeat || bean.status === DOWN) {
        // 获取通知配置列表
        const notificationList = await Monitor.getNotificationList(monitor);

        // 构建消息文本
        let text = bean.status === UP ? "✅ Up" : "🔴 Down";
        let msg = `[${monitor.name}] [${text}] ${bean.msg}`;

        // 准备通知数据
        const heartbeatJSON = await bean.toJSONAsync({ decodeResponse: true });

        // 服务恢复时计算停机时间
        if (bean.status === UP && monitor.id) {
            try {
                // 查询最近一次重要的 DOWN 心跳（状态转换点）
                const lastDownHeartbeat = await R.getRow(
                    "SELECT time FROM heartbeat WHERE monitor_id = ? AND status = ? AND important = 1 ORDER BY time DESC LIMIT 1",
                    [monitor.id, DOWN]
                );
                if (lastDownHeartbeat && lastDownHeartbeat.time) {
                    heartbeatJSON["lastDownTime"] = lastDownHeartbeat.time;
                }
            } catch (error) {
                // 静默失败
            }
        }

        // 遍历所有通知方式发送
        for (let notification of notificationList) {
            try {
                await Notification.send(
                    JSON.parse(notification.config),
                    msg,
                    monitor.toJSON(preloadData, false),
                    heartbeatJSON
                );
            } catch (e) {
                log.error("monitor", "Cannot send notification to " + notification.name);
            }
        }
    }
}
```

**通知内容**：
- 状态图标：`✅ Up` 或 `🔴 Down`
- 监控名称
- 详细消息（来自 `bean.msg`）
- 心跳时间（服务器时区和本地时间）
- 服务恢复时：上次故障时间（`lastDownTime`）

---

## 5. Uptime 计算与统计写入

### 5.1 UptimeCalculator 类

Uptime 计算由 `server/uptime-calculator.js` 中的 `UptimeCalculator` 类负责。

#### 5.1.1 数据存储结构

| 统计类型 | 时间粒度 | 保留时间 | 表名 |
|---------|---------|---------|------|
| 分钟级 | 1 分钟 | 24 小时 | `stat_minutely` |
| 小时级 | 1 小时 | 30 天 | `stat_hourly` |
| 天级 | 1 天 | 365 天 | `stat_daily` |

#### 5.1.2 状态扁平化

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

### 5.2 统计写入流程

#### 5.2.1 统计写入时序

**阶段 1：获取 UptimeCalculator 实例**

```javascript
let uptimeCalculator = await UptimeCalculator.getUptimeCalculator(this.id);

// getUptimeCalculator 内部逻辑:
if (!UptimeCalculator.list[monitorID]) {
    UptimeCalculator.list[monitorID] = new UptimeCalculator();
    await UptimeCalculator.list[monitorID].init(monitorID);
    // init(): 从数据库加载历史统计数据
}
```

**阶段 2：更新统计数据**

```javascript
let endTimeDayjs = await uptimeCalculator.update(bean.status, parseFloat(bean.ping));

// update() 内部详细流程:

// 2.1 状态扁平化
let flatStatus = this.flatStatus(status);

// 2.2 计算时间槽 key
let divisionKey = this.getMinutelyKey(date);  // 分钟级
let hourlyKey = this.getHourlyKey(date);        // 小时级
let dailyKey = this.getDailyKey(date);          // 天级

// 2.3 根据状态更新计数

// 维护模式：单独计数
if (status === MAINTENANCE) {
    minutelyData.maintenance = minutelyData.maintenance ? minutelyData.maintenance + 1 : 1;
    // ... 同样更新 hourly 和 daily
}

// UP 状态：增加 up 计数，更新 ping 统计
else if (flatStatus === UP) {
    minutelyData.up += 1;
    
    // 更新 ping 统计（仅 UP 状态有效）
    if (!isNaN(ping)) {
        if (minutelyData.up === 1) {
            // 该分钟第一次：直接赋值
            minutelyData.avgPing = ping;
            minutelyData.minPing = ping;
            minutelyData.maxPing = ping;
        } else {
            // 该分钟后续：计算平均值
            minutelyData.avgPing = (minutelyData.avgPing * (minutelyData.up - 1) + ping) / minutelyData.up;
            minutelyData.minPing = Math.min(minutelyData.minPing, ping);
            minutelyData.maxPing = Math.max(minutelyData.maxPing, ping);
        }
    }
    // ... 同样更新 hourly 和 daily
}

// DOWN 状态：增加 down 计数
else if (flatStatus === DOWN) {
    minutelyData.down += 1;
    // ... 同样更新 hourly 和 daily
}

// 2.4 存储到数据库

// 天级统计（总是存储）
let dailyStatBean = await this.getDailyStatBean(dailyKey);
dailyStatBean.up = dailyData.up;
dailyStatBean.down = dailyData.down;
dailyStatBean.ping = dailyData.avgPing;
dailyStatBean.pingMin = dailyData.minPing;
dailyStatBean.pingMax = dailyData.maxPing;
await R.store(dailyStatBean);

// 小时级统计（最近 30 天）
if (date.isAfter(currentDate.subtract(this.statHourlyKeepDay, "day"))) {
    let hourlyStatBean = await this.getHourlyStatBean(hourlyKey);
    // ... 更新字段
    await R.store(hourlyStatBean);
}

// 分钟级统计（最近 24 小时）
if (date.isAfter(currentDate.subtract(this.statMinutelyKeepHour, "hour"))) {
    let minutelyStatBean = await this.getMinutelyStatBean(divisionKey);
    // ... 更新字段
    await R.store(minutelyStatBean);
}

// 2.5 清理过期数据
if (!this.migrationMode) {
    // 删除超过 24 小时的分钟级统计
    await R.exec("DELETE FROM stat_minutely WHERE monitor_id = ? AND timestamp < ?", [...]);
    
    // 删除超过 30 天的小时级统计
    await R.exec("DELETE FROM stat_hourly WHERE monitor_id = ? AND timestamp < ?", [...]);
}
```

**阶段 3：心跳记录存储**

```javascript
bean.end_time = R.isoDateTimeMillis(endTimeDayjs);
await R.store(bean);  // 存储 heartbeat 记录
```

**阶段 4：前端推送**

```javascript
// 发送心跳事件到前端
io.to(this.user_id).emit("heartbeat", bean.toJSON());

// 发送统计数据（24h、30d、1y uptime 和 avgPing）
Monitor.sendStats(io, this.id, this.user_id);

// sendStats 内部:
let data24h = await uptimeCalculator.get24Hour();
io.to(userID).emit("avgPing", monitorID, data24h.avgPing);
io.to(userID).emit("uptime", monitorID, 24, data24h.uptime);
// ... 同样发送 30d 和 1y 的数据
```

**阶段 5：Prometheus 指标更新（如果启用）**

```javascript
const data24h = uptimeCalculator.get24Hour();
const data30d = uptimeCalculator.get30Day();
const data1y = uptimeCalculator.get1Year();

this.prometheus?.update(bean, tlsInfo, {
    data24h, data30d, data1y
});
```

### 5.3 Uptime 计算公式

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

### 5.4 常用统计方法

| 方法 | 说明 | 时间范围 |
|------|------|---------|
| `get24Hour()` | 24 小时 uptime | 最近 1440 分钟 |
| `get7Day()` | 7 天 uptime | 最近 168 小时 |
| `get30Day()` | 30 天 uptime | 最近 30 天 |
| `get1Year()` | 1 年 uptime | 最近 365 天 |

---

## 6. 完整时序图

### 6.1 单次心跳完整时序

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           单次心跳完整时序图                                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │ 调度器    │    │ 参数解析  │    │ 状态判定  │    │ 通知处理  │    │ 统计存储  │       │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘       │
│       │                │                │                │                │              │
│       │  1. 启动 beat()│                │                │                │              │
│       │───────────────>│                │                │                │              │
│       │                │                │                │                │              │
│       │                │  2. 初始化状态 │                │                │              │
│       │                │  bean.status=  │                │                │              │
│       │                │  DOWN (+翻转)  │                │                │              │
│       │                │                │                │                │              │
│       │                │  3. 维护检测  │                │                │              │
│       │                │  isUnder-     │                │                │              │
│       │                │  Maintenance? │                │                │              │
│       │                │       │        │                │                │              │
│       │                │      是│否     │                │                │              │
│       │                │       └───────>│                │                │              │
│       │                │                │                │                │              │
│       │                │                │  4. 执行检查  │                │              │
│       │                │                │  (HTTP/Ping/  │                │              │
│       │                │                │   插件等)     │                │              │
│       │                │                │       │        │                │              │
│       │                │                │     成│功     │                │              │
│       │                │                │       └───────>│                │              │
│       │                │                │                │                │              │
│       │                │                │  5. 检查后翻转 │                │              │
│       │                │                │  (Upside Down) │                │              │
│       │                │                │  bean.status = │                │              │
│       │                │                │  flipStatus(..)│                │              │
│       │                │                │       │        │                │              │
│       │                │                │   翻转后│ DOWN？│                │              │
│       │                │                │       └────────>│                │              │
│       │                │                │                │                │              │
│       │                │                │  6. 重试判定  │                │              │
│       │                │                │  maxretries>0  │                │              │
│       │                │                │  && retries <  │                │              │
│       │                │                │  maxretries?   │                │              │
│       │                │                │       │        │                │              │
│       │                │                │      是│否     │                │              │
│       │                │                │       │        │                │              │
│       │                │                │  PENDING│DOWN  │                │              │
│       │                │                │       │        │                │              │
│       │                │                │       └───────>│                │              │
│       │                │                │                │                │              │
│       │                │                │                │  7. 重要心跳   │              │
│       │                │                │                │  isImportant?  │              │
│       │                │                │                │       │        │              │
│       │                │                │                │      是│否     │              │
│       │                │                │                │       │        │              │
│       │                │                │                │  发送 │检查   │              │
│       │                │                │                │  通知 │持续   │              │
│       │                │                │                │       │DOWN?  │              │
│       │                │                │                │       │        │              │
│       │                │                │                │       └───────>│              │
│       │                │                │                │                │              │
│       │                │                │                │                │  8. 更新统计 │
│       │                │                │                │                │  UptimeCalc │
│       │                │                │                │                │  .update()   │
│       │                │                │                │                │              │
│       │                │                │                │                │  9. 存储心跳 │
│       │                │                │                │                │  R.store()   │
│       │                │                │                │                │              │
│       │                │                │                │                │  10. 推送前端│
│       │                │                │                │                │  Socket.io   │
│       │                │                │                │                │              │
│       │                │                │                │                │  11. 更新    │
│       │                │                │                │                │  Prometheus  │
│       │                │                │                │                │              │
│       │ <────────────────────────────────────────────────────────────────│              │
│       │  12. 调度下一次检查                                              │              │
│       │  setTimeout(safeBeat, intervalRemainingMs)                      │              │
│       │                                                                   │              │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 状态转换完整链路

#### 6.2.1 正常模式（UP → DOWN → UP）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    正常模式状态转换链路                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  初始状态: UP                                                                │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────┐                                                        │
│  │ 检查失败        │                                                        │
│  │ maxretries = 3 │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐    重要？ ❌ 否                                       │
│  │ 状态: PENDING   │────────────────────────────> 不触发通知              │
│  │ retries: 1      │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐    重要？ ❌ 否                                       │
│  │ 状态: PENDING   │────────────────────────────> 不触发通知              │
│  │ retries: 2      │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐    重要？ ❌ 否                                       │
│  │ 状态: PENDING   │────────────────────────────> 不触发通知              │
│  │ retries: 3      │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐    重要？ ✅ 是（PENDING→DOWN）                      │
│  │ 状态: DOWN      │────────────────────────────> 触发通知                │
│  │ retries: 4      │   通知？ ✅ 是                                       │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐    重要？ ❌ 否                                       │
│  │ 状态: DOWN      │────────────────────────────> 检查持续 DOWN 重发     │
│  │ retries: 5      │   resendInterval > 0 ?                               │
│  └────────┬────────┘                                                        │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐    重要？ ✅ 是（DOWN→UP）                           │
│  │ 状态: UP        │────────────────────────────> 触发通知                │
│  │ retries: 0      │   通知？ ✅ 是                                       │
│  │ (检查成功重置)   │                                                        │
│  └─────────────────┘                                                        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### 6.2.2 Upside Down 模式

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Upside Down 模式状态转换链路                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  核心逻辑: 成功 → 翻转成 DOWN → 视为"故障"                                   │
│           失败 → 翻转成 UP → 视为"正常"                                     │
│                                                                              │
│  初始状态: UP (原始检查失败后翻转)                                            │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────┐                                    │
│  │ 原始检查: 成功 (UP)                  │                                    │
│  │ 检查后翻转: 翻转成 DOWN              │                                    │
│  │ 抛出错误: "Flip UP to DOWN"          │                                    │
│  └─────────────────┬───────────────────┘                                    │
│                    │                                                         │
│                    ▼                                                         │
│  ┌─────────────────────────────────────┐                                    │
│  │ catch 块处理                         │                                    │
│  │ 此时 bean.status = DOWN (翻转后)     │                                    │
│  │ 注意: isUpsideDown() && bean.status  │                                    │
│  │       === UP 条件不成立 (当前是 DOWN) │                                    │
│  │       → 进入通用重试逻辑             │                                    │
│  └─────────────────┬───────────────────┘                                    │
│                    │                                                         │
│                    ▼                                                         │
│  ┌─────────────────────────────────────┐                                    │
│  │ 状态: PENDING (如果 maxretries > 0) │                                    │
│  │ 或: DOWN (如果 maxretries = 0)      │                                    │
│  └─────────────────┬───────────────────┘                                    │
│                    │                                                         │
│                    ▼                                                         │
│  ┌─────────────────────────────────────┐                                    │
│  │ 原始检查: 失败 (DOWN)                │                                    │
│  │ 检查后翻转: 翻转成 UP                │                                    │
│  │ 不抛出错误 (bean.status !== DOWN)    │                                    │
│  └─────────────────┬───────────────────┘                                    │
│                    │                                                         │
│                    ▼                                                         │
│  ┌─────────────────────────────────────┐                                    │
│  │ catch 块处理                         │                                    │
│  │ 此时 bean.status = UP (翻转后)       │                                    │
│  │ 条件: isUpsideDown() && bean.status  │                                    │
│  │       === UP → 成立                  │                                    │
│  │ 操作: retries = 0 (重置)             │                                    │
│  │ 状态: 保持 UP (视为"正常")           │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                              │
│  总结:                                                                       │
│  - 原始检查成功 → 翻转后 DOWN → 视为故障 → 触发重试/通知                   │
│  - 原始检查失败 → 翻转后 UP → 视为正常 → 不触发重试/通知                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 附录：关键代码位置速查表

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|----------|
| 监控启动 | `server/model/monitor.js` | 409-1149 |
| 心跳执行 | `server/model/monitor.js` | 421-1120 |
| 状态码校验 | `server/util-server.js` | 552-585 |
| 重要心跳判定 | `server/model/monitor.js` | 1419-1445 |
| 通知触发判定 | `server/model/monitor.js` | 1454-1477 |
| 发送通知 | `server/model/monitor.js` | 1486-1553 |
| 维护状态检测 | `server/model/monitor.js` | 1630-1652 |
| 状态常量 | `src/util.js` | 21-24 |
| Uptime 计算 | `server/uptime-calculator.js` | 全文 |
| 后台任务 | `server/jobs.js` | 全文 |
| 监控类型基类 | `server/monitor-types/monitor-type.js` | 全文 |
| TCP 监控类型 | `server/monitor-types/tcp.js` | 全文 |
