# Uptime Kuma 服务器启动流程与环境配置分析报告

## 1. 概述

本报告详细分析了 Uptime Kuma 应用的启动流程、环境配置加载机制以及不同部署模式下的初始化流程分叉。Uptime Kuma 是一个现代化的状态监控工具，支持多种部署方式和数据库配置。

## 2. 入口与基础初始化

### 2.1 启动入口

Uptime Kuma 的服务器端入口文件是 `server/server.js`，通过 `node server/server.js` 命令启动。在 `package.json` 中定义了多个启动脚本：

- **生产启动**: `npm run start` → `npm run start-server` → `node server/server.js`
- **开发模式**: `npm run dev` → 同时启动前端开发服务器和后端服务器
- **开发后端**: `npm run start-server-dev` → `cross-env NODE_ENV=development node server/server.js`

### 2.2 启动基础流程

`server.js:1-160` 的基础初始化流程：

1. **环境变量加载**: 使用 `dotenv` 库从 `.env` 文件加载环境变量（`server.js:15`）
2. **Node.js 版本检查**: 验证 Node.js 版本是否符合要求（`>= 20.4.0`），禁止使用不兼容版本
3. **命令行参数解析**: 使用 `args-parser` 解析命令行参数
4. **默认环境设置**:
   - 如果 `NODE_ENV` 未设置，默认为 `"production"`（`server.js:57-59`）
   - 如果 `UPTIME_KUMA_WS_ORIGIN_CHECK` 未设置，默认为 `"cors-like"`（`server.js:61-63`）

## 3. 环境配置加载机制

### 3.1 配置加载优先级

Uptime Kuma 的配置加载遵循以下优先级顺序（从高到低）：

1. **命令行参数** (args-parser)
2. **环境变量** (process.env)
3. **默认值**

### 3.2 核心配置文件 `server/config.js`

`server/config.js` 负责加载服务器的核心配置，包括：

#### 3.2.1 主机和端口配置

```javascript
// config.js:9-14
let hostEnv = isFreeBSD ? null : process.env.HOST;
const hostname = args.host || process.env.UPTIME_KUMA_HOST || hostEnv;

const port = [args.port, process.env.UPTIME_KUMA_PORT, process.env.PORT, 3001]
    .map((portValue) => parseInt(portValue))
    .find((portValue) => !isNaN(portValue));
```

**优先级**:
- 主机: `args.host` > `UPTIME_KUMA_HOST` > `HOST` (非 FreeBSD)
- 端口: `args.port` > `UPTIME_KUMA_PORT` > `PORT` > 默认 3001

#### 3.2.2 SSL/TLS 配置

```javascript
// config.js:16-24
const sslKey = args["ssl-key"] || process.env.UPTIME_KUMA_SSL_KEY || process.env.SSL_KEY || undefined;
const sslCert = args["ssl-cert"] || process.env.UPTIME_KUMA_SSL_CERT || process.env.SSL_CERT || undefined;
const sslKeyPassphrase = args["ssl-key-passphrase"] || process.env.UPTIME_KUMA_SSL_KEY_PASSPHRASE || process.env.SSL_KEY_PASSPHRASE || undefined;
const isSSL = sslKey && sslCert;
```

**优先级**:
- SSL 密钥: `args["ssl-key"]` > `UPTIME_KUMA_SSL_KEY` > `SSL_KEY`
- SSL 证书: `args["ssl-cert"]` > `UPTIME_KUMA_SSL_CERT` > `SSL_CERT`
- SSL 密钥密码: `args["ssl-key-passphrase"]` > `UPTIME_KUMA_SSL_KEY_PASSPHRASE` > `SSL_KEY_PASSPHRASE`

#### 3.2.3 其他配置

- **演示模式**: `args["demo"]` → `config.js:38`
- **本地 WebSocket URL**: 根据 SSL 配置和主机端口动态生成 → `config.js:30-36`

### 3.3 数据目录配置

`server/database.js:135-162` 中的 `initDataDir` 方法：

```javascript
Database.dataDir = process.env.DATA_DIR || args["data-dir"] || "./data/";
```

**优先级**:
- 数据目录: `DATA_DIR` > `args["data-dir"]` > 默认 `./data/`

数据目录下会自动创建以下子目录：
- `./data/upload/` - 用户上传文件
- `./data/screenshots/` - 浏览器监控截图
- `./data/docker-tls/` - Docker TLS 证书
- `./data/mariadb/` - 嵌入式 MariaDB 数据（如使用）

### 3.4 主要环境变量列表

| 环境变量 | 说明 | 默认值 | 使用位置 |
|---------|------|-------|---------|
| `NODE_ENV` | 运行环境 | `production` | 全局 |
| `DATA_DIR` | 数据目录 | `./data/` | `database.js:137` |
| `UPTIME_KUMA_HOST` | 监听主机 | 空 | `config.js:10` |
| `UPTIME_KUMA_PORT` | 监听端口 | `3001` | `config.js:12` |
| `UPTIME_KUMA_SSL_KEY` | SSL 密钥路径 | 空 | `config.js:16` |
| `UPTIME_KUMA_SSL_CERT` | SSL 证书路径 | 空 | `config.js:17` |
| `UPTIME_KUMA_DB_TYPE` | 数据库类型 | 空（首次启动需配置） | `setup-database.js:98` |
| `UPTIME_KUMA_DB_HOSTNAME` | 数据库主机 | 空 | `setup-database.js:102` |
| `UPTIME_KUMA_DB_PORT` | 数据库端口 | 空 | `setup-database.js:103` |
| `UPTIME_KUMA_DB_NAME` | 数据库名 | 空 | `setup-database.js:104` |
| `UPTIME_KUMA_DB_USERNAME` | 数据库用户名 | 空 | `setup-database.js:105` |
| `UPTIME_KUMA_DB_PASSWORD` | 数据库密码 | 空 | `setup-database.js:106` |
| `UPTIME_KUMA_DB_SOCKET` | 数据库 Socket 路径 | 空 | `setup-database.js:107` |
| `UPTIME_KUMA_DB_SSL` | 启用数据库 SSL | `false` | `setup-database.js:108` |
| `UPTIME_KUMA_DB_CA` | 数据库 CA 证书 | 空 | `setup-database.js:109` |
| `UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB` | 启用嵌入式 MariaDB | `false` | `setup-database.js:127` |
| `UPTIME_KUMA_DB_POOL_MAX_CONNECTIONS` | 数据库连接池大小 | `10` | `database.js:225` |
| `UPTIME_KUMA_SQLITE_SINGLE_CONNECTION` | SQLite 单连接模式 | `true` | `database.js:275` |
| `UPTIME_KUMA_IS_CONTAINER` | 是否在容器中运行 | `false` | `server.js:66` |
| `UPTIME_KUMA_WS_ORIGIN_CHECK` | WebSocket 源检查模式 | `cors-like` | `server.js:62` |
| `UPTIME_KUMA_DISABLE_FRAME_SAMEORIGIN` | 禁用 X-Frame-Options | `false` | `server.js:147` |
| `UPTIME_KUMA_CLOUDFLARED_TOKEN` | Cloudflared 隧道令牌 | 空 | `server.js:148` |
| `TZ` | 时区 | 系统默认 | `uptime-kuma-server.js:206` |
| `SQL_LOG` | 启用 SQL 日志 | `false` | `database.js:390` |
| `SET_MIGRATE_AGGREGATE_TABLE_TO_TRUE` | 跳过聚合表迁移 | `false` | `database.js:823` |

### 3.5 Docker Secrets 支持

`server/setup-database.js:19-36` 中实现了 Docker Secrets 支持，通过 `getEnvOrFile` 函数：

```javascript
function getEnvOrFile(envName) {
    const directValue = process.env[envName];
    const fileValue = process.env[envName + "_FILE"];

    if (directValue && fileValue) {
        throw new Error(`Both ${envName} and ${envName}_FILE are set. Please use only one.`);
    }

    if (fileValue) {
        try {
            return fs.readFileSync(fileValue, "utf8").trim();
        } catch (err) {
            throw new Error(`Failed to read ${envName}_FILE at ${fileValue}: ${err.message}`);
        }
    }

    return directValue;
}
```

支持的数据库敏感配置可通过 `_FILE` 后缀从 Docker Secrets 读取：
- `UPTIME_KUMA_DB_USERNAME` / `UPTIME_KUMA_DB_USERNAME_FILE`
- `UPTIME_KUMA_DB_PASSWORD` / `UPTIME_KUMA_DB_PASSWORD_FILE`
- `UPTIME_KUMA_DB_SSL` / `UPTIME_KUMA_DB_SSL_FILE`
- `UPTIME_KUMA_DB_CA` / `UPTIME_KUMA_DB_CA_FILE`

## 4. 数据库配置流程

### 4.1 数据库配置初始化流程

`server/server.js:212-229` 中的主要流程：

```javascript
(async () => {
    // 1. 初始化数据目录
    Database.initDataDir(args);

    // 2. 检查是否需要数据库配置
    let setupDatabase = new SetupDatabase(args, server);
    if (setupDatabase.isNeedSetup()) {
        // 启动配置页面，等待用户选择数据库类型
        await setupDatabase.start(hostname, port);
    }

    // 3. 连接数据库
    try {
        await initDatabase(testMode);
    } catch (e) {
        log.error("server", "Failed to prepare your database: " + e.message);
        process.exit(1);
    }
    // ...
})();
```

### 4.2 SetupDatabase 类的配置逻辑

`server/setup-database.js:66-112` 中的配置优先级：

```javascript
constructor(args, server) {
    // 优先级: 环境变量 > db-config.json
    // 如果提供了环境变量，写入 db-config.json
    // 如果找到 db-config.json，检查是否有效
    // 如果 db-config.json 不存在或无效，检查是否存在 kuma.db
    // 如果 kuma.db 不存在，显示配置页面

    let dbConfig;

    try {
        dbConfig = Database.readDBConfig(); // 从 data/db-config.json 读取
        this.needSetup = false;
    } catch (e) {
        // 检查是否是 v1.x 的用户（只有 kuma.db）
        if (fs.existsSync(path.join(Database.dataDir, "kuma.db"))) {
            this.needSetup = false;
            Database.writeDBConfig({ type: "sqlite" }); // 自动创建配置
        } else {
            this.needSetup = true;
        }
    }

    // 环境变量覆盖
    if (process.env.UPTIME_KUMA_DB_TYPE) {
        this.needSetup = false;
        dbConfig.type = process.env.UPTIME_KUMA_DB_TYPE;
        // ... 加载其他数据库配置
        Database.writeDBConfig(dbConfig);
    }
}
```

### 4.3 数据库配置流程分叉图

```
启动服务器
    │
    ▼
初始化数据目录
    │
    ▼
检查是否已有数据库配置？
    │
    ├─ 是（db-config.json 有效）
    │       │
    │       ▼
    │   检查环境变量是否覆盖？
    │       │
    │       ├─ 是 → 用环境变量覆盖配置，写入 db-config.json
    │       │
    │       └─ 否 → 使用现有配置
    │
    └─ 否
            │
            ▼
        检查是否有旧版本的 kuma.db？
            │
            ├─ 是 → 自动创建 sqlite 配置，写入 db-config.json
            │
            └─ 否
                    │
                    ▼
                检查 UPTIME_KUMA_DB_TYPE 环境变量？
                    │
                    ├─ 是 → 从环境变量读取配置，写入 db-config.json
                    │
                    └─ 否 → 启动配置页面，等待用户选择数据库类型
```

### 4.4 支持的数据库类型

1. **SQLite** (默认)
   - 文件存储: `./data/kuma.db`
   - 连接池: 默认单连接（可通过 `UPTIME_KUMA_SQLITE_SINGLE_CONNECTION=false` 改为多连接）
   - WAL 模式，支持增量 vacuum

2. **MariaDB/MySQL**
   - 支持 TCP 连接和 Socket 连接
   - 支持 SSL/TLS 加密
   - 连接池大小可配置（默认 10）

3. **嵌入式 MariaDB**
   - 仅在 Docker 容器中支持
   - 通过 `UPTIME_KUMA_ENABLE_EMBEDDED_MARIADB=1` 启用
   - 自动管理 MariaDB 进程生命周期

## 5. 不同部署模式的初始化流程

### 5.1 部署模式分类

Uptime Kuma 支持多种部署模式，主要包括：

1. **直接运行模式** (本地开发/测试)
2. **Docker 容器模式** (推荐生产环境)
3. **PM2 进程管理模式**
4. **测试模式**

### 5.2 直接运行模式

**启动命令**:
```bash
npm run start-server
# 或
node server/server.js
```

**环境特点**:
- `UPTIME_KUMA_IS_CONTAINER` 不为 `1`
- 默认使用 SQLite 数据库
- 数据目录默认 `./data/`

**初始化流程**:
1. 加载 `.env` 文件（如果存在）
2. 默认 `NODE_ENV=production`
3. 检查数据库配置
4. 如无配置且无 `kuma.db`，启动配置页面
5. 连接数据库并执行迁移
6. 初始化 JWT Secret
7. 检查是否需要用户设置（无用户时）
8. 启动主服务器

### 5.3 Docker 容器模式

**配置**:
- Dockerfile: `docker/dockerfile`
- docker-compose: `compose.yaml`

**容器环境特点**:
- `UPTIME_KUMA_IS_CONTAINER=1`（在 Dockerfile 中设置）
- 使用 `dumb-init` 作为进程管理器
- 数据目录通过卷挂载: `./data:/app/data`
- 可通过环境变量完全配置数据库（无需首次配置页面）

**Dockerfile 关键配置**:
```dockerfile
# docker/dockerfile:34
ENV UPTIME_KUMA_IS_CONTAINER=1

# docker/dockerfile:42
CMD ["node", "server/server.js"]
```

**容器模式特殊处理**:

1. **浏览器监控路径调整** (`server/monitor-types/real-browser-monitor-type.js:119`):
   - 容器内使用固定的 Chromium 路径

2. **嵌入式 MariaDB 支持** (`server/embedded-mariadb.js`):
   - 仅在容器中可用
   - 检查运行用户必须是 `node` 或 `root`

3. **信息显示** (`server/client.js:154`):
   - 向前端传递 `isContainer` 标志

**Docker 部署的数据库配置示例**:

```yaml
# compose.yaml 扩展版本
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    environment:
      - UPTIME_KUMA_DB_TYPE=mariadb
      - UPTIME_KUMA_DB_HOSTNAME=mariadb
      - UPTIME_KUMA_DB_PORT=3306
      - UPTIME_KUMA_DB_NAME=uptime_kuma
      - UPTIME_KUMA_DB_USERNAME=uptime
      - UPTIME_KUMA_DB_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - ./data:/app/data
    ports:
      - "3001:3001"
    depends_on:
      - mariadb

  mariadb:
    image: mariadb:10
    environment:
      - MYSQL_ROOT_PASSWORD=rootpass
      - MYSQL_DATABASE=uptime_kuma
      - MYSQL_USER=uptime
      - MYSQL_PASSWORD=secret
    volumes:
      - ./mariadb-data:/var/lib/mysql
```

### 5.4 PM2 进程管理模式

**配置文件**: `ecosystem.config.js`

```javascript
module.exports = {
    apps: [
        {
            name: "uptime-kuma",
            script: "./server/server.js",
        },
    ],
};
```

**启动命令**:
```bash
pm2 start ecosystem.config.js
```

**特点**:
- 进程自动重启
- 日志管理
- 集群模式支持（可配置）
- 环境变量可通过 PM2 配置传递

### 5.5 开发模式

**启动命令**:
```bash
npm run dev
```

**配置特点**:
- `NODE_ENV=development`
- 同时启动 Vite 前端开发服务器和后端服务器
- 前端热重载
- CORS 开放（`uptime-kuma-server.js:138-142`）

**开发模式特殊处理**:

1. **CORS 配置** (`uptime-kuma-server.js:138-142`):
```javascript
if (isDev) {
    cors = {
        origin: "*",
    };
}
```

2. **测试端点** (`server.js:278-321`):
   - `/test-webhook` - Webhook 测试
   - `/test-x-www-form-urlencoded` - 表单测试
   - `/_e2e/take-sqlite-snapshot` - E2E 测试快照
   - `/_e2e/restore-sqlite-snapshot` - E2E 测试恢复

3. **前端资源处理** (`uptime-kuma-server.js:103-110`):
   - 开发模式不要求 `dist/index.html` 存在

### 5.6 测试模式

**触发条件**:
- 命令行参数 `--test`
- 或环境变量 `TEST_BACKEND`

**特点**:
- SQLite 使用 MEMORY 模式（`database.js:426-427`）
- 禁用某些持久化操作
- 用于自动化测试

## 6. 完整启动流程图

```
node server/server.js
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 1. 基础初始化                                        │
│   - 加载 dotenv                                     │
│   - 检查 Node.js 版本                               │
│   - 解析命令行参数                                   │
│   - 设置默认环境变量 (NODE_ENV, WS_ORIGIN_CHECK)    │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 2. 创建服务器实例 (UptimeKumaServer)                │
│   - 创建 Express 应用                                │
│   - 创建 HTTP/HTTPS 服务器                           │
│   - 加载前端 index.html (生产模式必须存在)           │
│   - 初始化 Socket.io                                │
│   - 注册所有监控类型                                  │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 3. 初始化数据目录                                    │
│   - 确定 DATA_DIR                                   │
│   - 创建必要的子目录                                 │
│   - 确定数据路径                                      │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 4. 数据库配置检查 (SetupDatabase)                    │
│   ├─ 检查 db-config.json                            │
│   ├─ 检查环境变量覆盖                                │
│   ├─ 检查旧版 kuma.db (v1.x 迁移)                   │
│   └─ 需要配置？→ 启动配置页面 → 等待用户选择         │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 5. 数据库连接与初始化 (initDatabase)                │
│   - 根据配置连接数据库 (SQLite/MariaDB/嵌入式)      │
│   - 执行数据库迁移 (patch)                          │
│   - 初始化/加载 JWT Secret                          │
│   - 检查是否需要首次用户设置                          │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 6. 服务器初始化 (initAfterDatabaseReady)           │
│   - 设置时区                                         │
│   - 加载维护任务列表                                 │
│   - 初始化 Prometheus                               │
│   - 加载状态页面域名映射                              │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 7. 路由与处理器注册                                  │
│   - 注册 Express 路由                                │
│   - 注册 Socket.io 事件处理器                        │
│   - 注册 API 路由器                                  │
│   - 注册状态页面路由器                                │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 8. 启动后台服务                                      │
│   - 启动所有活动监控                                  │
│   - 启动定时任务 (清理旧数据等)                      │
│   - 启动 Cloudflared 隧道 (如配置)                  │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 9. 服务器就绪                                        │
│   - 监听端口                                         │
│   - 输出访问地址                                     │
│   - 接受请求                                         │
└─────────────────────────────────────────────────────┘
```

## 7. 关键分叉点分析

### 7.1 数据库配置分叉

在 `SetupDatabase` 构造函数中，根据配置存在情况有三个主要分支：

1. **已有有效配置** → 直接使用
2. **无配置但有 v1.x 数据库** → 自动迁移配置
3. **无配置且无数据库** → 检查环境变量或启动配置页面

### 7.2 运行环境分叉

根据 `NODE_ENV` 和 `UPTIME_KUMA_IS_CONTAINER`：

1. **生产环境** (`NODE_ENV=production`):
   - 要求 `dist/index.html` 必须存在
   - 严格的 CORS 策略
   - 禁用测试端点

2. **开发环境** (`NODE_ENV=development`):
   - 不要求 `dist/index.html`
   - 开放 CORS
   - 启用测试端点

3. **容器环境** (`UPTIME_KUMA_IS_CONTAINER=1`):
   - 支持嵌入式 MariaDB
   - 特定的浏览器路径
   - 信息标识

### 7.3 数据库类型分叉

在 `Database.connect()` 中根据 `dbConfig.type`：

1. **SQLite**:
   - 复制模板数据库（如不存在）
   - 配置单连接或多连接池
   - 初始化 SQLite PRAGMA

2. **MariaDB (外部)**:
   - 测试连接
   - 创建数据库（如不存在）
   - 配置连接池

3. **嵌入式 MariaDB**:
   - 启动 MariaDB 子进程
   - 通过 Socket 连接
   - 自动管理进程生命周期

## 8. 关键文件位置

| 功能 | 文件路径 | 行号范围 |
|------|---------|---------|
| 入口文件 | `server/server.js` | 1-160 |
| 基础配置 | `server/config.js` | 1-50 |
| 数据库配置 | `server/setup-database.js` | 43-112 |
| 数据库管理 | `server/database.js` | 1-412 |
| 服务器实例 | `server/uptime-kuma-server.js` | 1-196 |
| 嵌入式 MariaDB | `server/embedded-mariadb.js` | 1-150 |
| 启动脚本 | `package.json` | 21-27 |
| Docker 配置 | `docker/dockerfile` | 29-42 |
| PM2 配置 | `ecosystem.config.js` | 1-8 |
| Docker Compose | `compose.yaml` | 1-9 |

## 9. 总结

Uptime Kuma 的启动流程设计考虑了多种部署场景，具有以下特点：

1. **灵活的配置系统**: 支持命令行参数、环境变量、配置文件和 Docker Secrets，优先级明确
2. **平滑的升级路径**: 自动检测 v1.x 的 `kuma.db` 并迁移配置
3. **容器友好**: 专门为 Docker 环境优化，支持嵌入式数据库和 Secrets
4. **开发友好**: 开发模式提供热重载和测试端点
5. **数据库无关性**: 通过统一的抽象层支持 SQLite、MariaDB 和嵌入式 MariaDB

不同部署模式的主要差异体现在：
- 环境变量的预设值
- 数据库配置的自动程度
- 浏览器路径和系统级功能
- 开发/测试工具的可用性

这种设计使得 Uptime Kuma 能够在个人电脑、开发服务器、容器平台和生产环境中灵活部署，同时保持一致的核心功能。
