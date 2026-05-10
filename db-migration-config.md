# Uptime Kuma 数据库迁移与配置持久化分析报告

## 1. 概述

本文档分析 Uptime Kuma 项目中的数据库迁移机制、应用配置在 SQLite 中的持久化方式，以及跨版本升级时的兼容处理策略。

Uptime Kuma 支持多种数据库类型（SQLite 和 MariaDB），并采用了成熟的迁移框架来管理数据库结构的演进。

---

## 2. 数据库迁移执行机制

### 2.1 迁移系统的演进

Uptime Kuma 的数据库迁移系统经历了三个阶段的演进：

1. **第一阶段：版本号控制** - 使用 `database_version` 设置追踪版本（最高版本 10）
2. **第二阶段：补丁文件追踪** - 使用 `patchX.sql` 和 `databasePatchedFiles` 设置
3. **第三阶段：Knex 迁移框架** - 当前主流方式，使用 Knex.js 迁移系统

### 2.2 核心迁移类

数据库迁移的核心逻辑位于 `server/database.js` 中的 `Database` 类。

#### 2.2.1 迁移入口

迁移的入口是 `Database.patch()` 方法（`server/database.js:468`）：

```javascript
static async patch(port = undefined, hostname = undefined) {
    // 处理旧版本 SQLite 的迁移
    if (Database.dbConfig.type === "sqlite") {
        await this.patchSqlite();
    }

    // 使用 Knex 迁移系统
    try {
        // SQLite 需要临时禁用外键检查
        if (Database.dbConfig.type === "sqlite") {
            await R.exec("PRAGMA foreign_keys = OFF");
        }

        // 执行所有未应用的 Knex 迁移
        await R.knex.migrate.latest({
            directory: Database.knexMigrationsPath,
        });

        // 恢复外键检查
        if (Database.dbConfig.type === "sqlite") {
            await R.exec("PRAGMA foreign_keys = ON");
        }

        // 聚合表迁移（V1 到 V2 的大版本升级）
        await this.migrateAggregateTable(port, hostname);
    } catch (e) {
        // 处理降级场景
        if (e.message.includes("the following files are missing:")) {
            log.warn("db", "Database migration failed, you may be downgrading Uptime Kuma.");
        } else {
            throw e;
        }
    }
}
```

#### 2.2.2 Knex 迁移系统

**迁移文件位置**：`db/knex_migrations/`

**文件命名规范**（参考 `db/knex_migrations/README.md`）：
- 格式：`YYYY-MM-DD-HHMM-patch-name.js`
- 所有表必须有 `id` 主键
- 避免原生 SQL，使用 Knex 方法以兼容 SQLite 和 MariaDB

**标准迁移模板**：
```javascript
exports.up = function (knex) {};

exports.down = function (knex) {};

// 可选：禁用事务
// exports.config = { transaction: false };
```

**示例迁移**（`db/knex_migrations/2023-08-18-0301-heartbeat.js`）：
```javascript
exports.up = function (knex) {
    return knex.schema.alterTable("heartbeat", function (table) {
        table.datetime("end_time").nullable().defaultTo(null);
    });
};

exports.down = function (knex) {
    return knex.schema.alterTable("heartbeat", function (table) {
        table.dropColumn("end_time");
    });
};
```

### 2.3 旧版迁移系统（向后兼容）

#### 2.3.1 版本号系统

```javascript
static latestVersion = 10;  // 已弃用

static async patchSqlite() {
    let version = parseInt(await setting("database_version"));
    
    if (!version) {
        version = 0;
    }
    
    // 执行版本 1 到 10 的 SQL 补丁
    for (let i = version + 1; i <= this.latestVersion; i++) {
        const sqlFile = `./db/old_migrations/patch${i}.sql`;
        await Database.importSQLFile(sqlFile);
        await setSetting("database_version", i);
    }
    
    // 继续执行补丁文件系统
    await this.patchSqlite2();
}
```

#### 2.3.2 补丁文件系统

使用 `patchList` 配置（`server/database.js:71`）：

```javascript
static patchList = {
    "patch-setting-value-type.sql": true,
    "patch-add-other-auth.sql": { 
        parents: ["patch-monitor-basic-auth.sql"]  // 依赖关系
    },
    // ... 更多补丁
};
```

递归执行补丁的逻辑（`server/database.js:676`）：

```javascript
static async patch2Recursion(sqlFilename, databasePatchedFiles) {
    // 检查是否已修补
    if (!databasePatchedFiles[sqlFilename]) {
        // 先执行依赖
        if (value.parents) {
            for (let parentSQLFilename of value.parents) {
                await this.patch2Recursion(parentSQLFilename, databasePatchedFiles);
            }
        }
        
        // 执行当前补丁
        await this.importSQLFile("./db/old_migrations/" + sqlFilename);
        databasePatchedFiles[sqlFilename] = true;
    }
}
```

---

## 3. 应用配置持久化方式

### 3.1 数据库配置文件

Uptime Kuma 使用 `db-config.json` 存储数据库连接配置（`server/database.js:170-193`）：

```javascript
// 读取配置
static readDBConfig() {
    let dbConfigString = fs.readFileSync(
        path.join(Database.dataDir, "db-config.json")
    ).toString("utf-8");
    return JSON.parse(dbConfigString);
}

// 写入配置
static writeDBConfig(dbConfig) {
    fs.writeFileSync(
        path.join(Database.dataDir, "db-config.json"), 
        JSON.stringify(dbConfig, null, 4)
    );
}
```

**配置优先级**（`server/setup-database.js:69-111`）：
1. 环境变量（`UPTIME_KUMA_DB_*`）
2. `db-config.json` 文件
3. 默认配置（SQLite）

### 3.2 设置表（setting）

应用配置存储在 `setting` 表中，由 `Settings` 类管理（`server/settings.js`）。

#### 3.2.1 表结构

```javascript
// db/knex_init_db.js:383
await knex.schema.createTable("setting", (table) => {
    table.increments("id");
    table.string("key", 200).notNullable().unique();
    table.text("value");
    table.string("type", 20);
});
```

#### 3.2.2 核心方法

**读取配置**（`server/settings.js:28`）：
```javascript
static async get(key) {
    // 检查缓存
    if (key in Settings.cacheList) {
        return Settings.cacheList[key].value;
    }
    
    // 从数据库读取
    let value = await R.getCell("SELECT `value` FROM setting WHERE `key` = ? ", [key]);
    
    // 尝试 JSON 解析
    try {
        const v = JSON.parse(value);
        Settings.cacheList[key] = {
            value: v,
            timestamp: Date.now(),
        };
        return v;
    } catch (e) {
        return value;  // 返回原始字符串
    }
}
```

**写入配置**（`server/settings.js:73`）：
```javascript
static async set(key, value, type = null) {
    let bean = await R.findOne("setting", " `key` = ? ", [key]);
    if (!bean) {
        bean = R.dispense("setting");
        bean.key = key;
    }
    bean.type = type;
    bean.value = JSON.stringify(value);  // 值序列化为 JSON
    await R.store(bean);
    
    Settings.deleteCache([key]);
}
```

#### 3.2.3 缓存机制

- 所有读取的值都缓存在 `Settings.cacheList`
- 缓存每 60 秒自动清理过期项
- 写入时立即清除对应缓存

```javascript
if (!Settings.cacheCleaner) {
    Settings.cacheCleaner = setInterval(() => {
        for (key in Settings.cacheList) {
            if (Date.now() - Settings.cacheList[key].timestamp > 60 * 1000) {
                delete Settings.cacheList[key];
            }
        }
    }, 60 * 1000);
}
```

#### 3.2.4 按类型管理

支持按类型批量读写设置（`server/settings.js:91-136`）：

```javascript
// 获取某类型的所有设置
static async getSettings(type) {
    let list = await R.getAll("SELECT `key`, `value` FROM setting WHERE `type` = ? ", [type]);
    let result = {};
    for (let row of list) {
        try {
            result[row.key] = JSON.parse(row.value);
        } catch (e) {
            result[row.key] = row.value;
        }
    }
    return result;
}

// 批量设置某类型的设置
static async setSettings(type, data) {
    let keyList = Object.keys(data);
    let promiseList = [];
    
    for (let key of keyList) {
        let bean = await R.findOne("setting", " `key` = ? ", [key]);
        if (bean == null) {
            bean = R.dispense("setting");
            bean.type = type;
            bean.key = key;
        }
        if (bean.type === type) {
            bean.value = JSON.stringify(data[key]);
            promiseList.push(R.store(bean));
        }
    }
    await Promise.all(promiseList);
    Settings.deleteCache(keyList);
}
```

### 3.3 数据目录结构

数据目录默认位置：`./data/`（可通过 `DATA_DIR` 环境变量或 `--data-dir` 参数指定）

```
data/
├── kuma.db              # SQLite 数据库文件
├── db-config.json       # 数据库配置
├── upload/              # 用户上传文件
├── screenshots/         # 浏览器监控截图
└── docker-tls/          # Docker TLS 证书
```

---

## 4. 跨版本升级兼容处理策略

### 4.1 三层迁移系统

Uptime Kuma 采用三层迁移系统确保向后兼容：

| 层级 | 系统 | 使用场景 | 追踪方式 |
|------|------|---------|---------|
| 第 1 层 | 版本号系统 | 最早版本（1-10） | `database_version` |
| 第 2 层 | 补丁文件系统 | 中间版本 | `databasePatchedFiles` |
| 第 3 层 | Knex 迁移系统 | 当前版本 | `knex_migrations` 表 |

**执行顺序**（`server/database.js:468-503`）：
1. SQLite：`patchSqlite()` → `patchSqlite2()`
2. 所有数据库：`R.knex.migrate.latest()`
3. 数据迁移：`migrateAggregateTable()`

### 4.2 数据库类型初始化

#### 4.2.1 SQLite 初始化

从模板数据库复制（`server/database.js:258-260`）：
```javascript
if (dbConfig.type === "sqlite") {
    if (!fs.existsSync(Database.sqlitePath)) {
        log.info("server", "Copying Database");
        fs.copyFileSync(Database.templatePath, Database.sqlitePath);
    }
    // ...
}
```

#### 4.2.2 MariaDB 初始化

首次连接时创建表结构（`server/database.js:450-459`）：
```javascript
static async initMariaDB() {
    let hasTable = await R.hasTable("docker_host");
    if (!hasTable) {
        const { createTables } = require("../db/knex_init_db");
        await createTables();
    }
}
```

### 4.3 数据迁移示例

#### 4.3.1 状态页配置迁移

将设置表中的状态页配置迁移到独立的 `status_page` 表（`server/database.js:610-666`）：

```javascript
static async migrateNewStatusPage() {
    // 修复空 slug
    await R.exec("UPDATE status_page SET slug = 'empty-slug-recover' WHERE TRIM(slug) = ''");
    
    let title = await setting("title");
    
    if (title) {
        // 检查是否已迁移
        let statusPageCheck = await R.findOne("status_page", " slug = 'default' ");
        if (statusPageCheck !== null) {
            return;  // 已迁移，跳过
        }
        
        // 创建默认状态页
        let statusPage = R.dispense("status_page");
        statusPage.slug = "default";
        statusPage.title = title;
        statusPage.description = await setting("description");
        statusPage.icon = await setting("icon");
        // ... 更多字段
        let id = await R.store(statusPage);
        
        // 更新关联数据
        await R.exec("UPDATE incident SET status_page_id = ? WHERE status_page_id IS NULL", [id]);
        await R.exec("UPDATE [group] SET status_page_id = ? WHERE status_page_id IS NULL", [id]);
        
        // 清理旧设置
        await R.exec("DELETE FROM setting WHERE type = 'statusPage'");
        
        // 更新入口页设置
        let entryPage = await setting("entryPage");
        if (entryPage === "statusPage") {
            await setSetting("entryPage", "statusPage-default", "general");
        }
    }
}
```

#### 4.3.2 聚合表迁移（V1 → V2）

将 heartbeat 表的数据迁移到新的统计聚合表（`server/database.js:819-946`）：

```javascript
static async migrateAggregateTable(port, hostname) {
    // 检查迁移状态
    let migrateState = await Settings.get("migrateAggregateTableState");
    if (migrateState === "migrated") {
        return;  // 已完成
    }
    if (migrateState === "migrating") {
        throw new Error("Aggregate table migration is already in progress");
    }
    
    // 启动迁移进度服务器（可选）
    let migrationServer;
    if (port) {
        migrationServer = new SimpleMigrationServer();
        await migrationServer.start(port, hostname);
    }
    
    // 设置为迁移中
    await Settings.set("migrateAggregateTableState", "migrating");
    
    // 遍历所有监控，重新计算统计数据
    let monitors = await R.getAll(`SELECT DISTINCT monitor_id FROM heartbeat`);
    for (const [i, monitor] of monitors.entries()) {
        let dates = await R.getAll(
            `SELECT DISTINCT DATE(time) AS date FROM heartbeat WHERE monitor_id = ?`,
            [monitor.monitor_id]
        );
        
        for (const [dateIndex, date] of dates.entries()) {
            let calculator = new UptimeCalculator();
            calculator.monitorID = monitor.monitor_id;
            calculator.setMigrationMode(true);
            
            let heartbeats = await R.getAll(
                `SELECT status, ping, time FROM heartbeat WHERE monitor_id = ? AND DATE(time) = ?`,
                [monitor.monitor_id, date.date]
            );
            
            for (let heartbeat of heartbeats) {
                await calculator.update(
                    heartbeat.status, 
                    parseFloat(heartbeat.ping), 
                    dayjs(heartbeat.time)
                );
            }
        }
    }
    
    // 清理旧数据
    await Database.clearHeartbeatData(true);
    
    // 标记完成
    await Settings.set("migrateAggregateTableState", "migrated");
    await migrationServer?.stop();
}
```

### 4.4 降级处理

当数据库版本高于应用版本时（降级场景）（`server/database.js:494-502`）：

```javascript
catch (e) {
    if (e.message.includes("the following files are missing:")) {
        log.warn("db", e.message);
        log.warn("db", "Database migration failed, you may be downgrading Uptime Kuma.");
    } else {
        throw e;
    }
}
```

### 4.5 数据库连接兼容性

#### 4.5.1 SQLite 特殊配置

```javascript
// server/database.js:419-443
static async initSQLite(rawConn, testMode) {
    const asyncRun = (sql) => {
        return new Promise((resolve, reject) => 
            rawConn.run(sql, (err) => (err ? reject(err) : resolve()))
        );
    };
    
    // 日志模式
    if (testMode) {
        await asyncRun("PRAGMA journal_mode = MEMORY");
    } else {
        await asyncRun("PRAGMA journal_mode = WAL");
    }
    
    await asyncRun("PRAGMA foreign_keys = ON");
    await asyncRun("PRAGMA cache_size = -12000");
    await asyncRun("PRAGMA auto_vacuum = INCREMENTAL");
    await asyncRun("PRAGMA busy_timeout = 5000");
    await asyncRun("PRAGMA synchronous = NORMAL");
}
```

#### 4.5.2 连接池配置

```javascript
// SQLite 默认单连接（避免 SQLITE_BUSY）
if (process.env.UPTIME_KUMA_SQLITE_SINGLE_CONNECTION !== "false") {
    poolConfig = {
        min: 1,
        max: 1,
    };
}

// MariaDB 连接池
let mariadbPoolConfig = {
    min: 0,
    max: parsedMaxPoolConnections,  // 默认 10，最大 100
    idleTimeoutMillis: 30000,
};
```

### 4.6 迁移服务器（进度展示）

对于大版本升级，提供可视化迁移进度（`server/utils/simple-migration-server.js`）：

```javascript
// 启动迁移服务器
if (port) {
    migrationServer = new SimpleMigrationServer();
    await migrationServer.start(port, hostname);
}

// 更新进度
migrationServer?.update(msg);
```

---

## 5. 核心设计要点总结

### 5.1 迁移系统设计原则

1. **渐进式迁移**：从简单的版本号系统演进到 Knex 迁移框架
2. **向后兼容**：所有旧迁移系统保持可用，支持从任意版本升级
3. **依赖管理**：补丁文件支持依赖声明，确保执行顺序正确
4. **幂等性**：迁移检查确保同一迁移不会重复执行
5. **状态追踪**：使用数据库设置追踪迁移进度

### 5.2 配置持久化设计

1. **JSON 序列化**：所有配置值统一序列化为 JSON 存储
2. **类型分组**：支持按类型批量管理设置
3. **缓存策略**：60 秒缓存减少数据库查询
4. **灵活读取**：自动尝试 JSON 解析，失败则返回原始值

### 5.3 跨版本兼容策略

1. **多数据库支持**：使用 Knex 抽象层兼容 SQLite 和 MariaDB
2. **数据迁移**：大版本升级时提供专用数据迁移逻辑
3. **降级支持**：缺失迁移文件时给出警告而非崩溃
4. **进度反馈**：大迁移提供可视化进度展示
5. **安全措施**：
   - SQLite 迁移时临时禁用外键
   - 使用事务保护（可配置禁用）
   - 迁移前检查目标表是否为空

---

## 6. 相关文件索引

| 文件路径 | 职责 |
|---------|------|
| `server/database.js` | 数据库核心类，迁移执行逻辑 |
| `server/settings.js` | 设置表管理，缓存机制 |
| `server/setup-database.js` | 数据库配置引导 |
| `server/config.js` | 服务器配置（端口、SSL 等） |
| `db/knex_init_db.js` | MariaDB 初始表结构 |
| `db/knex_migrations/` | Knex 迁移脚本目录 |
| `db/knex_migrations/README.md` | 迁移脚本规范 |
| `db/old_migrations/` | 旧版 SQL 补丁（向后兼容） |
| `server/utils/simple-migration-server.js` | 大迁移进度展示服务器 |

---

## 7. 版本历史

- **v1.x**：使用 `patchX.sql` 和 `database_version`
- **过渡版本**：引入 `patchList` 和 `databasePatchedFiles`
- **v2.x+**：采用 Knex.js 迁移框架，引入聚合表统计

---

**报告生成日期**：2026-05-10
