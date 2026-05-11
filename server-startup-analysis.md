# Uptime Kuma 服务器启动流程与环境配置分析报告

## 1. 概述

本报告基于代码真实逻辑，详细分析 Uptime Kuma 应用的启动流程、环境配置加载机制以及不同运行模式下的初始化分叉点。

## 2. 启动入口与基础初始化

### 2.1 启动入口

Uptime Kuma 的服务器端入口文件是 `server/server.js`。所有部署方式最终都调用 `node server/server.js`：

| 启动方式 | 实际执行命令 |
|---------|-------------|
| `npm run start` | `node server/server.js` |
| `npm run start-server-dev` | `cross-env NODE_ENV=development node server/server.js` |
| `npm run start-server-dev:watch` | `cross-env NODE_ENV=development node --watch server/server.js` |
| `pm2 start server/server.js` | `node server/server.js`（PM2 包装） |
| Docker (`docker/dockerfile:42`) | `node server/server.js` |

**关键发现**：从代码层面看，所有部署方式的入口都是同一个文件，差异主要体现在**环境变量设置**上。

### 2.2 基础初始化流程 (`server/server.js:1-160`)

```
1. 加载 dotenv → 读取 .env 文件
2. 检查 Node.js 版本 → 禁止不兼容版本
3. 解析命令行参数 (args-parser)
4. 设置默认环境变量:
   - NODE_ENV 未设置 → 设为 "production" (server.js:57-59)
   - UPTIME_KUMA_WS_ORIGIN_CHECK 未设置 → 设为 "cors-like" (server.js:61-63)
5. 创建 UptimeKumaServer 单例
```

## 3. 环境配置加载机制

### 3.1 配置加载优先级

```
命令行参数 (args-parser) > 环境变量 (process.env) > 默认值
```

### 3.2 核心配置 `server/config.js`

#### 3.2.1 主机和端口

```javascript
// config.js:9-14
let hostEnv = isFreeBSD ? null : process.env.HOST;
const hostname = args.host || process.env.UPTIME_KUMA_HOST || hostEnv;

const port = [args.port, process.env.UPTIME_KUMA_PORT, process.env.PORT, 3001]
    .map((portValue) => parseInt(portValue))
    .find((portValue) => !isNaN(portValue));
```

**优先级**：
- 主机: `args.host` > `UPTIME_KUMA_HOST` > `HOST` (非 FreeBSD)
- 端口: `args.port` > `UPTIME_KUMA_PORT` > `PORT` > 默认 3001

#### 3.2.2 SSL/TLS

```javascript
// config.js:16-24
const sslKey = args["ssl-key"] || process.env.UPTIME_KUMA_SSL_KEY || process.env.SSL_KEY;
const sslCert = args["ssl-cert"] || process.env.UPTIME_KUMA_SSL_CERT || process.env.SSL_CERT;
const sslKeyPassphrase = args["ssl-key-passphrase"] || process.env.UPTIME_KUMA_SSL_KEY_PASSPHRASE || process.env.SSL_KEY_PASSPHRASE;
const isSSL = sslKey && sslCert;
```

### 3.3 数据目录

```javascript
// database.js:137
Database.dataDir = process.env.DATA_DIR || args["data-dir"] || "./data/";
```

自动创建子目录：`upload/`, `screenshots/`, `docker-tls/`, `mariadb/`（嵌入式）

### 3.4 主要环境变量

| 环境变量 | 说明 | 默认值 | 使用位置 |
|---------|------|-------|---------|
| `NODE_ENV` | 运行模式 | `production` | 全局 |
| `DATA_DIR` | 数据目录 | `./data/` | `database.js:137` |
| `UPTIME_KUMA_HOST` | 监听主机 | 空 | `config.js:10` |
| `UPTIME_KUMA_PORT` | 监听端口 | `3001` | `config.js:12` |
| `UPTIME_KUMA_SSL_KEY` | SSL 密钥 | 空 | `config.js:16` |
| `UPTIME_KUMA_SSL_CERT` | SSL 证书 | 空 | `config.js:17` |
| `UPTIME_KUMA_DB_TYPE` | 数据库类型 | 空（首次需配置） | `setup-database.js:98` |
| `UPTIME_KUMA_DB_HOSTNAME` | 数据库主机 | 空 | `setup-database.js:102` |
| `UPTIME_KUMA_DB_PORT` | 数据库端口 | 空 | `setup-database.js:103` |
| `UPTIME_KUMA_DB_NAME` | 数据库名 | 空 | `setup-database.js:104` |
| `UPTIME_KUMA_DB_USERNAME` | 数据库用户 | 空 | `setup-database.js:105` |
| `UPTIME_KUMA_DB_PASSWORD` | 数据库密码 | 空 | `setup-database.js:106` |
| `UPTIME_KUMA_DB_SOCKET` | 数据库 Socket | 空 | `setup-database.js:107` |
| `UPTIME_KUMA_DB_SSL` | 数据库 SSL | `false` | `setup-database.js:108` |
| `UPTIME_KUMA_DB_CA` | 数据库 CA | 空 | `setup-database.js:109` |
| `UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB` | 嵌入式 MariaDB | `false` | `setup-database.js:127` |
| `UPTIME_KUMA_DB_POOL_MAX_CONNECTIONS` | 连接池大小 | `10` | `database.js:225` |
| `UPTIME_KUMA_SQLITE_SINGLE_CONNECTION` | SQLite 单连接 | `true` | `database.js:275` |
| `UPTIME_KUMA_IS_CONTAINER` | 容器标识（判断方式不一致） | 未设置 (`undefined`) | 详见 6.3 节 |
| `UPTIME_KUMA_WS_ORIGIN_CHECK` | WS 源检查 | `cors-like` | `server.js:62` |
| `UPTIME_KUMA_DISABLE_FRAME_SAMEORIGIN` | 禁用 X-Frame | `false` | `server.js:147` |
| `UPTIME_KUMA_CLOUDFLARED_TOKEN` | Cloudflared | 空 | `server.js:148` |
| `TZ` | 时区 | 系统默认 | `uptime-kuma-server.js:206` |
| `SQL_LOG` | SQL 日志 | `false` | `database.js:390` |
| `SET_MIGRATE_AGGREGATE_TABLE_TO_TRUE` | 跳过聚合表迁移 | `false` | `database.js:823` |

### 3.5 Docker Secrets 支持

`server/setup-database.js:19-36` 的 `getEnvOrFile` 函数支持通过 `_FILE` 后缀从 Docker Secrets 读取敏感配置：

- `UPTIME_KUMA_DB_USERNAME` / `UPTIME_KUMA_DB_USERNAME_FILE`
- `UPTIME_KUMA_DB_PASSWORD` / `UPTIME_KUMA_DB_PASSWORD_FILE`
- `UPTIME_KUMA_DB_SSL` / `UPTIME_KUMA_DB_SSL_FILE`
- `UPTIME_KUMA_DB_CA` / `UPTIME_KUMA_DB_CA_FILE`

## 4. 数据库配置流程

### 4.1 初始化流程 (`server/server.js:212-229`)

```javascript
(async () => {
    // 1. 初始化数据目录
    Database.initDataDir(args);

    // 2. 检查是否需要数据库配置
    let setupDatabase = new SetupDatabase(args, server);
    if (setupDatabase.isNeedSetup()) {
        await setupDatabase.start(hostname, port); // 启动配置页面
    }

    // 3. 连接数据库
    await initDatabase(testMode);
})();
```

### 4.2 SetupDatabase 配置决策 (`setup-database.js:66-112`)

```
构造 SetupDatabase 实例
    │
    ├─ 尝试读取 data/db-config.json
    │   ├─ 成功 → needSetup = false
    │   └─ 失败
    │       ├─ 存在 data/kuma.db (v1.x 遗留)
    │       │   └─ 自动创建 {type: "sqlite"}，needSetup = false
    │       └─ 不存在 kuma.db
    │           └─ needSetup = true
    │
    └─ 检查 UPTIME_KUMA_DB_TYPE 环境变量
        └─ 已设置 → 从环境变量读取所有 DB 配置，写入 db-config.json，needSetup = false
```

**关键结论**：
- `UPTIME_KUMA_DB_TYPE` 环境变量可以**绕过首次配置页面**，直接初始化数据库
- v1.x 用户的 `kuma.db` 会被自动识别并迁移

### 4.3 数据库配置流程分叉图

```
启动服务器
    │
    ▼
初始化数据目录 (DATA_DIR)
    │
    ▼
检查 db-config.json
    │
    ├─ 存在且有效 ─────────────┐
    │                          │
    ├─ 不存在或无效             │
    │   ├─ 存在 kuma.db (v1.x)  │
    │   │   └─ 自动创建 sqlite 配置
    │   └─ 无 kuma.db           │
    │       └─ needSetup = true │
    │                          │
    ▼                          │
检查 UPTIME_KUMA_DB_TYPE       │
    │                          │
    ├─ 已设置 ──────────────────┼──┐
    │   └─ 从环境变量读取配置    │  │
    │                          │  │
    ▼                          ▼  ▼
needSetup = false?
    │
    ├─ 否 ──→ 启动配置页面 (SetupDatabase.start)
    │            等待用户选择数据库类型
    │            写入 db-config.json
    │            │
    │            ▼
    └─ 是 ──→ 继续数据库连接
```

### 4.4 数据库类型

1. **SQLite** (默认)
   - 文件: `data/kuma.db`
   - 连接池: 默认单连接（`UPTIME_KUMA_SQLITE_SINGLE_CONNECTION=false` 可改为多连接）
   - WAL 模式 + 增量 vacuum

2. **MariaDB/MySQL**
   - 支持 TCP 或 Socket 连接
   - 连接池默认 10（`UPTIME_KUMA_DB_POOL_MAX_CONNECTIONS` 可调）
   - 支持 SSL/TLS

3. **嵌入式 MariaDB** (`embedded-mariadb.js`)
   - **仅 Docker 容器支持**（检查用户必须是 `node` 或 `root`）
   - 通过 `UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB=1` 启用
   - 自动管理 MariaDB 进程生命周期

## 5. 核心测试标识澄清：--test 与 TEST_BACKEND

### 5.1 两者是完全独立的概念

| 特性 | `--test` (命令行参数) | `TEST_BACKEND` (环境变量) |
|-----|----------------------|-------------------------|
| 类型 | 命令行参数 | 环境变量 |
| 解析方式 | args-parser | process.env |
| 作用范围 | 数据库连接层 | 后端测试框架层 |
| 关联 | 无 | 无 |

### 5.2 --test 的作用链路

```
命令行: node server/server.js --test
    │
    ▼
server.js:160
    const testMode = !!args["test"] || false;
    │
    ▼
server.js:225
    await initDatabase(testMode);
    │
    ▼
server.js:1846
    await Database.connect(testMode);
    │
    ▼
database.js:294
    this.initSQLite(rawConn, testMode)
    │
    ▼
database.js:425-430
    if (testMode) {
        await asyncRun("PRAGMA journal_mode = MEMORY");  // 内存模式
    } else {
        await asyncRun("PRAGMA journal_mode = WAL");     // 正常模式
    }
```

**实际效果**：
- `--test` 只影响 **SQLite 的 journal_mode**
- 设为 MEMORY 模式（非持久化）
- 对 MariaDB 无效
- **与 `TEST_BACKEND` 无任何关联**

### 5.3 TEST_BACKEND 的作用

#### 5.3.1 util-server.js:850-857 - 导出私有函数供测试

```javascript
if (process.env.TEST_BACKEND) {
    module.exports.__test = {
        parseCertificateInfo,
    };
    module.exports.__getPrivateFunction = (functionName) => {
        return module.exports.__test[functionName];
    };
}
```

**作用**：将内部私有函数暴露给测试代码调用。

#### 5.3.2 uptime-calculator.js:105-115 - 可控制的日期方法

```javascript
constructor() {
    if (process.env.TEST_BACKEND) {
        // 允许测试代码通过 UptimeCalculator.currentDate 控制"当前时间"
        this.getCurrentDate = () => {
            if (UptimeCalculator.currentDate) {
                return UptimeCalculator.currentDate;
            } else {
                return dayjs.utc();
            }
        };
    }
}
```

**作用**：允许测试代码模拟特定时间点。

#### 5.3.3 uptime-calculator.js:297-300 - 跳过数据持久化

```javascript
// Don't store data in test mode
if (process.env.TEST_BACKEND) {
    log.debug("uptime_calc", "Skip storing data in test mode");
    return date;  // 直接返回，不写入数据库
}
```

**作用**：测试运行时不写入统计数据，避免污染数据库。

### 5.4 使用场景

```bash
# 场景1: 只想让 SQLite 用内存模式（不持久化）
node server/server.js --test

# 场景2: 运行后端单元测试（需要私有函数访问和模拟时间）
TEST_BACKEND=1 node --test --test-reporter=spec test/backend-test

# 场景3: 两者同时使用（独立生效，互不影响）
TEST_BACKEND=1 node server/server.js --test
```

**重要结论**：
- `--test` 和 `TEST_BACKEND` **没有任何关联**
- `--test` 是数据库层的配置（SQLite MEMORY 模式）
- `TEST_BACKEND` 是测试框架层的配置（导出函数、模拟时间、跳过存储）
- 两者可以独立使用，也可以同时使用

## 6. 运行模式的真实分叉点

### 6.1 分叉维度

代码中实际存在的分叉维度只有两个：

| 维度 | 标识 | 影响范围 |
|-----|------|---------|
| **运行模式** | `NODE_ENV === "development"` | 开发特性开关 |
| **容器环境** | `UPTIME_KUMA_IS_CONTAINER === "1"` | 容器特定功能 |

**PM2、Docker、本地运行** 本质上都是运行 `node server/server.js`，它们的差异主要体现在：
- 环境变量的预设值
- 进程管理方式（PM2/系统d/docker）
- 数据持久化方式（卷挂载）

### 6.2 运行模式分叉：开发 vs 生产

#### 6.2.1 isDev 的定义

```javascript
// src/util.ts:22 (编译后 src/util.js:17)
export const isDev = process.env.NODE_ENV === "development";
```

#### 6.2.2 开发模式分叉点

**分叉点 1: dist/index.html 检查** (`uptime-kuma-server.js:103-110`)

```javascript
try {
    this.indexHTML = fs.readFileSync("./dist/index.html").toString();
} catch (e) {
    // 开发模式不要求 dist/index.html 存在（前端由 Vite 提供）
    if (process.env.NODE_ENV !== "development") {
        log.error("server", "Error: Cannot find 'dist/index.html'");
        process.exit(1);
    }
}
```

- **生产**: 必须存在，否则退出
- **开发**: 不存在也不退出（Vite 开发服务器提供前端）

**分叉点 2: Socket.io CORS 配置** (`uptime-kuma-server.js:136-142`)

```javascript
let cors = undefined;
if (isDev) {
    cors = {
        origin: "*",  // 允许所有来源
    };
}
```

- **生产**: 严格的 Origin 检查（cors-like 模式）
- **开发**: 允许所有来源，方便前端热重载

**分叉点 3: 测试端点** (`server.js:278-321`)

```javascript
if (isDev) {
    app.use(express.urlencoded({ extended: true }));
    
    // Webhook 测试端点
    app.post("/test-webhook", ...);
    app.post("/test-x-www-form-urlencoded", ...);
    
    // E2E 测试快照端点
    app.get("/_e2e/take-sqlite-snapshot", ...);
    app.get("/_e2e/restore-sqlite-snapshot", ...);
}
```

- **生产**: 无这些端点
- **开发**: 提供测试辅助端点

**分叉点 4: 响应头 CORS** (`util-server.js:614-617`)

```javascript
exports.allowDevAllOrigin = (res) => {
    if (process.env.NODE_ENV === "development") {
        exports.allowAllOrigin(res);  // 允许跨域
    }
};
```

### 6.3 容器环境分叉点

#### 6.3.1 代码中所有判断位置汇总

基于代码真实逻辑，`UPTIME_KUMA_IS_CONTAINER` 在以下 4 个位置被使用：

| 文件位置 | 判断方式 | 功能 |
|---------|---------|------|
| `uptime-kuma-server.js:508` | `if (process.env.UPTIME_KUMA_IS_CONTAINER)` | 启动 NSCD 服务 |
| `uptime-kuma-server.js:523` | `if (process.env.UPTIME_KUMA_IS_CONTAINER)` | 停止 NSCD 服务 |
| `real-browser-monitor-type.js:119` | `if (process.env.UPTIME_KUMA_IS_CONTAINER)` | 浏览器可执行路径 |
| `server.js:66` | `process.env.UPTIME_KUMA_IS_CONTAINER === "1"` | 日志调试输出 |
| `client.js:154` | `process.env.UPTIME_KUMA_IS_CONTAINER === "1"` | 前端 isContainer 标识 |

---

#### 6.3.2 两类判断方式分类

**A 类：非空即触发**（真值判断 `if (env)`）

任何非空字符串（包括 `"0"`、`"false"`、`"true"`）都会触发这些分支。

**分支 A1：NSCD 服务启动/停止** (`uptime-kuma-server.js:507-531`)

```javascript
async startNSCDServices() {
    if (process.env.UPTIME_KUMA_IS_CONTAINER) {  // 真值判断
        try {
            log.info("services", "Starting nscd");
            await childProcessAsync.exec("sudo service nscd start");
        } catch (e) {
            log.info("services", "Failed to start nscd");
        }
    }
}
```

**分支 A2：浏览器可执行路径** (`real-browser-monitor-type.js:118-124`)

```javascript
} else if (!executablePath) {
    if (process.env.UPTIME_KUMA_IS_CONTAINER) {  // 真值判断
        executablePath = "/usr/bin/chromium";
        await installChromiumViaApt(executablePath);
    } else {
        executablePath = await findChrome(allowedList);
    }
}
```

---

**B 类：仅等于 1 才触发**（严格相等 `env === "1"`）

只有精确等于字符串 `"1"` 才会触发这些分支。

**分支 B1：日志容器标识** (`server.js:66`)

```javascript
log.debug("server", "Inside Container: " + (process.env.UPTIME_KUMA_IS_CONTAINER === "1"));
```

**分支 B2：前端信息标识** (`client.js:154`)

```javascript
info.isContainer = process.env.UPTIME_KUMA_IS_CONTAINER === "1";
```

---

#### 6.3.3 不同值的行为对照总表

| `UPTIME_KUMA_IS_CONTAINER` 值 | A 类 (NSCD/浏览器) | B 类 (日志/前端) | 综合效果 |
|-----------------------------|-------------------|-----------------|---------|
| **未设置** (`undefined`) | ❌ 不触发 | ❌ 不触发 | **本地运行** ✅ |
| **空字符串** (`""`) | ❌ 不触发 | ❌ 不触发 | 本地运行 |
| **`"1"`** | ✅ 触发 | ✅ 触发 | **完整容器模式** ✅ |
| **`"0"`** | ✅ **触发** ⚠️ | ❌ 不触发 | **不一致**：行为是容器，显示非容器 |
| **`"false"`** | ✅ **触发** ⚠️ | ❌ 不触发 | **不一致**：行为是容器，显示非容器 |
| **`"true"`** | ✅ **触发** ⚠️ | ❌ 不触发 | **不一致**：行为是容器，显示非容器 |
| **其他非空值** | ✅ **触发** ⚠️ | ❌ 不触发 | **不一致**：行为是容器，显示非容器 |

**A 类详细行为**：

| 值 | NSCD 服务 | 浏览器路径 |
|---|---------|-----------|
| `undefined` | 不启动，跳过 | 系统搜索 Chrome/Chromium |
| `""` | 不启动，跳过 | 系统搜索 Chrome/Chromium |
| `"1"` | 启动 nscd | 使用 `/usr/bin/chromium`，apt 安装 |
| `"0"` | **启动 nscd** | **使用容器路径** |
| `"false"` | **启动 nscd** | **使用容器路径** |

**B 类详细行为**：

| 值 | 日志 `Inside Container` | 前端 `info.isContainer` |
|---|------------------------|------------------------|
| `undefined` | `false` | `false` |
| `""` | `false` | `false` |
| `"1"` | `true` | `true` |
| `"0"` | `false` | `false` |
| `"false"` | `false` | `false` |

---

#### 6.3.4 正确使用方式

| 场景 | 环境变量设置 | 说明 |
|-----|-------------|------|
| **本地运行** | 不设置 `UPTIME_KUMA_IS_CONTAINER` | 保持 `undefined`，A 类和 B 类都不触发 |
| **容器模式** | `UPTIME_KUMA_IS_CONTAINER=1` | A 类和 B 类都触发，行为一致 |
| **⚠️ 错误用法** | `UPTIME_KUMA_IS_CONTAINER=0` | A 类触发，B 类不触发，行为不一致 |
| **⚠️ 错误用法** | `UPTIME_KUMA_IS_CONTAINER=false` | A 类触发，B 类不触发，行为不一致 |

**Dockerfile 中的正确设置** (`docker/dockerfile:34`)：

```dockerfile
ENV UPTIME_KUMA_IS_CONTAINER=1
```

---

#### 6.3.5 嵌入式 MariaDB 的特别说明

嵌入式 MariaDB 的启用条件**与 `UPTIME_KUMA_IS_CONTAINER` 无关**。

**启用条件**: `UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB === "1"` (`setup-database.js:127`)

```javascript
isEnabledEmbeddedMariaDB() {
    return process.env.UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB === "1";
}
```

**用户限制**: `embedded-mariadb.js:57-64`

```javascript
start() {
    this.username = require("os").userInfo().username;
    if (this.username !== "node" && this.username !== "root") {
        throw new Error("Embedded Mariadb supports only 'node' or 'root' user");
    }
}
```

- 容器中用户是 `node`，可以使用
- 本地环境通常不是 `node`/`root`，无法使用

### 6.4 各种部署方式的真实代码分叉

#### 本地直接运行

```bash
# 生产模式
node server/server.js
# 等价于: NODE_ENV=production UPTIME_KUMA_IS_CONTAINER=0 node server/server.js

# 开发模式
cross-env NODE_ENV=development node server/server.js
# 等价于: NODE_ENV=development UPTIME_KUMA_IS_CONTAINER=0 node server/server.js
```

**分叉**: 仅 `NODE_ENV` 决定开发/生产特性

#### PM2 运行

```bash
pm2 start server/server.js --name uptime-kuma
# 或
pm2 start ecosystem.config.js
```

**代码层面**：PM2 只是进程管理器，代码内没有任何 `pm2` 相关的检查。所有分叉逻辑与本地运行完全一致。

**PM2 的作用（非代码层面）**：
- 进程自动重启
- 日志管理
- 集群模式（可配置）
- 开机自启

**重要结论**：从代码初始化流程看，**PM2 与本地直接运行没有任何分叉**。

#### Docker 运行

```dockerfile
# docker/dockerfile:34
ENV UPTIME_KUMA_IS_CONTAINER=1

# docker/dockerfile:42
CMD ["node", "server/server.js"]
```

**代码层面的分叉**：
- `UPTIME_KUMA_IS_CONTAINER=1` → 触发容器特定逻辑（NSCD、浏览器路径、嵌入式 MariaDB）
- 默认 `NODE_ENV` 由 `server.js:57-59` 设为 `production`

**Docker 的额外特性（非代码层面）**：
- 镜像预装依赖（Chromium、MariaDB 等）
- 数据卷挂载
- 网络隔离
- 健康检查

#### 开发模式 (npm run dev)

```json
// package.json:21
"dev": "concurrently -k -r \"wait-on tcp:3000 && npm run start-server-dev \" \"npm run start-frontend-dev\"",
"start-frontend-dev": "cross-env NODE_ENV=development vite --host",
"start-server-dev": "cross-env NODE_ENV=development node server/server.js",
```

**代码层面的分叉**：
- `NODE_ENV=development` → 触发所有开发模式特性
- 同时运行 Vite 前端开发服务器（端口 3000）和后端（端口 3001）

## 7. 完整启动流程

### 7.1 流程图

```
node server/server.js
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 1: 基础初始化                                  │
│   - 加载 dotenv                                     │
│   - 检查 Node.js 版本                               │
│   - 解析命令行参数                                   │
│   - 设置默认环境变量                                 │
│     • NODE_ENV = "production" (如未设置)            │
│     • UPTIME_KUMA_WS_ORIGIN_CHECK = "cors-like"     │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 2: 创建服务器实例 (UptimeKumaServer)          │
│                                                     │
│ 【分叉点: NODE_ENV】                                 │
│   dist/index.html 检查                               │
│   ├─ production: 不存在则退出                        │
│   └─ development: 不存在也继续                       │
│                                                     │
│ 【分叉点: NODE_ENV (isDev)】                         │
│   Socket.io CORS 配置                               │
│   ├─ production: 严格检查                           │
│   └─ development: origin: "*"                       │
│                                                     │
│   - 初始化 Socket.io                                │
│   - 注册所有监控类型                                  │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 3: 数据库配置                                  │
│   - 初始化数据目录                                   │
│   - SetupDatabase 决策                               │
│     • 读取 db-config.json                           │
│     • 检查遗留 kuma.db                              │
│     • 检查 UPTIME_KUMA_DB_TYPE                      │
│   - 需要配置？→ 启动配置页面                         │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 4: 数据库连接                                  │
│                                                     │
│ 【分叉点: --test 命令行参数】                         │
│   SQLite journal_mode                               │
│   ├─ testMode=true:  MEMORY (非持久化)              │
│   └─ testMode=false: WAL (持久化)                   │
│                                                     │
│   - 连接数据库 (SQLite/MariaDB/嵌入式)              │
│   - 执行数据库迁移 (patch)                          │
│   - 初始化/加载 JWT Secret                          │
│   - 检查是否需要首次用户设置                          │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 5: 服务器初始化                                │
│   - 设置时区                                         │
│   - 加载维护任务列表                                 │
│   - 初始化 Prometheus                               │
│   - 加载状态页面域名映射                              │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 6: 路由注册                                    │
│                                                     │
│ 【分叉点: NODE_ENV (isDev)】                         │
│   测试端点注册                                       │
│   ├─ production: 无                                 │
│   └─ development: /test-webhook, /_e2e/*            │
│                                                     │
│   - 注册 Express 路由                                │
│   - 注册 Socket.io 事件处理器                        │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ Phase 7: 启动服务                                    │
│                                                     │
│ 【分叉点: UPTIME_KUMA_IS_CONTAINER】                 │
│   NSCD 服务启动                                      │
│   ├─ container: 启动 nscd                           │
│   └─ 非 container: 跳过                              │
│                                                     │
│   - server.start()                                  │
│   - server.httpServer.listen()                      │
│   - 启动所有活动监控                                  │
│   - 启动后台任务                                      │
│   - 启动 Cloudflared (如配置)                        │
└─────────────────────────────────────────────────────┘
    │
    ▼
服务器就绪
```

### 7.2 关键分叉点总结表

| 阶段 | 检查条件 | 分叉代码位置 | 生产/容器 | 开发/非容器 |
|-----|---------|-------------|----------|------------|
| 服务器实例化 | `NODE_ENV !== "development"` | `uptime-kuma-server.js:106` | 要求 dist/index.html | 不要求 |
| Socket.io CORS | `isDev` | `uptime-kuma-server.js:138` | 严格检查 | `origin: "*"` |
| SQLite 模式 | `testMode` (--test) | `database.js:425` | WAL | MEMORY (如果 --test) |
| 测试端点 | `isDev` | `server.js:278` | 无端点 | 注册端点 |
| NSCD 服务 | `UPTIME_KUMA_IS_CONTAINER` | `uptime-kuma-server.js:508` | 启动 nscd | 跳过 |
| 浏览器路径 | `UPTIME_KUMA_IS_CONTAINER` | `real-browser-monitor-type.js:119` | `/usr/bin/chromium` | 系统搜索 |
| 测试函数导出 | `TEST_BACKEND` | `util-server.js:850` | 不导出 | 导出 __test |
| 时间模拟 | `TEST_BACKEND` | `uptime-calculator.js:105` | 真实时间 | 可模拟 |
| 数据存储跳过 | `TEST_BACKEND` | `uptime-calculator.js:297` | 正常存储 | 跳过存储 |

## 8. 不同部署方式的启动命令对比

### 8.1 启动命令汇总

| 部署方式 | 启动命令 | 环境变量预设 | 代码分叉来源 |
|---------|---------|-------------|-------------|
| 本地生产 | `node server/server.js` | 无 | `NODE_ENV` 默认为 `production` |
| 本地开发 | `npm run dev` | `NODE_ENV=development` | `NODE_ENV=development` |
| PM2 | `pm2 start server/server.js` | 无（继承环境） | 同本地运行 |
| Docker | `docker run louislam/uptime-kuma` | `UPTIME_KUMA_IS_CONTAINER=1` | `UPTIME_KUMA_IS_CONTAINER=1` |
| 后端测试 | `npm run test-backend` | `TEST_BACKEND=1` | `TEST_BACKEND=1` |

### 8.2 部署方式本质分析

```
                    ┌─────────────────────────────────────┐
                    │         node server/server.js       │
                    │         (唯一真实入口)              │
                    └─────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
     ┌────────────┐   ┌────────────┐   ┌────────────┐
     │ PM2 包装    │   │ Docker 包装 │   │ 直接运行    │
     │ (进程管理)  │   │ (容器隔离)  │   │            │
     └────────────┘   └────────────┘   └────────────┘
              │               │               │
              └───────────────┼───────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   环境变量差异     │
                    │                   │
                    │ NODE_ENV          │
                    │ UPTIME_KUMA_*     │
                    │ TEST_BACKEND      │
                    └───────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   代码内分叉逻辑   │
                    │  (isDev, isContainer) │
                    └───────────────────┘
```

**核心结论**：
- 所有部署方式的代码入口都是 `server/server.js`
- PM2 和 Docker 只是**进程/环境包装层**
- 真正的代码分叉由**环境变量**触发，而不是部署工具
- PM2 在代码层面没有任何特殊逻辑

## 9. 关键文件位置

| 功能 | 文件路径 | 关键行号 |
|------|---------|---------|
| 入口文件 | `server/server.js` | 1-160, 212-229, 1737-1760 |
| 基础配置 | `server/config.js` | 1-50 |
| isDev 定义 | `src/util.ts` | 22 |
| 数据库配置 | `server/setup-database.js` | 66-112 |
| 数据库管理 | `server/database.js` | 135-162, 202-444 |
| 服务器实例 | `server/uptime-kuma-server.js` | 78-196, 482-531 |
| --test 处理 | `server/server.js` | 160, 225, 1844-1846 |
| --test SQLite | `server/database.js` | 419-444 |
| TEST_BACKEND | `server/util-server.js` | 850-857 |
| TEST_BACKEND | `server/uptime-calculator.js` | 105-115, 297-300 |
| Dockerfile | `docker/dockerfile` | 34, 42 |
| PM2 配置 | `ecosystem.config.js` | 1-8 |
| 启动脚本 | `package.json` | 21-27, 31-33 |

## 10. 总结与澄清

### 10.1 关键澄清

| 之前的误解 | 代码真实情况 |
|-----------|-------------|
| `--test` 和 `TEST_BACKEND` 相关 | **完全独立，无关联** |
| PM2 有特殊初始化逻辑 | **没有**，只是进程管理器 |
| Docker 与本地运行有本质区别 | **没有**，只是 `UPTIME_KUMA_IS_CONTAINER=1` |
| 部署模式 = 运行模式 | **不同维度**：部署是环境包装，运行模式由 `NODE_ENV` 决定 |

### 10.2 真实存在的代码分叉

代码中实际存在的分叉条件：

#### 严格相等判断 (===)

1. **`NODE_ENV === "development"` (isDev)**
   - dist/index.html 检查
   - Socket.io CORS 策略
   - 测试端点注册
   - 响应头 CORS

2. **`UPTIME_KUMA_IS_CONTAINER === "1"` (B类)**
   - 日志容器标识 (`server.js:66`)
   - 前端容器标识 (`client.js:154`)

3. **`UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB === "1"`**
   - 嵌入式 MariaDB 启用判断

#### 真值判断 (if (env))

4. **`process.env.UPTIME_KUMA_IS_CONTAINER` (A类)**
   - NSCD 服务启停
   - 浏览器可执行路径
   - **注意**：任何非空字符串（包括 `"0"`、`"false"`）都会触发

5. **`process.env.TEST_BACKEND`**
   - 导出私有测试函数
   - 可模拟当前时间
   - 跳过统计数据存储
   - **注意**：任何非空字符串都会触发

#### 命令行参数

6. **`testMode = !!args["test"]` (--test 命令行参数)**
   - SQLite journal_mode (MEMORY vs WAL)

### 10.3 部署方式的本质

```
部署方式只是"启动器"，决定：
├── 环境变量预设值
├── 进程管理方式
└── 资源隔离方式

代码分叉只看"环境变量值"，不关心"是谁启动的"
```

### 10.4 重要警告：UPTIME_KUMA_IS_CONTAINER 判断不一致

**代码中存在严重的判断不一致问题**：

| 功能 | 判断方式 | `"1"` | `"0"` 或 `"false"` | `undefined` |
|-----|---------|-------|-------------------|------------|
| NSCD 服务启动 | `if (env)` | ✅ 启动 | ✅ 启动 | ❌ 不启动 |
| 浏览器路径 | `if (env)` | ✅ 容器路径 | ✅ 容器路径 | ❌ 系统搜索 |
| 日志容器标识 | `env === "1"` | ✅ 显示容器 | ❌ 不显示 | ❌ 不显示 |
| 前端容器标识 | `env === "1"` | ✅ 显示容器 | ❌ 不显示 | ❌ 不显示 |

**潜在问题**：
- `UPTIME_KUMA_IS_CONTAINER=0` 或 `UPTIME_KUMA_IS_CONTAINER=false` → 会**错误触发** NSCD 和浏览器容器路径，但日志/前端显示为非容器
- 这可能导致调试困难，因为实际行为与日志显示不一致

**正确使用方式**：
- 启用容器模式：`UPTIME_KUMA_IS_CONTAINER=1`
- 禁用容器模式：**不设置**这个环境变量（不要设为 `"0"` 或 `"false"`）

### 10.5 设计优点
- 本地开发和生产环境的代码行为一致（仅由 `NODE_ENV` 控制）
- Docker 容器只是添加了容器特定的优化（NSCD、预安装依赖）
- PM2 等进程管理器完全透明，无需代码适配
- 测试标识完全隔离，不会互相干扰
