# Uptime-Kuma Push 类型监控机制分析

## 1. 概述

Uptime-Kuma 的 push 类型监控采用了**被动接收**的模式，与传统的主动轮询（如 HTTP、ping 类型）不同。在这种模式下：

- **外部服务**主动向 Uptime-Kuma 推送心跳信号
- **Uptime-Kuma** 接收并验证这些心跳，根据时间窗口判断服务状态
- 适用于无法被外部直接访问的内部服务，或者需要主动上报状态的场景

## 2. 外部推送 Gateway 接入机制

### 2.1 接入端点

Uptime-Kuma 提供了统一的 HTTP API 端点用于接收外部推送：

```
/api/push/:pushToken
```

- **位置**: `server/routers/api-router.js:47-146`
- **HTTP 方法**: 支持所有 HTTP 方法（GET、POST、PUT 等）
- **认证方式**: 通过 URL 路径中的 `pushToken` 进行身份验证

### 2.2 接入流程

1. **创建 Push 类型监控**：在 Uptime-Kuma 界面创建类型为 "Push" 的监控器
2. **获取 Push Token**：系统自动生成一个唯一的 `pushToken`
3. **外部服务配置**：在外部服务中配置定时任务，向 `/api/push/{pushToken}` 发送请求

### 2.3 请求参数

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| pushToken | string | 是（URL 路径） | - | 监控器的唯一标识令牌 |
| msg | string | 否 | "OK" | 心跳消息内容 |
| ping | number | 否 | null | 延迟时间（毫秒），范围 0 ~ 100,000,000,000 ms |
| status | string | 否 | "up" | 服务状态，可选值: "up" 或 "down" |

### 2.4 接入示例

**使用 curl 推送心跳：**
```bash
# 基本用法
curl "http://your-uptime-kuma/api/push/your-push-token"

# 带参数
curl "http://your-uptime-kuma/api/push/your-push-token?msg=Service%20Healthy&ping=23&status=up"

# POST 请求
curl -X POST "http://your-uptime-kuma/api/push/your-push-token" \
  -d "msg=Service%20Running&ping=15"
```

**在脚本中使用（Python）：**
```python
import requests
import time

PUSH_TOKEN = "your-push-token"
UPTIME_KUMA_URL = "http://your-uptime-kuma"

def send_heartbeat():
    try:
        # 执行健康检查逻辑
        is_healthy = check_service_health()
        
        params = {
            "msg": "Service is running" if is_healthy else "Service has issues",
            "status": "up" if is_healthy else "down"
        }
        
        response = requests.get(
            f"{UPTIME_KUMA_URL}/api/push/{PUSH_TOKEN}",
            params=params,
            timeout=10
        )
        print(f"Heartbeat sent: {response.json()}")
    except Exception as e:
        print(f"Failed to send heartbeat: {e}")

# 定时发送心跳（建议间隔小于监控器配置的间隔）
while True:
    send_heartbeat()
    time.sleep(30)  # 每 30 秒发送一次
```

### 2.5 Token 验证机制

```javascript
// server/routers/api-router.js:62
let monitor = await R.findOne("monitor", " push_token = ? AND active = 1 ", [pushToken]);

if (!monitor) {
    throw new Error("Monitor not found or not active.");
}
```

- 系统会在数据库中查找匹配 `pushToken` 且状态为 `active=1` 的监控器
- 如果找不到或者监控器未激活，返回 404 错误

## 3. 后端心跳处理流程

### 3.1 整体处理流程

当外部服务向 `/api/push/:pushToken` 发送请求时，后端的处理流程如下：

```
1. 接收请求
     ↓
2. 解析参数（msg, ping, status）
     ↓
3. 验证 ping 值范围
     ↓
4. 通过 pushToken 查找监控器
     ↓
5. 获取上次心跳记录
     ↓
6. 检查是否处于维护状态
     ↓
7. 确定最终状态（determineStatus）
     ↓
8. 计算 uptime
     ↓
9. 标记重要心跳（isImportantBeat）
     ↓
10. 检查是否需要发送通知
     ↓
11. 保存心跳到数据库
     ↓
12. 通过 Socket.io 推送到前端
     ↓
13. 更新 Prometheus 指标
     ↓
14. 返回响应 { ok: true }
```

### 3.2 心跳数据结构

每次心跳会创建一个 `heartbeat` 记录，包含以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| time | string | ISO 格式时间戳（毫秒级） |
| monitor_id | number | 关联的监控器 ID |
| ping | number | 延迟时间（毫秒） |
| msg | string | 心跳消息 |
| status | number | 状态码（0=DOWN, 1=UP, 2=PENDING, 3=MAINTENANCE） |
| duration | number | 距上次心跳的时间间隔（秒） |
| retries | number | 重试次数 |
| downCount | number | 连续 DOWN 计数（用于通知重发） |
| important | boolean | 是否为重要心跳（状态变化） |
| end_time | string | uptime 计算的结束时间 |

### 3.3 状态码定义

```javascript
// src/util.js
const UP = 1;
const DOWN = 0;
const PENDING = 2;
const MAINTENANCE = 3;
```

## 4. 状态判定逻辑

### 4.1 状态确定函数（determineStatus）

位置：`server/routers/api-router.js:576-619`

该函数根据以下因素综合确定最终状态：

1. **请求中的 status 参数**（默认为 "up"）
2. **上次心跳状态**（previousHeartbeat.status）
3. **最大重试次数**（maxretries）
4. **是否为反转模式**（isUpsideDown）

### 4.2 状态转换规则

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

#### 场景 3：服务恢复（DOWN → UP）

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

### 4.3 反转模式（Upside Down）

Uptime-Kuma 支持"反转模式"，即：
- 正常情况下报告为 DOWN
- 异常情况下报告为 UP

适用于监控"某种不应该发生的事件"（例如：错误日志出现）。

```javascript
// server/routers/api-router.js:577-579
if (isUpsideDown) {
    status = flipStatus(status);
}

// flipStatus 实现
const flipStatus = (status) => {
    if (status === UP) return DOWN;
    if (status === DOWN) return UP;
    return status;
};
```

## 5. 时间窗口判定机制

### 5.1 核心概念

Push 类型监控的核心在于**时间窗口判定**。即使外部服务推送了心跳，后端仍会独立检查是否在规定时间内收到心跳。

位置：`server/model/monitor.js:726-763`

### 5.2 判定流程

监控器启动后，会在 `beat()` 函数中执行以下逻辑：

```javascript
// server/model/monitor.js:726-763
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

### 5.3 关键参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| bufferTime | 1000 ms | 缓冲时间，用于适应时钟差异和网络延迟 |
| beatInterval | 监控器配置的 interval | 检查间隔（秒） |
| msSinceLastBeat | 动态计算 | 距上次心跳的毫秒数 |

### 5.4 判定条件

#### 条件 1：有之前的心跳记录

当满足以下**任一**条件时，判定为**超时（DOWN）**：

```
条件 A: 上次心跳状态不是 UP（考虑反转模式）
   说明：服务之前已经异常，继续保持异常状态

条件 B: msSinceLastBeat > beatInterval * 1000 + bufferTime
   说明：距离上次心跳的时间超过了（检查间隔 + 缓冲时间）
```

#### 条件 2：无之前的心跳记录

```
直接判定为超时
错误消息："No heartbeat in the time window"
```

### 5.5 正常情况的处理

当在时间窗口内收到心跳时：

1. **计算下一次检查时间**：
   ```javascript
   timeout = beatInterval * 1000 - msSinceLastBeat + bufferTime
   ```
   
2. **设置定时器**：
   ```javascript
   this.heartbeatInterval = setTimeout(safeBeat, timeout);
   ```

3. **直接返回**：
   - 不插入新的心跳记录（因为外部推送已经处理）
   - retries 重置为 0

### 5.6 时间窗口图示

假设监控器配置的 interval = 60 秒：

```
时间轴：
0s          60s        120s       180s
│           │          │          │
├───────────┼──────────┼──────────┼──►
│           │          │          │
▼           ▼          ▼          ▼
检查点1    检查点2    检查点3    检查点4

外部推送时间点：
   30s        90s       150s
    │          │          │
    ▼          ▼          ▼
   推送1      推送2      推送3

判定逻辑：
检查点2（60s）:
  msSinceLastBeat = 60s - 30s = 30s
  30s <= 60s + 1s（buffer）→ 正常，不报错

检查点3（120s）:
  msSinceLastBeat = 120s - 90s = 30s
  30s <= 61s → 正常

如果外部推送延迟到 130s：
检查点3（120s）:
  msSinceLastBeat = 120s - 90s = 30s（假设上次是 90s）
  但如果上次是 30s：
  msSinceLastBeat = 120s - 30s = 90s
  90s > 61s → 判定为超时，报错
```

## 6. 重要心跳与通知机制

### 6.1 重要心跳判定（isImportantBeat）

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

### 6.2 状态转换重要性表

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

### 6.3 通知触发判定（isImportantForNotification）

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

### 6.4 通知触发场景

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

### 6.5 通知重发机制

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

## 7. 完整架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        外部服务（External Services）              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │  Service │  │  Service │  │  Service │                       │
│  │    A     │  │    B     │  │    C     │                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│       │              │              │                              │
│       │ 定时推送      │ 定时推送      │ 定时推送                     │
│       ▼              ▼              ▼                              │
└───────┼──────────────┼──────────────┼─────────────────────────────┘
        │              │              │
        │   HTTP 请求   │   HTTP 请求   │   HTTP 请求
        │ /api/push/   │ /api/push/   │ /api/push/
        │ {pushToken}  │ {pushToken}  │ {pushToken}
        ▼              ▼              ▼
┌───────┴──────────────┴──────────────┴─────────────────────────────┐
│                    Uptime-Kuma 后端服务                             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    API Router (api-router.js)                 │ │
│  │  ┌────────────────────────────────────────────────────────┐  │ │
│  │  │              /api/push/:pushToken 端点                   │  │ │
│  │  │  1. 解析参数：msg, ping, status                          │  │ │
│  │  │  2. 验证 pushToken → 查找 monitor                        │  │ │
│  │  │  3. 检查维护状态                                         │  │ │
│  │  │  4. determineStatus() → 确定最终状态                     │  │ │
│  │  │  5. 保存 heartbeat 到数据库                               │  │ │
│  │  │  6. Socket.io 推送到前端                                  │  │ │
│  │  │  7. 返回 { ok: true }                                    │  │ │
│  │  └────────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    Monitor 定时检查 (monitor.js)              │ │
│  │  ┌────────────────────────────────────────────────────────┐  │ │
│  │  │              Push 类型监控的时间窗口判定                  │  │ │
│  │  │  1. 获取 previousBeat（上次心跳）                        │  │ │
│  │  │  2. 计算 msSinceLastBeat                                 │  │ │
│  │  │  3. 判定条件：                                           │  │ │
│  │  │     - 上次状态非 UP？                                     │  │ │
│  │  │     - 超时？(msSinceLastBeat > interval + buffer)       │  │ │
│  │  │  4. 超时 → 报错 "No heartbeat in the time window"        │  │ │
│  │  │  5. 正常 → 计算下一次检查时间，设置定时器                  │  │ │
│  │  └────────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                      数据库层 (RedBeanNode)                   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │ │
│  │  │   monitor    │  │  heartbeat   │  │ monitor_maintenance│  │ │
│  │  │  表：存储监控  │  │  表：存储心跳  │  │    表：维护关联     │  │ │
│  │  │  器配置      │  │  历史记录     │  │                  │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘  │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
└───────┬──────────────────────────────────────────────────────────────┘
        │
        │ Socket.io 推送
        ▼
┌───────┴──────────────────────────────────────────────────────────────┐
│                        前端 UI / 通知渠道                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │   Web 界面   │  │   移动 App   │  │  第三方通知（Email/Slack/ │  │
│  │              │  │              │  │  Telegram/DingTalk 等）  │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

## 8. 最佳实践建议

### 8.1 推送频率设置

建议外部服务的推送间隔 **小于** 监控器配置的 interval：

```
推荐配置：
监控器 interval: 60 秒
外部推送间隔: 30-45 秒

原因：
1. 预留缓冲时间应对网络延迟
2. 避免因为时钟偏差导致误判
3. 给系统留出处理时间
```

### 8.2 高可用部署考虑

对于关键服务，建议：

1. **多区域推送**：从不同的网络位置推送心跳
2. **本地缓存**：如果 Uptime-Kuma 不可用，本地缓存心跳，恢复后补发
3. **监控 Uptime-Kuma 本身**：使用另一个监控系统监控 Uptime-Kuma 的可用性

### 8.3 安全建议

1. **保护 pushToken**：
   - 不要在代码中硬编码 pushToken
   - 使用环境变量或密钥管理服务
   - 定期轮换 pushToken

2. **网络安全**：
   - 使用 HTTPS 加密传输
   - 考虑 IP 白名单限制
   - 对于内网服务，使用 VPN 或内网通道

3. **监控配置**：
   - 合理设置 maxretries，避免瞬态网络问题导致误报
   - 配置合适的通知方式和重发间隔
   - 启用维护模式，避免计划内维护触发告警

## 9. 关键代码位置总结

| 功能模块 | 文件路径 | 行号范围 |
|---------|---------|---------|
| Push API 端点 | `server/routers/api-router.js` | 47-146 |
| 状态确定函数 | `server/routers/api-router.js` | 576-619 |
| Push 类型时间窗口判定 | `server/model/monitor.js` | 726-763 |
| 重要心跳判定 | `server/model/monitor.js` | 1419-1445 |
| 通知触发判定 | `server/model/monitor.js` | 1454-1477 |
| 获取上次心跳 | `server/model/monitor.js` | 1621-1623 |
| 心跳数据模型 | `server/model/heartbeat.js` | 1-82 |
| 状态码定义 | `src/util.js` | - |

---

*分析基于 Uptime-Kuma 代码库，生成时间：2026-05-05*
