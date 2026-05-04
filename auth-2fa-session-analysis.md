# Uptime Kuma 认证、会话、2FA 和权限深度分析

## 1. 核心概念澄清

### 1.1 "Session" 在 Uptime Kuma 中的真实含义

Uptime Kuma **不使用传统的服务器端 Session**（如 express-session），而是采用 **JWT (JSON Web Token) 无状态机制**。

| 传统 Session 机制 | Uptime Kuma JWT 机制 |
|-----------------|---------------------|
| 服务器存储 session 数据 | 无状态，Token 自包含信息 |
| Session ID 存在 Cookie | JWT 存在 localStorage/sessionStorage |
| 每次请求查 Session 存储 | 每次请求验证 JWT 签名 |
| 服务器可主动使 Session 失效 | 依赖密码修改或 Secret 重置来失效 |

### 1.2 Socket 连接与鉴权的关系

**关键理解：Socket 连接 ≠ 已鉴权**

```
时序流程：
1. Socket.io 连接建立成功 → 此时 socket 是未鉴权状态 (socket.userID = undefined)
2. 服务端发送 loginRequired 事件（或前端检测需要登录）
3. 前端调用 loginByToken(token) 进行鉴权
4. 服务端验证 JWT → 设置 socket.userID
5. 此后的操作才能通过 checkLogin(socket) 检查
```

`afterLogin()` 是关键衔接函数：

```javascript
// server/server.js:1804-1818
async function afterLogin(socket, user) {
    socket.userID = user.id;           // 核心：设置 socket 级别的用户标识
    socket.join(user.id);              // 加入用户房间（用于广播）
    
    // 发送初始化数据...
}
```

---

## 2. 三层权限检查机制

### 2.1 权限检查函数定义

| 检查层级 | 函数 | 检查逻辑 | 适用场景 |
|---------|------|---------|---------|
| L1 基础认证 | `checkLogin(socket)` | 检查 `socket.userID` 是否存在 | 所有敏感操作入口 |
| L2 资源归属 | `checkOwner(userID, monitorID)` | SQL: `WHERE id = ? AND user_id = ?` | 监控的增删改查（部分） |
| L3 双重验密 | `doubleCheckPassword(socket, password)` | 验证当前密码是否正确 | 高危操作：启用/禁用 2FA、禁用认证 |

### 2.2 各函数实现细节

#### checkLogin (L1 基础认证)

```javascript
// server/util-server.js:637-641
exports.checkLogin = (socket) => {
    if (!socket.userID) {
        throw new Error("You are not logged in.");
    }
};
```

**说明**：仅检查 socket 连接是否已通过登录流程设置了 `userID`。这是最基础的防线。

#### checkOwner (L2 资源归属)

```javascript
// server/server.js:1789-1795
async function checkOwner(userID, monitorID) {
    let row = await R.getRow(
        "SELECT id FROM monitor WHERE id = ? AND user_id = ? ", 
        [monitorID, userID]
    );
    if (!row) {
        throw new Error("You do not own this monitor.");
    }
}
```

**说明**：检查特定 monitor 是否属于当前用户。这是为多用户架构预留的设计。

#### doubleCheckPassword (L3 双重验密)

```javascript
// server/util-server.js:651-663
exports.doubleCheckPassword = async (socket, currentPassword) => {
    if (typeof currentPassword !== "string") {
        throw new Error("Wrong data type?");
    }
    let user = await R.findOne("user", " id = ? AND active = 1 ", [socket.userID]);
    if (!user || !passwordHash.verify(currentPassword, user.password)) {
        throw new Error("Incorrect current password");
    }
    return user;
};
```

**说明**：要求用户再次输入当前密码进行验证，用于防止 CSRF 和会话劫持后的高危操作。

---

## 3. 系统设置入口的二次验密条件

### 3.1 关键发现：仅一种场景需要双重验密

**文件位置**: `server/server.js:1475-1535`

```javascript
socket.on("setSettings", async (data, currentPassword, callback) => {
    try {
        checkLogin(socket);  // L1 检查

        // 注释非常重要：
        // Disabled Auth + Want to Disable Auth => No Check
        // Disabled Auth + Want to Enable Auth => No Check
        // Enabled Auth + Want to Disable Auth => Check!!  ← 唯一需要双重验密的场景
        // Enabled Auth + Want to Enable Auth => No Check
        
        const currentDisabledAuth = await setting("disableAuth");
        if (!currentDisabledAuth && data.disableAuth) {
            // 条件：当前认证已启用 (!currentDisabledAuth) 
            //       且用户想要禁用认证 (data.disableAuth)
            await doubleCheckPassword(socket, currentPassword);
        }

        // 其他设置修改不需要双重验密
        await setSettings("general", data);
        // ...
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 3.2 场景决策表

| 当前认证状态 | 操作意图 | 是否需要双重验密 | 原因 |
|------------|---------|-----------------|------|
| 已禁用 | 保持禁用 | 否 | 无安全风险变化 |
| 已禁用 | 启用认证 | 否 | 提升安全性，不需要额外验证 |
| **已启用** | **禁用认证** | **是** | **降低安全性，高危操作，必须验证身份** |
| 已启用 | 保持启用 | 否 | 无安全风险变化 |

### 3.3 为什么禁用认证需要双重验密？

禁用认证 (`disableAuth = true`) 会导致：

1. **所有 API 端点不再需要认证**
2. **任何人都可以访问 Dashboard**
3. **任何人都可以修改监控配置**
4. **任何人都可以修改系统设置**

这是一个**不可逆的高危操作**，因此要求用户再次确认密码，防止：
- CSRF 攻击
- 会话劫持后的恶意操作
- 误操作

### 3.4 其他系统设置操作

| 操作 | 权限检查 | 是否需要双重验密 |
|-----|---------|-----------------|
| `getSettings` | `checkLogin` | 否 |
| `setSettings` (修改时区) | `checkLogin` | 否 |
| `setSettings` (修改入口页) | `checkLogin` | 否 |
| `setSettings` (修改 Chrome 路径) | `checkLogin` | 否 |
| `setSettings` (**禁用认证**) | `checkLogin` + `doubleCheckPassword` | **是** |

---

## 4. 监控操作：仅登录校验 vs 资源归属校验

### 4.1 发现的关键差异

监控相关操作的权限检查**不一致**，存在两种模式：

1. **仅登录校验 (checkLogin only)**：只检查用户是否登录，不检查资源归属
2. **资源归属校验 (checkLogin + 所有权检查)**：检查登录 + 确认用户是资源所有者

### 4.2 监控操作权限检查明细表

#### 模式 A：资源归属校验（相对安全）

| Socket 事件 | 权限检查方式 | 代码位置 | 检查逻辑 |
|------------|-------------|---------|---------|
| `getMonitor` | SQL 条件隐式检查 | server.js:993 | `WHERE id = ? AND user_id = ?` |
| `editMonitor` | 显式比较 | server.js:807-809 | `if (bean.user_id !== socket.userID)` |
| `deleteMonitor` | SQL 条件隐式检查 | server.js:1116 | `WHERE id = ? AND user_id = ?` |
| `resumeMonitor` | `checkOwner` 函数 | server.js:1067-1068 | 内部调用 `checkOwner` |
| `pauseMonitor` | `checkOwner` 函数 | server.js:1085-1086 | 内部调用 `checkOwner` |

#### editMonitor 的显式检查示例

```javascript
// server/server.js:800-809
socket.on("editMonitor", async (monitor, callback) => {
    try {
        checkLogin(socket);  // L1 检查
        
        let bean = await R.findOne("monitor", " id = ? ", [monitor.id]);
        // 显式检查所有权
        if (bean.user_id !== socket.userID) {
            throw new Error("Permission denied.");
        }
        // ... 修改逻辑
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

---

#### 模式 B：仅登录校验（潜在安全隐患）

| Socket 事件 | 权限检查方式 | 代码位置 | 是否检查资源归属 |
|------------|-------------|---------|-----------------|
| `add` (新建监控) | `checkLogin` only | server.js:727 | 否（新建，合理） |
| `getMonitorList` | `checkLogin` only | server.js:973 | 否（SQL 过滤返回，合理） |
| `getMonitorBeats` | `checkLogin` only | server.js:1031 | **否（隐患）** |
| `addMonitorTag` | `checkLogin` only | server.js:1284 | **否（隐患）** |
| `deleteMonitorTag` | `checkLogin` only | server.js:1334 | **否（隐患）** |

#### getMonitorBeats 的隐患示例

```javascript
// server/server.js:1029-1062
socket.on("getMonitorBeats", async (monitorID, period, callback) => {
    try {
        checkLogin(socket);  // 只检查是否登录
        
        log.info("monitor", `Get Monitor Beats: ${monitorID} User ID: ${socket.userID}`);

        // ⚠️ 没有检查 monitorID 是否属于 socket.userID！
        // SQL 中只过滤了 monitor_id，没有 user_id
        let list = await R.getAll(
            `SELECT * FROM heartbeat 
             WHERE monitor_id = ? AND time > ${sqlHourOffset}
             ORDER BY time ASC`,
            [monitorID, -period]
        );
        
        callback({ ok: true, data: list });
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

#### addMonitorTag 的隐患示例

```javascript
// server/server.js:1282-1307
socket.on("addMonitorTag", async (tagID, monitorID, value, callback) => {
    try {
        checkLogin(socket);  // 只检查是否登录
        
        // ⚠️ 没有检查 monitorID 是否属于 socket.userID！
        await R.exec(
            "INSERT INTO monitor_tag (tag_id, monitor_id, value) VALUES (?, ?, ?)",
            [tagID, monitorID, value]
        );
        
        await server.sendUpdateMonitorIntoList(socket, monitorID);
        callback({ ok: true, msg: "successAdded", msgi18n: true });
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 4.3 标签操作的特殊性

标签系统分为两层：

| 层级 | 操作 | 权限检查 | 设计意图 |
|-----|------|---------|---------|
| 标签定义 | `addTag`, `editTag`, `deleteTag` | `checkLogin` only | 标签是系统级资源，不属于特定用户 |
| 标签关联 | `addMonitorTag`, `deleteMonitorTag` | `checkLogin` only | **应该检查 monitor 归属，但实际没有** |

**问题**：当前系统是单用户设计，`checkOwner` 是为多用户预留的。但在多用户场景下：
- 用户 A 可以为用户 B 的 monitor 添加/删除标签
- 用户 A 可以查看用户 B 的 monitor 的心跳历史

### 4.4 监控操作权限检查差异图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          监控操作权限检查差异                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                     资源归属校验（相对安全）                          │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │  │
│  │  │ getMonitor│ │editMonitor│ │deleteMonitor│ │resumeMonitor│ │pauseMonitor│ │  │
│  │  │          │ │          │ │          │ │          │ │          │ │  │
│  │  │ SQL隐式  │ │ 显式比较  │ │ SQL隐式  │ │checkOwner│ │checkOwner│ │  │
│  │  │ AND user_id│ │bean.user_id│ │ AND user_id│ │  函数   │ │  函数   │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                     仅登录校验（潜在隐患）                            │  │
│  │  ┌──────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │  │
│  │  │   add    │ │getMonitorList│ │getMonitorBeats│ │addMonitorTag │ │  │
│  │  │ (新建)   │ │ (SQL过滤返回) │ │   ⚠️ 隐患    │ │  ⚠️ 隐患    │ │  │
│  │  └──────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │  │
│  │                                                        ┌──────────────┐│  │
│  │                                                        │deleteMonitorTag││  │
│  │                                                        │  ⚠️ 隐患    ││  │
│  │                                                        └──────────────┘│  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  说明：                                                                      │
│  - "新建"操作不需要归属检查是合理的                                          │
│  - "getMonitorList" 通过 SQL user_id 条件过滤返回结果，也是安全的           │
│  - 但 "getMonitorBeats"、"addMonitorTag"、"deleteMonitorTag" 没有检查      │
│    monitor 是否属于当前用户，在多用户场景下存在越权风险                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 状态页管理的保护边界

### 5.1 状态页权限检查现状

**文件位置**: `server/socket-handlers/status-page-socket-handler.js`

| 操作类型 | Socket 事件 | 权限检查 | 资源归属检查 |
|---------|------------|---------|-------------|
| 读取（公开） | `getIncidentHistory` | 无 | 否 |
| 读取（管理） | `getStatusPage` | `checkLogin` | 否 |
| 写入 | `postIncident`, `editIncident`, `deleteIncident` | `checkLogin` | 否 |
| 写入 | `resolveIncident`, `unpinIncident` | `checkLogin` | 否 |
| 配置管理 | `saveStatusPage`, `addStatusPage`, `deleteStatusPage` | `checkLogin` | 否 |

### 5.2 状态页的公开访问设计

```javascript
// server/socket-handlers/status-page-socket-handler.js:103-122
socket.on("getIncidentHistory", async (slug, cursor, callback) => {
    try {
        let statusPageID = await StatusPage.slugToID(slug);
        if (!statusPageID) {
            throw new Error("slug is not found");
        }

        // 关键设计：未登录用户也可以访问
        const isPublic = !socket.userID;
        const result = await StatusPage.getIncidentHistory(statusPageID, cursor, isPublic);
        callback({ ok: true, ...result });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

### 5.3 状态页管理的保护边界分析

#### 保护边界 1：公开读取 vs 管理操作

```
┌─────────────────────────────────────────────────────────────────┐
│                    状态页访问边界                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  公开访问区（无需登录）                                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  GET /status/{slug}                                      │  │
│  │  socket: getIncidentHistory (isPublic = true)           │  │
│  │  显示监控状态、事件历史（仅已发布的活跃事件）               │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  管理操作区（需要 checkLogin）                                  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  状态页配置：addStatusPage, saveStatusPage, deleteStatusPage │
│  │  事件管理：postIncident, editIncident, deleteIncident   │  │
│  │  事件状态：resolveIncident, unpinIncident               │  │
│  │  配置读取：getStatusPage                                 │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 保护边界 2：缺乏资源归属检查

与监控操作不同，状态页操作**完全没有资源归属检查**：

```javascript
// server/socket-handlers/status-page-socket-handler.js:34-82
socket.on("postIncident", async (slug, incident, callback) => {
    try {
        checkLogin(socket);  // 只检查是否登录
        
        let statusPageID = await StatusPage.slugToID(slug);
        // ⚠️ 没有检查状态页的"所有者"
        // 实际上 status_page 表根本没有 user_id 字段！
        
        let incidentBean = R.dispense("incident");
        incidentBean.title = incident.title;
        incidentBean.content = incident.content;
        incidentBean.status_page_id = statusPageID;
        await R.store(incidentBean);
        
        callback({ ok: true, incident: incidentBean.toPublicJSON() });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

### 5.4 状态页与监控的保护边界对比

| 维度 | 监控 (Monitor) | 状态页 (Status Page) |
|-----|---------------|---------------------|
| 数据模型 | `monitor.user_id` 字段 | `status_page` 无 `user_id` |
| 归属检查 | 部分操作有 `checkOwner` | **完全没有** |
| 设计目标 | 为多用户预留 | 单用户系统级资源 |
| 公开访问 | 无 | 有（状态页展示） |

### 5.5 对保护边界的影响

当前设计的前提假设：**Uptime Kuma 是单用户系统**

在这个假设下：
- 状态页没有 `user_id` 字段是合理的
- 监控的 `checkOwner` 检查在单用户场景下永远通过
- `getMonitorBeats` 等操作的隐患不会实际触发

**但如果未来转向多用户架构**：
- 状态页表需要添加 `user_id` 字段
- 状态页操作需要添加归属检查
- 监控操作的不一致性需要统一修复

---

## 6. 二次验证 (2FA) 机制

### 6.1 2FA 操作的权限要求

所有 2FA 相关操作**都需要双重验密**：

| 操作 | 权限检查 | 说明 |
|-----|---------|------|
| `prepare2FA` | `checkLogin` + `doubleCheckPassword` | 准备启用 2FA |
| `save2FA` | `checkLogin` + `doubleCheckPassword` | 确认启用 2FA |
| `disable2FA` | `checkLogin` + `doubleCheckPassword` | 禁用 2FA |
| `verifyToken` | `checkLogin` + `doubleCheckPassword` | 验证 2FA 令牌 |

### 6.2 2FA 启用流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           2FA 启用流程                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. prepare2FA (准备阶段)                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  socket.on("prepare2FA", async (currentPassword, callback) => {      │  │
│  │      checkLogin(socket);                                               │  │
│  │      await doubleCheckPassword(socket, currentPassword);  // 双重验密 │  │
│  │                                                                       │  │
│  │      // 生成 TOTP 密钥                                                 │  │
│  │      let newSecret = genSecret();                                      │  │
│  │      let encodedSecret = base32.encode(newSecret).replace(/=/g, "");  │  │
│  │      let uri = `otpauth://totp/Uptime%20Kuma:${user.username}?secret=│  │
│  │                ${encodedSecret}`;                                       │  │
│  │                                                                       │  │
│  │      // 暂存密钥到数据库                                                │  │
│  │      await R.exec("UPDATE `user` SET twofa_secret = ? WHERE id = ? ", │  │
│  │          [newSecret, socket.userID]);                                   │  │
│  │                                                                       │  │
│  │      callback({ ok: true, uri: uri });  // 返回二维码链接               │  │
│  │  });                                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  2. save2FA (确认阶段)                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  socket.on("save2FA", async (currentPassword, callback) => {         │  │
│  │      checkLogin(socket);                                               │  │
│  │      await doubleCheckPassword(socket, currentPassword);  // 再次双重验密 │  │
│  │                                                                       │  │
│  │      // 标记 2FA 为已启用                                               │  │
│  │      await R.exec("UPDATE `user` SET twofa_status = 1 WHERE id = ? ", │  │
│  │          [socket.userID]);                                              │  │
│  │                                                                       │  │
│  │      callback({ ok: true, msg: "2faEnabled", msgi18n: true });        │  │
│  │  });                                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 登录时的 2FA 验证

```javascript
// server/server.js:432-510
socket.on("login", async (data, callback) => {
    let user = await login(data.username, data.password);

    if (user) {
        // 场景1：未启用 2FA
        if (user.twofa_status === 0) {
            await afterLogin(socket, user);
            callback({ ok: true, token: User.createJWT(user, server.jwtSecret) });
        }

        // 场景2：已启用 2FA，但未提供 token
        if (user.twofa_status === 1 && !data.token) {
            callback({ tokenRequired: true });  // 告诉前端需要 2FA
        }

        // 场景3：已启用 2FA，且提供了 token
        if (data.token) {
            // 验证 TOTP
            let verify = notp.totp.verify(data.token, user.twofa_secret, twoFAVerifyOptions);
            
            // 防止重放：检查 token 是否与上一次相同
            if (user.twofa_last_token !== data.token && verify) {
                await afterLogin(socket, user);
                
                // 记录本次使用的 token
                await R.exec(
                    "UPDATE `user` SET twofa_last_token = ? WHERE id = ? ",
                    [data.token, socket.userID]
                );
                
                callback({ ok: true, token: User.createJWT(user, server.jwtSecret) });
            } else {
                callback({ ok: false, msg: "authInvalidToken", msgi18n: true });
            }
        }
    }
});
```

---

## 7. 完整的认证链路梳理

### 7.1 登录认证链路

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           登录认证链路                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  前端 (Vue + Socket.io)                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  1. 用户输入账号密码 → 调用 socket.emit("login", {username, password})│  │
│  │                           │                                           │  │
│  │                           ▼                                           │  │
│  │  2. 服务端返回 {tokenRequired: true}  (如果启用了 2FA)               │  │
│  │                           │                                           │  │
│  │                           ▼                                           │  │
│  │  3. 用户输入 2FA 令牌 → 调用 socket.emit("login", {username, password,│  │
│  │                                                    token: "123456"})  │  │
│  │                           │                                           │  │
│  │                           ▼                                           │  │
│  │  4. 服务端返回 {ok: true, token: "jwt_token"}                       │  │
│  │                           │                                           │  │
│  │                           ▼                                           │  │
│  │  5. 前端存储 Token: this.storage().token = res.token                 │  │
│  │     (remember=true → localStorage, 否则 sessionStorage)              │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  服务端 (Node.js)                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  socket.on("login", async (data, callback) => {                      │  │
│  │      // L0: 速率限制                                                  │  │
│  │      if (!(await loginRateLimiter.pass(callback))) return;           │  │
│  │                                                                       │  │
│  │      // L1: 验证账号密码                                              │  │
│  │      let user = await login(data.username, data.password);            │  │
│  │                                                                       │  │
│  │      if (user) {                                                      │  │
│  │          // L2: 检查 2FA 状态                                         │  │
│  │          if (user.twofa_status === 0) {                               │  │
│  │              // 无 2FA：直接登录                                      │  │
│  │              await afterLogin(socket, user);                           │  │
│  │              callback({ ok: true, token: User.createJWT(...) });      │  │
│  │          } else if (!data.token) {                                    │  │
│  │              // 需要 2FA                                               │  │
│  │              callback({ tokenRequired: true });                        │  │
│  │          } else {                                                      │  │
│  │              // L3: 验证 2FA 令牌                                     │  │
│  │              let verify = notp.totp.verify(data.token, user.twofa_secret│  │
│  │                                               twoFAVerifyOptions);     │  │
│  │                                                                       │  │
│  │              // L4: 防止重放攻击                                       │  │
│  │              if (user.twofa_last_token !== data.token && verify) {    │  │
│  │                  await afterLogin(socket, user);                       │  │
│  │                  // 更新 last_token                                    │  │
│  │                  await R.exec("UPDATE `user` SET twofa_last_token = ? │  │
│  │                          WHERE id = ? ", [data.token, socket.userID]);│  │
│  │                                                                       │  │
│  │                  callback({ ok: true, token: User.createJWT(...) });  │  │
│  │              }                                                         │  │
│  │          }                                                             │  │
│  │      }                                                                 │  │
│  │  });                                                                    │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 自动重连认证链路

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        自动重连 (loginByToken) 链路                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  前端触发时机：                                                              │
│  - Socket 断开重连后，服务端发送 "loginRequired" 事件                       │
│  - 前端检查 storage 中是否有 token                                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  // src/mixins/socket.js:137-145                                      │  │
│  │  socket.on("loginRequired", () => {                                   │  │
│  │      let token = this.storage().token;                                 │  │
│  │      if (token && token !== "autoLogin") {                            │  │
│  │          this.loginByToken(token);  // 使用存储的 token 重连           │  │
│  │      } else {                                                          │  │
│  │          this.$root.storage().removeItem("token");                    │  │
│  │          this.allowLoginDialog = true;  // 显示登录框                  │  │
│  │      }                                                                 │  │
│  │  });                                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  服务端验证逻辑：                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  // server/server.js:383-430                                          │  │
│  │  socket.on("loginByToken", async (token, callback) => {              │  │
│  │      try {                                                             │  │
│  │          // L1: 验证 JWT 签名                                          │  │
│  │          let decoded = jwt.verify(token, server.jwtSecret);           │  │
│  │                                                                       │  │
│  │          // L2: 查找用户                                               │  │
│  │          let user = await R.findOne("user", " username = ? AND active │  │
│  │                                      = 1 ", [decoded.username]);       │  │
│  │                                                                       │  │
│  │          if (user) {                                                   │  │
│  │              // L3: 关键！检查密码是否修改                              │  │
│  │              // JWT payload 中的 h 字段是密码的 SHAKE256 哈希          │  │
│  │              if (decoded.h !== shake256(user.password, SHAKE256_LENGTH))│  │
│  │              {                                                         │  │
│  │                  throw new Error("The token is invalid due to password │  │
│  │                              change or old token");                    │  │
│  │              }                                                         │  │
│  │                                                                       │  │
│  │              // L4: 完成登录                                           │  │
│  │              await afterLogin(socket, user);                           │  │
│  │              callback({ ok: true });                                   │  │
│  │          }                                                             │  │
│  │      } catch (error) {                                                 │  │
│  │          callback({ ok: false, msg: "authInvalidToken", msgi18n: true│  │
│  │          });                                                           │  │
│  │      }                                                                 │  │
│  │  });                                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  关键点：                                                                    │
│  - JWT 没有过期时间，通过绑定密码哈希实现"修改密码即失效所有 Token"         │
│  - 这是一种巧妙的设计，无需维护 Token 黑名单                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 各模块保护边界总结

### 8.1 权限检查矩阵

| 模块 | 操作 | L1 checkLogin | L2 归属检查 | L3 双重验密 | 触发条件 |
|-----|------|--------------|------------|------------|---------|
| **监控** | add | ✓ | - | - | 新建操作 |
| | getMonitorList | ✓ | SQL 过滤 | - | 返回时过滤 |
| | getMonitor | ✓ | SQL 条件 | - | `WHERE ... AND user_id = ?` |
| | getMonitorBeats | ✓ | **✗** | - | **隐患：无归属检查** |
| | editMonitor | ✓ | 显式比较 | - | `bean.user_id !== socket.userID` |
| | deleteMonitor | ✓ | SQL 条件 | - | `WHERE ... AND user_id = ?` |
| | resumeMonitor | ✓ | checkOwner | - | 函数内部检查 |
| | pauseMonitor | ✓ | checkOwner | - | 函数内部检查 |
| **监控标签** | addTag | ✓ | - | - | 系统级标签 |
| | editTag | ✓ | - | - | 系统级标签 |
| | deleteTag | ✓ | - | - | 系统级标签 |
| | addMonitorTag | ✓ | **✗** | - | **隐患：无归属检查** |
| | deleteMonitorTag | ✓ | **✗** | - | **隐患：无归属检查** |
| **状态页** | getIncidentHistory (公开) | ✗ | - | - | 公开访问 |
| | getStatusPage (管理) | ✓ | **✗** | - | 无 user_id 字段 |
| | postIncident | ✓ | **✗** | - | 无 user_id 字段 |
| | saveStatusPage | ✓ | **✗** | - | 无 user_id 字段 |
| | addStatusPage | ✓ | **✗** | - | 无 user_id 字段 |
| | deleteStatusPage | ✓ | **✗** | - | 无 user_id 字段 |
| **系统设置** | getSettings | ✓ | - | - | 读取配置 |
| | setSettings (普通) | ✓ | - | - | 修改时区等 |
| | setSettings (禁用认证) | ✓ | - | **✓** | 仅当"启用→禁用"时 |
| **2FA** | prepare2FA | ✓ | - | **✓** | 始终需要 |
| | save2FA | ✓ | - | **✓** | 始终需要 |
| | disable2FA | ✓ | - | **✓** | 始终需要 |
| | verifyToken | ✓ | - | **✓** | 始终需要 |

### 8.2 保护边界设计原则

#### 原则 1：单用户假设

当前 Uptime Kuma 设计上是**单用户系统**：
- 状态页表没有 `user_id` 字段
- 部分监控操作的 `checkOwner` 检查在单用户场景下冗余
- 标签是系统级资源，不属于特定用户

#### 原则 2：高危操作需要双重验密

需要 `doubleCheckPassword` 的操作：
1. **禁用认证** (`disableAuth = true`)：降低系统安全等级
2. **启用 2FA**：提升账户安全等级
3. **禁用 2FA**：降低账户安全等级
4. **验证 2FA 令牌**：确认身份

**共同点**：都是改变安全边界的操作。

#### 原则 3：JWT 绑定密码哈希

```javascript
// server/model/user.js:41-49
static createJWT(user, jwtSecret) {
    return jwt.sign(
        {
            username: user.username,
            h: shake256(user.password, SHAKE256_LENGTH),  // 密码哈希绑定
        },
        jwtSecret
    );
}
```

**优势**：
- 无需维护 Token 黑名单
- 密码修改后，所有历史 Token 自动失效
- 实现简单，无状态

**劣势**：
- JWT 本身没有过期时间
- 如果密码从未修改，Token 永久有效

### 8.3 保护边界差异的影响

#### 对状态页管理的影响

状态页的设计目标是**公开可访问，管理需登录**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    状态页保护边界                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  外部用户 (未登录)                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  ✓ 访问 /status/{slug} 页面                              │  │
│  │  ✓ 查看监控状态 (UP/DOWN)                                 │  │
│  │  ✓ 查看已发布的事件历史                                    │  │
│  │  ✗ 修改任何配置                                           │  │
│  │  ✗ 发布/编辑/删除事件                                     │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  管理员 (已登录)                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  ✓ 所有外部用户权限                                       │  │
│  │  ✓ 创建/编辑/删除状态页                                   │  │
│  │  ✓ 发布/编辑/删除事件                                     │  │
│  │  ✓ 配置状态页外观、监控分组                                │  │
│  │  ⚠️ 无资源归属检查（单用户假设）                           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**多用户架构下需要修改**：
- `status_page` 表添加 `user_id` 字段
- 所有状态页操作添加归属检查
- 状态页可以设置为"私有"或"公开"

#### 对系统设置的影响

系统设置的保护边界**非常精细**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    系统设置保护边界                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  低风险设置（仅需 checkLogin）                                   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  ✓ serverTimezone: 服务器时区                            │  │
│  │  ✓ entryPage: 入口页面                                   │  │
│  │  ✓ chromeExecutable: Chrome 路径                         │  │
│  │  ✓ nscd: NSCD 服务状态                                   │  │
│  │  ✓ tlsExpiryNotifyDays: TLS 过期通知天数                 │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  高风险设置（需要 doubleCheckPassword）                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  ⚠️ disableAuth: 禁用认证（仅当"启用→禁用"时）           │  │
│  │                                                         │  │
│  │  原因：                                                  │  │
│  │  - 禁用后系统完全开放                                    │  │
│  │  - 任何人都可以修改监控配置                              │  │
│  │  - 任何人都可以修改系统设置                              │  │
│  │  - 这是不可逆的安全降级操作                              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  注意："禁用→启用"认证不需要双重验密，因为这是安全升级操作。    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. 关键文件索引

| 功能 | 文件路径 | 关键函数/行号 |
|-----|---------|-------------|
| 基础登录认证 | `server/auth.js:15-34` | `login()` |
| JWT 生成 | `server/model/user.js:41-49` | `createJWT()` |
| JWT Secret 管理 | `server/util-server.js:42-53` | `initJWTSecret()` |
| L1 登录检查 | `server/util-server.js:637-641` | `checkLogin()` |
| L2 归属检查 | `server/server.js:1789-1795` | `checkOwner()` |
| L3 双重验密 | `server/util-server.js:651-663` | `doubleCheckPassword()` |
| Socket 登录处理 | `server/server.js:432-510` | `socket.on("login")` |
| Token 登录处理 | `server/server.js:383-430` | `socket.on("loginByToken")` |
| 登录后处理 | `server/server.js:1804-1818` | `afterLogin()` |
| 系统设置保存 | `server/server.js:1475-1535` | `socket.on("setSettings")` |
| 获取监控详情 | `server/server.js:987-1006` | `socket.on("getMonitor")` |
| 获取监控心跳 | `server/server.js:1029-1062` | `socket.on("getMonitorBeats")` |
| 编辑监控 | `server/server.js:800-900` | `socket.on("editMonitor")` |
| 删除监控 | `server/server.js:1103-1191` | `socket.on("deleteMonitor")` |
| 添加监控标签 | `server/server.js:1282-1307` | `socket.on("addMonitorTag")` |
| 删除监控标签 | `server/server.js:1332-1350` | `socket.on("deleteMonitorTag")` |
| 状态页管理 | `server/socket-handlers/status-page-socket-handler.js` | 多处 `checkLogin()` |
| 2FA 准备 | `server/server.js:526-567` | `socket.on("prepare2FA")` |
| 2FA 保存 | `server/server.js:569-597` | `socket.on("save2FA")` |
| 2FA 禁用 | `server/server.js:599-626` | `socket.on("disable2FA")` |
| 前端 Socket 管理 | `src/mixins/socket.js` | `login()`, `loginByToken()` |

---

## 10. 发现的潜在问题

### 10.1 监控操作权限检查不一致

| 操作 | 是否有归属检查 | 风险等级 |
|-----|--------------|---------|
| `getMonitor` | ✓ (SQL 条件) | 低 |
| `editMonitor` | ✓ (显式比较) | 低 |
| `deleteMonitor` | ✓ (SQL 条件) | 低 |
| `resumeMonitor` | ✓ (checkOwner) | 低 |
| `pauseMonitor` | ✓ (checkOwner) | 低 |
| `getMonitorBeats` | **✗** | **中** |
| `addMonitorTag` | **✗** | **中** |
| `deleteMonitorTag` | **✗** | **中** |

### 10.2 建议

在当前单用户架构下，这些问题不会实际触发风险。但如果未来转向多用户架构，建议：

1. **统一监控操作的归属检查**：
   - `getMonitorBeats` 添加 `user_id` 条件或调用 `checkOwner`
   - `addMonitorTag`/`deleteMonitorTag` 添加归属检查

2. **状态页添加用户归属**：
   - `status_page` 表添加 `user_id` 字段
   - 所有状态页操作添加归属检查

3. **考虑 JWT 过期时间**：
   - 当前 JWT 永久有效（只要密码不修改）
   - 可以考虑添加 `exp`  claim，实现 Token 自动过期

---

## 11. 总结

### 11.1 核心设计理念

Uptime Kuma 的认证权限设计围绕**单用户假设**展开：

1. **JWT 无状态会话**：简化部署，支持水平扩展
2. **密码哈希绑定**：实现"修改密码即失效所有 Token"
3. **三层权限检查**：`checkLogin` → `checkOwner` → `doubleCheckPassword`
4. **精细的双重验密**：仅在改变安全边界时触发

### 11.2 关键澄清

1. **系统设置的双重验密条件**：
   - **只有一种场景需要**：当前认证已启用，且用户想要禁用认证
   - 其他系统设置修改只需要 `checkLogin`

2. **监控操作的权限差异**：
   - **资源归属校验**：`getMonitor`, `editMonitor`, `deleteMonitor`, `resumeMonitor`, `pauseMonitor`
   - **仅登录校验**：`getMonitorBeats`, `addMonitorTag`, `deleteMonitorTag`（多用户场景隐患）

3. **状态页的保护边界**：
   - 公开读取：无需登录
   - 管理操作：仅需 `checkLogin`
   - 无资源归属检查：因为 `status_page` 表没有 `user_id` 字段（单用户设计）

### 11.3 认证链路与 Socket 鉴权的关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Login Session 与 Socket 鉴权的关系                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  概念澄清：                                                                  │
│  - Uptime Kuma 没有传统的服务器端 Session                                    │
│  - "Session" 在 Uptime Kuma 中 = JWT Token + socket.userID                 │
│                                                                             │
│  关系图示：                                                                  │
│                                                                             │
│  首次登录：                                                                  │
│  ┌──────────┐    login(username, password)     ┌──────────┐               │
│  │  前端    │ ───────────────────────────────► │  服务端  │               │
│  │          │                                  │          │               │
│  │          │ ◄──── JWT Token + 完成登录 ──── │          │               │
│  │          │      (afterLogin 设置 userID)    │          │               │
│  └──────────┘                                  └──────────┘               │
│       │                                              │                     │
│       │ 存储到 localStorage/sessionStorage           │                     │
│       ▼                                              │                     │
│  ┌──────────────┐                                    │                     │
│  │  JWT Token   │                                    │                     │
│  │  - username  │                                    │                     │
│  │  - h (密码哈希)│◄─────────────────────────────────┘                     │
│  │  - 签名      │                                                          │
│  └──────────────┘                                                          │
│                                                                             │
│  Socket 重连：                                                              │
│  ┌──────────┐    loginByToken(token)         ┌──────────┐               │
│  │  前端    │ ───────────────────────────────► │  服务端  │               │
│  │          │                                  │          │               │
│  │          │     验证步骤：                    │          │               │
│  │          │     1. JWT 签名验证              │          │               │
│  │          │     2. 用户状态检查 (active=1)   │          │               │
│  │          │     3. 密码哈希匹配 (h 字段)     │          │               │
│  │          │     4. afterLogin 设置 userID    │          │               │
│  │          │                                  │          │               │
│  │          │ ◄──────── {ok: true} ─────────── │          │               │
│  └──────────┘                                  └──────────┘               │
│                                                                             │
│  关键理解：                                                                  │
│  - Socket 连接建立 ≠ 已鉴权                                                 │
│  - 每次 Socket 重连都需要重新调用 loginByToken                              │
│  - socket.userID 是所有权限检查的核心依据                                   │
│  - JWT 中的 h 字段实现了"密码修改即失效所有 Token"                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
