# Uptime-Kuma Push 类型监控机制分析

## 1. 概述

Uptime-Kuma 的 push 类型监控采用了**被动接收 + 主动验证**的双重检查模式，与传统的主动轮询（如 HTTP、ping 类型）不同。在这种模式下：

- **外部服务**主动向 Uptime-Kuma 推送心跳信号
- **Uptime-Kuma API 层**接收并记录这些心跳
- **Uptime-Kuma Monitor 层**独立进行时间窗口验证
- 适用于无法被外部直接访问的内部服务，或者需要主动上报状态的场景

## 2. 外部推送 Gateway 完整接入链路

### 2.1 接入链路总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           完整接入链路流程图                                   │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   前端 UI     │────▶│  生成 Token  │────▶│  显示 Push   │────▶│  外部服务    │
│  创建监控    │     │  (32位随机)  │     │    URL      │     │  配置推送    │
└──────────────┘     └──────────────┘     └──────────────┘     └──────┬───────┘
                                                                       │
                                                                       │ HTTP GET/POST
                                                                       │ URL 查询参数
                                                                       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Socket.io   │◀────│  保存心跳    │◀────│  状态判定    │◀────│  API 端点    │
│  推送前端    │     │  到数据库   │     │              │     │ /api/push/   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                       │
                                              ┌────────────────────────┘
                                              │
                                              ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  触发通知     │◀────│  超时则报错   │◀────│  时间窗口    │◀────│  Monitor     │
│  (邮件/Slack)│     │"No heartbeat"│    │  验证检查   │     │  定时 beat() │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘

注意：这是两条独立的链路！
- 链路 A：外部推送 → API 记录心跳
- 链路 B：Monitor 定时检查 → 验证时间窗口
```

### 2.2 第一步：前端创建 Push 类型监控

#### 2.2.1 监控类型选择

用户在前端页面创建或编辑监控时，选择类型为 "Push"：

```vue
<!-- src/pages/EditMonitor.vue:69 -->
<option value="push">Push</option>
```

#### 2.2.2 Push Token 自动生成

当监控类型为 "push" 时，前端会自动检查并生成 `pushToken`：

```javascript
// src/pages/EditMonitor.vue:3570-3576
if (this.monitor.type === "push") {
    if (!this.monitor.pushToken) {
        // ideally this would require checking if the generated token is already used
        // it's very unlikely to get a collision though (62^32 ~ 2.27265788 * 10^57 unique tokens)
        this.monitor.pushToken = genSecret(pushTokenLength);
    }
}
```

#### 2.2.3 Token 生成算法

```javascript
// src/pages/EditMonitor.vue:3060
const pushTokenLength = 32;

// src/util.ts:555-563
export function genSecret(length = 64) {
    let secret = "";
    const chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    const charsLength = chars.length;
    for (let i = 0; i < length; i++) {
        secret += chars.charAt(getCryptoRandomInt(0, charsLength - 1));
    }
    return secret;
}
```

**Token 特性**：
- **长度**：32 位字符
- **字符集**：大小写字母 + 数字（共 62 种字符）
- **随机性**：使用加密安全的随机数生成器 (`getCryptoRandomInt`)
- **碰撞概率**：62^32 ≈ 2.27 × 10^57 种可能，碰撞概率极低

#### 2.2.4 克隆监控时重置 Token

当克隆一个 Push 类型监控时，会重置 Token 避免冲突：

```javascript
// src/pages/EditMonitor.vue:3791-3794
// Reset push token for cloned monitors
if (res.monitor.type === "push") {
    res.monitor.pushToken = undefined;
}
```

#### 2.2.5 手动重置 Token

用户可以点击 "Reset Token" 按钮手动重置 Token：

```javascript
// src/pages/EditMonitor.vue:4006-4008
resetToken() {
    this.monitor.pushToken = genSecret(pushTokenLength);
}
```

### 2.3 第二步：获取 Push URL

前端会自动构建完整的 Push URL 供用户使用：

```javascript
// src/pages/EditMonitor.vue:3271-3273
pushURL() {
    return this.$root.baseURL + "/api/push/" + this.monitor.pushToken + "?status=up&msg=OK&ping=";
}
```

**示例 Push URL**：
```
https://your-uptime-kuma.example.com/api/push/abc123def456ghi789jkl012mno345pq?status=up&msg=OK&ping=
```

**URL 结构说明：
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  https://your-uptime-kuma.example.com/api/push/{pushToken}?status=up&msg=OK │
│  └─────────────────────────────┘└──────────┘└──────────┘└──────────────────┘
│              │                    │            │            │
│              ▼                    ▼            ▼            ▼
│         基础 URL           API 路径    Token(32位)    URL 查询参数
│         (协议+域名)        固定值      随机值        (状态/消息/延迟)
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 第三步：外部服务推送心跳

#### 2.4.1 接入端点

```
/api/push/:pushToken
```

- **位置**: `server/routers/api-router.js:47-146`
- **HTTP 方法**: 支持所有 HTTP 方法（GET、POST、PUT、DELETE 等）
- **认证方式**: 通过 URL 路径中的 `pushToken`

#### 2.4.2 ⚠️ 重要：参数来源说明

**关键发现**：所有参数都来自 **URL 查询字符串**，**不是**请求体！

```javascript
// server/routers/api-router.js:50-52
let msg = request.query.msg || "OK";
let ping = parseFloat(request.query.ping) || null;
let statusString = request.query.status || "up";
```

这意味着：
- ✅ GET 请求：参数放在 URL 正常工作
- ⚠️ POST 请求：**参数必须放在 URL 查询字符串中，** `request.body` 中的参数会被忽略！

#### 2.4.3 请求参数详解

| 参数名 | 来源 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|------|--------|------|
| pushToken | URL 路径 | string | 是 | - | 32 位监控器唯一标识令牌 |
| msg | URL 查询 | string | 否 | "OK" | 心跳消息内容 |
| ping | URL 查询 | number | 否 | null | 延迟时间（毫秒） |
| status | URL 查询 | string | 否 | "up" | 服务状态："up" 或 "down" |

#### 2.4.4 ping 值验证

```javascript
// server/routers/api-router.js:55-60
// Validate ping value - max 100 billion ms (~3.17 years)
// Fits safely in both BIGINT and FLOAT(20,2)
const MAX_PING_MS = 100000000000;
if (ping !== null && (ping < 0 || ping > MAX_PING_MS)) {
    throw new Error(`Invalid ping value. Must be between 0 and ${MAX_PING_MS} ms.`);
}
```

#### 2.4.5 正确的接入示例

**使用 curl 推送心跳：**

```bash
# ✅ 正确：GET 请求，参数在 URL 中
curl "http://your-uptime-kuma/api/push/abc123def456ghi789jkl012mno345pq"

# ✅ 正确：带参数的 GET 请求
curl "http://your-uptime-kuma/api/push/abc123def456ghi789jkl012mno345pq?msg=Service%20Healthy&ping=23&status=up"

# ✅ 正确：POST 请求，但参数仍在 URL 中
curl -X POST "http://your-uptime-kuma/api/push/abc123def456ghi789jkl012mno345pq?status=up&msg=OK"

# ❌ 错误：POST 请求，参数在 body 中（会被忽略！）
curl -X POST "http://your-uptime-kuma/api/push/abc123def456ghi789jkl012mno345pq" \
  -d "status=up&msg=Service%20Running"  # ❌ 这些参数会被忽略！
```

**在脚本中使用（Python）：

```python
import requests
import time

PUSH_TOKEN = "abc123def456ghi789jkl012mno345pq"
UPTIME_KUMA_URL = "http://your-uptime-kuma"

def send_heartbeat():
    try:
        # 执行健康检查逻辑
        is_healthy = check_service_health()
        
        # ✅ 正确：使用 params 参数（会放在 URL 查询字符串中）
        params = {
            "msg": "Service is running" if is_healthy else "Service has issues",
            "status": "up" if is_healthy else "down",
            "ping": 25  # 可选：延迟时间（毫秒）
        }
        
        # 方式 1：GET 请求（推荐）
        response = requests.get(
            f"{UPTIME_KUMA_URL}/api/push/{PUSH_TOKEN}",
            params=params,  # ✅ 正确
            timeout=10
        )
        
        # 方式 2：POST 请求（参数仍在 URL 中）
        # response = requests.post(
        #     f"{UPTIME_KUMA_URL}/api/push/{PUSH_TOKEN}",
        #     params=params,  # ✅ 仍然使用 params，不是 json 或 data
        #     timeout=10
        # )
        
        result = response.json()
        print(f"Heartbeat sent: {result}")  # {"ok": true}
    except Exception as e:
        print(f"Failed to send heartbeat: {e}")

# 定时发送心跳（建议间隔小于监控器配置的 interval）
while True:
    send_heartbeat()
    time.sleep(30)  # 每 30 秒发送一次
```

### 2.5 第四步：后端 API 处理推送

#### 2.5.1 Token 验证

```javascript
// server/routers/api-router.js:62-66
let monitor = await R.findOne("monitor", " push_token = ? AND active = 1 ", [pushToken]);

if (!monitor) {
    throw new Error("Monitor not found or not active.");
}
```

- 在数据库中查找匹配 `pushToken` 且 `active=1` 的监控器
- 如果找不到或监控器未激活，返回 404 错误

#### 2.5.2 API 处理流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    /api/push/:pushToken 处理流程                                │
└─────────────────────────────────────────────────────────────────────────────┘

1. 接收请求
   ↓
2. 解析 URL 查询参数（msg, ping, status）
   ↓
3. 验证 ping 值范围（0 ~ 100,000,000,000 ms）
   ↓
4. 通过 pushToken 查找 active=1 的监控器
   ↓
5. 获取上次心跳记录（getPreviousHeartbeat）
   ↓
6. 检查是否处于维护状态
   ├─ 是 → status = MAINTENANCE, msg = "Monitor under maintenance"
   └─ 否 → 继续
   ↓
7. 确定最终状态（determineStatus）
   ├─ 考虑 maxretries 重试机制
   └─ 考虑 isUpsideDown 反转模式
   ↓
8. 计算 uptime（UptimeCalculator）
   ↓
9. 标记重要心跳（isImportantBeat）
   ↓
10. 检查是否需要发送通知（isImportantForNotification）
    ├─ 需要 → 发送通知，重置 downCount
    └─ 不需要 → 检查是否需要重发通知
   ↓
11. 保存心跳到数据库（R.store）
   ↓
12. Socket.io 推送到前端（io.to(user_id).emit）
   ↓
13. 更新 Prometheus 指标
   ↓
14. 返回响应 { ok: true }
```

## 3. 状态判定逻辑

### 3.1 状态确定函数（determineStatus）

位置：`server/routers/api-router.js:576-619

```javascript
function determineStatus(status, previousHeartbeat, maxretries, isUpsideDown, bean) {
    // 反转模式处理
    if (isUpsideDown) {
        status = flipStatus(status);
    }

    if (previousHeartbeat) {
        // UP → DOWN：检查重试
        if (previousHeartbeat.status === UP && status === DOWN) {
            if (maxretries > 0 && previousHeartbeat.retries < maxretries) {
                // 还有重试次数
                bean.retries = previousHeartbeat.retries + 1;
                bean.status = PENDING;
            } else {
                // 无重试次数
                bean.retries = 0;
                bean.status = DOWN;
            }
        } 
        // PENDING → DOWN：继续重试
        else if (previousHeartbeat.status === PENDING && status === DOWN && previousHeartbeat.retries < maxretries) {
            bean.retries = previousHeartbeat.retries + 1;
            bean.status = PENDING;
        }
        // 其他情况
        else {
            if (status === DOWN) {
                bean.retries = previousHeartbeat.retries + 1;
                bean.status = status;
            } else {
                bean.retries = 0;
                bean.status = status;
            }
        }
    } else {
        // 首次心跳
        if (status === DOWN && maxretries > 0) {
            bean.retries = 1;
            bean.status = PENDING;
        } else {
            bean.retries = 0;
            bean.status = status;
        }
    }
}
```

### 3.2 状态转换规则

#### 场景 1：正常上报（UP → UP）

```
外部服务推送 status=up
     ↓
检查 previousHeartbeat.status
     ↓
如果上次是 UP → 保持 UP 状态
     ↓
retries 重置为 0
```

#### 场景 2：服务异常（UP → DOWN）

```
外部服务推送 status=down 或 未按时推送
     ↓
检查 maxretries 配置
     ↓
如果 maxretries > 0 且 retries < maxretries:
    → 状态变为 PENDING
    → retries + 1
否则:
    → 状态变为 DOWN
    → retries 重置为 0
```

#### 场景 3：服务恢复（DOWN/PENDING → UP）

```
外部服务推送 status=up
     ↓
检查 previousHeartbeat.status
     ↓
如果上次是 DOWN 或 PENDING → 状态变为 UP
     ↓
retries 重置为 0
```

#### 场景 4：维护模式

```
检查监控器是否处于维护时段
     ↓
如果是:
    → 状态强制设置为 MAINTENANCE
    → msg 设置为 "Monitor under maintenance"
```

### 3.3 反转模式（Upside Down）

适用于监控"某种不应该发生的事件"（例如：错误日志出现）。

```javascript
const flipStatus = (status) => {
    if (status === UP) return DOWN;
    if (status === DOWN) return UP;
    return status;
};
```

| 正常模式 | 反转模式 | 适用场景 |
|---------|---------|---------|
| 推送 up → UP | 推送 up → DOWN | 监控"错误不应出现" |
| 推送 down → DOWN | 推送 down → UP | 错误出现时通知 |

## 4. 时间窗口判定机制（Monitor 层）

### 4.1 核心概念

**这是 Push 类型监控最关键的机制**：即使外部服务推送了心跳，后端 Monitor 层仍会独立进行时间窗口验证。

这是一条**完全独立**于 API 层的第二条检查链路**。

### 4.2 判定逻辑

位置：`server/model/monitor.js:726-763`

```javascript
} else if (this.type === "push") {
    // Type: Push
    log.debug(
        "monitor",
        `[${this.name}] Checking monitor at ${dayjs().format("YYYY-MM-DD HH:mm:ss.SSS")}`
    );
    const bufferTime = 1000; // 1s buffer to accommodate clock differences

    if (previousBeat) {
        const msSinceLastBeat = dayjs.utc().valueOf() - dayjs.utc(previousBeat.time).valueOf();

        log.debug("monitor", `[${this.name}] msSinceLastBeat = ${msSinceLastBeat}`);

        // 如果之前的心跳是 DOWN 或 PENDING，使用常规的 beatInterval/retryInterval
        if (
            previousBeat.status !== (this.isUpsideDown() ? DOWN : UP) ||
            msSinceLastBeat > beatInterval * 1000 + bufferTime
        ) {
            bean.duration = Math.round(msSinceLastBeat / 1000);
            throw new Error("No heartbeat in the time window");
        } else {
            // 在时间窗口内收到心跳，计算下一次检查时间
            let timeout = beatInterval * 1000 - msSinceLastBeat;
            if (timeout < 0) {
                timeout = bufferTime;
            } else {
                timeout += bufferTime;
            }
            // 不需要为 push 类型插入成功的心跳，因为外部推送已经处理了
            retries = 0;
            log.debug("monitor", `[${this.name}] timeout = ${timeout}`);
            this.heartbeatInterval = setTimeout(safeBeat, timeout);
            return;
        }
    } else {
        // 没有之前的心跳记录
        bean.duration = beatInterval;
        throw new Error("No heartbeat in the time window");
    }
}
```

### 4.3 判定条件详解

#### 条件 1：有之前的心跳记录

当满足以下**任一**条件时，判定为**超时（DOWN）**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  条件 A: 上次心跳状态不是 UP（考虑反转模式）                                  │
│  ─────────────────────────────────────────────────────────────────────────  │
│  代码逻辑：                                                                   │
│  previousBeat.status !== (this.isUpsideDown() ? DOWN : UP)                 │
│                                                                              │
│  正常模式（isUpsideDown=false）：                                            │
│    检查 previousBeat.status !== UP                                            │
│    → 如果上次是 DOWN 或 PENDING，条件成立                                   │
│                                                                              │
│  反转模式（isUpsideDown=true）：                                             │
│    检查 previousBeat.status !== DOWN                                          │
│    → 如果上次是 UP 或 PENDING，条件成立（反转模式下 UP 表示异常）          │
│                                                                              │
│  含义：服务之前已经异常，继续保持异常状态，不需要等待时间窗口                  │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  条件 B: 距离上次心跳的时间超过（检查间隔 + 缓冲时间）                        │
│  ─────────────────────────────────────────────────────────────────────────  │
│  代码逻辑：                                                                   │
│  msSinceLastBeat > beatInterval * 1000 + bufferTime                        │
│                                                                              │
│  参数说明：                                                                   │
│  - msSinceLastBeat：当前时间 - 上次心跳时间（毫秒）                          │
│  - beatInterval：监控器配置的检查间隔（秒）                                  │
│  - bufferTime：1000ms（1秒缓冲）                                             │
│                                                                              │
│  示例：                                                                       │
│  如果 interval = 60 秒：                                                     │
│    超时阈值 = 60 * 1000 + 1000 = 61000 ms = 61 秒                         │
│                                                                              │
│  含义：外部服务没有在规定时间内推送心跳，判定为超时                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 条件 2：无之前的心跳记录

```
直接判定为超时
错误消息："No heartbeat in the time window"

含义：新创建的 Push 监控器，如果没有收到任何心跳，直接判定为 DOWN
```

### 4.4 正常情况的处理

当在时间窗口内收到心跳时：

```javascript
// 计算下一次检查时间
let timeout = beatInterval * 1000 - msSinceLastBeat;
if (timeout < 0) {
    timeout = bufferTime;
} else {
    timeout += bufferTime;
}

// 设置定时器
this.heartbeatInterval = setTimeout(safeBeat, timeout);

// 直接返回，不插入新的心跳记录
return;
```

**为什么不插入心跳记录？**
- 因为外部推送已经通过 API 层插入了心跳记录
- Monitor 层只负责验证时间窗口，不负责记录成功的心跳
- 这样避免了重复记录

### 4.5 时间窗口图示

假设监控器配置的 interval = 60 秒：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           时间窗口判定图示（interval = 60s, buffer = 1s）      │
└─────────────────────────────────────────────────────────────────────────────┘

时间轴：
0s                    60s                   120s                   180s
│                      │                     │                      │
├──────────────────────┼─────────────────────┼──────────────────────┼──►
│                      │                     │                      │
▼                      ▼                     ▼                      ▼
Monitor 检查点 1      检查点 2             检查点 3              检查点 4

外部推送时间点：
   30s                   90s                    150s
    │                     │                      │
    ▼                     ▼                      ▼
   推送 1                 推送 2                 推送 3

┌─────────────────────────────────────────────────────────────────────────────┐
│  检查点 2（60s 时）判定逻辑：                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│  msSinceLastBeat = 60s - 30s = 30s = 30000 ms                              │
│  超时阈值 = 60 * 1000 + 1000 = 61000 ms                                    │
│                                                                              │
│  30000 ms < 61000 ms → ✅ 在时间窗口内，正常                               │
│                                                                              │
│  下一次检查时间计算：                                                          │
│  timeout = 60000 - 30000 + 1000 = 31000 ms = 31s                         │
│  → 检查点 3 实际上会在 60s + 31s = 91s 时执行                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  异常情况：外部推送延迟到 130s                                                 │
│  ─────────────────────────────────────────────────────────────────────────  │
│  检查点 3（假设在 120s 时执行）：                                             │
│  msSinceLastBeat = 120s - 90s = 30s（如果上次推送是 90s）→ 正常           │
│                                                                              │
│  但如果上次推送是 30s：                                                      │
│  msSinceLastBeat = 120s - 30s = 90s = 90000 ms                            │
│  90000 ms > 61000 ms → ❌ 超时，报错 "No heartbeat in the time window"  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.6 动态检查间隔

注意：Monitor 的检查间隔不是固定的 `interval`，而是**动态计算**的：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  静态间隔 vs 动态间隔                                                          │
└─────────────────────────────────────────────────────────────────────────────┘

静态思维（其他监控类型）：
每次检查间隔固定为 interval 秒

动态思维（Push 类型）：
根据上次心跳时间动态计算下一次检查时间

计算公式：
timeout = beatInterval * 1000 - msSinceLastBeat + bufferTime

示例（interval = 60s）：
┌──────────────────────────────────────────────────────────────────────────┐
│  场景 1：上次心跳在 30s 前                                              │
│  ────────────────────────────────────────────────────────────────────  │
│  msSinceLastBeat = 30000 ms                                             │
│  timeout = 60000 - 30000 + 1000 = 31000 ms = 31s                    │
│  下一次检查：现在 + 31s                                                    │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  场景 2：上次心跳在 55s 前（接近超时）                                       │
│  ────────────────────────────────────────────────────────────────────  │
│  msSinceLastBeat = 55000 ms                                             │
│  timeout = 60000 - 55000 + 1000 = 6000 ms = 6s                        │
│  下一次检查：现在 + 6s                                                     │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  场景 3：上次心跳在 70s 前（已超时）                                        │
│  ────────────────────────────────────────────────────────────────────  │
│  msSinceLastBeat = 70000 ms > 61000 ms → ❌ 超时，直接报错              │
└──────────────────────────────────────────────────────────────────────────┘
```

## 5. 双重检查机制详解

### 5.1 两条独立链路

Push 类型监控有两条**完全独立**的检查链路：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           双重检查机制架构                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  链路 A：外部推送 → API 记录心跳                                               │
│  ────────────────────────────────────────────────────────────────────────────│
│                                                                               │
│   外部服务                                                                     │
│       │                                                                        │
│       │ HTTP 请求（URL 查询参数）                                               │
│       ▼                                                                        │
│   ┌─────────────────────┐                                                        │
│   │ /api/push/:token │                                                        │
│   │  1. 验证 Token │                                                        │
│   │  2. 解析参数    │                                                        │
│   │  3. 确定状态     │                                                        │
│   │  4. 保存心跳    │                                                        │
│   │  5. 推送前端    │                                                        │
│   └─────────┬───────────┘                                                        │
│             │                                                                    │
│             ▼                                                                    │
│   ┌─────────────────────┐                                                        │
│   │  heartbeat 表       │                                                        │
│   │  - 记录每次推送     │                                                        │
│   │  - status = UP/DOWN  │                                                        │
│   └─────────────────────┘                                                        │
│                                                                               │
│  特点：被动接收，只记录外部服务主动推送的心跳                                │
│  不进行时间窗口验证                                                           │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  链路 B：Monitor 定时检查 → 时间窗口验证                                       │
│  ────────────────────────────────────────────────────────────────────────────│
│                                                                               │
│   Monitor.beat()                                                              │
│   (定时执行，间隔动态计算)                                                      │
│       │                                                                        │
│       │ 读取上次心跳记录                                                        │
│       ▼                                                                        │
│   ┌─────────────────────┐                                                        │
│   │ 时间窗口验证         │                                                        │
│   │ 1. 获取 previousBeat│                                                        │
│   │ 2. 计算 msSinceLast │                                                        │
│   │ 3. 检查条件 A/B     │                                                        │
│   └─────────┬───────────┘                                                        │
│             │                                                                    │
│     ┌───────┴───────┐                                                            │
│     │                 │                                                            │
│     ▼                 ▼                                                            │
│  超时              正常                                                          │
│     │                 │                                                            │
│     ▼                 ▼                                                            │
│  报错: "No       计算下次检查时间                                            │
│   heartbeat"        动态设置定时器                                              │
│  插入 DOWN        不插入心跳                                                  │
│  心跳记录                                                            │
│                                                                               │
│  特点：主动检查，验证时间窗口                                                │
│  即使外部没有推送，也能检测到超时                                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 两条链路的协作

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      两条链路如何协作                                             │
└─────────────────────────────────────────────────────────────────────────────┘

场景 1：外部服务正常推送

时间线：
0s:    Monitor 启动，无心跳记录 → 检查点 1 报错 "No heartbeat"
30s:    外部推送 status=up → API 记录心跳（status=UP）
61s:    Monitor 检查点 2 → 读取 30s 的心跳 → 正常，计算下次检查时间
90s:    外部推送 status=up → API 记录心跳（status=UP）
91s:    Monitor 检查点 3 → 读取 90s 的心跳 → 正常

结果：服务状态为 UP ✓

───────────────────────────────────────────────────────────────────────────────

场景 2：外部服务停止推送

时间线：
0s:    Monitor 启动
30s:    外部推送 status=up → API 记录心跳（status=UP）
61s:    Monitor 检查点 2 → 读取 30s 的心跳 → 正常
90s:    ❌ 外部未推送（服务宕机）
91s:    Monitor 检查点 3 → 读取 30s 的心跳
         msSinceLastBeat = 91s - 30s = 61s
         61s == 超时阈值（60+1）→ ❌ 超时！
         → 报错 "No heartbeat in the time window"
         → 插入 DOWN 心跳记录
         → 触发通知

结果：服务状态变为 DOWN ✓

───────────────────────────────────────────────────────────────────────────────

场景 3：外部服务推送 status=down

时间线：
0s:    Monitor 启动
30s:    外部推送 status=down → API 记录心跳
         → determineStatus 处理
         → 如果有 maxretries，可能变为 PENDING
         → 否则直接 DOWN
61s:    Monitor 检查点 2
         → 读取 30s 的心跳
         → 条件 A：上次状态不是 UP → ✅ 成立
         → 报错 "No heartbeat in the time window"
         → 但实际上外部已经是 DOWN/PENDING，保持状态

结果：服务状态为 DOWN/PENDING ✓
```

### 5.3 为什么需要两条链路？

| 链路 | 作用 | 单独使用的问题 |
|------|------|---------------|
| 链路 A（API 记录） | 接收外部推送的状态 | 如果外部服务宕机，无法推送状态，API 层无法知道服务已死 |
| 链路 B（时间窗口） | 检测超时 | 无法区分"服务正常但想主动报告 DOWN" |

**只有两条链路结合才能覆盖所有场景：

1. **服务正常**：外部推送 up → API 记录 UP → Monitor 验证时间窗口正常
2. **服务宕机**：外部不推送 → Monitor 检测超时 → 记录 DOWN
3. **服务异常但能推送**：外部推送 down → API 记录 DOWN/PENDING → Monitor 保持异常状态

## 6. 心跳数据结构

每次心跳会创建一个 `heartbeat` 记录，包含以下字段：

| 字段 | 类型 | 说明 | 来源 |
|------|------|------|------|
| time | string | ISO 格式时间戳（毫秒级） | 服务器当前时间 |
| monitor_id | number | 关联的监控器 ID | 通过 pushToken 查找 |
| ping | number | 延迟时间（毫秒） | URL 查询参数 `ping` |
| msg | string | 心跳消息 | URL 查询参数 `msg`（或维护模式） |
| status | number | 状态码 | determineStatus 计算 |
| duration | number | 距上次心跳的时间间隔（秒） | 计算 |
| retries | number | 重试次数 | determineStatus 计算 |
| downCount | number | 连续 DOWN 计数（用于通知重发） | 通知逻辑 |
| important | boolean | 是否为重要心跳（状态变化） | isImportantBeat |
| end_time | string | uptime 计算的结束时间 | UptimeCalculator |

### 6.1 状态码定义

```javascript
// src/util.js
const UP = 1;
const DOWN = 0;
const PENDING = 2;
const MAINTENANCE = 3;
```

## 7. 重要心跳与通知机制

### 7.1 重要心跳判定（isImportantBeat）

位置：`server/model/monitor.js:1419-1445`

用于判断是否需要记录为"重要心跳"（影响前端显示和历史记录）：

```javascript
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

### 7.2 状态转换重要性表

| 上次状态 | 当前状态 | 是否重要 | 说明 |
|---------|---------|---------|------|
| 无（首次） | 任意 | ✅ 是 | 第一次心跳 |
| UP | UP | ❌ 否 | 状态不变 |
| UP | PENDING | ❌ 否 | 开始重试 |
| UP | DOWN | ✅ 是 | 服务宕机 |
| UP | MAINTENANCE | ✅ 是 | 进入维护 |
| PENDING | PENDING | ❌ 否 | 重试中 |
| PENDING | DOWN | ✅ 是 | 重试失败，确认宕机 |
| PENDING | UP | ❌ 否 | 重试中恢复（不算重要） |
| DOWN | PENDING | - | 不存在此情况 |
| DOWN | DOWN | ❌ 否 | 持续宕机 |
| DOWN | UP | ✅ 是 | 服务恢复 |
| DOWN | MAINTENANCE | ✅ 是 | 进入维护 |
| MAINTENANCE | MAINTENANCE | ❌ 否 | 持续维护 |
| MAINTENANCE | UP | ✅ 是 | 维护结束恢复 |
| MAINTENANCE | DOWN | ✅ 是 | 维护结束但异常 |

### 7.3 通知触发判定（isImportantForNotification）

位置：`server/model/monitor.js:1454-1477`

用于判断是否需要发送通知（比 isImportantBeat 更严格）：

```javascript
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

### 7.4 通知触发场景

| 场景 | 是否发通知 | 说明 |
|------|-----------|------|
| 首次心跳 | ✅ 是 | 初始化通知 |
| UP → DOWN | ✅ 是 | 服务宕机告警 |
| PENDING → DOWN | ✅ 是 | 重试失败告警 |
| DOWN → UP | ✅ 是 | 服务恢复通知 |
| MAINTENANCE → DOWN | ✅ 是 | 维护结束但异常 |
| 进入 MAINTENANCE | ❌ 否 | 维护模式不通知 |
| 退出 MAINTENANCE 到 UP | ❌ 否 | 维护结束正常恢复 |
| 状态不变 | ❌ 否 | 持续状态不通知 |

### 7.5 通知重发机制

当服务持续 DOWN 时，支持按间隔重发通知：

```javascript
// server/routers/api-router.js:108-123
} else {
    if (bean.status === DOWN && monitor.resendInterval > 0) {
        ++bean.downCount;
        if (bean.downCount >= monitor.resendInterval) {
            // 再次发送通知，因为仍然 DOWN
            log.debug(
                "monitor",
                `[${monitor.name}] sendNotification again: Down Count: ${bean.downCount} | Resend Interval: ${monitor.resendInterval}`
            );
            await Monitor.sendNotification(isFirstBeat, monitor, bean);
            
            // 重置计数
            bean.downCount = 0;
        }
    }
}
```

**工作原理**：
1. `downCount` 记录连续 DOWN 的次数
2. 每次心跳如果状态为 DOWN 且 `resendInterval > 0`，`downCount + 1`
3. 当 `downCount >= resendInterval` 时，发送通知并重置计数

**示例**：
- resendInterval = 5
- 表示每 5 次 DOWN 心跳发送一次通知
- 如果 interval = 60s，那么每 5 分钟重发一次告警

## 8. 完整架构图

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      Uptime-Kuma Push 监控完整架构                                 │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      外部服务层（External Services）                                │
│                                                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                                     │
│  │  内部服务 A   │     │  内部服务 B   │     │  定时脚本     │                                     │
│  │ (无法外网访问)│     │ (主动上报状态)│     │ (cron/job)  │                                     │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘                                     │
│         │                       │                       │                                               │
│         │ 定时推送               │ 定时推送               │ 定时推送                                        │
│         │ HTTP GET/POST         │ HTTP GET/POST         │ HTTP GET/POST                                  │
│         │ URL 查询参数          │ URL 查询参数          │ URL 查询参数                                   │
│         ▼                       ▼                       ▼                                               │
│         │                       │                       │                                               │
│         └───────────────────────┼───────────────────────┘                                               │
│                                 │                                                                       │
│                                 ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐     │
│  │  推送 URL 示例：                                                                             │     │
│  │  https://your-uptime-kuma/api/push/abc123def456ghi789jkl012mno345pq?status=up&msg=OK   │     │
│  └──────────────────────────────────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ HTTPS（推荐）/ HTTP
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              Uptime-Kuma 后端服务（Backend）                                       │
│                                                                                                     │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │                                    Express.js Web 服务器                                      │ │
│  │  ┌──────────────────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │                           API Router (api-router.js)                                   │ │ │
│  │  │                                                                                         │                                                             │ │ │
│  │  │  ┌────────────────────────────────────────────────────────────────────────────────┐  │ │ │
│  │  │  │                    /api/push/:pushToken 端点（位置：47-146）                      │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  1. 解析参数（全部来自 request.query，不是 request.body！）                         │  │ │ │
│  │  │  │     - msg: request.query.msg || "OK"                                              │  │ │ │
│  │  │  │     - ping: parseFloat(request.query.ping) || null                                   │  │ │ │
│  │  │  │     - status: request.query.status || "up"                                          │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  2. 验证 pushToken（必须匹配 active=1 的 monitor）                                   │  │ │ │
│  │  │  │     R.findOne("monitor", " push_token = ? AND active = 1 ", [pushToken])          │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  3. 获取上次心跳（getPreviousHeartbeat）                                              │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  4. 检查维护状态（isUnderMaintenance）                                                │  │ │ │
│  │  │  │     - 是：status = MAINTENANCE, msg = "Monitor under maintenance"                  │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  5. 确定最终状态（determineStatus，位置：576-619）                                   │  │ │ │
│  │  │  │     - 考虑 maxretries 重试机制                                                       │  │ │ │
│  │  │  │     - 考虑 isUpsideDown 反转模式                                                     │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  6. 计算 uptime（UptimeCalculator）                                                │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  7. 标记重要心跳（isImportantBeat）                                                  │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  8. 检查通知（isImportantForNotification）                                            │  │ │ │
│  │  │  │     - 需要：发送通知，重置 downCount                                                   │  │ │ │
│  │  │  │     - 不需要：检查 resendInterval 重发逻辑                                            │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  9. 保存心跳到数据库（R.store(bean)）                                                │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  10. Socket.io 推送到前端                                                            │  │ │ │
│  │  │  │     io.to(monitor.user_id).emit("heartbeat", bean.toJSON())                          │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  11. 更新 Prometheus 指标                                                            │  │ │ │
│  │  │  │                                                                                     │  │ │ │
│  │  │  │  12. 返回响应 { ok: true }                                                            │  │ │ │
│  │  │  └────────────────────────────────────────────────────────────────────────────────┘  │ │ │
│  │  └──────────────────────────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                     │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              Monitor 定时检查（monitor.js）                                    │ │
│  │  ┌──────────────────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │                         Push 类型监控的时间窗口验证（位置：726-763）                                │ │ │
│  │  │                                                                                       │                                                               │ │ │
│  │  │  执行时机：定时执行，间隔动态计算（不是固定的 interval）                                   │ │ │
│  │  │                                                                                       │ │ │
│  │  │  1. 获取 previousBeat（上次心跳记录）                                                   │ │ │
│  │  │     R.findOne("heartbeat", " monitor_id = ? ORDER BY time DESC", [this.id])             │ │ │
│  │  │                                                                                       │ │ │
│  │  │  2. 计算 msSinceLastBeat = 当前时间 - 上次心跳时间                                      │ │ │
│  │  │                                                                                       │ │ │
│  │  │  3. 判定条件（满足任一条件则超时）：                                                      │ │ │
│  │  │     ┌─────────────────────────────────────────────────────────────────────────────┐ │ │ │
│  │  │     │ 条件 A：previousBeat.status !== (isUpsideDown ? DOWN : UP)                    │ │ │ │
│  │  │     │ 含义：上次状态不是正常状态（考虑反转模式）                                          │ │ │
│  │  │     ├─────────────────────────────────────────────────────────────────────────────┤ │ │ │
│  │  │     │ 条件 B：msSinceLastBeat > beatInterval * 1000 + bufferTime                  │ │ │ │
│  │  │     │ 其中：bufferTime = 1000ms（1秒缓冲）                                             │ │ │ │
│  │  │     │ 含义：距离上次心跳超过了（检查间隔 + 缓冲时间）                                      │ │ │ │
│  │  │     └─────────────────────────────────────────────────────────────────────────────┘ │ │ │
│  │  │                                                                                       │ │ │
│  │  │  4. 超时处理：                                                                         │ │ │
│  │  │     - 抛出错误："No heartbeat in the time window"                                         │ │ │
│  │  │     - 插入 DOWN 心跳记录                                                                 │ │ │
│  │  │     - 触发通知                                                                           │ │ │
│  │  │                                                                                       │ │ │
│  │  │  5. 正常处理：                                                                         │ │ │
│  │  │     - 计算下一次检查时间：                                                               │ │ │
│  │  │       timeout = beatInterval * 1000 - msSinceLastBeat + bufferTime                      │ │ │
│  │  │     - 设置定时器：this.heartbeatInterval = setTimeout(safeBeat, timeout)                  │ │ │
│  │  │     - 直接返回，不插入心跳记录（API 层已经处理）                                          │ │ │
│  │  │     - retries 重置为 0                                                                 │ │ │
│  │  └──────────────────────────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                     │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │                                    数据库层（RedBeanNode / SQLite）                           │ │
│  │                                                                                             │ │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐               │ │
│  │  │    monitor 表     │  │   heartbeat 表    │  │ monitor_maintenance 表   │               │ │
│  │  │                  │  │                  │  │                          │               │ │
│  │  │  - id            │  │  - id            │  │  - monitor_id             │               │ │
│  │  │  - name          │  │  - monitor_id    │  │  - maintenance_id         │               │ │
│  │  │  - type          │  │  - time          │  │                          │               │ │
│  │  │  - interval      │  │  - status        │  │                          │               │ │
│  │  │  - active        │  │  - msg           │  │                          │               │ │
│  │  │  - push_token    │  │  - ping          │  │                          │               │ │
│  │  │  - maxretries    │  │  - duration      │  │                          │               │ │
│  │  │  - resendInterval│  │  - retries       │  │                          │               │ │
│  │  │  - ...           │  │  - important     │  │                          │               │ │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────────────┘               │ │
│  │                                                                                             │ │
│  │  查询示例：                                                                                    │ │
│  │  - 验证 pushToken：                                                                       │ │
│  │    R.findOne("monitor", " push_token = ? AND active = 1 ", [pushToken])                       │ │
│  │                                                                                             │ │
│  │  - 获取上次心跳：                                                                            │ │
│  │    R.findOne("heartbeat", " monitor_id = ? ORDER BY time DESC", [monitorId])                 │ │
│  │                                                                                             │ │
│  │  - 保存心跳：                                                                                │ │
│  │    R.store(bean)                                                                          │ │
│  └────────────────────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ Socket.io 实时推送
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              前端 UI / 通知渠道（Frontend & Notifications）                        │
│                                                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────────────────────────┐   │
│  │    Web 界面      │  │    移动 App       │  │         第三方通知渠道                        │   │
│  │                  │  │                  │  │                                              │   │
│  │  - Dashboard     │  │  - 实时状态      │  │  - Email（邮件）                              │   │
│  │  - 监控详情页    │  │  - 推送通知      │  │  - Slack / Teams / Discord                   │   │
│  │  - 历史记录      │  │                  │  │  - Telegram / WeChat / DingTalk               │   │
│  │  - Push URL 显示  │  │                  │  │  - SMS / Phone Call                          │   │
│  │  - Token 重置    │  │                  │  │  - Webhook / Feishu / Line                   │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 9. 关键代码位置总结

| 功能模块 | 文件路径 | 行号范围 | 说明 |
|---------|---------|---------|------|
| Push API 端点 | `server/routers/api-router.js` | 47-146 | 接收外部推送，记录心跳 |
| 参数解析（重要！） | `server/routers/api-router.js` | 50-52 | **全部来自 request.query** |
| 状态确定函数 | `server/routers/api-router.js` | 576-619 | 处理重试、反转模式 |
| Push Token 生成 | `src/pages/EditMonitor.vue` | 3570-3576 | 前端生成 32 位 Token |
| Push URL 构建 | `src/pages/EditMonitor.vue` | 3271-3273 | 构建完整推送 URL |
| genSecret 算法 | `src/util.ts` | 555-563 | 加密安全随机字符串 |
| Push 类型时间窗口判定 | `server/model/monitor.js` | 726-763 | **核心：时间窗口验证** |
| 重要心跳判定 | `server/model/monitor.js` | 1419-1445 | 状态变化判定 |
| 通知触发判定 | `server/model/monitor.js` | 1454-1477 | 通知发送判定 |
| 获取上次心跳 | `server/model/monitor.js` | 1621-1623 | 数据库查询 |
| 心跳数据模型 | `server/model/heartbeat.js` | 1-82 | 数据结构定义 |
| 状态码定义 | `src/util.js` | - | UP/DOWN/PENDING/MAINTENANCE |

## 10. 最佳实践建议

### 10.1 推送频率设置

建议外部服务的推送间隔 **小于** 监控器配置的 interval：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  推荐配置                                                                      │
└─────────────────────────────────────────────────────────────────────────────┘

监控器 interval: 60 秒
外部推送间隔: 30-45 秒

原因：
1. 预留缓冲时间应对网络延迟
2. 避免因为时钟偏差导致误判
3. 给系统留出处理时间

极端情况示例：
监控器 interval = 60s，buffer = 1s
超时阈值 = 61s

如果外部推送间隔 = 60s：
- 第 0s: 推送
- 第 60s: 推送（刚好）
- 但如果网络延迟 2s，第 62s 才到达
- Monitor 在第 61s 检查时，msSinceLastBeat = 61s
- 61s == 超时阈值 → ❌ 误判！

所以推送间隔必须小于 interval！
```

### 10.2 参数传递方式（重要！）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ⚠️ 重要提醒：参数必须放在 URL 查询字符串中！                                   │
└─────────────────────────────────────────────────────────────────────────────┘

✅ 正确方式：
curl "http://your-uptime-kuma/api/push/token?status=up&msg=OK&ping=23"

requests.get(url, params={"status": "up", "msg": "OK"})

❌ 错误方式：
curl -X POST url -d "status=up&msg=OK"  # body 中的参数会被忽略！

requests.post(url, json={"status": "up"})  # json body 会被忽略！
```

### 10.3 高可用部署考虑

对于关键服务，建议：

1. **多区域推送**：从不同的网络位置推送心跳
2. **本地缓存**：如果 Uptime-Kuma 不可用，本地缓存心跳，恢复后补发
3. **监控 Uptime-Kuma 本身**：使用另一个监控系统监控 Uptime-Kuma 的可用性

### 10.4 安全建议

1. **保护 pushToken**：
   - 不要在代码中硬编码 pushToken
   - 使用环境变量或密钥管理服务
   - 定期轮换 pushToken（点击 "Reset Token" 按钮）

2. **网络安全**：
   - 使用 HTTPS 加密传输
   - 考虑 IP 白名单限制
   - 对于内网服务，使用 VPN 或内网通道

3. **监控配置**：
   - 合理设置 maxretries，避免瞬态网络问题导致误报
   - 配置合适的通知方式和重发间隔
   - 启用维护模式，避免计划内维护触发告警

## 11. 常见问题解答

### Q1: 为什么外部服务推送了心跳，但状态还是 DOWN？

可能的原因：

1. **参数传递方式错误**：
   - 参数放在了 request.body 中，而不是 URL 查询字符串
   - 检查代码：确认使用 `?status=up&msg=OK` 格式

2. **推送间隔超过了 interval**：
   - 监控器 interval = 60s
   - 外部推送间隔 = 70s
   - 超时阈值 = 61s
   - 70s > 61s → 超时！

3. **Token 不匹配**：
   - 检查 pushToken 是否正确
   - 检查监控器是否 active=1

4. **处于维护模式**：
   - 状态会显示为 MAINTENANCE

### Q2: POST 请求可以用吗？

可以用，但参数必须放在 **URL 查询字符串**中：

```bash
# ✅ 正确：POST 请求，但参数在 URL 中
curl -X POST "http://your-uptime-kuma/api/push/token?status=up"

# ❌ 错误：POST 请求，参数在 body 中
curl -X POST "http://your-uptime-kuma/api/push/token" -d "status=up"
```

### Q3: 为什么需要两条检查链路？

| 场景 | 只有 API 链路 | 只有 Monitor 链路 | 两条链路都有 |
|------|-------------|------------------|-------------|
| 服务正常推送 | ✅ 正常 | ✅ 正常 | ✅ 正常 |
| 服务宕机（不推送） | ❌ 不知道 | ✅ 检测超时 | ✅ 检测超时 |
| 服务异常（推送 down） | ✅ 记录 DOWN | ✅ 保持 DOWN | ✅ 正常 |

### Q4: 如何测试 Push 监控？

1. **立即测试：
   - 复制 Push URL 到浏览器访问
   - 应该返回 `{"ok": true}`
   - 检查监控状态是否变为 UP

2. **测试超时**：
   - 停止外部推送
   - 等待 interval + buffer 时间
   - 检查监控状态是否变为 DOWN

---

*分析基于 Uptime-Kuma 代码库，生成时间：2026-05-05*

*关键更正记录：*
- *更正：参数来源为 `request.query`（URL 查询参数），不是 `request.body`*
- *补充：完整的接入链路（前端创建 → Token 生成 → 外部推送 → 双重检查）*
- *详细说明：两条独立检查链路的协作机制*
