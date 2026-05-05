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

### 2.5 令牌数据绑定链路（数据库 → 前端页面）

Push Token 从数据库记录到前端编辑页和详情页的完整数据流转链路涉及多个层次：

#### 2.5.1 数据层：数据库字段映射

数据库中存储的字段名为 `push_token`（snake_case），在后端模型中转换为 `pushToken`（camelCase）：

**文件位置：** `server/model/monitor.js:233`

```javascript
// 在 toJSON() 方法中，pushToken 被包含在敏感数据中返回
if (includeSensitiveData) {
    data = {
        ...data,
        // ... 其他字段
        pushToken: this.pushToken,  // 数据库字段是 push_token，通过 ORM 映射
        // ... 其他字段
    };
}
```

**关键机制：**
- 使用 RedBean ORM，数据库字段 `push_token` 自动映射到 bean 属性 `pushToken`
- `toJSON()` 方法默认包含敏感数据（`includeSensitiveData = true`），所以 pushToken 会被返回

#### 2.5.2 后端：监控列表获取

后端通过 `sendMonitorList` 和 `sendUpdateMonitorIntoList` 方法将监控数据发送到前端：

**文件位置：** `server/uptime-kuma-server.js:219-234, 256-277`

```javascript
// 发送完整监控列表
async sendMonitorList(socket) {
    let list = await this.getMonitorJSONList(socket.userID);
    this.io.to(socket.userID).emit("monitorList", list);  // 发送 monitorList 事件
    return list;
}

// 发送更新的单个监控到列表
async sendUpdateMonitorIntoList(socket, monitorID) {
    let list = await this.getMonitorJSONList(socket.userID, monitorID);
    if (list && list[monitorID]) {
        this.io.to(socket.userID).emit("updateMonitorIntoList", list);
    }
}

// 构建监控数据列表
async getMonitorJSONList(userID, monitorID = null) {
    // ... 查询数据库 ...
    let monitorList = await R.find("monitor", query + "ORDER BY weight DESC, name", queryParams);
    
    // ... 预处理数据 ...
    const preloadData = await Monitor.preparePreloadData(monitorData);
    
    // 调用 toJSON() 转换，包含 pushToken
    const result = {};
    monitorList.forEach((monitor) => (result[monitor.id] = monitor.toJSON(preloadData)));
    return result;
}
```

#### 2.5.3 后端：单个监控获取（编辑页使用）

当用户进入编辑页面时，前端通过 `getMonitor` 事件获取单个监控的详细数据：

**文件位置：** `server/server.js:987-1000`

```javascript
socket.on("getMonitor", async (monitorID, callback) => {
    try {
        checkLogin(socket);

        log.info("monitor", `Get Monitor: ${monitorID} User ID: ${socket.userID}`);

        // 从数据库查询监控
        let monitor = await R.findOne("monitor", " id = ? AND user_id = ? ", [monitorID, socket.userID]);
        const monitorData = [{ id: monitor.id, active: monitor.active }];
        const preloadData = await Monitor.preparePreloadData(monitorData);
        
        // 调用 toJSON()，包含 pushToken（因为 includeSensitiveData 默认为 true）
        callback({
            ok: true,
            monitor: monitor.toJSON(preloadData),
        });
        // ...
    } catch (e) {
        // ... 错误处理
    }
});
```

#### 2.5.4 后端：保存监控时的令牌处理

当用户保存编辑后的监控时，`editMonitor` 事件处理函数将 `pushToken` 保存回数据库：

**文件位置：** `server/server.js:800-881, 940-954`

```javascript
socket.on("editMonitor", async (monitor, callback) => {
    try {
        // ... 权限检查 ...
        
        let bean = await R.findOne("monitor", " id = ? ", [monitor.id]);
        
        // ... 其他字段赋值 ...
        
        // pushToken 从前端 monitor 对象赋值到数据库 bean
        bean.pushToken = monitor.pushToken;  // 关键绑定！
        
        // ... 其他字段赋值 ...
        
        bean.validate();
        await R.store(bean);  // 保存到数据库
        
        // ... 重启监控（如需要）...
        
        // 发送更新到前端监控列表
        await server.sendUpdateMonitorIntoList(socket, bean.id);
        
        callback({
            ok: true,
            msg: "Saved.",
            msgi18n: true,
            monitorID: bean.id,
        });
    } catch (e) {
        // ... 错误处理
    }
});
```

#### 2.5.5 前端：Socket 接收监控列表

前端 `socket.js` mixin 监听 `monitorList` 和 `updateMonitorIntoList` 事件：

**文件位置：** `src/mixins/socket.js:147-163`

```javascript
// 接收完整监控列表
socket.on("monitorList", (data) => {
    this.assignMonitorUrlParser(data);
    this.monitorList = data;  // 存储到 $root.monitorList
});

// 接收单个监控更新
socket.on("updateMonitorIntoList", (data) => {
    this.assignMonitorUrlParser(data);
    Object.entries(data).forEach(([monitorID, updatedMonitor]) => {
        this.monitorList[monitorID] = updatedMonitor;  // 增量更新
    });
});
```

`monitorList` 是 Vue 根实例的响应式数据，包含所有监控的完整信息，包括 `pushToken`。

#### 2.5.6 前端：编辑页面数据绑定流程

**文件位置：** `src/pages/EditMonitor.vue:3764-3854`

编辑页面在 `init()` 方法中根据路由判断是添加、编辑还是克隆模式：

```javascript
methods: {
    init() {
        if (this.isAdd) {
            // === 添加模式 ===
            this.monitor = {
                ...monitorDefaults,
                // ... 默认值
            };
            // 此时 monitor.pushToken 为 undefined
            // 在提交前会检查类型并生成令牌（见 2.1 节）
        } else if (this.isEdit || this.isClone) {
            // === 编辑或克隆模式 ===
            // 通过 Socket 从后端获取单个监控数据
            this.$root.getSocket().emit("getMonitor", this.$route.params.id, (res) => {
                if (res.ok) {
                    if (this.isClone) {
                        // 克隆时：清除 pushToken，后续会重新生成
                        if (res.monitor.type === "push") {
                            res.monitor.pushToken = undefined;
                        }
                        // 克隆时还会清除其他不可继承的属性
                        this.monitor.id = undefined;
                        // ...
                    }

                    // === 关键绑定：将后端数据赋值给本地 this.monitor ===
                    this.monitor = res.monitor;  // 包含 pushToken（编辑模式）
                    
                    // ... 其他数据处理
                } else {
                    this.$root.toastError(res.msg);
                }
            });
        }
        // ...
    },
}
```

#### 2.5.7 前端：编辑页面 Push URL 显示

编辑页面通过计算属性 `pushURL` 动态构建完整的 Push 端点 URL：

**文件位置：** `src/pages/EditMonitor.vue:3271-3272`

```javascript
computed: {
    // ... 其他计算属性
    
    pushURL() {
        // 使用 monitor.pushToken 构建 URL
        return this.$root.baseURL + "/api/push/" + this.monitor.pushToken + "?status=up&msg=OK&ping=";
    },
    
    // ...
}
```

**模板中的使用：** `src/pages/EditMonitor.vue:256-267`

```vue
<div v-if="monitor.type === 'push'" class="my-3">
    <label for="push-url" class="form-label">{{ $t("PushUrl") }}</label>
    
    <!-- 绑定到计算属性 pushURL -->
    <CopyableInput id="push-url" v-model="pushURL" type="url" disabled="disabled" />
    
    <div class="form-text">
        {{ $t("needPushEvery", [monitor.interval]) }}
        <br />
        {{ $t("pushOptionalParams", ["status, msg, ping"]) }}
    </div>
    
    <!-- 重置令牌按钮 -->
    <button class="btn btn-primary" type="button" @click="resetToken">
        {{ $t("Reset Token") }}
    </button>
</div>
```

#### 2.5.8 前端：编辑页面令牌重置

当用户点击"Reset Token"按钮时：

**文件位置：** `src/pages/EditMonitor.vue:4006-4008`

```javascript
resetToken() {
    // 生成新的 32 位随机令牌
    this.monitor.pushToken = genSecret(pushTokenLength);
}
```

这会：
1. 更新 `this.monitor.pushToken` 的值
2. 触发 `pushURL` 计算属性重新计算（响应式更新）
3. UI 中的 Push URL 立即显示新令牌
4. 用户需要点击"Save"按钮才会保存到数据库

#### 2.5.9 前端：详情页面数据绑定

详情页面从 `$root.monitorList` 获取监控数据：

**详情页 Push URL 显示模板：** `src/pages/Details.vue:97-100`

```vue
<span v-if="monitor.type === 'push'">
    Push:
    <a :href="pushURL" target="_blank" rel="noopener noreferrer">{{ pushURL }}</a>
</span>
```

**详情页计算属性：** `src/pages/Details.vue:583-585`

```javascript
computed: {
    // ...
    
    pushURL() {
        // 使用 this.monitor.pushToken
        return this.$root.baseURL + "/api/push/" + this.monitor.pushToken + "?status=up&msg=OK&ping=";
    },
    
    // ...
}
```

**详情页数据来源：**
详情页的 `monitor` 数据通常来自 `$root.monitorList`（通过路由参数 ID 索引），或者在某些情况下从后端获取。当监控列表通过 Socket 更新时，详情页的数据会自动响应式更新。

#### 2.5.10 令牌数据绑定链路总览图

```
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│                         Push Token 完整数据绑定链路                                                │
├────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                    数据层 (Database)                                        │  │
│  │                                                                                              │  │
│  │   ┌────────────────────────────────────────────────────────────────────────────────────┐   │  │
│  │   │                           monitor 表                                                  │   │  │
│  │   │   ┌─────────────┬─────────────┬─────────────┬───────────────────┐                │   │  │
│  │   │   │     id      │    name     │    type     │    push_token     │                │   │  │
│  │   │   ├─────────────┼─────────────┼─────────────┼───────────────────┤                │   │  │
│  │   │   │      1      │ "My Push"   │   "push"    │ "abc123...xyz"   │                │   │  │
│  │   │   └─────────────┴─────────────┴─────────────┴───────────────────┘                │   │  │
│  │   │                                    │                                                  │   │  │
│  │   │                                    ▼ RedBean ORM 自动映射                            │   │  │
│  │   │                           bean.pushToken                                              │   │  │
│  │   └────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                              │                                                     │
│                                              ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                  后端层 (Backend)                                          │  │
│  │                                                                                              │  │
│  │   ┌────────────────────────────────────────────────────────────────────────────────────┐   │  │
│  │   │                         monitor.toJSON()                                            │   │  │
│  │   │                                                                                        │   │  │
│  │   │   toJSON(preloadData, includeSensitiveData = true) {                                │   │  │
│  │   │       // includeSensitiveData 默认为 true                                            │   │  │
│  │   │       return {                                                                        │   │  │
│  │   │           id: this.id,                                                                │   │  │
│  │   │           name: this.name,                                                            │   │  │
│  │   │           type: this.type,                                                            │   │  │
│  │   │           // ... 其他字段 ...                                                         │   │  │
│  │   │           pushToken: this.pushToken,    // ← 包含在返回数据中                        │   │  │
│  │   │           // ...                                                                      │   │  │
│  │   │       };                                                                               │   │  │
│  │   │   }                                                                                    │   │  │
│  │   └────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                              │                                                 │  │
│  │              ┌───────────────────────────────┴───────────────────────────────┐              │  │
│  │              │                                                               │              │  │
│  │              ▼                                                               ▼              │  │
│  │   ┌──────────────────────┐                                     ┌──────────────────────┐   │  │
│  │   │  getMonitorJSONList  │                                     │    getMonitor (单条)  │   │  │
│  │   │  (监控列表)           │                                     │    (编辑/详情页)      │   │  │
│  │   └──────────┬───────────┘                                     └──────────┬───────────┘   │  │
│  │              │                                                               │              │  │
│  │              ▼ Socket.io 事件                                                ▼ Socket.io 事件  │  │
│  │   ┌──────────────────────────────┐              ┌──────────────────────────────────────┐   │  │
│  │   │ io.to(userID).emit(         │              │ callback({                           │   │  │
│  │   │   "monitorList", list       │              │   ok: true,                          │   │  │
│  │   │ )                           │              │   monitor: monitor.toJSON()          │   │  │
│  │   │                             │              │ })                                   │   │  │
│  │   │ 或                          │              └──────────────────────────────────────┘   │  │
│  │   │ io.to(userID).emit(         │                                                           │  │
│  │   │   "updateMonitorIntoList",  │                                                           │  │
│  │   │   list                      │                                                           │  │
│  │   │ )                           │                                                           │  │
│  │   └──────────────────────────────┘                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                              │                                                     │
│                                              ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                  前端层 (Frontend)                                         │  │
│  │                                                                                              │  │
│  │   ┌────────────────────────────────────────────────────────────────────────────────────┐   │  │
│  │   │                     Vue 根实例 ($root) 响应式数据                                    │   │  │
│  │   │                                                                                        │   │  │
│  │   │   this.monitorList = {                                                                │   │  │
│  │   │       "1": {                                                                          │   │  │
│  │   │           id: 1,                                                                       │   │  │
│  │   │           name: "My Push",                                                             │   │  │
│  │   │           type: "push",                                                                │   │  │
│  │   │           pushToken: "abc123...xyz",    // ← 响应式数据                               │   │  │
│  │   │           // ...                                                                       │   │  │
│  │   │       },                                                                                │   │  │
│  │   │       // ... 其他监控 ...                                                               │   │  │
│  │   │   };                                                                                    │   │  │
│  │   │                                                                                        │   │  │
│  │   │   // Socket 事件监听                                                                    │   │  │
│  │   │   socket.on("monitorList", (data) => {                                                │   │  │
│  │   │       this.monitorList = data;    // 响应式更新整个列表                                │   │  │
│  │   │   });                                                                                   │   │  │
│  │   │                                                                                        │   │  │
│  │   │   socket.on("updateMonitorIntoList", (data) => {                                     │   │  │
│  │   │       // 增量更新                                                                      │   │  │
│  │   │       Object.entries(data).forEach(([id, monitor]) => {                              │   │  │
│  │   │           this.monitorList[id] = monitor;                                             │   │  │
│  │   │       });                                                                               │   │  │
│  │   │   });                                                                                   │   │  │
│  │   └────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                              │                                                 │  │
│  │              ┌───────────────────────────────┴───────────────────────────────┐              │  │
│  │              │                                                               │              │  │
│  │              ▼                                                               ▼              │  │
│  │   ┌──────────────────────┐                                     ┌──────────────────────┐   │  │
│  │   │   编辑页面            │                                     │     详情页面          │   │  │
│  │   │   EditMonitor.vue    │                                     │    Details.vue       │   │  │
│  │   └──────────┬───────────┘                                     └──────────┬───────────┘   │  │
│  │              │                                                               │              │  │
│  │              ▼ 数据获取方式                                                  ▼ 数据获取方式   │  │
│  │   ┌──────────────────────────────────┐              ┌──────────────────────────────────┐   │  │
│  │   │ // 编辑模式：从后端获取单条        │              │ // 从 $root.monitorList 获取     │   │  │
│  │   │ this.$root.getSocket().emit(     │              │ // 或通过 props/路由获取         │   │  │
│  │   │   "getMonitor", monitorID,        │              │ this.monitor =                     │   │  │
│  │   │   (res) => {                      │              │   this.$root.monitorList[monitorID]│   │  │
│  │   │       this.monitor = res.monitor; │              │                                    │   │  │
│  │   │       // 包含 pushToken            │              │ // 包含 pushToken                  │   │  │
│  │   │   }                               │              └──────────────────────────────────┘   │  │
│  │   │ );                              │                                                   │  │
│  │   │                                 │                                                   │  │
│  │   │ // 克隆模式：清除后重新生成      │                                                   │  │
│  │   │ if (this.isClone &&             │                                                   │  │
│  │   │     res.monitor.type === "push")│                                                   │  │
│  │   │   res.monitor.pushToken =       │                                                   │  │
│  │   │     undefined;                   │                                                   │  │
│  │   │ // 提交时自动生成新令牌          │                                                   │  │
│  │   └──────────────────────────────────┘                                                   │  │
│  │              │                                                               │              │  │
│  │              ▼ 数据绑定                                                      ▼ 数据绑定   │  │
│  │   ┌──────────────────────────────────┐              ┌──────────────────────────────────┐   │  │
│  │   │ // 本地响应式数据                 │              │ // 本地响应式数据                 │   │  │
│  │   │ this.monitor = {                 │              │ this.monitor = {                 │   │  │
│  │   │     id: 1,                        │              │     id: 1,                        │   │  │
│  │   │     pushToken: "abc123...xyz",   │              │     pushToken: "abc123...xyz",   │   │  │
│  │   │     // ...                        │              │     // ...                        │   │  │
│  │   │ };                               │              │ };                               │   │  │
│  │   │                                 │              │                                    │   │  │
│  │   │ // 计算属性                       │              │ // 计算属性                       │   │  │
│  │   │ pushURL() {                      │              │ pushURL() {                      │   │  │
│  │   │     return this.$root.baseURL +  │              │     return this.$root.baseURL +  │   │  │
│  │   │       "/api/push/" +             │              │       "/api/push/" +             │   │  │
│  │   │       this.monitor.pushToken +   │              │       this.monitor.pushToken +   │   │  │
│  │   │       "?status=up&msg=OK&ping="; │              │       "?status=up&msg=OK&ping="; │   │  │
│  │   │ }                                │              │ }                                │   │  │
│  │   └──────────────────────────────────┘              └──────────────────────────────────┘   │  │
│  │              │                                                               │              │  │
│  │              ▼ UI 显示                                                       ▼ UI 显示    │  │
│  │   ┌──────────────────────────────────────────────────────────────────────────────────┐   │  │
│  │   │                              Vue 模板 (Template)                                   │   │  │
│  │   │                                                                                      │   │  │
│  │   │   <!-- 编辑页 / 详情页共同的显示方式 -->                                           │   │  │
│  │   │   <div v-if="monitor.type === 'push'">                                             │   │  │
│  │   │       <!-- 绑定到计算属性 pushURL -->                                               │   │  │
│  │   │       <CopyableInput v-model="pushURL" disabled />                                 │   │  │
│  │   │       <!-- 或 -->                                                                   │   │  │
│  │   │       <a :href="pushURL">{{ pushURL }}</a>                                         │   │  │
│  │   │                                                                                      │   │  │
│  │   │       <!-- 编辑页独有：重置按钮 -->                                                 │   │  │
│  │   │       <button @click="resetToken">Reset Token</button>                             │   │  │
│  │   │   </div>                                                                             │   │  │
│  │   │                                                                                      │   │  │
│  │   │   // resetToken 方法（编辑页）                                                       │   │  │
│  │   │   resetToken() {                                                                     │   │  │
│  │   │       this.monitor.pushToken = genSecret(32);  // 生成新令牌                       │   │  │
│  │   │       // 触发 pushURL 重新计算 → UI 立即更新                                        │   │  │
│  │   │       // 需点击 Save 才保存到数据库                                                  │   │  │
│  │   │   }                                                                                  │   │  │
│  │   └──────────────────────────────────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                              │                                                     │
│                                              ▼ 保存流程                                            │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                              保存时的数据流 (编辑页)                                        │  │
│  │                                                                                              │  │
│  │   用户点击 "Save" 按钮                                                                       │  │
│  │          │                                                                                   │  │
│  │          ▼                                                                                   │  │
│  │   this.$root.getSocket().emit(                                                              │  │
│  │       "editMonitor",                                                                         │  │
│  │       this.monitor,    // 包含 pushToken                                                    │  │
│  │       (res) => { ... }                                                                      │  │
│  │   );                                                                                         │  │
│  │          │                                                                                   │  │
│  │          ▼                                                                                   │  │
│  │   后端 server.js:800-881                                                                     │  │
│  │   socket.on("editMonitor", async (monitor, callback) => {                                  │  │
│  │       let bean = await R.findOne("monitor", " id = ? ", [monitor.id]);                    │  │
│  │                                                                                              │  │
│  │       // 关键绑定：前端 monitor.pushToken → 数据库 bean.pushToken                          │  │
│  │       bean.pushToken = monitor.pushToken;   // ← 保存到数据库                              │  │
│  │                                                                                              │  │
│  │       await R.store(bean);                    // 持久化                                    │  │
│  │                                                                                              │  │
│  │       // 发送更新到所有前端客户端                                                             │  │
│  │       await server.sendUpdateMonitorIntoList(socket, bean.id);                             │  │
│  │       // → 触发 "updateMonitorIntoList" 事件                                               │  │
│  │       // → 所有打开的浏览器标签页的 $root.monitorList 自动更新                             │  │
│  │       // → 详情页的 UI 自动响应式更新                                                       │  │
│  │   });                                                                                        │  │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.5.11 关键代码位置汇总（令牌绑定链路）

| 层级 | 功能 | 文件路径 | 关键行号 |
|------|------|----------|----------|
| 数据层 | 数据库字段 | 表 `monitor.push_token` | - |
| ORM 层 | 字段映射 | RedBean ORM 自动映射 | - |
| 后端模型 | toJSON 包含 pushToken | `server/model/monitor.js` | 218, 233 |
| 后端 | 监控列表获取 | `server/uptime-kuma-server.js` | 219-234, 256-277 |
| 后端 | 单条监控获取 | `server/server.js` | 987-1000 |
| 后端 | 保存监控 | `server/server.js` | 800-881, 940-954 |
| 前端根实例 | Socket 监听 | `src/mixins/socket.js` | 147-163 |
| 前端编辑页 | 初始化获取数据 | `src/pages/EditMonitor.vue` | 3764-3854 |
| 前端编辑页 | pushURL 计算属性 | `src/pages/EditMonitor.vue` | 3271-3272 |
| 前端编辑页 | 令牌重置 | `src/pages/EditMonitor.vue` | 4006-4008 |
| 前端详情页 | pushURL 计算属性 | `src/pages/Details.vue` | 583-585 |

### 2.6 Push Token 可见性边界

Uptime Kuma 通过 `toJSON()` 方法的 `includeSensitiveData` 参数和 `toPublicJSON()` 方法来控制 Push Token 等敏感数据的可见性边界。

#### 2.6.1 核心控制机制

**`toJSON()` 方法的 `includeSensitiveData` 参数：**

**文件位置：** `server/model/monitor.js:117, 218-250`

```javascript
// toJSON() 方法签名
toJSON(preloadData = {}, includeSensitiveData = true) {
    // ... 基础数据字段（不包含敏感数据）...
    
    if (includeSensitiveData) {
        // 仅当 includeSensitiveData = true 时才包含这些字段
        data = {
            ...data,
            // 认证凭据
            headers: this.headers,
            body: this.body,
            basic_auth_user: this.basic_auth_user,
            basic_auth_pass: this.basic_auth_pass,
            // OAuth 凭据
            oauth_client_id: this.oauth_client_id,
            oauth_client_secret: this.oauth_client_secret,
            oauth_token_url: this.oauth_token_url,
            oauth_scopes: this.oauth_scopes,
            // Push Token（敏感！）
            pushToken: this.pushToken,           // ← 关键字段
            // 数据库连接字符串
            databaseConnectionString: this.databaseConnectionString,
            // Radius 凭据
            radiusUsername: this.radiusUsername,
            radiusPassword: this.radiusPassword,
            radiusSecret: this.radiusSecret,
            // MQTT 凭据
            mqttUsername: this.mqttUsername,
            mqttPassword: this.mqttPassword,
            // TLS 证书密钥
            tlsCa: this.tlsCa,
            tlsCert: this.tlsCert,
            tlsKey: this.tlsKey,
            // Kafka SASL 选项
            kafkaProducerSaslOptions: JSON.parse(this.kafkaProducerSaslOptions),
            // RabbitMQ 凭据
            rabbitmqUsername: this.rabbitmqUsername,
            rabbitmqPassword: this.rabbitmqPassword,
        };
    }
    
    data.includeSensitiveData = includeSensitiveData;  // 标记数据来源
    return data;
}
```

**`toPublicJSON()` 方法（用于公共访问）：**

**文件位置：** `server/model/monitor.js:85-108`

```javascript
// toPublicJSON() 只返回最小必要的公开数据
async toPublicJSON(showTags = false, certExpiry = false) {
    let obj = {
        id: this.id,
        name: this.name,
        sendUrl: this.sendUrl,
        type: this.type,
    };

    if (this.sendUrl) {
        obj.url = this.customUrl ?? this.url;
    }

    // 可选的扩展字段，仍不包含敏感数据
    if (showTags) {
        obj.tags = await this.getTags();
    }

    if (certExpiry) {
        const { certExpiryDaysRemaining, validCert } = await this.getCertExpiry(this.id);
        obj.certExpiryDaysRemaining = certExpiryDaysRemaining;
        obj.validCert = validCert;
    }

    return obj;  // 绝对不包含 pushToken
}
```

#### 2.6.2 包含 Push Token 的链路（Owner 可见的管理链路）

以下链路中 `includeSensitiveData = true`（默认值），**Push Token 会被带出**：

| 链路类型 | 调用方式 | 文件位置 | 说明 |
|----------|----------|----------|------|
| **监控列表 Socket 事件** | `monitor.toJSON(preloadData)` | `server/uptime-kuma-server.js:275` | 登录用户接收 `monitorList` 事件 |
| **监控更新 Socket 事件** | `monitor.toJSON(preloadData)` | `server/uptime-kuma-server.js:275` | 登录用户接收 `updateMonitorIntoList` 事件 |
| **编辑页获取单条监控** | `monitor.toJSON(preloadData)` | `server/server.js:998` | `getMonitor` Socket 事件响应 |

**详细代码分析：**

**1. 监控列表获取（所有登录用户可见）：**

**文件位置：** `server/uptime-kuma-server.js:256-277`

```javascript
async getMonitorJSONList(userID, monitorID = null) {
    // ... 数据库查询 ...
    let monitorList = await R.find("monitor", query + "ORDER BY weight DESC, name", queryParams);
    
    // ... 预处理数据 ...
    const preloadData = await Monitor.preparePreloadData(monitorData);
    
    // 关键：调用 toJSON() 时未传入第二个参数，使用默认值 includeSensitiveData = true
    const result = {};
    monitorList.forEach((monitor) => (result[monitor.id] = monitor.toJSON(preloadData)));
    // result 中的每个 monitor 都包含 pushToken
    return result;
}
```

**触发的 Socket 事件：**
- `io.to(userID).emit("monitorList", list)` → 完整列表推送
- `io.to(userID).emit("updateMonitorIntoList", list)` → 增量更新

**2. 编辑页获取单条监控：**

**文件位置：** `server/server.js:987-1000`

```javascript
socket.on("getMonitor", async (monitorID, callback) => {
    try {
        checkLogin(socket);  // 验证已登录
        
        let monitor = await R.findOne("monitor", " id = ? AND user_id = ? ", [monitorID, socket.userID]);
        const monitorData = [{ id: monitor.id, active: monitor.active }];
        const preloadData = await Monitor.preparePreloadData(monitorData);
        
        // 关键：toJSON() 使用默认的 includeSensitiveData = true
        callback({
            ok: true,
            monitor: monitor.toJSON(preloadData),  // 包含 pushToken
        });
    } catch (e) {
        // ... 错误处理
    }
});
```

#### 2.6.3 不包含 Push Token 的链路（通知和非管理链路）

以下链路中 **Push Token 会被过滤掉**：

| 链路类型 | 调用方式 | 过滤机制 | 文件位置 |
|----------|----------|----------|----------|
| **所有通知发送** | `monitor.toJSON(preloadData, false)` | `includeSensitiveData = false` | `server/model/monitor.js:1544` |
| **公共状态页 Heartbeat API** | `heartbeat.toPublicJSON()` | `toPublicJSON()` 方法 | `server/routers/status-page-router.js:97` |
| **公共状态页 Incident API** | `incident.toPublicJSON()` | `toPublicJSON()` 方法 | `server/socket-handlers/status-page-socket-handler.js:74, 176, 257` |
| **公共状态页 Config** | `statusPage.toPublicJSON()` | `toPublicJSON()` 方法 | `server/model/status_page.js:268` |

**详细代码分析：**

**1. 通知发送（关键安全边界）：**

**文件位置：** `server/model/monitor.js:1486-1552`

```javascript
static async sendNotification(isFirstBeat, monitor, bean) {
    if (!isFirstBeat || bean.status === DOWN) {
        const notificationList = await Monitor.getNotificationList(monitor);

        // 构建通知消息
        let text;
        if (bean.status === UP) {
            text = "✅ Up";
        } else {
            text = "🔴 Down";
        }
        let msg = `[${monitor.name}] [${text}] ${bean.msg}`;

        // ... 构建 heartbeatJSON ...

        const monitorData = [{ id: monitor.id, active: monitor.active, name: monitor.name }];
        const preloadData = await Monitor.preparePreloadData(monitorData);

        // 遍历所有通知提供者（Webhook、Email、Telegram、Slack 等）
        for (let notification of notificationList) {
            try {
                await Notification.send(
                    JSON.parse(notification.config),
                    msg,
                    // 关键！显式传入 includeSensitiveData = false
                    monitor.toJSON(preloadData, false),  // ← 不包含 pushToken！
                    heartbeatJSON
                );
            } catch (e) {
                log.error("monitor", "Cannot send notification to " + notification.name);
                log.error("monitor", e);
            }
        }
    }
}
```

**影响范围：**
- **Webhook 通知**：请求体中 `monitorJSON` 不包含 `pushToken`
- **Email 通知**：模板变量中不包含 `pushToken`
- **Telegram/Discord/Slack 等聊天通知**：消息中不包含 `pushToken`
- **所有其他通知提供者**：都无法获取 `pushToken`

**2. 公共状态页 API：**

**文件位置：** `server/routers/status-page-router.js:64-110`

```javascript
// 公共状态页心跳数据 API（无需登录）
router.get("/api/status-page/heartbeat/:slug", cache("1 minutes"), async (request, response) => {
    // ... 查询公开的监控组 ...
    
    let monitorIDList = await R.getCol(
        `
        SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
        WHERE monitor_group.group_id = \`group\`.id
        AND public = 1    -- 仅公开的分组
        AND \`group\`.status_page_id = ?
    `,
        [statusPageID]
    );

    for (let monitorID of monitorIDList) {
        let list = await R.getAll(
            `SELECT * FROM heartbeat WHERE monitor_id = ? ORDER BY time DESC LIMIT 100`,
            [monitorID]
        );

        list = R.convertToBeans("heartbeat", list);
        // 关键：使用 toPublicJSON()，不包含敏感数据
        heartbeatList[monitorID] = list.reverse().map((row) => row.toPublicJSON());
    }

    response.json({
        heartbeatList,  // 所有心跳数据都已过滤
        uptimeList,
    });
});
```

**3. 公共状态页事件处理：**

**文件位置：** `server/socket-handlers/status-page-socket-handler.js:64-82`

```javascript
// 保存事件后返回公共数据
await R.store(incidentBean);

callback({
    ok: true,
    incident: incidentBean.toPublicJSON(),  // 不包含敏感数据
});
```

#### 2.6.4 可见性边界总览表

| 访问场景 | 身份认证 | Push Token 可见 | 数据序列化方式 |
|----------|----------|-----------------|----------------|
| **管理后台（仪表板）** | 已登录 Owner | ✅ 可见 | `toJSON(preloadData)` 包含敏感数据 |
| **编辑页面** | 已登录 Owner | ✅ 可见 | `toJSON(preloadData)` 包含敏感数据 |
| **详情页面** | 已登录 Owner | ✅ 可见 | 从 `$root.monitorList` 获取（已包含） |
| **所有通知（Webhook/Email/Telegram 等）** | 通知接收者 | ❌ **不可见** | `toJSON(preloadData, false)` 过滤敏感数据 |
| **公共状态页** | 匿名访问者 | ❌ **不可见** | `toPublicJSON()` 最小化数据 |
| **公共状态页 RSS** | 匿名访问者 | ❌ **不可见** | `toPublicJSON()` 最小化数据 |
| **状态页徽章 API** | 匿名访问者 | ❌ **不可见** | 直接查询状态值，不暴露监控详情 |
| **外部心跳 API** | 外部系统 | 需**携带** token 才能调用 | 反向：外部系统**必须知道** token 才能发送心跳 |

#### 2.6.5 安全边界流程图

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              Push Token 安全边界架构                                              │
├──────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐│
│  │                              安全边界之内（Owner 可见）                                     ││
│  │                                                                                              ││
│  │   ┌────────────────────────────────────────────────────────────────────────────────────┐  ││
│  │   │                         后端 API 层（已登录用户）                                    │  ││
│  │   │                                                                                        │  ││
│  │   │  Socket 事件:                                                                          │  ││
│  │   │  ┌──────────────────┐     ┌────────────────────────────┐                            │  ││
│  │   │  │   monitorList    │     │   updateMonitorIntoList    │                            │  ││
│  │   │  │   (完整列表)      │     │   (增量更新)                │                            │  ││
│  │   │  │                  │     │                            │                            │  ││
│  │   │  │ toJSON(preloadData) │  │ toJSON(preloadData)     │                            │  ││
│  │   │  │ includeSensitiveData=true │ includeSensitiveData=true │                         │  ││
│  │   │  │                  │     │                            │                            │  ││
│  │   │  │ ✅ 包含 pushToken │     │ ✅ 包含 pushToken         │                            │  ││
│  │   │  └────────┬─────────┘     └─────────────┬──────────────┘                            │  ││
│  │   │           │                               │                                           │  ││
│  │   │           └───────────────┬───────────────┘                                           │  ││
│  │   │                           │                                                           │  ││
│  │   │                           ▼                                                           │  ││
│  │   │              ┌────────────────────────────┐                                            │  ││
│  │   │              │   前端: $root.monitorList │                                            │  ││
│  │   │              │   (Vue 响应式数据)         │                                            │  ││
│  │   │              │   ✅ 包含 pushToken        │                                            │  ││
│  │   │              └─────────────┬──────────────┘                                            │  ││
│  │   │                            │                                                            │  ││
│  │   │            ┌───────────────┼───────────────┐                                            │  ││
│  │   │            │               │               │                                            │  ││
│  │   │            ▼               ▼               ▼                                            │  ││
│  │   │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                                  │  ││
│  │   │   │ 仪表板页面    │ │ 编辑页面      │ │ 详情页面      │                                  │  ││
│  │   │   │ Dashboard     │ │ EditMonitor  │ │ Details      │                                  │  ││
│  │   │   │              │ │              │ │              │                                  │  ││
│  │   │   │ 显示状态列表  │ │ 显示 Push URL│ │ 显示 Push URL│                                  │  ││
│  │   │   │ ✅ 可用于查看 │ │ ✅ 可编辑令牌 │ │ ✅ 可复制 URL │                                  │  ││
│  │   │   └──────────────┘ └──────────────┘ └──────────────┘                                  │  ││
│  │   │                                                                                        │  ││
│  │   └────────────────────────────────────────────────────────────────────────────────────┘  ││
│  └──────────────────────────────────────────────────────────────────────────────────────────┘│
│                                              │                                                   │
│                                              │ 安全边界                                          │
│                                              ▼                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐│
│  │                            安全边界之外（敏感数据被过滤）                                   ││
│  │                                                                                              ││
│  │   ┌────────────────────────────────────────────────────────────────────────────────────┐  ││
│  │   │                              通知系统                                                  │  ││
│  │   │                                                                                        │  ││
│  │   │   sendNotification() 调用:                                                            │  ││
│  │   │   monitor.toJSON(preloadData, false)  ← includeSensitiveData = false                 │  ││
│  │   │                                                                                        │  ││
│  │   │   ❌ 不包含 pushToken, headers, body, basic_auth_pass 等敏感字段                      │  ││
│  │   │                                                                                        │  ││
│  │   │   影响范围:                                                                             │  ││
│  │   │   ┌─────────────┬─────────────┬─────────────┬──────────────────┐                    │  ││
│  │   │   │  Webhook    │   Email     │  Telegram   │ Slack/Discord    │                    │  ││
│  │   │   │             │             │             │ Teams/...         │                    │  ││
│  │   │   │ ❌ 无 token │ ❌ 无 token │ ❌ 无 token │ ❌ 无 token      │                    │  ││
│  │   │   └─────────────┴─────────────┴─────────────┴──────────────────┘                    │  ││
│  │   │                                                                                        │  ││
│  │   └────────────────────────────────────────────────────────────────────────────────────┘  ││
│  │                                                                                              ││
│  │   ┌────────────────────────────────────────────────────────────────────────────────────┐  ││
│  │   │                            公共状态页（匿名访问）                                      │  ││
│  │   │                                                                                        │  ││
│  │   │   使用 toPublicJSON() 方法:                                                            │  ││
│  │   │   ┌─────────────────────┐                                                              │  ││
│  │   │   │  id, name, type,    │ ← 仅返回最小必要字段                                        │  ││
│  │   │   │  url (可选)          │                                                              │  ││
│  │   │   │  tags (可选)         │                                                              │  ││
│  │   │   │  certExpiry (可选)  │                                                              │  ││
│  │   │   └─────────────────────┘                                                              │  ││
│  │   │                                                                                        │  ││
│  │   │   API 端点（无需认证）:                                                                 │  ││
│  │   │   ┌──────────────────────────────┐ ┌──────────────────────────────┐                    │  ││
│  │   │   │ /api/status-page/:slug        │ │ /api/status-page/:slug/badge │                    │  ││
│  │   │   │ /api/status-page/:slug/rss    │ │ /api/status-page/heartbeat/  │                    │  ││
│  │   │   │ /api/status-page/heartbeat/   │ │ /api/status-page/:slug/      │                    │  ││
│  │   │   │ :slug                          │ │ incident-history             │                    │  ││
│  │   │   │                              │ │                              │                    │  ││
│  │   │   │ ❌ 所有监控详情均过滤敏感数据  │ │ ❌ 仅返回状态值             │                    │  ││
│  │   │   └──────────────────────────────┘ └──────────────────────────────┘                    │  ││
│  │   │                                                                                        │  ││
│  │   └────────────────────────────────────────────────────────────────────────────────────┘  ││
│  └──────────────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐│
│  │                          特殊：外部心跳 API（反向访问）                                      ││
│  │                                                                                              ││
│  │   端点: POST /api/push/:pushToken                                                          ││
│  │                                                                                              ││
│  │   ┌────────────────────────────────────────────────────────────────────────────────────┐  ││
│  │   │  外部系统必须**预先知道** pushToken 才能调用此 API                                     │  ││
│  │   │                                                                                        │  ││
│  │   │  流程:                                                                                 │  ││
│  │   │  1. Owner 在管理后台查看/复制 Push URL（包含 token）                                   │  ││
│  │   │  2. Owner 将 token 配置到外部 cron/脚本中                                             │  ││
│  │   │  3. 外部系统携带 token 调用 /api/push/:token                                          │  ││
│  │   │  4. 后端验证 token 对应的 monitor 并处理心跳                                          │  ││
│  │   │                                                                                        │  ││
│  │   │  关键点: token 由 Owner 主动分发到外部系统，而非 API 暴露                              │  ││
│  │   └────────────────────────────────────────────────────────────────────────────────────┘  ││
│  └──────────────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.6.6 关键安全设计总结

| 设计原则 | 实现方式 | 安全意义 |
|----------|----------|----------|
| **最小权限原则** | `toPublicJSON()` 只返回必要字段 | 公共 API 不会意外泄露敏感数据 |
| **显式控制** | `includeSensitiveData` 默认为 `true` 但通知时显式传 `false` | 防止在通知、Webhook 等扩展点泄露 |
| **数据来源标记** | `data.includeSensitiveData = includeSensitiveData` | 前端可判断数据是否包含敏感信息，克隆时清除 |
| **令牌分发可控** | token 仅在管理界面显示，需 Owner 主动复制 | 外部系统无法通过 API 主动获取 token |

#### 2.6.7 相关代码位置汇总（可见性边界）

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| `toJSON()` 方法定义 | `server/model/monitor.js` | 117, 218-250 |
| `toPublicJSON()` 方法定义 | `server/model/monitor.js` | 85-108 |
| 通知发送时过滤敏感数据 | `server/model/monitor.js` | 1544 |
| 监控列表获取（包含敏感数据） | `server/uptime-kuma-server.js` | 275 |
| 编辑页获取单条监控（包含敏感数据） | `server/server.js` | 998 |
| 公共状态页心跳 API（过滤） | `server/routers/status-page-router.js` | 97 |
| 公共状态页事件处理（过滤） | `server/socket-handlers/status-page-socket-handler.js` | 74, 176, 257 |
| 前端克隆时清除敏感数据标记 | `src/pages/EditMonitor.vue` | 3805 |

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
