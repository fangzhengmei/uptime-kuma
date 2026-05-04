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

## 3. 实例信息到 Prometheus 标签的转换规则

### 3.1 整体转换流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Monitor 实例信息 → Prometheus 标签                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  阶段 1: 全局初始化 (Prometheus.init())                                       │
│  ─────────────────────────────────────────                                  │
│  数据库 tag 表                                                                │
│       │                                                                      │
│       ▼                                                                      │
│  sanitizeForPrometheus(tag.name)  ──► 清洗标签名                             │
│       │                                                                      │
│       ▼                                                                      │
│  filter(tagName !== "")  ──► 过滤空值                                        │
│       │                                                                      │
│       ▼                                                                      │
│  sort(sortTags)  ──► 不区分大小写字母排序                                     │
│       │                                                                      │
│       ▼                                                                      │
│  new Set()  ──► 去重 (处理清洗后重名的情况)                                   │
│       │                                                                      │
│       ▼                                                                      │
│  合并内置标签 [monitor_id, monitor_name, ...]                               │
│       │                                                                      │
│       ▼                                                                      │
│  commonLabels  ──► 注册到 Gauge 的 labelNames                                │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  阶段 2: 每个 Monitor 实例 (Prometheus 构造函数)                             │
│  ──────────────────────────────────────────────────────                     │
│  Monitor 对象 + Tags 数组                                                     │
│       │                                                                      │
│       ▼                                                                      │
│  mapTagsToLabels(tags)  ──► 转换自定义标签                                    │
│       │                                                                      │
│       ▼                                                                      │
│  合并内置标签值                                                                │
│  { monitor_id, monitor_name, monitor_type, ... }                            │
│       │                                                                      │
│       ▼                                                                      │
│  monitorLabelValues  ──► 用于 .set() 时的标签值                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 标签名/标签值清洗规则

**核心函数**: `server/prometheus.js:105-109`

```javascript
static sanitizeForPrometheus(text) {
    text = text.replace(/[^a-zA-Z0-9_]/g, "");  // 第1步
    text = text.replace(/^[^a-zA-Z_]+/, "");       // 第2步
    return text;
}
```

#### 清洗规则详解

| 步骤 | 正则表达式 | 作用 | 示例 |
|------|-----------|------|------|
| 1 | `[^a-zA-Z0-9_]` | 移除非字母、数字、下划线的所有字符 | `"环境@prod"` → `"环境prod"` |
| 2 | `^[^a-zA-Z_]+` | 确保开头是字母或下划线 | `"123_test"` → `"test"` |

**注意**: 步骤1 会移除中文等非 ASCII 字符！

#### 清洗示例表

| 原始值 (tag.name 或 tag.value) | 步骤1 结果 | 步骤2 结果 | 最终是否有效 |
|-------------------------------|------------|------------|-------------|
| `"environment"` | `"environment"` | `"environment"` | ✅ 有效 |
| `"env-prod"` | `"envprod"` | `"envprod"` | ✅ 有效 |
| `"环境"` | `""` | `""` | ❌ 空，被过滤 |
| `"环境@prod"` | `"环境prod"` → `""` (中文被移除) | `""` | ❌ 空，被过滤 |
| `"123_test"` | `"123_test"` | `"test"` | ✅ 有效 |
| `"_private"` | `"_private"` | `"_private"` | ✅ 有效 |
| `"@special"` | `"special"` | `"special"` | ✅ 有效 |
| `"12345"` | `"12345"` | `""` | ❌ 空，被过滤 |
| `"my tag 123"` | `"mytag123"` | `"mytag123"` | ✅ 有效 |

#### 设计意图

根据代码注释 (prometheus.js:101):
> See https://github.com/louislam/uptime-kuma/pull/4704#issuecomment-2366524692

这是为了符合 **Prometheus 标签命名规范**:
- 标签名必须匹配正则: `[a-zA-Z_][a-zA-Z0-9_]*`
- 必须以字母或下划线开头
- 只能包含字母、数字、下划线

### 3.3 去重规则

**触发时机**: `Prometheus.init()` 阶段

**代码位置**: `server/prometheus.js:38-50`

```javascript
const tags = new Set(
    (await R.findAll("tag"))
        .map((tag) => {
            return Prometheus.sanitizeForPrometheus(tag.name);
        })
        .filter((tagName) => {
            return tagName !== "";
        })
        .sort(this.sortTags)
);
```

#### 去重场景说明

根据代码注释 (prometheus.js:40):
> "use Set to remove possible duplicates (for when multiple tags contain non-ascii characters, and thus are sanitized to the same label)"

**典型场景**: 不同的原始标签名，清洗后变成相同名称。

```
示例:

数据库中的 tag 记录:
┌────┬──────────────────┬─────────┐
│ id │ name             │ value   │
├────┼──────────────────┼─────────┤
│ 1  │ 环境@prod        │ web     │  ← 原始标签名1
│ 2  │ 环境#test        │ api     │  ← 原始标签名2
│ 3  │ 环境%staging     │ db      │  ← 原始标签名3
└────┴──────────────────┴─────────┘

清洗过程:
tag.name = "环境@prod"
  步骤1: 移除 @ → "环境prod"
  步骤2: 中文被步骤1的 [^a-zA-Z0-9_] 移除 → "prod"
  结果: "prod"

tag.name = "环境#test"
  步骤1: 移除 # → "环境test"
  步骤2: 中文被移除 → "test"
  结果: "test"  (这个例子结果不同，让我找一个真正会重名的)

更好的重名示例:
数据库 tag 记录:
┌────┬──────────────────┬─────────┐
│ id │ name             │ value   │
├────┼──────────────────┼─────────┤
│ 1  │ 中文@env         │ prod    │  ← 原始标签名1
│ 2  │ 中文#env         │ test    │  ← 原始标签名2
└────┴──────────────────┴─────────┘

清洗后:
- "中文@env" → 步骤1移除中文和@ → "env"
- "中文#env" → 步骤1移除中文和# → "env"

结果: 两个不同的原始标签，清洗后都变成 "env"
      使用 new Set() 后只保留一个 "env"
```

#### 去重的影响

| 阶段 | 影响 |
|------|------|
| **Gauge labelNames 注册** | 只保留一个 `env` 作为标签名 |
| **Monitor 实例标签值** | 两个 tag 的 value 会被合并到同一个 `env` 标签下，变成数组 |

### 3.4 排序规则

#### 标签名排序

**函数**: `server/prometheus.js:265-278`

```javascript
sortTags(a, b) {
    const aLowerCase = a.toLowerCase();
    const bLowerCase = b.toLowerCase();

    if (aLowerCase < bLowerCase) {
        return -1;
    }

    if (aLowerCase > bLowerCase) {
        return 1;
    }

    return 0;
}
```

**规则**: 不区分大小写的字母顺序排序

**排序示例**:
```
原始标签名列表: ["Zebra", "apple", "Banana", "_internal", "monitor_id"]

排序后 (不区分大小写):
["_internal", "apple", "Banana", "monitor_id", "Zebra"]

注意: 下划线 '_' 的 ASCII 码 (95) 小于小写字母 'a' (97)
      所以 "_internal" 排在最前面
```

#### 标签值排序

**位置**: `server/prometheus.js:133`

```javascript
mappedTags[sanitizedTag] = mappedTags[sanitizedTag].sort();
```

**规则**: JavaScript 默认字符串排序 (按 Unicode 码点)

**排序示例**:
```
同一标签下的多个值: ["prod", "Test", "123", "staging"]

排序后:
["123", "Test", "prod", "staging"]

解释:
- "123" (ASCII 49) < "Test" (ASCII 84) < "prod" (ASCII 112) < "staging" (ASCII 115)
- 注意: 大写字母 (65-90) < 小写字母 (97-122)
```

### 3.5 空值处理策略

#### 标签名空值处理

**两处过滤**:

1. **`init()` 阶段** (prometheus.js:46-48):
```javascript
.filter((tagName) => {
    return tagName !== "";
})
```

2. **`mapTagsToLabels()` 阶段** (prometheus.js:120-122):
```javascript
let sanitizedTag = Prometheus.sanitizeForPrometheus(tag.name);
if (sanitizedTag === "") {
    return; // Skip empty tag names
}
```

**空值场景**:
- 原始 tag.name 全是非 ASCII 字符 (如纯中文)
- 原始 tag.name 清洗后开头全是数字且没有字母/下划线

#### 标签值空值处理

**位置**: `server/prometheus.js:128-131`

```javascript
let tagValue = Prometheus.sanitizeForPrometheus(tag.value || "");
if (tagValue !== "") {
    mappedTags[sanitizedTag].push(tagValue);
}
```

**处理逻辑**:
1. `tag.value || ""` - null/undefined 转为空字符串
2. 清洗后仍为空的，不加入标签值数组

**空值场景示例**:
```
Tag 记录: { name: "env", value: "生产环境" }

清洗过程:
tag.value = "生产环境"
sanitizeForPrometheus("生产环境") → 中文被移除 → ""

结果: 标签名 "env" 存在，但该 tag 不贡献任何值
      如果这是唯一的 tag，mappedTags["env"] = [] (空数组)
```

### 3.6 自定义标签映射完整流程

**函数**: `server/prometheus.js:116-143`

```javascript
mapTagsToLabels(tags) {
    let mappedTags = {};
    tags.forEach((tag) => {
        // 1. 清洗标签名
        let sanitizedTag = Prometheus.sanitizeForPrometheus(tag.name);
        if (sanitizedTag === "") {
            return; // 跳过空标签名
        }

        // 2. 初始化标签值数组
        if (mappedTags[sanitizedTag] === undefined) {
            mappedTags[sanitizedTag] = [];
        }

        // 3. 清洗并添加标签值
        let tagValue = Prometheus.sanitizeForPrometheus(tag.value || "");
        if (tagValue !== "") {
            mappedTags[sanitizedTag].push(tagValue);
        }

        // 4. 标签值排序
        mappedTags[sanitizedTag] = mappedTags[sanitizedTag].sort();
    });

    // 5. 标签名排序后返回
    return Object.keys(mappedTags)
        .sort(this.sortTags)
        .reduce((obj, key) => {
            obj[key] = mappedTags[key];
            return obj;
        }, {});
}
```

#### 完整映射示例

```
输入 tags 数组:
[
    { name: "环境", value: "生产" },           // ← 纯中文，清洗后空
    { name: "env-prod", value: "web-1" },       // ← 正常
    { name: "Env-test", value: "api-2" },       // ← 正常
    { name: "env-prod", value: "db-3" },        // ← 同名标签，值追加
    { name: "123test", value: "cache" },         // ← 数字开头
    { name: "region", value: "中国-北京" }       // ← 值含中文
]

处理过程:

1. { name: "环境", value: "生产" }
   sanitizeForPrometheus("环境") → ""
   → 跳过，不处理

2. { name: "env-prod", value: "web-1" }
   sanitizeForPrometheus("env-prod") → "envprod"
   sanitizeForPrometheus("web-1") → "web1"
   → mappedTags["envprod"] = ["web1"]

3. { name: "Env-test", value: "api-2" }
   sanitizeForPrometheus("Env-test") → "Envtest"
   sanitizeForPrometheus("api-2") → "api2"
   → mappedTags["Envtest"] = ["api2"]

4. { name: "env-prod", value: "db-3" }
   sanitizeForPrometheus("env-prod") → "envprod"
   sanitizeForPrometheus("db-3") → "db3"
   → mappedTags["envprod"].push("db3")
   → 排序: ["db3", "web1"]

5. { name: "123test", value: "cache" }
   sanitizeForPrometheus("123test") → 步骤2移除开头数字 → "test"
   sanitizeForPrometheus("cache") → "cache"
   → mappedTags["test"] = ["cache"]

6. { name: "region", value: "中国-北京" }
   sanitizeForPrometheus("region") → "region"
   sanitizeForPrometheus("中国-北京") → 中文被移除 → ""
   → 值为空，不添加
   → mappedTags["region"] = [] (空数组)

标签名排序 (不区分大小写):
["Envtest", "envprod", "region", "test"]

最终 mappedTags:
{
    "Envtest": ["api2"],
    "envprod": ["db3", "web1"],  // 已排序
    "region": [],                 // 空数组
    "test": ["cache"]
}
```

### 3.7 内置标签与自定义标签的合并

**位置**: `server/prometheus.js:19-29` (构造函数)

```javascript
constructor(monitor, tags) {
    this.monitorLabelValues = {
        ...this.mapTagsToLabels(tags),  // 自定义标签 (可能是数组)
        monitor_id: monitor.id,           // 内置标签: 数字
        monitor_name: monitor.name,       // 内置标签: 字符串 (未清洗！)
        monitor_type: monitor.type,       // 内置标签: 字符串
        monitor_url: monitor.url,         // 内置标签: 字符串 (可能含特殊字符)
        monitor_hostname: monitor.hostname, // 内置标签: 字符串
        monitor_port: monitor.port,       // 内置标签: 数字
    };
}
```

**⚠️ 重要发现**: 内置标签 (`monitor_name`, `monitor_url` 等) **没有经过 `sanitizeForPrometheus` 清洗**！

这意味着:
- 如果 `monitor.name` 包含空格、中文、特殊字符，会直接作为标签值
- 这依赖于 `prom-client` 库的内部处理

### 3.8 标签处理关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| 标签清洗函数 | `server/prometheus.js` | 105-109 |
| 自定义标签映射 | `server/prometheus.js` | 116-143 |
| 全局标签初始化+去重+排序 | `server/prometheus.js` | 38-50 |
| 标签名排序函数 | `server/prometheus.js` | 265-278 |
| 内置标签合并 | `server/prometheus.js` | 19-29 |

---

## 4. 非数字 ping 值的指标写入策略

### 4.1 代码逻辑

**位置**: `server/prometheus.js:226-232`

```javascript
try {
    if (typeof heartbeat.ping === "number") {
        monitorResponseTime.set(this.monitorLabelValues, heartbeat.ping);
    } else {
        // Is it good?
        monitorResponseTime.set(this.monitorLabelValues, -1);
    }
} catch (e) {
    log.error("prometheus", "Caught error");
    log.error("prometheus", e);
}
```

### 4.2 策略详解

| 条件 | 写入值 | 说明 |
|------|--------|------|
| `typeof heartbeat.ping === "number"` | `heartbeat.ping` | 正常的响应时间 (毫秒) |
| 其他情况 (非 number) | `-1` | 哨兵值，表示无有效响应时间 |

### 4.3 严格类型判断

使用 `typeof === "number"` 严格判断，而非 `!isNaN()` 或其他方式:

| heartbeat.ping 值 | typeof 结果 | 是否写入原值 | 写入值 |
|-------------------|-------------|-------------|--------|
| `45` | `"number"` | ✅ 是 | `45` |
| `0` | `"number"` | ✅ 是 | `0` |
| `null` | `"object"` | ❌ 否 | `-1` |
| `undefined` | `"undefined"` | ❌ 否 | `-1` |
| `""` | `"string"` | ❌ 否 | `-1` |
| `"45"` | `"string"` | ❌ 否 | `-1` |
| `NaN` | `"number"` | ⚠️ 是 | `NaN` (这是个问题？) |

**注意**: `typeof NaN === "number"` 返回 `true`！所以 `NaN` 会被直接写入。

### 4.4 什么情况下 ping 非数字？

根据 `monitor.js` 中的逻辑，以下场景 `ping` 可能非数字:

1. **DOWN 状态**:
   ```javascript
   // uptime-calculator.js:219-221
   if (flatStatus === DOWN && ping > 0) {
       log.debug("uptime_calc", "The ping is not effective when the status is DOWN");
   }
   ```
   DOWN 状态下的 ping 值可能不被更新或为 null

2. **某些监控类型**:
   - 部分监控类型 (如某些自定义插件) 可能不返回 ping 值
   - 错误处理路径中可能没有设置 ping

3. **MAINTENANCE 状态**:
   - 维护状态下可能没有实际的网络请求

### 4.5 代码注释中的疑问

代码中有一行注释:
```javascript
// Is it good?
```

这表明开发者对 `-1` 作为哨兵值的设计有疑问。可能的考虑:

| 方案 | 优点 | 缺点 |
|------|------|------|
| **当前方案: -1** | 简单直接，数字类型 | 与正常的 0ms 或低延迟值容易混淆 |
| **方案: null** | 语义清晰，表示无值 | prom-client Gauge 不支持 null |
| **方案: 不设置该标签组合** | Prometheus 中无该时间序列 | 可能导致图表中断，告警规则复杂 |
| **方案: 0** | 简单 | 语义模糊 (是真的 0ms 还是无数据？) |

### 4.6 Prometheus 中查询时如何处理 -1

如果需要排除无有效响应时间的数据，可以在 PromQL 中过滤:

```promql
# 只查询有有效响应时间的监控
monitor_response_time{monitor_id="1"} > 0

# 或者转换为 NaN (在 Grafana 中不会显示)
monitor_response_time{monitor_id="1"} > 0 or on() vector(NaN)

# 统计平均响应时间时排除 -1
avg_over_time(monitor_response_time{monitor_id="1"}[5m] > 0)
```

---

## 5. 数据更新流程

### 5.1 整体数据流

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
│  - 绑定 monitor 的标签信息 (执行 mapTagsToLabels)                │
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

### 5.2 Prometheus.update() 详细逻辑

`server/prometheus.js:153-238`

```javascript
update(heartbeat, tlsInfo, uptime) {
    // 1. TLS 证书相关 metrics (实时)
    if (typeof tlsInfo !== "undefined") {
        try {
            monitorCertIsValid.set(this.monitorLabelValues, tlsInfo.valid ? 1 : 0);
        } catch (e) {
            log.error("prometheus", "Caught error", e);
        }
        
        try {
            if (tlsInfo.certInfo != null) {
                monitorCertDaysRemaining.set(this.monitorLabelValues, tlsInfo.certInfo.daysRemaining);
            }
        } catch (e) {
            log.error("prometheus", "Caught error", e);
        }
    }
    
    // 2. 历史聚合 metrics (基于滑动窗口)
    if (uptime) {
        // 24 小时窗口
        monitorAverageResponseTimeSeconds.set(
            {...this.monitorLabelValues, window: "1d"}, 
            uptime.data24h.avgPing / 1000
        );
        monitorUptimeRatio.set({...this.monitorLabelValues, window: "1d"}, uptime.data24h.uptime);
        
        // 30 天窗口
        monitorAverageResponseTimeSeconds.set(
            {...this.monitorLabelValues, window: "30d"}, 
            uptime.data30d.avgPing / 1000
        );
        monitorUptimeRatio.set({...this.monitorLabelValues, window: "30d"}, uptime.data30d.uptime);
        
        // 365 天窗口
        monitorAverageResponseTimeSeconds.set(
            {...this.monitorLabelValues, window: "365d"}, 
            uptime.data1y.avgPing / 1000
        );
        monitorUptimeRatio.set({...this.monitorLabelValues, window: "365d"}, uptime.data1y.uptime);
    }
    
    // 3. 实时状态 metrics (基于最新 heartbeat)
    if (heartbeat) {
        try {
            monitorStatus.set(this.monitorLabelValues, heartbeat.status);
        } catch (e) {
            log.error("prometheus", "Caught error");
            log.error("prometheus", e);
        }

        try {
            if (typeof heartbeat.ping === "number") {
                monitorResponseTime.set(this.monitorLabelValues, heartbeat.ping);
            } else {
                // Is it good?
                monitorResponseTime.set(this.monitorLabelValues, -1);
            }
        } catch (e) {
            log.error("prometheus", "Caught error");
            log.error("prometheus", e);
        }
    }
}
```

---

## 6. 实时状态 vs 历史聚合的边界

### 6.1 核心分界点

| 维度 | 实时状态 | 历史聚合 |
|------|----------|----------|
| **数据来源** | 最新的单条 heartbeat 记录 | UptimeCalculator 滑动窗口聚合 |
| **时间粒度** | 瞬时值 (当前心跳) | 多时间窗口 (24h/30d/365d) |
| **更新时机** | 每次 heartbeat 后立即更新 | 每次 heartbeat 后累加聚合 |
| **Prometheus Metrics** | `monitor_status`, `monitor_response_time` | `monitor_uptime_ratio`, `monitor_response_time_seconds` |

### 6.2 实时状态数据流程

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

### 6.3 历史聚合数据流程

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

### 6.4 边界示意图

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

---

## 7. UptimeCalculator 深入分析

### 7.1 初始化时的数据加载

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

### 7.2 时间桶对齐策略

| 粒度 | 对齐方式 | Key 计算 | 示例 |
|------|----------|----------|------|
| 分钟级 | 开始 of minute | `date.startOf("minute").unix()` | 2026-05-04T10:30:00 |
| 小时级 | 开始 of hour | `date.startOf("hour").unix()` | 2026-05-04T10:00:00 |
| 天级 | 开始 of day (UTC) | `date.utc().startOf("day").unix()` | 2026-05-04T00:00:00 |

**注意**: 天级使用 UTC 时间，避免时区切换导致统计异常。

### 7.3 滑动平均计算逻辑

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

---

## 8. 关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Prometheus 初始化 | `server/server.js` | 237 |
| /metrics 路由注册 | `server/server.js` | 337 |
| Metrics 定义 | `server/prometheus.js` | 62-96 |
| Metrics 更新逻辑 | `server/prometheus.js` | 153-238 |
| **标签清洗函数** | `server/prometheus.js` | 105-109 |
| **自定义标签映射** | `server/prometheus.js` | 116-143 |
| **全局标签初始化+去重+排序** | `server/prometheus.js` | 38-50 |
| **标签名排序函数** | `server/prometheus.js` | 265-278 |
| **内置标签合并** | `server/prometheus.js` | 19-29 |
| **ping 非数字处理** | `server/prometheus.js` | 226-232 |
| Monitor 中创建 Prometheus 实例 | `server/model/monitor.js` | 416 |
| Monitor 中调用 prometheus.update | `server/model/monitor.js` | 1101-1105 |
| UptimeCalculator 初始化 | `server/uptime-calculator.js` | 123-203 |
| UptimeCalculator 更新聚合 | `server/uptime-calculator.js` | 212-374 |
| 24 小时统计获取 | `server/uptime-calculator.js` | 806-808 |
| 30 天统计获取 | `server/uptime-calculator.js` | 820-822 |
| 365 天统计获取 | `server/uptime-calculator.js` | 827-829 |

---

## 9. 总结

### 9.1 标签处理核心设计

1. **清洗严格**:
   - 标签名和值都必须符合 Prometheus 规范 (`[a-zA-Z_][a-zA-Z0-9_]*`)
   - 非 ASCII 字符 (如中文) 会被完全移除，可能导致标签名/值为空

2. **去重机制**:
   - 使用 `new Set()` 处理清洗后重名的标签
   - 典型场景: 不同原始标签名含非 ASCII 字符，清洗后变成相同名称

3. **双重排序**:
   - 标签名: 不区分大小写的字母顺序
   - 标签值: 默认字符串排序 (Unicode 码点)

4. **空值过滤**:
   - 清洗后为空的标签名直接跳过
   - 清洗后为空的标签值不加入数组

5. **⚠️ 潜在问题**:
   - 内置标签 (`monitor_name`, `monitor_url` 等) **没有经过清洗**
   - 中文标签名/值会被完全移除，可能导致标签丢失

### 9.2 ping 非数字处理策略

1. **哨兵值 `-1`**:
   - 非数字类型的 ping 统一写入 `-1`
   - 代码注释 `// Is it good?` 表明这是一个有争议的设计决策

2. **严格类型判断**:
   - 使用 `typeof === "number"` 而非 `!isNaN()`
   - 但 `NaN` 的 `typeof` 也是 `"number"`，会被直接写入

3. **PromQL 过滤建议**:
   - 使用 `monitor_response_time > 0` 排除无有效数据的点

### 9.3 实时状态 vs 历史聚合的边界

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

4. **边界清晰的好处**:
   - **告警准确性**: 实时状态直接触发告警，避免聚合延迟
   - **统计稳定性**: 历史聚合使用滑动窗口，抗单点波动
   - **资源隔离**: 实时路径和聚合路径独立，互不影响
   - **Prometheus 查询友好**: 不同用途的 metrics 有明确的命名和标签区分
