# Uptime Kuma 通知渠道系统说明

本文档详细说明 Uptime Kuma 中多通知渠道（Slack、Telegram、邮件、Webhook 等）的注册、配置、路由、模板选择以及重复通知抑制机制。

## 目录

1. [通知渠道的注册机制](#通知渠道的注册机制)
2. [通知渠道的配置与管理](#通知渠道的配置与管理)
3. [通知配置的生效条件](#通知配置的生效条件)
4. [监控状态变化时的通知路由](#监控状态变化时的通知路由)
5. [通知渠道最终命中逻辑](#通知渠道最终命中逻辑)
6. [模板选择与渲染机制](#模板选择与渲染机制)
7. [重复通知抑制机制](#重复通知抑制机制)

---

## 1. 通知渠道的注册机制

### 1.1 后端注册

Uptime Kuma 的通知渠道系统采用**工厂模式**设计，所有通知提供商都继承自 `NotificationProvider` 基类。

#### 基类定义

`server/notification-providers/notification-provider.js` 定义了所有通知提供商必须实现的接口：

```javascript
class NotificationProvider {
    // 通知提供商名称，必须唯一
    name = undefined;

    // 发送通知的核心方法，必须被子类重写
    async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
        throw new Error("Have to override Notification.send(...)");
    }

    // 工具方法：从 monitor 对象中提取地址
    extractAddress(monitorJSON) { ... }

    // 工具方法：渲染 Liquid 模板
    async renderTemplate(template, msg, monitorJSON, heartbeatJSON) { ... }

    // 工具方法：处理 Axios 错误
    throwGeneralAxiosError(error) { ... }

    // 工具方法：获取带代理配置的 Axios 配置
    getAxiosConfigWithProxy(axiosConfig = {}) { ... }
}
```

#### 通知提供商的注册流程

在 `server/notification.js` 中，所有通知提供商通过**硬编码导入和实例化**的方式进行注册：

```javascript
// 1. 导入所有通知提供商
const Slack = require("./notification-providers/slack");
const Telegram = require("./notification-providers/telegram");
const Webhook = require("./notification-providers/webhook");
const SMTP = require("./notification-providers/smtp");
// ... 其他 80+ 通知提供商

class Notification {
    providerList = {};

    // 2. 初始化所有通知提供商
    static init() {
        this.providerList = {};

        const list = [
            new Slack(),
            new Telegram(),
            new Webhook(),
            new SMTP(),
            // ... 其他 80+ 通知提供商实例
        ];

        // 3. 注册到 providerList，以 name 为键
        for (let item of list) {
            if (!item.name) {
                throw new Error("Notification provider without name");
            }

            if (this.providerList[item.name]) {
                throw new Error("Duplicate notification provider name");
            }
            this.providerList[item.name] = item;
        }
    }
}
```

### 1.2 前端注册

前端组件在 `src/components/notifications/index.js` 中进行注册：

```javascript
// 导入所有通知表单组件
import Slack from "./Slack.vue";
import Telegram from "./Telegram.vue";
import Webhook from "./Webhook.vue";
import STMP from "./SMTP.vue";
// ... 其他组件

// 注册到 NotificationFormList，键名与后端的 name 对应
const NotificationFormList = {
    slack: Slack,
    telegram: Telegram,
    webhook: Webhook,
    smtp: STMP,
    // ... 其他组件
};
```

### 1.3 通知提供商示例：Slack

`server/notification-providers/slack.js` 展示了一个典型的通知提供商实现：

```javascript
class Slack extends NotificationProvider {
    name = "slack"; // 唯一标识符

    async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
        const okMsg = "Sent Successfully.";

        try {
            let config = this.getAxiosConfigWithProxy({});
            
            // 支持自定义模板
            if (notification.slackUseTemplate) {
                const renderedText = await this.renderTemplate(
                    notification.slackTemplate,
                    msg,
                    monitorJSON,
                    heartbeatJSON
                );
                // 发送渲染后的消息
                await axios.post(notification.slackwebhookURL, {
                    text: renderedText,
                    channel: notification.slackchannel,
                    // ... 其他配置
                }, config);
                return okMsg;
            }

            // 构建 Slack Block Kit 消息
            const blocks = this.buildBlocks(baseURL, monitorJSON, heartbeatJSON, title, msg, includeGroupName);
            
            await axios.post(notification.slackwebhookURL, {
                text: msg,
                channel: notification.slackchannel,
                attachments: [{
                    color: heartbeatJSON["status"] === UP ? "#2eb886" : "#e01e5a",
                    blocks: blocks,
                }],
            }, config);
            
            return okMsg;
        } catch (error) {
            this.throwGeneralAxiosError(error);
        }
    }
}
```

---

## 2. 通知渠道的配置与管理

### 2.1 数据模型

通知配置存储在数据库的 `notification` 表中，通过 `monitor_notification` 关联表与监控项关联。

#### 通知表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER | 主键 |
| name | VARCHAR | 通知名称（用户自定义） |
| user_id | INTEGER | 所属用户 ID |
| config | TEXT | JSON 格式的配置数据 |
| is_default | BOOLEAN | 是否为默认通知 |

#### 关联表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| monitor_id | INTEGER | 监控项 ID |
| notification_id | INTEGER | 通知配置 ID |

### 2.2 配置保存与管理

在 `server/notification.js` 中实现了通知的保存、删除等管理功能：

```javascript
class Notification {
    // 保存通知配置
    static async save(notification, notificationID, userID) {
        let bean;

        if (notificationID) {
            // 更新现有通知
            bean = await R.findOne("notification", " id = ? AND user_id = ? ", [notificationID, userID]);
            if (!bean) {
                throw new Error("notification not found");
            }
        } else {
            // 创建新通知
            bean = R.dispense("notification");
        }

        // applyExisting 是一次性操作，不保存到数据库
        const applyExisting = notification.applyExisting || false;
        notification.applyExisting = false;

        bean.name = notification.name;
        bean.user_id = userID;
        bean.config = JSON.stringify(notification); // 完整配置保存为 JSON
        bean.is_default = notification.isDefault || false;
        await R.store(bean);

        // 如果勾选了"应用到所有现有监控"
        if (applyExisting) {
            await applyNotificationEveryMonitor(bean.id, userID);
        }

        return bean;
    }

    // 删除通知
    static async delete(notificationID, userID) {
        let bean = await R.findOne("notification", " id = ? AND user_id = ? ", [notificationID, userID]);
        if (!bean) {
            throw new Error("notification not found");
        }
        await R.trash(bean);
    }
}

// 将通知应用到所有现有监控
async function applyNotificationEveryMonitor(notificationID, userID) {
    let monitors = await R.getAll("SELECT id FROM monitor WHERE user_id = ?", [userID]);

    for (let i = 0; i < monitors.length; i++) {
        // 检查是否已关联
        let checkNotification = await R.findOne(
            "monitor_notification", 
            " monitor_id = ? AND notification_id = ? ", 
            [monitors[i].id, notificationID]
        );

        if (!checkNotification) {
            let relation = R.dispense("monitor_notification");
            relation.monitor_id = monitors[i].id;
            relation.notification_id = notificationID;
            await R.store(relation);
        }
    }
}
```

### 2.3 获取监控关联的通知列表

`server/model/monitor.js` 中提供了获取监控关联通知列表的方法：

```javascript
static async getNotificationList(monitor) {
    let notificationList = await R.getAll(
        "SELECT notification.* FROM notification, monitor_notification " +
        "WHERE monitor_id = ? AND monitor_notification.notification_id = notification.id ",
        [monitor.id]
    );
    return notificationList;
}
```

---

## 3. 通知配置的生效条件

### 3.1 通知配置的启停机制

Uptime Kuma 的通知配置**没有单独的启用/禁用开关**。通知的"启用"或"禁用"是通过**监控项与通知配置的关联关系**来控制的。

#### 核心设计

- 通知配置本身存储在 `notification` 表中
- 通知是否对某个监控生效，取决于 `monitor_notification` 关联表中是否存在对应的记录
- 用户在编辑监控时，可以通过复选框选择启用/禁用哪些通知（即创建/删除关联关系）

#### 前端实现

在 `src/pages/EditMonitor.vue` 中，通知的启用状态通过 `monitor.notificationIDList` 控制：

```javascript
// 数据结构
monitor: {
    notificationIDList: {},  // 键为通知 ID，值为 true/false
    // ... 其他监控配置
}

// 模板中的复选框
<input
    v-model="monitor.notificationIDList[notification.id]"
    type="checkbox"
/>
```

#### 后端实现

在 `server/server.js` 中，`updateMonitorNotification` 函数负责更新关联关系：

```javascript
async function updateMonitorNotification(monitorID, notificationIDList) {
    // 1. 删除所有现有关联
    await R.exec("DELETE FROM monitor_notification WHERE monitor_id = ? ", [monitorID]);

    // 2. 重新创建被勾选的通知的关联
    for (let notificationID in notificationIDList) {
        if (notificationIDList[notificationID]) {  // 值为 true 表示启用
            let relation = R.dispense("monitor_notification");
            relation.monitor_id = monitorID;
            relation.notification_id = notificationID;
            await R.store(relation);
        }
    }
}
```

### 3.2 默认通知（isDefault）机制

通知配置有一个 `is_default` 字段，用于标记该通知是否为"默认通知"。

#### 默认通知的作用

默认通知只影响**新建的监控**：

- 当用户创建新监控时，前端会自动将所有标记为 `isDefault === true` 的通知添加到 `monitor.notificationIDList` 中
- 这意味着新监控会自动关联所有"默认通知"

#### 前端实现

在 `src/pages/EditMonitor.vue` 的 `init()` 方法中：

```javascript
if (this.isAdd) {
    // 新建监控时，自动启用所有默认通知
    for (let i = 0; i < this.$root.notificationList.length; i++) {
        if (this.$root.notificationList[i].isDefault === true) {
            this.monitor.notificationIDList[this.$root.notificationList[i].id] = true;
        }
    }
}
```

### 3.3 默认通知扩散到存量监控的规则

`isDefault` 标记**不会自动扩散到已有的监控**。如果需要将通知应用到现有监控，需要显式操作。

#### 方式一：创建/编辑通知时勾选"应用到所有现有监控"

在 `src/components/NotificationDialog.vue` 中，用户可以勾选 `applyExisting` 选项：

```html
<div class="form-check form-switch">
    <input v-model="notification.applyExisting" class="form-check-input" type="checkbox" />
    <label class="form-check-label">{{ $t("Apply on all existing monitors") }}</label>
</div>
```

当用户勾选此选项并保存时，后端会执行 `applyNotificationEveryMonitor()` 函数：

```javascript
// server/notification.js
async function applyNotificationEveryMonitor(notificationID, userID) {
    // 1. 获取用户的所有监控
    let monitors = await R.getAll("SELECT id FROM monitor WHERE user_id = ?", [userID]);

    // 2. 遍历每个监控，检查是否已关联该通知
    for (let i = 0; i < monitors.length; i++) {
        let checkNotification = await R.findOne(
            "monitor_notification", 
            " monitor_id = ? AND notification_id = ? ", 
            [monitors[i].id, notificationID]
        );

        // 3. 如果未关联，则创建关联
        if (!checkNotification) {
            let relation = R.dispense("monitor_notification");
            relation.monitor_id = monitors[i].id;
            relation.notification_id = notificationID;
            await R.store(relation);
        }
    }
}
```

#### 方式二：在每个监控的编辑页面手动启用

用户可以在每个监控的编辑页面，通过复选框手动启用/禁用特定的通知配置。

### 3.4 生效条件总结

| 场景 | 新建监控 | 现有监控 |
|------|----------|----------|
| `isDefault = true` | 自动启用 | 无影响 |
| `isDefault = false` | 不自动启用 | 无影响 |
| 创建时勾选 `applyExisting` | 自动启用 | 为所有现有监控启用 |
| 编辑监控时勾选/取消勾选 | - | 为该监控启用/禁用 |

---

## 4. 监控状态变化时的通知路由

### 4.1 心跳检查流程

监控项的心跳检查在 `server/model/monitor.js` 的 `start()` 方法中实现，核心逻辑在内部的 `beat()` 函数中：

```javascript
async start(io) {
    let previousBeat = null;
    let retries = 0;

    const beat = async () => {
        // ... 执行检查，设置 bean.status 和 bean.msg

        // 判断这次心跳是否重要（状态变化等）
        let isImportant = Monitor.isImportantBeat(isFirstBeat, previousBeat?.status, bean.status);

        if (isImportant) {
            bean.important = true;

            // 判断是否需要发送通知（排除维护模式等特殊情况）
            if (Monitor.isImportantForNotification(isFirstBeat, previousBeat?.status, bean.status)) {
                // 发送通知
                await Monitor.sendNotification(isFirstBeat, this, bean);
            }
        } else {
            bean.important = false;

            // 处理持续 DOWN 状态的重复通知
            if (bean.status === DOWN && this.resendInterval > 0) {
                ++bean.downCount;
                if (bean.downCount >= this.resendInterval) {
                    // 达到重发间隔，再次发送通知
                    await Monitor.sendNotification(isFirstBeat, this, bean);
                    bean.downCount = 0; // 重置计数器
                }
            }
        }

        // ... 保存心跳记录，准备下一次检查
    };
}
```

### 4.2 重要心跳判断

`isImportantBeat()` 方法判断一次心跳是否"重要"：

```javascript
static isImportantBeat(isFirstBeat, previousBeatStatus, currentBeatStatus) {
    // 状态转换规则：
    // * ? -> ANY STATUS = important [isFirstBeat]
    // UP -> PENDING = not important
    // * UP -> DOWN = important
    // UP -> UP = not important
    // PENDING -> PENDING = not important
    // * PENDING -> DOWN = important
    // PENDING -> UP = not important
    // DOWN -> PENDING = 不可能发生
    // DOWN -> DOWN = not important
    // * DOWN -> UP = important
    // MAINTENANCE -> MAINTENANCE = not important
    // * MAINTENANCE -> UP = important
    // * MAINTENANCE -> DOWN = important
    // * DOWN -> MAINTENANCE = important
    // * UP -> MAINTENANCE = important

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

### 4.3 通知触发判断

`isImportantForNotification()` 方法进一步过滤需要发送通知的情况：

```javascript
static isImportantForNotification(isFirstBeat, previousBeatStatus, currentBeatStatus) {
    // 与 isImportantBeat 的区别：
    // - 维护模式的进入和退出不发送通知
    // - 维护模式 -> UP 不发送通知
    // - UP -> 维护模式 不发送通知
    // - DOWN -> 维护模式 不发送通知

    return (
        isFirstBeat ||
        (previousBeatStatus === MAINTENANCE && currentBeatStatus === DOWN) ||  // 维护中检测到故障
        (previousBeatStatus === UP && currentBeatStatus === DOWN) ||             // 正常变故障
        (previousBeatStatus === DOWN && currentBeatStatus === UP) ||             // 故障恢复
        (previousBeatStatus === PENDING && currentBeatStatus === DOWN)            // 重试后仍故障
    );
}
```

### 4.4 发送通知

`sendNotification()` 方法实际执行通知发送：

```javascript
static async sendNotification(isFirstBeat, monitor, bean) {
    // 首次心跳且状态不是 DOWN 时不发送通知
    if (!isFirstBeat || bean.status === DOWN) {
        // 获取与该监控关联的所有通知配置
        const notificationList = await Monitor.getNotificationList(monitor);

        // 构建基础消息
        let text;
        if (bean.status === UP) {
            text = "✅ Up";
        } else {
            text = "🔴 Down";
        }
        let msg = `[${monitor.name}] [${text}] ${bean.msg}`;

        // 准备详细数据
        const heartbeatJSON = await bean.toJSONAsync({ decodeResponse: true });
        // 添加时区信息
        heartbeatJSON["timezone"] = await UptimeKumaServer.getInstance().getTimezone();
        heartbeatJSON["timezoneOffset"] = UptimeKumaServer.getInstance().getTimezoneOffset();
        heartbeatJSON["localDateTime"] = dayjs
            .utc(heartbeatJSON["time"])
            .tz(heartbeatJSON["timezone"])
            .format(SQL_DATETIME_FORMAT);

        // 如果是从 DOWN 恢复到 UP，计算并添加停机时间信息
        if (bean.status === UP && monitor.id) {
            try {
                const lastDownHeartbeat = await R.getRow(
                    "SELECT time FROM heartbeat WHERE monitor_id = ? AND status = ? AND important = 1 " +
                    "ORDER BY time DESC LIMIT 1",
                    [monitor.id, DOWN]
                );
                if (lastDownHeartbeat && lastDownHeartbeat.time) {
                    heartbeatJSON["lastDownTime"] = lastDownHeartbeat.time;
                }
            } catch (error) {
                // 如果计算停机时间失败，继续执行而不中断通知发送
            }
        }

        // 遍历所有关联的通知配置，逐一发送
        for (let notification of notificationList) {
            try {
                // 调用 Notification.send()，根据 type 分发到对应的 provider
                await Notification.send(
                    JSON.parse(notification.config),  // 通知配置
                    msg,                                // 基础消息
                    monitor.toJSON(preloadData, false), // 监控信息
                    heartbeatJSON                       // 心跳详情
                );
            } catch (e) {
                // 单个通知发送失败不影响其他通知
                log.error("monitor", "Cannot send notification to " + notification.name);
                log.error("monitor", e);
            }
        }
    }
}
```

### 4.5 通知分发

`server/notification.js` 中的 `send()` 方法根据通知类型分发到对应的 provider：

```javascript
static async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    if (this.providerList[notification.type]) {
        // 调用对应 provider 的 send 方法
        return this.providerList[notification.type].send(notification, msg, monitorJSON, heartbeatJSON);
    } else {
        throw new Error("Notification type is not supported");
    }
}
```

---

## 5. 通知渠道最终命中逻辑

### 5.1 完整的命中流程

当监控状态变化需要发送通知时，系统按照以下流程决定最终命中哪些通知渠道：

```
┌─────────────────────┐
│  监控状态变化        │
│  (需要发送通知)      │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  1. 获取监控关联的通知配置            │
│     Monitor.getNotificationList()    │
│     (查询 monitor_notification 表)   │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  2. 遍历每个通知配置                  │
│     for (notification of list) {     │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  3. 解析配置 JSON                     │
│     JSON.parse(notification.config)  │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  4. 检查 provider 是否存在            │
│     providerList[notification.type]  │
└──────────┬───────────────────────────┘
           │
      ┌────┴────┐
      │         │
     存在      不存在
      │         │
      ▼         ▼
┌─────────┐  ┌─────────────────┐
│5. 调用  │  │  抛出错误       │
│provider │  │  "Notification  │
│.send()  │  │   type not      │
└────┬────┘  │   supported"    │
     │       └─────────────────┘
     ▼
┌─────────────────┐
│  6. 发送通知成功 │
│  或记录失败日志  │
└─────────────────┘
```

### 5.2 关键步骤详解

#### 步骤 1：获取关联的通知配置

```javascript
// server/model/monitor.js
static async getNotificationList(monitor) {
    let notificationList = await R.getAll(
        "SELECT notification.* FROM notification, monitor_notification " +
        "WHERE monitor_id = ? AND monitor_notification.notification_id = notification.id ",
        [monitor.id]
    );
    return notificationList;
}
```

**关键点**：
- 只查询 `monitor_notification` 关联表中存在的记录
- **不考虑** `is_default` 字段（`is_default` 只在新建监控时自动建立关联）
- **不考虑**通知配置本身是否有"启用"状态（Uptime Kuma 没有这个概念）

#### 步骤 2-4：通知类型检查

```javascript
// server/notification.js
static async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    if (this.providerList[notification.type]) {
        return this.providerList[notification.type].send(notification, msg, monitorJSON, heartbeatJSON);
    } else {
        throw new Error("Notification type is not supported");
    }
}
```

**关键点**：
- `notification.type` 必须与 `providerList` 中的某个键匹配
- 这个 `type` 值存储在 `notification.config` JSON 中，由前端表单设置

#### 步骤 5-6：发送通知

每个通知提供商的 `send()` 方法实现不同，但都遵循相同的模式：

```javascript
// 以 Webhook 为例
async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    const okMsg = "Sent Successfully.";

    try {
        // 1. 从 notification 配置中提取参数
        const httpMethod = notification.httpMethod?.toLowerCase() || "post";
        const webhookURL = notification.webhookURL;
        // ... 其他配置

        // 2. 准备请求数据
        let data = {
            heartbeat: heartbeatJSON,
            monitor: monitorJSON,
            msg,
        };

        // 3. 发送请求
        if (httpMethod === "get") {
            await axios.get(webhookURL, config);
        } else {
            await axios.post(webhookURL, data, config);
        }

        return okMsg;
    } catch (error) {
        this.throwGeneralAxiosError(error);
    }
}
```

### 5.3 影响最终命中的因素

#### 因素 1：monitor_notification 关联关系

这是**最重要**的因素。只有在 `monitor_notification` 表中存在关联记录的通知配置才会被考虑。

建立关联的方式：
1. **新建监控时**：所有 `isDefault = true` 的通知会自动关联
2. **创建/编辑通知时**：勾选 `applyExisting` 会为所有现有监控建立关联
3. **编辑监控时**：用户手动勾选复选框建立/解除关联

#### 因素 2：notification.type 的有效性

`notification.type` 必须与后端注册的某个 provider 匹配。常见的 type 值：

| type 值 | 通知渠道 |
|---------|----------|
| `slack` | Slack |
| `telegram` | Telegram |
| `webhook` | Webhook |
| `smtp` | SMTP 邮件 |
| `discord` | Discord |
| `teams` | Microsoft Teams |
| ... | ... |

#### 因素 3：通知配置的完整性

某些通知渠道需要特定的配置参数才能正常工作：

- **Slack**：需要 `slackwebhookURL`
- **Telegram**：需要 `bottoken` 和 `chatid`
- **SMTP**：需要 `smtpHost`、`smtpPort`、`smtpSecure`、`smtpFrom`、`smtpTo` 等
- **Webhook**：需要 `webhookURL`

如果配置不完整，通知发送会失败，但不会影响其他通知的发送。

### 5.4 失败处理

单个通知发送失败不会阻止其他通知的发送：

```javascript
// server/model/monitor.js
for (let notification of notificationList) {
    try {
        await Notification.send(
            JSON.parse(notification.config),
            msg,
            monitor.toJSON(preloadData, false),
            heartbeatJSON
        );
    } catch (e) {
        // 记录错误，但继续处理下一个通知
        log.error("monitor", "Cannot send notification to " + notification.name);
        log.error("monitor", e);
    }
}
```

---

## 6. 模板选择与渲染机制

### 6.1 模板渲染引擎

Uptime Kuma 使用 **LiquidJS** 作为模板渲染引擎，在 `NotificationProvider.renderTemplate()` 方法中实现：

```javascript
async renderTemplate(template, msg, monitorJSON, heartbeatJSON) {
    const engine = new Liquid({
        root: "./no-such-directory-uptime-kuma",  // 禁用文件包含
        relativeReference: false,
        dynamicPartials: false,
    });
    const parsedTpl = engine.parse(template);

    // 准备上下文变量
    let monitorName = "Monitor Name not available";
    let monitorHostnameOrURL = "testing.hostname";

    if (monitorJSON !== null) {
        monitorName = monitorJSON["name"];
        monitorHostnameOrURL = this.extractAddress(monitorJSON);
    }

    let serviceStatus = "⚠️ Test";
    if (heartbeatJSON !== null) {
        serviceStatus = heartbeatJSON["status"] === DOWN ? "🔴 Down" : "✅ Up";
    }

    const context = {
        // v1 兼容变量（将在 v3 移除）
        STATUS: serviceStatus,
        NAME: monitorName,
        HOSTNAME_OR_URL: monitorHostnameOrURL,

        // 官方支持的变量
        status: serviceStatus,
        name: monitorName,
        hostnameOrURL: monitorHostnameOrURL,
        monitorJSON,      // 完整的监控对象
        heartbeatJSON,    // 完整的心跳对象
        msg,              // 基础消息文本
    };

    return engine.render(parsedTpl, context);
}
```

### 6.2 模板变量说明

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `status` | string | 服务状态（如 "🔴 Down" 或 "✅ Up"） |
| `name` | string | 监控项名称 |
| `hostnameOrURL` | string | 监控的主机名或 URL |
| `monitorJSON` | object | 完整的监控配置对象 |
| `heartbeatJSON` | object | 完整的心跳数据对象 |
| `msg` | string | 基础消息文本 |
| `STATUS` | string | 与 `status` 相同（v1 兼容） |
| `NAME` | string | 与 `name` 相同（v1 兼容） |
| `HOSTNAME_OR_URL` | string | 与 `hostnameOrURL` 相同（v1 兼容） |

### 6.3 Webhook 自定义模板示例

`server/notification-providers/webhook.js` 展示了如何使用自定义模板：

```javascript
async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    // ...

    if (notification.webhookContentType === "custom") {
        // 使用用户自定义的模板渲染请求体
        data = await this.renderTemplate(
            notification.webhookCustomBody,  // 用户定义的模板
            msg,
            monitorJSON,
            heartbeatJSON
        );
    }

    // ...
}
```

### 6.4 Slack 自定义模板示例

`server/notification-providers/slack.js` 中的模板使用：

```javascript
async send(notification, msg, monitorJSON = null, heartbeatJSON = null) {
    // 检查是否启用自定义模板
    if (notification.slackUseTemplate) {
        const renderedText = await this.renderTemplate(
            notification.slackTemplate,  // 用户定义的模板
            msg,
            monitorJSON,
            heartbeatJSON
        );

        let data = {
            text: renderedText,  // 使用渲染后的文本
            channel: notification.slackchannel,
            username: notification.slackusername,
            icon_emoji: notification.slackiconemo,
        };

        await axios.post(notification.slackwebhookURL, data, config);
        return okMsg;
    }
}
```

---

## 7. 重复通知抑制机制

### 7.1 状态变化驱动的通知

Uptime Kuma 的通知机制核心是**基于状态变化**，只有在以下情况才会发送通知：

1. **首次心跳且状态为 DOWN**
2. **UP → DOWN**：正常状态变为故障
3. **DOWN → UP**：故障恢复
4. **PENDING → DOWN**：重试后确认故障
5. **MAINTENANCE → DOWN**：维护模式中检测到故障

### 7.2 持续故障的重发机制

对于持续 DOWN 的状态，Uptime Kuma 提供了**定时重发**机制：

```javascript
// 在 beat() 函数中
if (!isImportant) {
    bean.important = false;

    // 只有当状态是 DOWN 且配置了重发间隔时才处理
    if (bean.status === DOWN && this.resendInterval > 0) {
        ++bean.downCount;  // 增加计数
        
        // 当计数达到重发间隔时，再次发送通知
        if (bean.downCount >= this.resendInterval) {
            log.debug(
                "monitor",
                `[${this.name}] sendNotification again: Down Count: ${bean.downCount} | Resend Interval: ${this.resendInterval}`
            );
            await Monitor.sendNotification(isFirstBeat, this, bean);
            
            // 重置计数器，为下一次重发做准备
            bean.downCount = 0;
        }
    }
}
```

#### 重发机制说明

| 配置项 | 说明 |
|--------|------|
| `resendInterval` | 重发间隔（心跳次数），0 表示不重发 |
| `downCount` | 当前连续 DOWN 的心跳计数，状态变化时重置为 0 |
| `bean.downCount` | 存储在上一次心跳记录中，用于恢复计数 |

#### 工作流程示例

假设 `resendInterval = 5`（每 5 次心跳重发一次）：

1. **心跳 1**：UP → DOWN（重要心跳，发送通知，`downCount = 0`）
2. **心跳 2**：DOWN → DOWN（非重要，`downCount = 1`）
3. **心跳 3**：DOWN → DOWN（非重要，`downCount = 2`）
4. **心跳 4**：DOWN → DOWN（非重要，`downCount = 3`）
5. **心跳 5**：DOWN → DOWN（非重要，`downCount = 4`）
6. **心跳 6**：DOWN → DOWN（非重要，`downCount = 5`，达到阈值，重发通知，`downCount = 0`）
7. **心跳 7**：DOWN → DOWN（非重要，`downCount = 1`）
8. ...

### 7.3 证书过期通知的抑制

证书过期通知使用 `notification_sent_history` 表来记录已发送的通知，避免重复发送：

```javascript
async sendCertNotificationByTargetDays(certCN, certType, daysRemaining, targetDays, notificationList) {
    // 检查是否已发送过该阈值的通知
    let row = await R.getRow(
        "SELECT * FROM notification_sent_history WHERE type = ? AND monitor_id = ? AND days <= ?",
        ["certificate", this.id, targetDays]
    );

    // 已发送过，不再发送
    if (row) {
        log.debug("monitor", "Sent already, no need to send again");
        return;
    }

    // 发送通知
    let sent = false;
    for (let notification of notificationList) {
        try {
            await Notification.send(
                JSON.parse(notification.config),
                `[${this.name}][${this.url}] ${certType} certificate ${certCN} will expire in ${daysRemaining} days`
            );
            sent = true;
        } catch (e) {
            log.error("monitor", "Cannot send cert notification to " + notification.name);
        }
    }

    // 记录发送历史
    if (sent) {
        await R.exec(
            "INSERT INTO notification_sent_history (type, monitor_id, days) VALUES(?, ?, ?)",
            ["certificate", this.id, targetDays]
        );
    }
}
```

#### 证书通知表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| type | VARCHAR | 通知类型（如 "certificate"） |
| monitor_id | INTEGER | 关联的监控 ID |
| days | INTEGER | 触发通知的剩余天数阈值 |

### 7.4 维护模式的通知抑制

维护模式下的状态变化会被特殊处理：

```javascript
// isImportantForNotification 中：
// - MAINTENANCE → UP 不发送通知（维护结束恢复正常）
// - UP → MAINTENANCE 不发送通知（进入维护）
// - DOWN → MAINTENANCE 不发送通知（故障期间进入维护）
// - 只有 MAINTENANCE → DOWN 会发送通知（维护中检测到新故障）
```

### 7.5 PENDING 状态的处理

PENDING（重试中）状态有特殊的通知规则：

```javascript
// isImportantBeat 中：
// - UP → PENDING 不重要（不发送通知）
// - PENDING → PENDING 不重要（继续重试）
// - PENDING → DOWN 重要（重试失败，确认为故障，发送通知）
// - PENDING → UP 不重要（重试成功，不发送通知，因为没有真正"故障"过）
```

---

## 附录

### A. 状态码定义

| 常量 | 值 | 说明 |
|------|---|------|
| `DOWN` | 0 | 故障状态 |
| `UP` | 1 | 正常状态 |
| `PENDING` | 2 | 重试中 |
| `MAINTENANCE` | 3 | 维护模式 |

### B. 关键文件位置

| 文件 | 说明 |
|------|------|
| `server/notification.js` | 通知系统主入口，provider 注册与管理 |
| `server/notification-providers/notification-provider.js` | 通知提供商基类 |
| `server/notification-providers/*.js` | 各通知提供商的具体实现 |
| `server/model/monitor.js` | 监控模型，包含心跳检查和通知触发逻辑 |
| `server/server.js` | 服务器主入口，包含监控通知关联更新逻辑 |
| `src/components/notifications/index.js` | 前端通知组件注册 |
| `src/components/notifications/*.vue` | 各通知提供商的前端配置表单 |
| `src/components/NotificationDialog.vue` | 通知配置对话框 |
| `src/pages/EditMonitor.vue` | 监控编辑页面 |

### C. 通知发送流程图

```
┌─────────────────┐
│   心跳检查完成   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│ isImportantBeat() 判断      │
│ （状态是否发生重要变化？）    │
└────────┬────────────────────┘
         │
    ┌────┴────┐
    │         │
   是         否
    │         │
    ▼         ▼
┌─────────┐  ┌──────────────────────────────┐
│标记重要 │  │ 状态是否为 DOWN？             │
└────┬────┘  │ resendInterval > 0？         │
     │       └─────────────┬────────────────┘
     │                     │
     ▼                ┌────┴────┐
┌──────────────────┐  │         │
│ isImportantFor   │ 是         否
│ Notification()   │  │         │
│ 判断             │  ▼         │
└────────┬─────────┘┌────────────────┐│
         │          │downCount++     ││
    ┌────┴────┐     │>= resendInterval?││
    │         │     └───────┬────────┘││
   是         否             │         ││
    │         │         ┌────┴────┐   ││
    ▼         │         │         │   ││
┌──────────┐  │        是         否  ││
│发送通知  │  │         │         │   ││
└──────────┘  │         ▼         │   ││
              │    ┌──────────┐   │   ││
              │    │发送通知  │   │   ││
              │    │downCount=0│  │   ││
              │    └──────────┘   │   ││
              │                   │   ││
              └───────────────────┴───┘│
                                        │
                                        ▼
                              ┌─────────────────┐
                              │   等待下次心跳   │
                              └─────────────────┘
```

### D. 通知渠道命中决策流程图

```
┌─────────────────────────────────────────────────────┐
│                   监控状态变化                        │
│         (满足 isImportantForNotification 条件)       │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
              ┌───────────────────────┐
              │ 查询 monitor_notification│
              │ 获取关联的通知配置列表   │
              └───────────┬───────────┘
                          │
            ┌─────────────┴─────────────┐
            │                             │
            ▼                             ▼
    ┌───────────────┐            ┌─────────────────┐
    │ 列表为空       │            │ 列表有通知配置   │
    │ (不发送任何通知)│            └────────┬────────┘
    └───────────────┘                     │
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                              ▼                       ▼
                    ┌───────────────┐       ┌───────────────────┐
                    │ 遍历每个通知   │       │ 解析 notification.  │
                    │ 配置          │──────▶│ config 获取 type    │
                    └───────────────┘       └─────────┬─────────┘
                                                        │
                                          ┌─────────────┴─────────────┐
                                          │                             │
                                          ▼                             ▼
                                ┌───────────────────┐         ┌─────────────────────┐
                                │ type 在 providerList│         │ type 不在 providerList │
                                │ 中存在             │         │ 中存在               │
                                └─────────┬─────────┘         └──────────┬──────────┘
                                          │                                │
                                          ▼                                ▼
                                ┌───────────────────┐         ┌─────────────────────┐
                                │ 调用 provider.send()│         │ 抛出错误并记录日志   │
                                │ 发送通知           │         │ (不影响其他通知)    │
                                └───────────────────┘         └─────────────────────┘
```
