# Uptime-Kuma Push 类型监控机制分析（精简版）

## 1. 核心架构

Push 类型监控采用 **双重检查机制**：

| 链路 | 触发方式 | 核心职责 | 代码位置 |
|------|---------|---------|---------|
| 链路 A（API） | 外部推送触发 | 接收并记录心跳状态 | `server/routers/api-router.js:47-146` |
| 链路 B（Monitor） | 定时触发 | 独立验证时间窗口 | `server/model/monitor.js:726-763` |

---

## 2. 外部推送接入链路

### 2.1 接入流程

```
前端创建 Push 监控 → 生成 32 位 Token → 构建 Push URL → 外部服务推送 → API 接收处理
```

### 2.2 Token 生成

**位置**：`src/pages/EditMonitor.vue:3570-3576`

```javascript
if (this.monitor.type === "push") {
    if (!this.monitor.pushToken) {
        this.monitor.pushToken = genSecret(32); // 32位随机字符串
    }
}
```

**Push URL 结构**：
```
{baseURL}/api/push/{32位Token}?status=up&msg=OK&ping=
```

### 2.3 ⚠️ 关键证据：参数来源

**所有参数都来自 URL 查询字符串，不是请求体！**

**位置**：`server/routers/api-router.js:50-52`

```javascript
let msg = request.query.msg || "OK";
let ping = parseFloat(request.query.ping) || null;
let statusString = request.query.status || "up";
```

### 2.4 请求参数

| 参数 | 来源 | 默认值 | 说明 |
|------|------|--------|------|
| pushToken | URL 路径 | - | 32 位唯一标识（必填） |
| status | URL 查询 | "up" | 状态："up" 或 "down" |
| msg | URL 查询 | "OK" | 心跳消息 |
| ping | URL 查询 | null | 延迟（毫秒，范围 0 ~ 1000亿） |

### 2.5 正确与错误示例

**✅ 正确方式（参数在 URL 中）**：
```bash
# GET
curl "http://uptime-kuma/api/push/{token}?status=up&msg=OK"

# POST（参数仍需在 URL 中！）
curl -X POST "http://uptime-kuma/api/push/{token}?status=up"
```

**❌ 错误方式（参数在 body 中，会被忽略）**：
```bash
curl -X POST "http://uptime-kuma/api/push/{token}" -d "status=up"
```

### 2.6 Token 验证

**位置**：`server/routers/api-router.js:62-66`

```javascript
let monitor = await R.findOne(
    "monitor", 
    " push_token = ? AND active = 1 ", 
    [pushToken]
);
if (!monitor) {
    throw new Error("Monitor not found or not active.");
}
```

---

## 3. 时间窗口判定逻辑（核心）

### 3.1 核心概念

**即使外部服务推送了心跳，Monitor 层仍会独立验证时间窗口。**

这是 Push 监控最关键的机制——确保外部服务宕机（无法推送）时也能被检测到。

### 3.2 核心代码

**位置**：`server/model/monitor.js:726-763`

```javascript
} else if (this.type === "push") {
    const bufferTime = 1000; // 1秒缓冲

    if (previousBeat) {
        const msSinceLastBeat = dayjs.utc().valueOf() - dayjs.utc(previousBeat.time).valueOf();

        // 超时判定：满足任一条件即超时
        if (
            previousBeat.status !== (this.isUpsideDown() ? DOWN : UP) ||  // 条件 A
            msSinceLastBeat > beatInterval * 1000 + bufferTime             // 条件 B
        ) {
            bean.duration = Math.round(msSinceLastBeat / 1000);
            throw new Error("No heartbeat in the time window");
        } else {
            // 正常：动态计算下次检查时间
            let timeout = beatInterval * 1000 - msSinceLastBeat;
            timeout = (timeout < 0) ? bufferTime : (timeout + bufferTime);
            retries = 0;
            this.heartbeatInterval = setTimeout(safeBeat, timeout);
            return; // 不插入心跳（API 层已处理）
        }
    } else {
        // 无上次心跳 → 直接超时
        bean.duration = beatInterval;
        throw new Error("No heartbeat in the time window");
    }
}
```

### 3.3 超时判定条件

| 条件 | 代码逻辑 | 说明 |
|------|---------|------|
| 条件 A | `previousBeat.status !== (isUpsideDown ? DOWN : UP)` | 上次状态不是正常状态（考虑反转模式） |
| 条件 B | `msSinceLastBeat > beatInterval * 1000 + bufferTime` | 距上次心跳超过（检查间隔 + 1秒缓冲） |

### 3.4 关键参数

| 参数 | 值 | 说明 |
|------|-----|------|
| `beatInterval` | 监控器配置的 `interval` | 检查间隔（秒） |
| `bufferTime` | 1000ms | 缓冲时间，适应时钟差异 |
| `msSinceLastBeat` | 动态计算 | 当前时间 - 上次心跳时间（毫秒） |

### 3.5 计算示例

假设 `interval = 60` 秒：

```
超时阈值 = 60 × 1000 + 1000 = 61000ms = 61秒
```

**场景判定**：

| 上次心跳时间 | msSinceLastBeat | 判定 | 原因 |
|-------------|-----------------|------|------|
| 30秒前 | 30000ms | ✅ 正常 | 30s < 61s |
| 60秒前 | 60000ms | ✅ 正常 | 60s < 61s |
| 61秒前 | 61000ms | ✅ 正常 | 61s > 61s 为 false（注意是 > 不是 >=） |
| 62秒前 | 62000ms | ❌ 超时 | 62s > 61s |

### 3.6 动态检查间隔

检查间隔不是固定的，而是**动态计算**：

```javascript
timeout = beatInterval * 1000 - msSinceLastBeat + bufferTime
```

**示例**（interval = 60s）：

| 上次心跳时间 | msSinceLastBeat | 下次检查时间 |
|-------------|-----------------|-------------|
| 30秒前 | 30000ms | 现在 + 31秒 |
| 55秒前 | 55000ms | 现在 + 6秒 |
| 70秒前 | 70000ms | 已超时 |

---

## 4. 双重检查协作

### 4.1 为什么需要两条链路？

| 场景 | 只有 API 链路 | 只有 Monitor 链路 | 两条链路都有 |
|------|-------------|------------------|-------------|
| 服务正常推送 | ✅ 正常 | ✅ 正常 | ✅ 正常 |
| 服务宕机（不推送） | ❌ 无法检测 | ✅ 检测超时 | ✅ 检测超时 |
| 服务主动报告 DOWN | ✅ 记录 DOWN | ✅ 保持 DOWN | ✅ 正常 |

### 4.2 协作流程

**场景：服务宕机（停止推送）**

```
时间线（interval = 60s）：

0s:  Monitor 启动
30s: 外部推送 status=up → API 记录 UP
61s: Monitor 检查 → 读取 30s 心跳 → 30s < 61s → 正常
90s: ❌ 外部未推送（服务宕机）
91s: Monitor 检查 → 读取 30s 心跳
     msSinceLastBeat = 91s - 30s = 61s
     61s > 61s? （代码是 >，不是 >=）
     
     实际上：由于执行延迟，通常 msSinceLastBeat > 61s
     → 报错 "No heartbeat in the time window"
     → 插入 DOWN 心跳
     → 触发通知

结果：服务状态变为 DOWN ✓
```

---

## 5. 关键代码位置汇总

| 功能 | 文件 | 行号 |
|------|------|------|
| API 端点定义 | `server/routers/api-router.js` | 47-146 |
| **参数来源（关键证据）** | `server/routers/api-router.js` | 50-52 |
| Token 验证 | `server/routers/api-router.js` | 62-66 |
| **时间窗口判定（核心）** | `server/model/monitor.js` | 726-763 |
| Token 生成 | `src/pages/EditMonitor.vue` | 3570-3576 |
| Push URL 构建 | `src/pages/EditMonitor.vue` | 3271-3273 |

---

## 6. 常见问题

### Q1: 为什么推送了心跳但状态还是 DOWN？

1. **参数放错位置**：参数放在了 `request.body` 中，应该放在 URL 查询字符串
2. **推送间隔超过 interval**：interval = 60s，但推送间隔 = 70s → 超时
3. **Token 不匹配**：检查 pushToken 和监控器 active 状态

### Q2: POST 请求可以用吗？

可以，但参数必须放在 **URL 查询字符串**中：

```bash
# ✅ 正确
curl -X POST "http://uptime-kuma/api/push/{token}?status=up"

# ❌ 错误（body 中的参数会被忽略）
curl -X POST "http://uptime-kuma/api/push/{token}" -d "status=up"
```

### Q3: 推送间隔建议？

建议推送间隔 **小于** interval：

```
推荐配置：
监控器 interval: 60 秒
外部推送间隔: 30-45 秒

原因：
- 预留缓冲时间
- 避免时钟偏差导致误判
```

---

*分析基于 Uptime-Kuma 代码库，生成时间：2026-05-05*

**关键证据摘要**：
1. **参数来源**：`request.query`（URL 查询参数），不是 `request.body`
2. **超时判定**：`msSinceLastBeat > beatInterval * 1000 + bufferTime`
3. **双重检查**：API 记录状态 + Monitor 独立验证时间窗口
