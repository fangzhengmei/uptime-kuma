# Uptime Kuma Push 类型监控机制分析报告

## 一、概述

Push 类型监控是 Uptime Kuma 中的一种被动监控方式。与主动监控（如 HTTP、Ping）不同，Push 监控不主动探测目标服务，而是等待外部系统（如 cron 任务、脚本）向 Uptime Kuma 发送心跳信号。这种模式特别适合监控内部服务、定时任务或无法从外部直接访问的服务。

## 二、令牌绑定机制

### 2.1 令牌生成

令牌（Push Token）在前端创建/编辑 Push 类型监控时生成：

**文件位置：** `src/pages/EditMonitor.vue:3060, 3570-3575`

```javascript
const pushTokenLength = 32;

// 当监控类型为 push 时生成令牌
if (this.monitor.type === "push") {
    if (!this.monitor.pushToken) {
        // 生成 32 位随机令牌
        // 碰撞概率极低: 62^32 ~ 2.27e57 种唯一组合
        this.monitor.pushToken = genSecret(pushTokenLength);
    }
}
```

### 2.2 令牌重置

用户可以通过前端界面重置令牌：

**文件位置：** `src/pages/EditMonitor.vue:4006-4008`

```javascript
resetToken() {
    this.monitor.pushToken = genSecret(pushTokenLength);
}
```

### 2.3 令牌存储

令牌存储在数据库的 `monitor` 表中，字段名为 `push_token`：

**数据库迁移：** `db/knex_migrations/2023-10-11-1915-push-token-to-32.js`

```javascript
exports.up = function (knex) {
    // 将 push_token 字段长度从 20 扩展到 32
    return knex.schema.alterTable("monitor", function (table) {
        table.string("push_token", 32).alter();
    });
};
```

### 2.4 克隆监控时的令牌处理

克隆 Push 类型监控时，原令牌会被清除，新监控会自动生成新令牌：

**文件位置：** `src/pages/EditMonitor.vue:3790-3794`

```javascript
if (this.isClone) {
    // 重置克隆监控的 push token
    if (res.monitor.type === "push") {
        res.monitor.pushToken = undefined;
    }
}
```

## 三、外部心跳接入链路

### 3.1 API 端点

外部系统通过以下 HTTP 端点发送心跳：

**文件位置：** `server/routers/api-router.js:47-146`

```javascript
router.all("/api/push/:pushToken", async (request, response) => {
    // 处理心跳请求
});
```

**支持的 HTTP 方法：** 所有方法（GET, POST, PUT, DELETE 等）

### 3.2 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| pushToken | string (URL Path) | 是 | - | 监控令牌，用于识别目标监控 |
| status | string (Query) | 否 | "up" | 状态值："up" 或 "down" |
| msg | string (Query) | 否 | "OK" | 状态消息 |
| ping | number (Query) | 否 | null | 延迟时间（毫秒），最大值 1000 亿 ms (~3.17 年) |

**完整 URL 示例：**
```
https://your-uptime-kuma.com/api/push/abc123xyz?status=up&msg=OK&ping=23
```

前端显示的 Push URL：

**文件位置：** `src/pages/EditMonitor.vue:3271-3272`
```javascript
pushURL() {
    return this.$root.baseURL + "/api/push/" + this.monitor.pushToken + "?status=up&msg=OK&ping=";
}
```

### 3.3 心跳处理流程

**文件位置：** `server/routers/api-router.js:47-146`

```
┌─────────────────────────────────────────────────────────────────┐
│                    外部心跳请求处理流程                            │
├─────────────────────────────────────────────────────────────────┤
│  1. 提取参数                                                      │
│     - pushToken: 从 URL 路径获取                                  │
│     - msg: 默认 "OK"                                              │
│     - ping: 解析为浮点数，验证范围 (0 ~ 1e11 ms)                 │
│     - status: "up" → UP(1), "down" → DOWN(0)                   │
├─────────────────────────────────────────────────────────────────┤
│  2. 查找监控                                                      │
│     - 查询条件: push_token = ? AND active = 1                   │
│     - 未找到或未激活: 返回 404 错误                               │
├─────────────────────────────────────────────────────────────────┤
│  3. 准备心跳数据 (Heartbeat Bean)                                 │
│     - time: 当前 UTC 时间                                         │
│     - monitor_id: 监控 ID                                         │
│     - ping: 请求传入的延迟值                                       │
│     - msg: 状态消息                                                │
│     - downCount: 继承上次心跳的值                                 │
│     - duration: 与上次心跳的时间间隔（秒）                        │
├─────────────────────────────────────────────────────────────────┤
│  4. 状态判定                                                      │
│     - 检查维护模式: 是 → status = MAINTENANCE(3)                │
│     - 调用 determineStatus() 确定最终状态                        │
├─────────────────────────────────────────────────────────────────┤
│  5. 更新可用性                                                    │
│     - 通过 UptimeCalculator 更新可用性计算                        │
│     - 设置 end_time                                               │
├─────────────────────────────────────────────────────────────────┤
│  6. 处理通知                                                      │
│     - 判断是否为重要心跳 (状态变化)                                │
│     - 重置 downCount 或处理重发间隔                               │
│     - 发送通知 (如需)                                              │
├─────────────────────────────────────────────────────────────────┤
│  7. 持久化与同步                                                  │
│     - 存储心跳到数据库                                            │
│     - 通过 Socket.io 发送心跳事件到前端                          │
│     - 更新统计数据                                                │
│     - 更新 Prometheus 指标                                        │
└─────────────────────────────────────────────────────────────────┘
```

## 四、状态判定逻辑

### 4.1 determineStatus 函数

**文件位置：** `server/routers/api-router.js:576-619`

```javascript
function determineStatus(status, previousHeartbeat, maxretries, isUpsideDown, bean) {
    // 1. 处理 Upside Down 模式
    if (isUpsideDown) {
        status = flipStatus(status);
    }

    if (previousHeartbeat) {
        // 有上次心跳记录
        if (previousHeartbeat.status === UP && status === DOWN) {
            // 情况 A: UP → DOWN
            if (maxretries > 0 && previousHeartbeat.retries < maxretries) {
                // 还有重试次数: 标记为 PENDING
                bean.retries = previousHeartbeat.retries + 1;
                bean.status = PENDING;
            } else {
                // 无重试次数: 标记为 DOWN
                bean.retries = 0;
                bean.status = DOWN;
            }
        } else if (previousHeartbeat.status === PENDING && status === DOWN && 
                   previousHeartbeat.retries < maxretries) {
            // 情况 B: PENDING → DOWN (还有重试)
            // 继续 PENDING 状态
            bean.retries = previousHeartbeat.retries + 1;
            bean.status = PENDING;
        } else {
            // 情况 C: 其他状态转换
            if (status === DOWN) {
                // DOWN 状态: 累计重试次数
                bean.retries = previousHeartbeat.retries + 1;
                bean.status = status;
            } else {
                // UP/PENDING 状态: 重置重试次数
                bean.retries = 0;
                bean.status = status;
            }
        }
    } else {
        // 首次心跳
        if (status === DOWN && maxretries > 0) {
            // 首次就是 DOWN 且有重试: 标记为 PENDING
            bean.retries = 1;
            bean.status = PENDING;
        } else {
            // 首次 UP 或无重试: 直接使用传入状态
            bean.retries = 0;
            bean.status = status;
        }
    }
}
```

### 4.2 状态转换图

```
                    ┌──────────────┐
                    │   首次心跳    │
                    └──────┬───────┘
                           │
           ┌───────────────┴───────────────┐
           │                                 │
           ▼                                 ▼
    ┌────────────┐                   ┌────────────┐
    │  status=UP │                   │ status=DOWN│
    └──────┬─────┘                   └──────┬─────┘
           │                                 │
           ▼                        ┌────────┴────────┐
      ┌────────┐                    │                 │
      │   UP   │                    ▼                 ▼
      └────────┘           ┌──────────────┐   ┌──────────┐
                           │ maxretries>0 │   │  无重试   │
                           └──────┬───────┘   └────┬─────┘
                                  │                 │
                                  ▼                 ▼
                           ┌──────────┐      ┌──────────┐
                           │ PENDING  │      │   DOWN   │
                           └────┬─────┘      └──────────┘
                                │
                    ┌───────────┴───────────┐
                    │                         │
                    ▼                         ▼
             ┌────────────┐           ┌────────────┐
             │ 收到 UP    │           │ 继续 DOWN  │
             │ retries<max│           │ retries<max│
             └──────┬─────┘           └──────┬─────┘
                    │                         │
                    ▼                         ▼
              ┌──────────┐              ┌──────────┐
              │    UP    │              │ PENDING  │
              └──────────┘              └──────────┘
                                              │
                                    ┌─────────┴─────────┐
                                    │                   │
                                    ▼                   ▼
                             ┌──────────┐       ┌──────────┐
                             │ 收到 UP  │       │ retries>=│
                             │          │       │  max     │
                             └────┬─────┘       └────┬─────┘
                                  │                   │
                                  ▼                   ▼
                            ┌──────────┐       ┌──────────┐
                            │    UP    │       │   DOWN   │
                            └──────────┘       └──────────┘
```

## 五、超时检测逻辑

### 5.1 Push 类型监控的特殊启动方式

与其他监控类型不同，Push 类型监控启动时会延迟执行第一次心跳检查：

**文件位置：** `server/model/monitor.js:1141-1148`

```javascript
// 延迟 Push 类型的启动
if (this.type === "push") {
    setTimeout(() => {
        safeBeat();
    }, this.interval * 1000);  // 延迟一个完整的间隔时间
} else {
    safeBeat();  // 其他类型立即执行
}
```

**设计意图：** 给外部系统足够的时间来发送第一次心跳。

### 5.2 超时判定逻辑

**文件位置：** `server/model/monitor.js:726-763`

```javascript
} else if (this.type === "push") {
    // Push 类型监控检查
    log.debug("monitor", `[${this.name}] Checking monitor at ${dayjs().format("YYYY-MM-DD HH:mm:ss.SSS")}`);
    
    const bufferTime = 1000; // 1秒缓冲时间，用于处理时钟差异

    if (previousBeat) {
        // 计算与上次心跳的时间差
        const msSinceLastBeat = dayjs.utc().valueOf() - dayjs.utc(previousBeat.time).valueOf();
        
        log.debug("monitor", `[${this.name}] msSinceLastBeat = ${msSinceLastBeat}`);

        // 判断条件：
        // 1. 上次状态不是 UP（考虑 upside down 模式）
        // 或
        // 2. 超过时间窗口（间隔 + 缓冲时间）
        if (
            previousBeat.status !== (this.isUpsideDown() ? DOWN : UP) ||
            msSinceLastBeat > beatInterval * 1000 + bufferTime
        ) {
            // 判定为超时
            bean.duration = Math.round(msSinceLastBeat / 1000);
            throw new Error("No heartbeat in the time window");
        } else {
            // 心跳在时间窗口内，计算下次检查时间
            let timeout = beatInterval * 1000 - msSinceLastBeat;
            if (timeout < 0) {
                timeout = bufferTime;
            } else {
                timeout += bufferTime;
            }
            
            // 注意：Push 类型在正常情况下不插入新的成功心跳
            // 只有当外部系统调用 /api/push/ 时才会写入心跳
            retries = 0;
            log.debug("monitor", `[${this.name}] timeout = ${timeout}`);
            this.heartbeatInterval = setTimeout(safeBeat, timeout);
            return;  // 提前返回，不执行后续的心跳存储逻辑
        }
    } else {
        // 没有上次心跳记录（首次）
        bean.duration = beatInterval;
        throw new Error("No heartbeat in the time window");
    }
}
```

### 5.3 超时逻辑流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Push 类型超时检查流程                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐                                                      │
│  │  启动监控     │                                                      │
│  └──────┬───────┘                                                      │
│         │                                                               │
│         ▼                                                               │
│  ┌─────────────────────────────────┐                                    │
│  │ 延迟 interval 秒后首次检查        │  <-- 给外部系统发送心跳的时间     │
│  └──────────────┬──────────────────┘                                    │
│                 │                                                       │
│                 ▼                                                       │
│  ┌──────────────────────────────────┐                                   │
│  │ 有 previousBeat（上次心跳记录）？ │                                   │
│  └──────────────┬───────────────────┘                                   │
│         ┌───────┴───────┐                                               │
│         │               │                                               │
│         ▼               ▼                                               │
│     ┌────────┐    ┌─────────────────┐                                   │
│     │   是   │    │       否        │                                   │
│     └───┬────┘    │ （首次/无记录）  │                                   │
│         │         └────────┬────────┘                                   │
│         │                  │                                             │
│         │                  ▼                                             │
│         │         ┌──────────────────┐                                   │
│         │         │ throw Error:     │                                   │
│         │         │ "No heartbeat in │                                   │
│         │         │ the time window" │                                   │
│         │         └────────┬─────────┘                                   │
│         │                  │                                             │
│         │                  ▼                                             │
│         │         ┌──────────────────┐                                   │
│         │         │ 状态: DOWN       │                                   │
│         │         │ 后续通知等处理    │                                   │
│         │         └──────────────────┘                                   │
│         │                                                                  │
│         ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │ 计算 msSinceLastBeat = 当前时间 - 上次心跳时间                  │       │
│  └──────────────────────────────┬───────────────────────────────┘       │
│                                 │                                          │
│                                 ▼                                          │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │ 判断超时条件：                                                  │       │
│  │   1. 上次状态 ≠ UP (考虑 upside down)                          │       │
│  │   或                                                            │       │
│  │   2. msSinceLastBeat > interval*1000 + 1000 (缓冲时间)        │       │
│  └──────────────────────────────┬───────────────────────────────┘       │
│                    ┌────────────┴────────────┐                           │
│                    │                         │                           │
│                    ▼                         ▼                           │
│           ┌────────────────┐        ┌────────────────┐                  │
│           │   条件满足     │        │   条件不满足   │                  │
│           │   → 超时      │        │   → 正常       │                  │
│           └───────┬────────┘        └───────┬────────┘                  │
│                   │                           │                           │
│                   ▼                           ▼                           │
│           ┌────────────────┐        ┌───────────────────────────────┐   │
│           │ throw Error:   │        │ 计算下次检查时间:              │   │
│           │ "No heartbeat  │        │ timeout =                      │   │
│           │ in the time    │        │   interval*1000 - msSinceLastBeat│
│           │ window"        │        │   + bufferTime (1s)           │   │
│           └───────┬────────┘        └───────────────┬───────────────┘   │
│                   │                                   │                   │
│                   ▼                                   ▼                   │
│           ┌────────────────┐        ┌───────────────────────────────┐   │
│           │ 后续异常处理   │        │ setTimeout(safeBeat, timeout) │   │
│           │ (通知、存储等) │        │ return; (不插入新心跳)         │   │
│           └────────────────┘        └───────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 两种心跳写入方式的区别

Push 类型监控有两种写入心跳的方式：

| 方式 | 触发条件 | 写入位置 | 状态 |
|------|----------|----------|------|
| **方式 A: 外部 API 调用** | 外部系统调用 `/api/push/:pushToken` | `server/routers/api-router.js` | UP 或 DOWN（由请求参数决定） |
| **方式 B: 超时检测** | 内部定时检查发现超时 | `server/model/monitor.js:beat()` | DOWN |

**关键设计：**
- 正常情况下，只有外部 API 调用会写入心跳
- 内部的 `beat()` 函数在心跳正常时**不写入**新的成功心跳，只是计算下次检查时间
- 只有当**超时发生**时，`beat()` 才会写入 DOWN 状态的心跳

这种设计确保了：
1. 心跳时间准确反映外部系统实际发送时间
2. 避免重复写入相同的成功状态
3. 超时检测独立于外部心跳

## 六、前端实时同步机制

### 6.1 Socket.io 通信架构

Uptime Kuma 使用 Socket.io 实现前后端实时通信。

**后端发送心跳事件：**

**文件位置：** `server/routers/api-router.js:127`
```javascript
// 当外部心跳到达时
io.to(monitor.user_id).emit("heartbeat", bean.toJSON());
```

**文件位置：** `server/model/monitor.js:1094`
```javascript
// 当监控检测（包括超时）时
io.to(this.user_id).emit("heartbeat", bean.toJSON());
```

**前端接收心跳事件：**

**文件位置：** `src/mixins/socket.js:204-234`

```javascript
socket.on("heartbeat", (data) => {
    // 初始化该监控的心跳列表
    if (!(data.monitorID in this.heartbeatList)) {
        this.heartbeatList[data.monitorID] = [];
    }

    // 将新心跳添加到列表末尾
    this.heartbeatList[data.monitorID].push(data);

    // 限制列表长度，最多保留 150 条
    if (this.heartbeatList[data.monitorID].length >= 150) {
        this.heartbeatList[data.monitorID].shift();
    }

    // 处理重要心跳（状态变化）
    if (data.important) {
        // 显示 Toast 通知
        if (this.monitorList[data.monitorID] !== undefined) {
            if (data.status === 0) {
                // DOWN 状态：红色错误提示
                toast.error(`[${this.monitorList[data.monitorID].name}] [DOWN] ${data.msg}`, {
                    timeout: getToastErrorTimeout(),
                });
            } else if (data.status === 1) {
                // UP 状态：绿色成功提示
                toast.success(`[${this.monitorList[data.monitorID].name}] [Up] ${data.msg}`, {
                    timeout: getToastSuccessTimeout(),
                });
            } else {
                // 其他状态：普通提示
                toast(`[${this.monitorList[data.monitorID].name}] ${data.msg}`);
            }
        }

        // 触发事件总线事件
        this.emitter.emit("newImportantHeartbeat", data);
    }
});
```

### 6.2 前端数据绑定

**心跳列表：**
```javascript
// 存储结构
this.heartbeatList = {
    monitorID1: [beat1, beat2, beat3, ...],
    monitorID2: [beat1, beat2, ...],
    ...
};
```

**计算属性：**

**文件位置：** `src/mixins/socket.js:744-832`

```javascript
computed: {
    // 获取每个监控的最后一次心跳
    lastHeartbeatList() {
        let result = {};
        for (let monitorID in this.heartbeatList) {
            let index = this.heartbeatList[monitorID].length - 1;
            result[monitorID] = this.heartbeatList[monitorID][index];
        }
        return result;
    },

    // 计算状态列表（用于 UI 显示）
    statusList() {
        let result = {};
        // 预定义状态样式
        let unknown = { text: this.$t("Unknown"), color: "secondary" };
        
        for (let monitorID in this.lastHeartbeatList) {
            let lastHeartBeat = this.lastHeartbeatList[monitorID];
            
            if (!lastHeartBeat) {
                result[monitorID] = unknown;
            } else if (lastHeartBeat.status === UP) {
                result[monitorID] = { text: this.$t("Up"), color: "primary" };
            } else if (lastHeartBeat.status === DOWN) {
                result[monitorID] = { text: this.$t("Down"), color: "danger" };
            } else if (lastHeartBeat.status === PENDING) {
                result[monitorID] = { text: this.$t("Pending"), color: "warning" };
            } else if (lastHeartBeat.status === MAINTENANCE) {
                result[monitorID] = { text: this.$t("statusMaintenance"), color: "maintenance" };
            } else {
                result[monitorID] = unknown;
            }
        }
        return result;
    },

    // 统计信息
    stats() {
        let result = {
            active: 0,
            up: 0,
            down: 0,
            maintenance: 0,
            pending: 0,
            unknown: 0,
            pause: 0,
        };
        // ... 计算各状态数量
    }
}
```

### 6.3 前端显示 Push URL

**编辑页面：** `src/pages/EditMonitor.vue:256-267`

```vue
<!-- Push URL 显示区域 -->
<div v-if="monitor.type === 'push'" class="my-3">
    <label for="push-url" class="form-label">{{ $t("PushUrl") }}</label>
    
    <!-- 可复制的 URL 输入框 -->
    <CopyableInput id="push-url" v-model="pushURL" type="url" disabled="disabled" />
    
    <!-- 说明文字 -->
    <div class="form-text">
        {{ $t("needPushEvery", [monitor.interval]) }}  <!-- "需要每 X 秒推送一次" -->
        <br />
        {{ $t("pushOptionalParams", ["status, msg, ping"]) }}  <!-- 可选参数 -->
    </div>
    
    <!-- 重置令牌按钮 -->
    <button class="btn btn-primary" type="button" @click="resetToken">
        {{ $t("Reset Token") }}
    </button>
</div>
```

**详情页面：** `src/pages/Details.vue:583-585`

```javascript
pushURL() {
    return this.$root.baseURL + "/api/push/" + this.monitor.pushToken + "?status=up&msg=OK&ping=";
}
```

### 6.4 实时同步流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Push 监控前端实时同步流程                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌──────────────────────┐                              ┌──────────────────┐   │
│  │   外部系统 (cron等)   │                              │   监控定时检查     │   │
│  │                      │                              │   (超时检测)       │   │
│  └──────────┬───────────┘                              └────────┬─────────┘   │
│             │                                                    │              │
│             ▼                                                    ▼              │
│  ┌────────────────────────┐                         ┌────────────────────────┐ │
│  │ POST /api/push/:token  │                         │ server/model/monitor.js │ │
│  │ (api-router.js)        │                         │ :beat() 超时检测        │ │
│  └───────────┬────────────┘                         └───────────┬────────────┘ │
│              │                                                    │              │
│              └────────────────────┬───────────────────────────────┘              │
│                                   │                                              │
│                                   ▼                                              │
│              ┌──────────────────────────────────────────────────────┐          │
│              │   后端 Socket.io 发送                                  │          │
│              │   io.to(user_id).emit("heartbeat", bean.toJSON())   │          │
│              └──────────────────────────┬───────────────────────────┘          │
│                                         │                                      │
│                                         ▼                                      │
│              ┌──────────────────────────────────────────────────────────────┐│
│              │              前端 Socket.io 接收                               ││
│              │   socket.on("heartbeat", (data) => { ... })                  ││
│              └──────────────────────────────────┬───────────────────────────┘│
│                                                 │                              │
│                                                 ▼                              │
│              ┌──────────────────────────────────────────────────────────────┐│
│              │                   更新前端数据                                  ││
│              │  1. this.heartbeatList[monitorID].push(data)                  ││
│              │  2. 限制列表长度 (最多 150 条)                                  ││
│              │  3. 响应式更新 computed 属性                                    ││
│              │     - lastHeartbeatList                                         ││
│              │     - statusList                                                ││
│              │     - stats                                                     ││
│              └──────────────────────────────────┬───────────────────────────┘│
│                                                 │                              │
│                          ┌──────────────────────┴──────────────────────┐     │
│                          │                                             │     │
│                          ▼                                             ▼     │
│              ┌───────────────────────┐                    ┌──────────────────┐│
│              │   UI 自动更新          │                    │   重要心跳处理     ││
│              │   (Vue 响应式系统)     │                    │                  ││
│              │                       │                    │ 1. Toast 通知    ││
│              │ - 监控列表状态颜色     │                    │    - UP: 绿色    ││
│              │ - 最后心跳时间         │                    │    - DOWN: 红色  ││
│              │ - 可用性统计           │                    │    - 其他: 普通  ││
│              │ - 详情页面数据         │                    │                  ││
│              │                       │                    │ 2. 事件总线       ││
│              │                       │                    │    emitter.emit() ││
│              └───────────────────────┘                    └──────────────────┘│
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

## 七、关键代码位置总结

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| Push 类型超时检测 | `server/model/monitor.js` | 726-763, 1141-1148 |
| 外部心跳 API 端点 | `server/routers/api-router.js` | 47-146 |
| 状态判定函数 | `server/routers/api-router.js` | 576-619 |
| 前端令牌生成 | `src/pages/EditMonitor.vue` | 3060, 3570-3575, 4006-4008 |
| 前端 Push URL 计算 | `src/pages/EditMonitor.vue` | 3271-3272 |
| 前端心跳接收处理 | `src/mixins/socket.js` | 204-234 |
| 前端状态计算 | `src/mixins/socket.js` | 744-832 |
| 数据库令牌字段迁移 | `db/knex_migrations/2023-10-11-1915-push-token-to-32.js` | 1-12 |

## 八、使用场景示例

### 8.1 Cron 任务集成

```bash
# 每分钟发送一次心跳
* * * * * curl -s "https://your-kuma.com/api/push/your-token-here?status=up&msg=OK&ping=" > /dev/null
```

### 8.2 脚本集成

```bash
#!/bin/bash
# backup-script.sh

# 执行备份任务
START_TIME=$(date +%s%N)
./do-backup.sh
BACKUP_EXIT_CODE=$?
END_TIME=$(date +%s%N)

# 计算耗时（毫秒）
PING=$((($END_TIME - $START_TIME)/1000000))

# 发送心跳到 Uptime Kuma
KUMA_URL="https://your-kuma.com/api/push/your-token-here"

if [ $BACKUP_EXIT_CODE -eq 0 ]; then
    curl -s "$KUMA_URL?status=up&msg=Backup%20completed&ping=$PING"
else
    curl -s "$KUMA_URL?status=down&msg=Backup%20failed&ping=$PING"
fi
```

### 8.3 Python 脚本示例

```python
import requests
import time

KUMA_PUSH_URL = "https://your-kuma.com/api/push/your-token-here"

def send_heartbeat(status="up", msg="OK", ping=None):
    params = {"status": status, "msg": msg}
    if ping is not None:
        params["ping"] = ping
    try:
        response = requests.get(KUMA_PUSH_URL, params=params, timeout=10)
        return response.json().get("ok", False)
    except Exception as e:
        print(f"Failed to send heartbeat: {e}")
        return False

# 使用示例
start = time.time()
# ... 执行任务 ...
ping_ms = int((time.time() - start) * 1000)

send_heartbeat(status="up", msg="Task completed", ping=ping_ms)
```

## 九、注意事项

1. **令牌安全性**：Push Token 相当于访问凭证，泄露后可能被恶意使用。建议定期重置令牌。

2. **时钟同步**：超时检测使用了 1 秒的缓冲时间来处理时钟差异，但建议外部系统与 Uptime Kuma 服务器保持时钟同步。

3. **缓冲时间**：`bufferTime = 1000ms` 是硬编码的，如果网络延迟较大，可能需要考虑更大的间隔时间。

4. **首次启动延迟**：Push 监控启动后会延迟一个完整的间隔时间才开始第一次检查，这是为了给外部系统发送首次心跳的机会。

5. **监控间隔与心跳频率**：外部系统发送心跳的频率应该**小于**监控的间隔时间。建议：
   - 监控间隔为 60 秒时，每 30-50 秒发送一次心跳
   - 或者使用 cron 时，调度频率为监控间隔的 1/2

6. **重试机制**：Push 监控支持与其他类型相同的重试机制（`maxretries`），当外部报告 DOWN 或超时时，会经历 PENDING 状态再变为 DOWN。
