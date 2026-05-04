# Uptime Kuma 认证、会话、2FA 和权限分析

## 1. 整体架构概览

Uptime Kuma 采用 **JWT (JSON Web Token)** 作为会话载体，配合 **Socket.io** 进行实时通信。所有敏感操作都通过 Socket 事件进行，并在服务端进行统一的权限检查。

```
┌─────────────┐         ┌──────────────────┐         ┌───────────────┐
│   前端      │         │   Socket.io      │         │   服务端      │
│ (Vue + JS)  │◄───────►│   通信层         │◄───────►│  (Node.js)    │
└─────────────┘         └──────────────────┘         └───────────────┘
       │                        │                           │
       │ 1. 登录 (username/pw)  │                           │
       │────────────────────────►                           │
       │                        │ 验证                       │
       │                        │─────────►                  │
       │                        │    JWT Token ◄────────────│
       │    JWT Token ◄─────────────────────────────────────│
       │                        │                           │
       │ 2. 后续操作 (带 Token) │                           │
       │────────────────────────►                           │
       │                        │ checkLogin(socket)        │
       │                        │─────────►                  │
       │                        │    checkOwner() (如有)    │
       │                        │    执行操作 ◄─────────────│
       │    结果 ◄──────────────────────────────────────────│
```

---

## 2. 登录认证机制

### 2.1 核心登录逻辑

**文件位置**: `server/auth.js:15-34`

```javascript
exports.login = async function (username, password) {
    if (typeof username !== "string" || typeof password !== "string") {
        return null;
    }

    let user = await R.findOne("user", "TRIM(username) = ? AND active = 1 ", [username.trim()]);

    if (user && passwordHash.verify(password, user.password)) {
        // 自动升级旧哈希到 bcrypt
        if (passwordHash.needRehash(user.password)) {
            await R.exec("UPDATE `user` SET password = ? WHERE id = ? ", [
                await passwordHash.generate(password),
                user.id,
            ]);
        }
        return user;
    }
    return null;
};
```

**关键点**:
- 只允许 `active = 1` 的用户登录
- 密码使用 **bcrypt** 哈希存储
- 支持自动升级旧版本哈希格式

### 2.2 登录速率限制

**文件位置**: `server/auth.js:106-123`

```javascript
function userAuthorizer(username, password, callback) {
    loginRateLimiter.pass(null, 0).then((pass) => {
        if (pass) {
            exports.login(username, password).then((user) => {
                callback(null, user != null);
                if (user == null) {
                    loginRateLimiter.removeTokens(1);  // 失败消耗 token
                }
            });
        } else {
            callback(null, false);  // 速率限制触发
        }
    });
}
```

---

## 3. JWT Session 机制

### 3.1 JWT Secret 管理

**文件位置**: `server/util-server.js:42-53`

```javascript
exports.initJWTSecret = async () => {
    let jwtSecretBean = await R.findOne("setting", " `key` = ? ", ["jwtSecret"]);

    if (!jwtSecretBean) {
        jwtSecretBean = R.dispense("setting");
        jwtSecretBean.key = "jwtSecret";
    }

    jwtSecretBean.value = await passwordHash.generate(genSecret());
    await R.store(jwtSecretBean);
    return jwtSecretBean;
};
```

**说明**:
- JWT Secret 存储在数据库 `setting` 表中
- 启动时初始化，保证集群环境一致性

### 3.2 JWT Token 生成

**文件位置**: `server/model/user.js:41-49`

```javascript
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

**重要设计**:
- Token Payload 包含 `h` 字段，是密码的 SHAKE256 哈希
- **密码修改后，所有旧 Token 自动失效**（因为 `h` 不匹配）

### 3.3 前端 Token 存储

**文件位置**: `src/mixins/socket.js:325-327`

```javascript
storage() {
    return this.remember ? localStorage : sessionStorage;
}
```

**说明**:
- "记住我" 勾选时，Token 存在 `localStorage`（持久化）
- 未勾选时，Token 存在 `sessionStorage`（会话级）

---

## 4. Socket 鉴权机制

### 4.1 鉴权入口：Login & LoginByToken

**文件位置**: `server/server.js:383-510`

#### 流程1：用户名密码登录

```javascript
socket.on("login", async (data, callback) => {
    // 1. 速率限制检查
    if (!(await loginRateLimiter.pass(callback))) {
        return;
    }

    // 2. 验证用户名密码
    let user = await login(data.username, data.password);

    if (user) {
        // 3. 检查 2FA 状态
        if (user.twofa_status === 0) {
            // 无 2FA：直接登录
            await afterLogin(socket, user);
            callback({
                ok: true,
                token: User.createJWT(user, server.jwtSecret),
            });
        }

        if (user.twofa_status === 1 && !data.token) {
            // 需要 2FA：返回 tokenRequired
            callback({ tokenRequired: true });
        }

        if (data.token) {
            // 验证 TOTP Token
            let verify = notp.totp.verify(data.token, user.twofa_secret, twoFAVerifyOptions);
            if (verify) {
                await afterLogin(socket, user);
                callback({
                    ok: true,
                    token: User.createJWT(user, server.jwtSecret),
                });
            }
        }
    }
});
```

#### 流程2：Token 登录（自动重连）

**文件位置**: `server/server.js:383-430`

```javascript
socket.on("loginByToken", async (token, callback) => {
    try {
        // 1. 验证 JWT 签名
        let decoded = jwt.verify(token, server.jwtSecret);

        // 2. 查找用户
        let user = await R.findOne("user", " username = ? AND active = 1 ", [decoded.username]);

        if (user) {
            // 3. 验证密码哈希绑定（防止旧 Token）
            if (decoded.h !== shake256(user.password, SHAKE256_LENGTH)) {
                throw new Error("The token is invalid due to password change or old token");
            }

            // 4. 完成登录
            await afterLogin(socket, user);
            callback({ ok: true });
        }
    } catch (error) {
        callback({ ok: false, msg: "authInvalidToken" });
    }
});
```

### 4.2 登录完成处理：afterLogin

**文件位置**: `server/server.js:1804-1818`

```javascript
async function afterLogin(socket, user) {
    socket.userID = user.id;           // 关键：设置 socket 级别的用户标识
    socket.join(user.id);              // 加入用户房间（用于广播）

    // 发送初始化数据
    let monitorList = await server.sendMonitorList(socket);
    await Promise.allSettled([
        sendInfo(socket),
        server.sendMaintenanceList(socket),
        sendNotificationList(socket),
        // ... 其他初始化数据
    ]);
}
```

**核心机制**:
- `socket.userID` 是后续所有权限检查的依据
- 加入用户房间 `socket.join(user.id)`，支持向同一用户的所有连接广播

### 4.3 权限检查：checkLogin

**文件位置**: `server/util-server.js:637-641`

```javascript
exports.checkLogin = (socket) => {
    if (!socket.userID) {
        throw new Error("You are not logged in.");
    }
};
```

**这是最核心的权限守卫**，所有需要认证的 Socket 事件处理函数第一行都调用它：

```javascript
socket.on("add", async (monitor, callback) => {
    try {
        checkLogin(socket);  // 第一行：检查登录
        // ... 后续操作
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

---

## 5. 二次验证 (2FA) 机制

### 5.1 2FA 状态存储

用户表 `user` 中的 2FA 相关字段：
- `twofa_status`: 0=未启用, 1=已启用
- `twofa_secret`: TOTP 密钥
- `twofa_last_token`: 上一次使用的 token（防止重放）

### 5.2 启用 2FA 流程

**文件位置**: `server/server.js:526-597`

```javascript
// 步骤1：准备 2FA（生成密钥）
socket.on("prepare2FA", async (currentPassword, callback) => {
    checkLogin(socket);
    await doubleCheckPassword(socket, currentPassword);  // 双重密码验证

    let user = await R.findOne("user", " id = ? AND active = 1 ", [socket.userID]);

    if (user.twofa_status === 0) {
        let newSecret = genSecret();
        let encodedSecret = base32.encode(newSecret).replace(/=/g, "");
        let uri = `otpauth://totp/Uptime%20Kuma:${user.username}?secret=${encodedSecret}`;

        await R.exec("UPDATE `user` SET twofa_secret = ? WHERE id = ? ", [newSecret, socket.userID]);
        callback({ ok: true, uri: uri });  // 返回二维码链接
    }
});

// 步骤2：确认启用 2FA
socket.on("save2FA", async (currentPassword, callback) => {
    checkLogin(socket);
    await doubleCheckPassword(socket, currentPassword);

    await R.exec("UPDATE `user` SET twofa_status = 1 WHERE id = ? ", [socket.userID]);
    callback({ ok: true, msg: "2faEnabled" });
});
```

### 5.3 禁用 2FA

**文件位置**: `server/server.js:599-626`

```javascript
socket.on("disable2FA", async (currentPassword, callback) => {
    checkLogin(socket);
    await doubleCheckPassword(socket, currentPassword);  // 必须验证当前密码
    await TwoFA.disable2FA(socket.userID);  // 设置 twofa_status = 0
    callback({ ok: true, msg: "2faDisabled" });
});
```

### 5.4 双重密码验证

**文件位置**: `server/util-server.js:651-663`

```javascript
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

**适用场景**:
- 启用/禁用 2FA
- 修改系统设置（部分敏感项）
- 修改密码

---

## 6. 各功能模块的保护方式

### 6.1 监控配置保护

**文件位置**: `server/server.js:724-1535`

#### 6.1.1 基础权限检查

所有监控相关操作都首先调用 `checkLogin(socket)`：

| Socket 事件 | 权限检查 | 说明 |
|------------|---------|------|
| `add` | checkLogin | 添加监控 |
| `getMonitorList` | checkLogin | 获取监控列表 |
| `getMonitor` | checkLogin + checkOwner | 获取单个监控详情 |
| `deleteMonitor` | checkLogin + checkOwner | 删除监控 |
| `addTag` | checkLogin | 添加标签 |
| `addMonitorTag` | checkLogin | 给监控加标签 |

#### 6.1.2 所有权检查：checkOwner

**文件位置**: `server/server.js:1789-1795`

```javascript
async function checkOwner(userID, monitorID) {
    let row = await R.getRow("SELECT id FROM monitor WHERE id = ? AND user_id = ? ", [monitorID, userID]);

    if (!row) {
        throw new Error("You do not own this monitor.");
    }
}
```

**使用示例** (`getMonitor`):

```javascript
socket.on("getMonitor", async (monitorID, callback) => {
    try {
        checkLogin(socket);
        await checkOwner(socket.userID, monitorID);  // 额外检查所有权

        let bean = await R.findOne("monitor", " id = ? ", [monitorID]);
        // ... 返回监控数据
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

### 6.2 状态页管理保护

**文件位置**: `server/socket-handlers/status-page-socket-handler.js`

#### 6.2.1 写入操作（需登录）

| Socket 事件 | 权限检查 | 说明 |
|------------|---------|------|
| `postIncident` | checkLogin | 发布/编辑事件 |
| `unpinIncident` | checkLogin | 取消置顶事件 |
| `editIncident` | checkLogin | 编辑事件 |
| `deleteIncident` | checkLogin | 删除事件 |
| `resolveIncident` | checkLogin | 标记事件已解决 |
| `getStatusPage` | checkLogin | 获取状态页配置（编辑用） |
| `saveStatusPage` | checkLogin | 保存状态页配置 |
| `addStatusPage` | checkLogin | 新建状态页 |
| `deleteStatusPage` | checkLogin | 删除状态页 |

#### 6.2.2 读取操作（公开访问）

```javascript
socket.on("getIncidentHistory", async (slug, cursor, callback) => {
    try {
        let statusPageID = await StatusPage.slugToID(slug);
        const isPublic = !socket.userID;  // 未登录用户也可以访问
        const result = await StatusPage.getIncidentHistory(statusPageID, cursor, isPublic);
        callback({ ok: true, ...result });
    } catch (error) {
        // ...
    }
});
```

**设计说明**:
- 状态页的**公开展示**部分不需要登录
- 状态页的**管理操作**（创建、编辑、删除、发布事件）都需要 `checkLogin`

### 6.3 系统设置保护

**文件位置**: `server/server.js:1454-1535`, `server/socket-handlers/database-socket-handler.js`

#### 6.3.1 读取设置

```javascript
socket.on("getSettings", async (callback) => {
    try {
        checkLogin(socket);
        const data = await getSettings("general");
        // ...
        callback({ ok: true, data });
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

#### 6.3.2 保存设置（双重验证）

```javascript
socket.on("setSettings", async (data, currentPassword, callback) => {
    try {
        checkLogin(socket);

        // 只有未禁用认证时才需要双重密码验证
        if (!(await Settings.get("disableAuth"))) {
            await doubleCheckPassword(socket, currentPassword);
        }

        await setSettings("general", data);
        callback({ ok: true });
    } catch (e) {
        callback({ ok: false, msg: e.message });
    }
});
```

#### 6.3.3 数据库操作

**文件位置**: `server/socket-handlers/database-socket-handler.js`

```javascript
socket.on("getDatabaseSize", async (callback) => {
    try {
        checkLogin(socket);
        callback({ ok: true, size: await Database.getSize() });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});

socket.on("shrinkDatabase", async (callback) => {
    try {
        checkLogin(socket);
        await Database.shrink();
        callback({ ok: true });
    } catch (error) {
        callback({ ok: false, msg: error.message });
    }
});
```

---

## 7. Login Session 与 Socket 鉴权的关系

### 7.1 核心关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           前端 (Vue + Socket.io Client)                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────┐      ┌──────────────────┐      ┌──────────────────┐  │
│  │  登录表单      │      │  localStorage/   │      │   Socket.io      │  │
│  │              │      │  sessionStorage  │      │    连接          │  │
│  │ username/pw  │      │                  │      │                  │  │
│  └──────┬───────┘      └────────┬─────────┘      └────────┬─────────┘  │
│         │                        │                         │            │
│         │ 1. 发送 login 事件     │                         │            │
│         ├────────────────────────┼─────────────────────────►            │
│         │                        │                         │            │
│         │                        │  2. 接收 JWT Token      │            │
│         │◄───────────────────────┼─────────────────────────┤            │
│         │                        │                         │            │
│         │                        │ 3. 存储 Token           │            │
│         │                        ├───────────────►         │            │
│         │                        │                         │            │
│         │                        │ 4. 后续请求携带 Token   │            │
│         │                        │                         │            │
│  ┌──────▼───────┐      ┌────────▼─────────┐      ┌────────▼─────────┐  │
│  │ 记住我勾选    │─────►│ 存储位置选择     │      │ 自动重连时       │  │
│  │              │      │                  │      │ loginByToken     │  │
│  │ ✓ = localStorage    │ ✗ = sessionStorage│      │                  │  │
│  └──────────────┘      └──────────────────┘      └──────────────────┘  │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Socket.io 通信
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           服务端 (Node.js + Socket.io Server)             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    公共 Socket API (无需鉴权)                       │  │
│  │  ┌──────────┐  ┌──────────────┐  ┌────────────────────────────┐  │  │
│  │  │  login   │  │ loginByToken │  │         logout             │  │  │
│  │  │ (用户密码)│  │  (JWT Token) │  │       (退出登录)           │  │  │
│  │  └────┬─────┘  └──────┬───────┘  └────────────┬───────────────┘  │  │
│  │       │               │                        │                   │  │
│  │       └───────────────┼────────────────────────┘                   │  │
│  │                       ▼                                             │  │
│  │              ┌─────────────────┐                                    │  │
│  │              │   afterLogin()  │                                    │  │
│  │              │  设置 socket.userID │                                    │  │
│  │              └────────┬────────┘                                    │  │
│  │                       │                                             │  │
│  └───────────────────────┼─────────────────────────────────────────────┘  │
│                          ▼                                                  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    认证后 API (需要 checkLogin)                     │  │
│  │                                                                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐  │  │
│  │  │  监控操作    │  │  状态页管理  │  │      系统设置            │  │  │
│  │  │ add/delete  │  │ saveStatus  │  │ getSettings/setSettings  │  │  │
│  │  │ getMonitor  │  │ postIncident│  │  (部分需 doubleCheck)    │  │  │
│  │  │  +checkOwner│  │             │  │                          │  │  │
│  │  └─────────────┘  └─────────────┘  └──────────────────────────┘  │  │
│  │                                                                      │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 关键概念澄清

#### 1. "Session" 在 Uptime Kuma 中的含义

Uptime Kuma **没有使用传统的服务器端 Session（如 express-session）**，而是使用 **JWT (无状态 Token)** 作为会话标识。

| 传统 Session | Uptime Kuma JWT |
|-------------|-----------------|
| 服务器存储 session data | 无状态，Token 自包含 |
| Session ID 存在 Cookie | JWT 存在 localStorage/sessionStorage |
| 每次请求查 Session 存储 | 每次请求验证 JWT 签名 |

#### 2. Socket 连接与鉴权的关系

**Socket 连接 ≠ 已鉴权**

```javascript
// 前端 socket.js 中的流程
socket.on("loginRequired", () => {
    let token = this.storage().token;
    if (token && token !== "autoLogin") {
        this.loginByToken(token);  // 必须显式调用鉴权
    } else {
        this.allowLoginDialog = true;  // 显示登录框
    }
});
```

**时序说明**:
1. Socket.io 连接建立成功 → **此时 socket 是未鉴权状态**
2. 服务端发送 `loginRequired` 事件（或前端检测需要登录）
3. 前端调用 `loginByToken(token)` 进行鉴权
4. 服务端验证通过后设置 `socket.userID`
5. 此后的操作才能通过 `checkLogin` 检查

#### 3. Token 失效机制

Uptime Kuma 的 JWT **没有设置过期时间**，但有以下失效机制：

| 失效场景 | 触发条件 | 实现方式 |
|---------|---------|---------|
| 密码修改 | 用户修改密码 | JWT Payload 中的 `h` 字段与新密码哈希不匹配 |
| 重新生成 JWT Secret | 数据库中 jwtSecret 被重置 | JWT 签名验证失败 |
| 用户被禁用 | `user.active = 0` | 登录查询时过滤 `active = 1` |
| 手动登出 | 用户点击退出 | 前端删除 Token + 服务端清除 `socket.userID` |

### 7.3 完整的认证生命周期

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           认证生命周期                                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [阶段1: 首次登录]                                                          │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐           │
│  │ 用户输入 │────►│ 服务端  │────►│ 生成    │────►│ 返回    │           │
│  │ 账号密码 │     │ 验证通过 │     │  JWT   │     │  Token  │           │
│  └─────────┘     └─────────┘     └─────────┘     └────┬────┘           │
│                                                          │                 │
│                                                          ▼                 │
│  [阶段2: Token 存储]                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  remember = true  ──►  localStorage  (关闭浏览器后保留)               │ │
│  │  remember = false ──►  sessionStorage (关闭浏览器后清除)              │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                          │                 │
│                                                          ▼                 │
│  [阶段3: Socket 重连鉴权]                                                    │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐           │
│  │ 断开重连 │────►│ 发送    │────►│ 验证    │────►│ 设置    │           │
│  │         │     │loginBy- │     │  JWT   │     │socket.  │           │
│  │         │     │  Token  │     │  + h    │     │  userID │           │
│  └─────────┘     └─────────┘     └─────────┘     └────┬────┘           │
│                                                          │                 │
│                                                          ▼                 │
│  [阶段4: 正常操作]                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  每次 Socket 事件:                                                    │ │
│  │  1. checkLogin(socket)  ──►  检查 socket.userID 是否存在            │ │
│  │  2. checkOwner(...)     ──►  监控操作额外检查所有权                   │ │
│  │  3. doubleCheckPassword ──►  敏感操作额外验证当前密码                 │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                          │                 │
│                                                          ▼                 │
│  [阶段5: 登出 / Token 失效]                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  主动登出: logout() 事件                                              │ │
│  │    - 前端: 删除 storage 中的 token                                   │ │
│  │    - 后端: socket.leave(userID), socket.userID = null              │ │
│  │                                                                       │ │
│  │  被动失效:                                                            │ │
│  │    - 密码修改: h 字段不匹配                                          │ │
│  │    - JWT Secret 重置: 签名验证失败                                   │ │
│  │    - 用户被禁用: active = 0                                          │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 关键文件索引

| 功能 | 文件路径 | 关键函数/行号 |
|-----|---------|-------------|
| 登录认证 | `server/auth.js:15-34` | `login()` |
| JWT 生成 | `server/model/user.js:41-49` | `createJWT()` |
| JWT Secret 管理 | `server/util-server.js:42-53` | `initJWTSecret()` |
| 登录检查 | `server/util-server.js:637-641` | `checkLogin()` |
| 双重密码验证 | `server/util-server.js:651-663` | `doubleCheckPassword()` |
| Socket 登录处理 | `server/server.js:432-510` | `socket.on("login")` |
| Token 登录处理 | `server/server.js:383-430` | `socket.on("loginByToken")` |
| 登录后处理 | `server/server.js:1804-1818` | `afterLogin()` |
| 监控所有权检查 | `server/server.js:1789-1795` | `checkOwner()` |
| 2FA 准备 | `server/server.js:526-567` | `socket.on("prepare2FA")` |
| 2FA 保存 | `server/server.js:569-597` | `socket.on("save2FA")` |
| 2FA 禁用 | `server/server.js:599-626` | `socket.on("disable2FA")` |
| 状态页管理 | `server/socket-handlers/status-page-socket-handler.js` | `checkLogin()` 多处 |
| 数据库操作 | `server/socket-handlers/database-socket-handler.js` | `checkLogin()` |
| 前端 Socket 管理 | `src/mixins/socket.js` | `login()`, `loginByToken()` |

---

## 9. 安全设计总结

### 9.1 多层防护机制

```
┌─────────────────────────────────────────────────────────────────┐
│                        安全防护层级                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Layer 1: 网络层 (Rate Limiter)                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  loginRateLimiter: 防止暴力破解密码                       │   │
│  │  twoFaRateLimiter: 防止暴力破解 2FA Token                │   │
│  │  apiRateLimiter: 限制 API 调用频率                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                              ▼                                    │
│  Layer 2: 认证层 (Authentication)                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  用户名 + 密码验证 (bcrypt)                               │   │
│  │  可选: 2FA TOTP 验证 (notp)                              │   │
│  │  JWT 签名验证 (jsonwebtoken)                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                              ▼                                    │
│  Layer 3: 会话层 (Session Binding)                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  JWT 绑定密码哈希 (h 字段)                                 │   │
│  │  密码修改后所有旧 Token 自动失效                           │   │
│  │  可选: disableAuth 模式（仅内网测试）                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                              ▼                                    │
│  Layer 4: 权限层 (Authorization)                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  checkLogin: 所有敏感操作入口检查                          │   │
│  │  checkOwner: 监控操作所有权检查                            │   │
│  │  doubleCheckPassword: 敏感操作二次密码验证                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                              ▼                                    │
│  Layer 5: 数据层 (Data Integrity)                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  密码 bcrypt 哈希存储                                      │   │
│  │  2FA Secret 单独存储                                      │   │
│  │  JWT Secret 数据库存储（集群一致性）                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 设计亮点

1. **JWT 绑定密码哈希**: 密码修改后所有会话自动失效，无需维护 Token 黑名单
2. **无状态设计**: 水平扩展友好，JWT Secret 统一存储在数据库
3. **2FA 重放防护**: `twofa_last_token` 防止同一个 TOTP Token 被重复使用
4. **精细粒度权限控制**: `checkLogin` + `checkOwner` + `doubleCheckPassword` 三层防护
5. **速率限制**: 多维度限流保护（登录、2FA、API）

### 9.3 注意事项

- JWT **没有过期时间**，依赖密码修改或 Secret 重置来失效
- `disableAuth` 模式下完全跳过认证，**仅限内网测试环境使用**
- 当前设计是**单用户系统**（所有监控属于同一用户），`checkOwner` 是为未来多用户预留
