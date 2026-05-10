# Uptime Kuma 数据库迁移与配置持久化分析报告

## 1. 概述

本文档深入分析 Uptime Kuma 项目中的数据库迁移机制、应用配置持久化方式，以及跨版本升级时的兼容处理策略。重点澄清三个核心问题：

1. **Setup 流程**：`db-config.json`、`kuma.db`、`needSetup` 的真实分支与优先级
2. **Setting 机制**：持久化方式、JSON 与非 JSON 值的差异、缓存条件
3. **聚合迁移**：中断、跳过、恢复场景下的语义与风险

---

## 2. Setup 流程：db-config.json、kuma.db、needSetup 的真实分支与优先级

### 2.1 决策流程图

`SetupDatabase` 类的构造函数（`server/setup-database.js:66-112`）实现了复杂的决策逻辑。以下是真实的优先级和分支流程：

```
启动 SetupDatabase 构造函数
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  第一阶段：读取 db-config.json                        │
└─────────────────────────────────────────────────────┘
    │
    ├─ 读取成功 ────► needSetup = false ───────────────────► 进入第二阶段
    │
    └─ 读取失败（文件不存在或无效）
        │
        ├─ 检查 kuma.db 是否存在
        │   │
        │   ├─ 存在（1.X 升级场景）
        │   │    │
        │   │    ├─ needSetup = false
        │   │    └─ 写入 db-config.json: {"type": "sqlite"}
        │   │
        │   └─ 不存在（全新安装）
        │        │
        │        └─ needSetup = true
        │
        └─ 进入第二阶段
             │
             ▼
┌─────────────────────────────────────────────────────┐
│  第二阶段：检查环境变量 UPTIME_KUMA_DB_TYPE           │
└─────────────────────────────────────────────────────┘
    │
    ├─ 环境变量已设置
    │   │
    │   ├─ needSetup = false（强制设为 false）
    │   └─ 用环境变量覆盖/写入 db-config.json
    │
    └─ 环境变量未设置 ────► 保持 needSetup 不变
```

### 2.2 优先级的真实顺序

**实际执行顺序**（重要：不是注释中说的 `env > db-config.json`）：

| 顺序 | 条件 | 操作 | needSetup 状态 |
|------|------|------|---------------|
| 1 | 读取 `db-config.json` 成功 | 跳过 kuma.db 检查 | `false` |
| 2 | `db-config.json` 失败 且 `kuma.db` 存在 | 自动生成 `db-config.json`（type: sqlite） | `false` |
| 3 | `db-config.json` 失败 且 `kuma.db` 不存在 | 不生成配置文件 | `true` |
| 4 | **无论之前什么状态**，`UPTIME_KUMA_DB_TYPE` 存在 | 用环境变量覆盖 `db-config.json` | **强制 `false`** |

**关键发现**：
- 环境变量的优先级体现在**第二阶段的强制覆盖**，而不是跳过第一阶段
- 如果 `db-config.json` 已存在且有效，`kuma.db` 检查根本不会执行
- `needSetup = true` 的唯一情况是：`db-config.json` 不存在/无效 **且** `kuma.db` 不存在 **且** 没有设置 `UPTIME_KUMA_DB_TYPE`

### 2.3 分支场景详解

#### 场景 A：全新安装（无任何遗留）

```
条件：
  - data/db-config.json 不存在
  - data/kuma.db 不存在
  - UPTIME_KUMA_DB_TYPE 未设置

执行路径：
  1. readDBConfig() 抛出异常 → 进入 catch
  2. fs.existsSync(kuma.db) 返回 false
  3. needSetup = true
  4. 检查 UPTIME_KUMA_DB_TYPE：未设置 → 不改变

结果：
  - needSetup = true
  - 显示设置页面，等待用户选择数据库类型
```

#### 场景 B：1.X 版本升级（只有 kuma.db）

```
条件：
  - data/db-config.json 不存在
  - data/kuma.db 存在（1.X 的数据库）
  - UPTIME_KUMA_DB_TYPE 未设置

执行路径：
  1. readDBConfig() 抛出异常 → 进入 catch
  2. fs.existsSync(kuma.db) 返回 true
  3. needSetup = false
  4. 写入 db-config.json: {"type": "sqlite"}
  5. 检查 UPTIME_KUMA_DB_TYPE：未设置 → 不改变

结果：
  - needSetup = false
  - 自动生成 db-config.json，指向现有的 kuma.db
  - 后续启动时会执行所有必要的数据库迁移
```

#### 场景 C：已配置环境（db-config.json 存在）

```
条件：
  - data/db-config.json 存在且有效
  - data/kuma.db 可能存在或不存在（取决于配置的数据库类型）

执行路径：
  1. readDBConfig() 成功
  2. needSetup = false
  3. **跳过 kuma.db 检查**（这是关键！）
  4. 如果有 UPTIME_KUMA_DB_TYPE，覆盖配置

结果：
  - needSetup = false
  - 直接使用 db-config.json 的配置
```

#### 场景 D：环境变量强制指定（Docker/容器部署）

```
条件：
  - UPTIME_KUMA_DB_TYPE 已设置（例如：UPTIME_KUMA_DB_TYPE=mariadb）
  - 其他文件可能存在或不存在

执行路径：
  1. 先执行第一阶段（读取配置或检查 kuma.db）
  2. 然后检查 UPTIME_KUMA_DB_TYPE
  3. needSetup = false（强制）
  4. 用环境变量值覆盖/写入 db-config.json

关键：
  - 无论第一阶段的结果如何，环境变量都会将 needSetup 设为 false
  - 环境变量的值会写入 db-config.json 文件
```

### 2.4 关键代码剖析

```javascript
// server/setup-database.js:77-96
try {
    dbConfig = Database.readDBConfig();
    log.debug("setup-database", "db-config.json is found and is valid");
    this.needSetup = false;
} catch (e) {
    log.info("setup-database", "db-config.json is not found or invalid: " + e.message);

    // 只有在 db-config.json 读取失败时，才检查 kuma.db
    if (fs.existsSync(path.join(Database.dataDir, "kuma.db"))) {
        this.needSetup = false;
        log.info("setup-database", "kuma.db is found, generate db-config.json");
        Database.writeDBConfig({
            type: "sqlite",
        });
    } else {
        this.needSetup = true;
    }
    dbConfig = {};
}

// server/setup-database.js:98-111
// 这个块是独立的，无论上面什么结果都会执行
if (process.env.UPTIME_KUMA_DB_TYPE) {
    this.needSetup = false;  // 强制设为 false！
    log.info("setup-database", "UPTIME_KUMA_DB_TYPE is provided by env, try to override db-config.json");
    // ... 用环境变量写入 db-config.json
    Database.writeDBConfig(dbConfig);
}
```

### 2.5 配置优先级总结

```
最终配置来源优先级（从高到低）：
1. UPTIME_KUMA_DB_* 环境变量（会写入 db-config.json）
2. data/db-config.json 文件
3. 自动生成（从 kuma.db 推断为 sqlite）
4. 用户通过设置页面输入

needSetup 决策优先级：
1. UPTIME_KUMA_DB_TYPE 存在 → false（最高优先级，强制覆盖）
2. db-config.json 有效 → false
3. kuma.db 存在 → false（自动生成 db-config.json）
4. 以上都不满足 → true（显示设置页面）
```

---

## 3. Setting 持久化与缓存机制

### 3.1 两个 Setting API

Uptime Kuma 有两套 Setting API，存在历史演进关系：

| API | 位置 | 实现 |
|-----|------|------|
| 旧 API（兼容性） | `server/util-server.js:384-397` | `setting()` / `setSetting()` → 委托给 Settings 类 |
| 新 API（推荐） | `server/settings.js` | `Settings` 类 |

```javascript
// server/util-server.js:384-397
exports.setting = async function (key) {
    return await Settings.get(key);
};

exports.setSetting = async function (key, value, type = null) {
    await Settings.set(key, value, type);
};
```

### 3.2 持久化机制：全 JSON 序列化

**重要修正**：所有值的写入都是 JSON 序列化，不存在"非 JSON 值"的写入路径。

```javascript
// server/settings.js:73-84
static async set(key, value, type = null) {
    let bean = await R.findOne("setting", " `key` = ? ", [key]);
    if (!bean) {
        bean = R.dispense("setting");
        bean.key = key;
    }
    bean.type = type;
    bean.value = JSON.stringify(value);  // 统一 JSON.stringify！
    await R.store(bean);
    
    Settings.deleteCache([key]);
}
```

**写入时的值处理**：

| 输入值 | JSON.stringify 结果 | 数据库存储 |
|--------|-------------------|-----------|
| `"hello"` | `"\"hello\""` | `text` 类型 |
| `123` | `"123"` | `text` 类型 |
| `true` | `"true"` | `text` 类型 |
| `{a: 1}` | `"{\"a\":1}"` | `text` 类型 |
| `null` | `"null"` | `text` 类型 |
| `undefined` | `"undefined"` | `text` 类型 |

**关键发现**：
- 无论输入什么类型，`set()` 都会用 `JSON.stringify()` 序列化
- 数据库的 `value` 列是 `text` 类型，所有值都存储为字符串
- 没有分支逻辑判断"是否需要 JSON 序列化"

### 3.3 读取机制：JSON 解析 + 失败回退

读取时才存在"JSON 值"和"非 JSON 值"的区分：

```javascript
// server/settings.js:28-64
static async get(key) {
    // 1. 启动缓存清理器（懒加载）
    if (!Settings.cacheCleaner) {
        Settings.cacheCleaner = setInterval(() => {
            for (key in Settings.cacheList) {
                if (Date.now() - Settings.cacheList[key].timestamp > 60 * 1000) {
                    delete Settings.cacheList[key];
                }
            }
        }, 60 * 1000);
    }

    // 2. 检查内存缓存
    if (key in Settings.cacheList) {
        const v = Settings.cacheList[key].value;
        log.debug("settings", `Get Setting (cache): ${key}: ${v}`);
        return v;  // 直接返回缓存值，不做任何处理
    }

    // 3. 从数据库读取原始字符串
    let value = await R.getCell("SELECT `value` FROM setting WHERE `key` = ? ", [key]);

    // 4. 尝试 JSON 解析
    try {
        const v = JSON.parse(value);
        log.debug("settings", `Get Setting: ${key}: ${v}`);
        
        // 缓存解析后的值
        Settings.cacheList[key] = {
            value: v,
            timestamp: Date.now(),
        };

        return v;
    } catch (e) {
        // 5. 解析失败：返回原始字符串（不缓存！）
        return value;
    }
}
```

**读取时的值处理**：

| 数据库存储 | JSON.parse 结果 | 返回值 | 是否缓存 |
|-----------|----------------|--------|---------|
| `"\"hello\""` | `"hello"`（字符串） | `"hello"` | ✅ 是 |
| `"123"` | `123`（数字） | `123` | ✅ 是 |
| `"true"` | `true`（布尔） | `true` | ✅ 是 |
| `"{\"a\":1}"` | `{a: 1}`（对象） | `{a: 1}` | ✅ 是 |
| `"null"` | `null` | `null` | ✅ 是 |
| `"undefined"` | `undefined` | `undefined` | ✅ 是 |
| `"hello"`（无引号） | SyntaxError | `"hello"`（原始字符串） | ❌ **否** |
| `""`（空字符串） | SyntaxError | `""`（原始字符串） | ❌ **否** |
| `undefined`（DB 无此 key） | SyntaxError | `undefined` | ❌ **否** |

### 3.4 缓存机制的准确描述

#### 缓存数据结构

```javascript
Settings.cacheList = {
    key1: {
        value: "解析后的值",      // JSON.parse 的结果
        timestamp: 1715347200000, // 写入缓存的时间戳
    },
    key2: {
        value: { a: 1, b: 2 },
        timestamp: 1715347200000,
    },
};
```

#### 缓存条件（写入缓存的时机）

**只有满足以下所有条件才会写入缓存**：

1. **通过 `Settings.get(key)` 读取**（`getSettings()` 方法的读取不经过此缓存）
2. **从数据库成功读取到值**（不是 `undefined`）
3. **JSON.parse() 解析成功**（没有抛出异常）

**不会缓存的情况**：

1. `JSON.parse()` 抛出异常（解析失败）
2. 数据库中不存在该 key（`R.getCell()` 返回 `undefined`）
3. 通过 `Settings.getSettings(type)` 批量读取（没有缓存逻辑）

#### 缓存生命周期

```
调用 Settings.get("someKey")
    │
    ├─ 检查缓存：命中 ───► 直接返回缓存值 ───► 结束（不更新 timestamp）
    │
    └─ 缓存未命中
         │
         ├─ 从 DB 读取
         │
         ├─ 尝试 JSON.parse()
         │   │
         │   ├─ 成功 ───► 写入缓存（timestamp = Date.now()）
         │   │               │
         │   │               └─ 缓存有效期：60 秒
         │   │
         │   └─ 失败 ───► 不写入缓存
         │
         └─ 返回值
```

#### 缓存清理机制

```javascript
// 每 60 秒执行一次
for (key in Settings.cacheList) {
    // 如果距离写入时间超过 60 秒，删除
    if (Date.now() - Settings.cacheList[key].timestamp > 60 * 1000) {
        delete Settings.cacheList[key];
    }
}
```

**清理特性**：
- 清理器是**懒启动**的：第一次调用 `get()` 时才启动
- 清理是**被动的**：只在清理器运行时才删除，不会自动过期
- 缓存命中时**不更新** timestamp：从写入时开始算 60 秒，不是从访问时

#### 缓存失效（主动删除）

```javascript
// server/settings.js:143-146
static deleteCache(keyList) {
    for (let key of keyList) {
        delete Settings.cacheList[key];
    }
}
```

**触发主动删除的时机**：

1. `Settings.set()` 调用时 → `Settings.deleteCache([key])`
2. `Settings.setSettings()` 调用时 → `Settings.deleteCache(keyList)`

**注意**：`deleteCache` 只清除内存缓存，不影响数据库。

### 3.5 getSettings() 的特殊行为

`getSettings()` 方法有独立的逻辑，**不使用** `cacheList` 缓存：

```javascript
// server/settings.js:91-105
static async getSettings(type) {
    // 直接查询数据库，不检查 cacheList
    let list = await R.getAll("SELECT `key`, `value` FROM setting WHERE `type` = ? ", [type]);

    let result = {};

    for (let row of list) {
        try {
            result[row.key] = JSON.parse(row.value);
        } catch (e) {
            // 解析失败也直接返回原始值，同样不缓存
            result[row.key] = row.value;
        }
    }

    return result;
}
```

**getSettings() vs get() 的区别**：

| 特性 | `get(key)` | `getSettings(type)` |
|------|-----------|--------------------|
| 缓存 | 使用 `cacheList` | 不使用任何缓存 |
| 查询方式 | 单条查询 | 按类型批量查询 |
| 解析失败处理 | 不缓存 | 不缓存（本来就没缓存） |
| 性能特征 | 热点数据快，冷数据慢 | 每次都查库，稳定 |

### 3.6 Setting 机制总结图

```
写入路径（统一 JSON 序列化）：
  set(key, value)
       │
       ▼
  JSON.stringify(value)
       │
       ▼
  写入 setting 表的 value 列（text 类型）
       │
       ▼
  deleteCache([key])  ───► 清除内存缓存


读取路径（JSON 解析 + 条件缓存）：
  get(key)
       │
       ├─ 命中 cacheList ──────────────────► 返回缓存值
       │
       └─ 未命中
            │
            ▼
       SELECT value FROM setting
            │
            ▼
       JSON.parse(value)
            │
            ├─ 成功 ───► 写入 cacheList（timestamp = now）
            │                  │
            │                  └─ 返回解析后的值
            │
            └─ 失败 ───► 返回原始字符串（不缓存）
```

---

## 4. 聚合迁移：中断、跳过、恢复场景的兼容语义与风险

### 4.1 聚合迁移概述

`migrateAggregateTable()` 是 V1 → V2 版本的**数据迁移**（不是 schema 迁移），负责将旧版 `heartbeat` 表的原始数据转换为新版的统计聚合表（`stat_minutely`、`stat_hourly`、`stat_daily`）。

**迁移核心逻辑**（`server/database.js:819-946`）：

```
┌─────────────────────────────────────────────────────────┐
│  migrateAggregateTable() 执行流程                        │
├─────────────────────────────────────────────────────────┤
│  1. 检查 SET_MIGRATE_AGGREGATE_TABLE_TO_TRUE 环境变量    │
│     └─ 如设为 1：直接设为 migrated，跳过所有逻辑          │
│                                                         │
│  2. 读取 migrateAggregateTableState 设置                 │
│     ├─ "migrated"  ──► 直接返回（已完成）                 │
│     ├─ "migrating" ──► throw Error（拒绝启动）           │
│     └─ 其他/空     ──► 继续                              │
│                                                         │
│  3. 检查 stat_* 表是否为空                               │
│     └─ 任一表有数据 ──► 直接返回（不启动迁移）             │
│                                                         │
│  4. 设置状态为 "migrating"                               │
│                                                         │
│  5. 遍历所有 monitor，遍历所有日期，重新计算统计           │
│     └─ 写入 stat_minutely / stat_hourly / stat_daily     │
│                                                         │
│  6. 清理旧的 heartbeat 数据                              │
│                                                         │
│  7. 设置状态为 "migrated"                                │
└─────────────────────────────────────────────────────────┘
```

### 4.2 状态机设计

迁移状态存储在 `setting` 表中，key 为 `migrateAggregateTableState`：

| 状态值 | 含义 | 后续行为 |
|--------|------|---------|
| `undefined` / `""` / 不存在 | 未开始迁移 | 尝试执行迁移 |
| `"migrating"` | 迁移中（或已中断） | **抛出错误，拒绝启动** |
| `"migrated"` | 迁移完成 | 跳过迁移 |

### 4.3 中断场景分析

#### 4.3.1 中断发生的时机

迁移过程是**非事务性**的（代码注释明确说明），中断可能发生在：

```
阶段 1：状态检查 ──► 阶段 2：写入 "migrating" ──► 阶段 3：数据处理 ──► 阶段 4：清理 ──► 阶段 5：写入 "migrated"
                        │                      │                      │
                        │                      │                      └─ 已部分/全部写入 stat_*
                        │                      │                      └─ 可能已清理部分 heartbeat
                        │                      │
                        │                      └─ 可能已部分写入 stat_*
                        │
                        └─ 状态卡住
```

#### 4.3.2 中断后的行为

如果在阶段 3-5 中断，下次启动时：

```javascript
// server/database.js:838-841
} else if (migrateState === "migrating") {
    log.warn("db", "Aggregate table migration is already in progress, or it was interrupted");
    throw new Error("Aggregate table migration is already in progress");
}
```

**结果**：
- 进程启动失败
- 日志警告：迁移可能已中断
- 抛出错误，需要人工干预

#### 4.3.3 中断的风险

| 中断时机 | stat_* 表状态 | heartbeat 表状态 | 数据一致性 |
|---------|--------------|-----------------|-----------|
| 阶段 2 刚完成 | 空 | 完整 | 一致（未开始） |
| 阶段 3 进行中 | 部分 monitor 已处理 | 完整 | **不一致** |
| 阶段 4（清理）进行中 | 可能完整 | 部分已清理 | **不一致** |
| 阶段 5 完成前 | 完整 | 已清理 | 实际一致，但状态卡住 |

**最危险的场景**：阶段 4 中断（`clearHeartbeatData()` 执行中）
- `stat_*` 已完整写入
- `heartbeat` 部分删除
- 如果强行设为 `migrated`：数据可能丢失
- 如果强行重置：需要重新计算已清理的 heartbeat

### 4.4 跳过场景分析

#### 4.4.1 跳过方式一：环境变量（永久跳过）

```javascript
// server/database.js:823-829
if (process.env.SET_MIGRATE_AGGREGATE_TABLE_TO_TRUE === "1") {
    log.warn("db", "SET_MIGRATE_AGGREGATE_TABLE_TO_TRUE is set to 1, skipping aggregate table migration forever");
    await Settings.set("migrateAggregateTableState", "migrated");
}
```

**用途**：为 2.0.0-dev 版本用户提供的逃逸口

**效果**：
- 将状态永久设为 `"migrated"`
- 以后启动都不再检查
- **不会**清理 `heartbeat` 数据
- **不会**写入任何 `stat_*` 数据

**风险**：
- 统计功能（图表、历史数据）将无法使用
- 依赖聚合表的功能（如 uptime 计算）会出错
- 不会有任何警告或错误，静默失效

#### 4.4.2 跳过方式二：stat_* 表非空（自动跳过）

```javascript
// server/database.js:867-878
for (let table of ["stat_minutely", "stat_hourly", "stat_daily"]) {
    let countResult = await R.getRow(`SELECT COUNT(*) AS count FROM ${table}`);
    let count = countResult.count;
    if (count > 0) {
        log.warn("db", `Aggregate table ${table} is not empty, migration will not be started (Maybe you were using 2.0.0-dev?)`);
        await migrationServer?.stop();
        return;  // 直接返回，不设置任何状态
    }
}
```

**用途**：检测 2.0.0-dev 用户（他们可能已有部分聚合数据）

**行为**：
- 直接返回，不设置 `migrateAggregateTableState`
- 下次启动仍会重复此检查

**风险**：
- 如果是中断后残留了部分数据，会被误认为是 dev 版本
- 状态仍为 `"migrating"` 或空，下次可能报错
- **不会**清理 heartbeat 数据
- **不会**补全缺失的聚合数据

#### 4.4.3 跳过方式三：状态已为 migrated

```javascript
// server/database.js:835-837
if (migrateState === "migrated") {
    log.debug("db", "Migrated aggregate table already, skip");
    return;
}
```

**这是正常的跳过路径**，表示迁移已成功完成。

### 4.5 恢复场景分析

#### 4.5.1 官方恢复工具

项目提供了专门的恢复脚本 `extra/reset-migrate-aggregate-table-state.js`：

```javascript
// extra/reset-migrate-aggregate-table-state.js:6-21
const main = async () => {
    console.log("Connecting the database");
    Database.initDataDir(args);
    await Database.connect(false, false, true);

    console.log("Deleting all data from aggregate tables");
    await R.exec("DELETE FROM stat_minutely");
    await R.exec("DELETE FROM stat_hourly");
    await R.exec("DELETE FROM stat_daily");

    console.log("Resetting the aggregate table state");
    await Settings.set("migrateAggregateTableState", "");

    await Database.close();
    console.log("Done");
};
```

**恢复步骤**：

```bash
npm run reset-migrate-aggregate-table-state
```

**恢复流程**：

```
┌─────────────────────────────────────────┐
│  1. 删除 stat_minutely 所有数据          │
│  2. 删除 stat_hourly 所有数据            │
│  3. 删除 stat_daily 所有数据             │
│  4. 将 migrateAggregateTableState 设为空 │
└─────────────────────────────────────────┘
```

**关键设计决策**：**清空所有聚合表，从头开始**

#### 4.5.2 官方方案的隐含假设

这个恢复方案假设：

1. **heartbeat 数据仍然完整**：如果 `clearHeartbeatData()` 已部分执行，这个假设不成立
2. **重新计算所有数据是安全的**：对于大数据量，可能耗时很长
3. **部分写入的聚合数据没有价值**：直接全部删除

#### 4.5.3 无法自动恢复的场景

**场景：中断发生在清理阶段**

```
时间线：
  T1: migrateAggregateTableState = "migrating"
  T2: 所有 monitor 的 stat_* 数据写入完成
  T3: 开始执行 clearHeartbeatData()
  T4: 为 monitor 1、2、3 删除了旧的 heartbeat
  T5: 进程中断

此时状态：
  - migrateAggregateTableState = "migrating"
  - stat_*: 完整（所有 monitor 都有数据）
  - heartbeat: monitor 1-3 已清理，monitor 4+ 还完整

问题：
  - 如果运行官方恢复脚本：
    → 删除所有 stat_*
    → 但 monitor 1-3 的 heartbeat 已丢失
    → 重新计算时，monitor 1-3 没有原始数据！

  - 如果强行设为 migrated：
    → 跳过迁移
    → 但 monitor 1-3 的 heartbeat 可能没删干净
    → 后续可能有数据重复或冲突
```

**这是无法自动恢复的场景**，需要人工判断和处理。

### 4.6 风险矩阵

| 场景 | 行为 | 数据一致性 | 可恢复性 | 风险等级 |
|------|------|-----------|---------|---------|
| 正常完成 | 状态设为 migrated，清理完成 | ✅ 一致 | 无需恢复 | 低 |
| 计算阶段中断 | 状态为 migrating，报错 | ❌ 部分不一致 | ✅ 用官方脚本（前提：heartbeat 完整） | 中 |
| 清理阶段中断 | 状态为 migrating，报错 | ❌ 可能丢失数据 | ⚠️ 需人工检查 heartbeat | **高** |
| 使用环境变量跳过 | 状态强制设为 migrated | ⚠️ 聚合表无数据 | ❌ 永久跳过，除非手动操作 | 中 |
| stat_* 非空自动跳过 | 直接返回，不改状态 | ⚠️ 可能不完整 | ⚠️ 需判断是否为 dev 版本 | 中 |

### 4.7 迁移的非事务性设计说明

代码中的重要注释（`server/database.js:812-814`）：

```javascript
/**
 *  Normally, it should be in transaction, but UptimeCalculator wasn't designed to be in transaction before that.
 *  I don't want to heavily modify the UptimeCalculator, so it is not in transaction.
 *  Run `npm run reset-migrate-aggregate-table-state` to reset, in case the migration is interrupted.
 */
```

**设计权衡**：
- **选择**：不使用事务
- **原因**：`UptimeCalculator` 类在设计时没有考虑事务
- **代价**：需要提供恢复工具，接受中断风险
- **补偿**：清晰的日志、官方恢复脚本

---

## 5. 数据库迁移执行机制

### 5.1 迁移系统的演进

Uptime Kuma 的数据库迁移系统经历了三个阶段的演进：

| 阶段 | 系统 | 版本范围 | 追踪方式 |
|------|------|---------|---------|
| 第 1 层 | 版本号系统 | 最早版本 | `database_version` setting |
| 第 2 层 | 补丁文件系统 | 过渡版本 | `databasePatchedFiles` setting |
| 第 3 层 | Knex 迁移框架 | 当前版本 | `knex_migrations` 表 |

### 5.2 核心迁移入口

迁移的入口是 `Database.patch()` 方法（`server/database.js:468-504`）：

```javascript
static async patch(port = undefined, hostname = undefined) {
    // 第一部分：旧版迁移（仅 SQLite）
    if (Database.dbConfig.type === "sqlite") {
        await this.patchSqlite();  // 版本号系统 + 补丁文件系统
    }

    // 第二部分：Knex 迁移（所有数据库）
    try {
        // SQLite 特殊处理：临时禁用外键
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

        // 第三部分：数据迁移（V1 → V2 聚合表）
        await this.migrateAggregateTable(port, hostname);
    } catch (e) {
        // 处理降级场景：缺失迁移文件
        if (e.message.includes("the following files are missing:")) {
            log.warn("db", e.message);
            log.warn("db", "Database migration failed, you may be downgrading Uptime Kuma.");
        } else {
            throw e;
        }
    }
}
```

### 5.3 Knex 迁移系统规范

**迁移文件位置**：`db/knex_migrations/`

**文件命名规范**（`db/knex_migrations/README.md`）：
- 格式：`YYYY-MM-DD-HHMM-patch-name.js`
- 所有表必须有 `id` 主键
- 避免原生 SQL，使用 Knex 方法以兼容 SQLite 和 MariaDB

**标准模板**：
```javascript
exports.up = function (knex) {
    // 迁移逻辑
};

exports.down = function (knex) {
    // 回滚逻辑
};

// 可选：禁用事务（大表操作等场景）
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

### 5.4 旧版迁移系统（向后兼容）

#### 版本号系统

```javascript
// server/database.js:122
static latestVersion = 10;  // 已弃用的版本号系统

// server/database.js:517-561
static async patchSqlite() {
    let version = parseInt(await setting("database_version"));
    
    if (!version) {
        version = 0;
    }
    
    // 按顺序执行 patch1.sql 到 patch10.sql
    if (version < this.latestVersion) {
        for (let i = version + 1; i <= this.latestVersion; i++) {
            const sqlFile = `./db/old_migrations/patch${i}.sql`;
            await Database.importSQLFile(sqlFile);
            await setSetting("database_version", i);
        }
    }
    
    // 继续执行补丁文件系统
    await this.patchSqlite2();
}
```

#### 补丁文件系统

支持依赖关系的补丁追踪：

```javascript
// server/database.js:71-116
static patchList = {
    "patch-setting-value-type.sql": true,
    "patch-add-other-auth.sql": { 
        parents: ["patch-monitor-basic-auth.sql"]  // 依赖声明
    },
    // ... 更多补丁
};

// 递归执行（处理依赖）
static async patch2Recursion(sqlFilename, databasePatchedFiles) {
    if (!databasePatchedFiles[sqlFilename]) {
        // 先执行所有父依赖
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

## 6. 跨版本升级兼容策略

### 6.1 三层迁移系统

| 层级 | 系统 | 使用场景 | 追踪方式 | 仅 SQLite |
|------|------|---------|---------|-----------|
| 第 1 层 | 版本号系统 | 最早版本（1-10） | `database_version` | ✅ 是 |
| 第 2 层 | 补丁文件系统 | 中间版本 | `databasePatchedFiles` | ✅ 是 |
| 第 3 层 | Knex 迁移系统 | 当前版本 | `knex_migrations` 表 | ❌ 否（所有数据库） |

**执行顺序**：
1. SQLite：`patchSqlite()`（版本号）→ `patchSqlite2()`（补丁文件）
2. 所有数据库：`R.knex.migrate.latest()`（Knex）
3. 所有数据库：`migrateAggregateTable()`（数据迁移）

### 6.2 数据库初始化

#### SQLite 初始化

```javascript
// server/database.js:258-260
if (dbConfig.type === "sqlite") {
    if (!fs.existsSync(Database.sqlitePath)) {
        log.info("server", "Copying Database");
        fs.copyFileSync(Database.templatePath, Database.sqlitePath);
    }
}
```

- 从模板数据库 `./db/kuma.db` 复制
- 模板已包含基础表结构

#### MariaDB 初始化

```javascript
// server/database.js:450-459
static async initMariaDB() {
    let hasTable = await R.hasTable("docker_host");
    if (!hasTable) {
        const { createTables } = require("../db/knex_init_db");
        await createTables();
    }
}
```

- 首次连接时动态创建表
- 使用 `db/knex_init_db.js` 的完整 schema

### 6.3 数据迁移示例

#### 状态页配置迁移

将设置表中的状态页配置迁移到独立的 `status_page` 表（`server/database.js:610-666`）：

```javascript
static async migrateNewStatusPage() {
    // 修复数据问题
    await R.exec("UPDATE status_page SET slug = 'empty-slug-recover' WHERE TRIM(slug) = ''");
    
    let title = await setting("title");
    
    if (title) {
        // 幂等检查
        let statusPageCheck = await R.findOne("status_page", " slug = 'default' ");
        if (statusPageCheck !== null) {
            return;  // 已迁移，跳过
        }
        
        // 创建新记录
        let statusPage = R.dispense("status_page");
        statusPage.slug = "default";
        statusPage.title = title;
        statusPage.description = await setting("description");
        statusPage.icon = await setting("icon");
        // ...
        
        let id = await R.store(statusPage);
        
        // 更新关联数据
        await R.exec("UPDATE incident SET status_page_id = ? WHERE status_page_id IS NULL", [id]);
        await R.exec("UPDATE [group] SET status_page_id = ? WHERE status_page_id IS NULL", [id]);
        
        // 清理旧数据
        await R.exec("DELETE FROM setting WHERE type = 'statusPage'");
        
        // 更新引用
        let entryPage = await setting("entryPage");
        if (entryPage === "statusPage") {
            await setSetting("entryPage", "statusPage-default", "general");
        }
    }
}
```

### 6.4 降级处理

当应用版本低于数据库版本时（缺失迁移文件）：

```javascript
// server/database.js:494-502
catch (e) {
    if (e.message.includes("the following files are missing:")) {
        log.warn("db", e.message);
        log.warn("db", "Database migration failed, you may be downgrading Uptime Kuma.");
    } else {
        throw e;
    }
}
```

**降级策略**：
- 检测到缺失迁移文件时给出警告
- 不抛出致命错误，允许应用继续启动
- **风险**：新 schema 可能与旧代码不兼容

### 6.5 数据库连接兼容性

#### SQLite 特殊配置

```javascript
// server/database.js:419-443
static async initSQLite(rawConn, testMode) {
    // 日志模式
    if (testMode) {
        await asyncRun("PRAGMA journal_mode = MEMORY");
    } else {
        await asyncRun("PRAGMA journal_mode = WAL");
    }
    
    await asyncRun("PRAGMA foreign_keys = ON");
    await asyncRun("PRAGMA cache_size = -12000");
    await asyncRun("PRAGMA auto_vacuum = INCREMENTAL");
    await asyncRun("PRAGMA busy_timeout = 5000");  // 避免 SQLITE_BUSY
    await asyncRun("PRAGMA synchronous = NORMAL");
}
```

#### 连接池配置

```javascript
// SQLite 默认单连接
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

---

## 7. 核心设计要点总结

### 7.1 Setup 流程设计

1. **多阶段决策**：先检查配置文件，再检查遗留数据库，最后检查环境变量
2. **自动升级路径**：检测到 `kuma.db` 时自动生成 `db-config.json`
3. **环境变量强制**：`UPTIME_KUMA_DB_TYPE` 可强制覆盖所有决策
4. **幂等性**：配置文件存在时跳过 `kuma.db` 检查

### 7.2 Setting 机制设计

1. **写入统一**：所有值通过 `JSON.stringify()` 序列化
2. **读取容错**：`JSON.parse()` 失败时返回原始字符串
3. **条件缓存**：只有解析成功的值才写入缓存
4. **懒启动清理**：缓存清理器在第一次读取时启动
5. **双 API 兼容**：保留旧 `setting()` API 委托给新 `Settings` 类

### 7.3 迁移系统设计

1. **渐进式迁移**：从简单的版本号系统演进到 Knex 框架
2. **向后兼容**：所有旧迁移系统保持可用
3. **依赖管理**：补丁文件支持依赖声明
4. **幂等检查**：迁移前检查目标状态，避免重复执行
5. **状态追踪**：使用数据库设置追踪迁移进度
6. **非事务性权衡**：复杂数据迁移接受中断风险，提供恢复工具

### 7.4 跨版本兼容策略

1. **多数据库抽象**：使用 Knex 兼容 SQLite 和 MariaDB
2. **数据迁移分离**：schema 迁移和数据迁移分开处理
3. **降级支持**：缺失迁移文件时警告而非崩溃
4. **进度反馈**：大迁移提供可视化进度服务器
5. **安全措施**：
   - SQLite 迁移时临时禁用外键
   - 支持禁用事务（`exports.config = { transaction: false }`）
   - 迁移前检查表状态

---

## 8. 相关文件索引

| 文件路径 | 职责 |
|---------|------|
| `server/database.js` | 数据库核心类，迁移执行逻辑，聚合迁移 |
| `server/settings.js` | 设置表管理，缓存机制 |
| `server/util-server.js` | 旧版 setting API（委托给 Settings） |
| `server/setup-database.js` | Setup 决策流程，needSetup 状态 |
| `server/config.js` | 服务器配置（端口、SSL 等） |
| `db/knex_init_db.js` | MariaDB 初始表结构 |
| `db/knex_migrations/` | Knex 迁移脚本目录 |
| `db/knex_migrations/README.md` | 迁移脚本规范 |
| `db/old_migrations/` | 旧版 SQL 补丁（向后兼容） |
| `db/old_migrations/README.md` | 旧补丁说明（指向新目录） |
| `server/utils/simple-migration-server.js` | 大迁移进度展示服务器 |
| `extra/reset-migrate-aggregate-table-state.js` | 聚合迁移恢复工具 |

---

## 9. 版本历史

- **v1.x**：使用 `patchX.sql` 和 `database_version`
- **过渡版本**：引入 `patchList` 和 `databasePatchedFiles`
- **v2.0+**：
  - 采用 Knex.js 迁移框架
  - 引入聚合表统计（`stat_minutely` / `stat_hourly` / `stat_daily`）
  - 引入 `db-config.json` 配置文件
  - 支持 MariaDB 数据库

---

**报告生成日期**：2026-05-10
**报告版本**：2.0（重大修订，重点澄清 setup 流程、setting 机制、聚合迁移风险）
