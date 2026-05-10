# Uptime Kuma 数据库迁移与配置持久化分析报告

## 1. 概述

本文档深入分析 Uptime Kuma 项目中的数据库迁移机制、应用配置持久化方式，以及跨版本升级时的兼容处理策略。重点澄清四个核心问题：

1. **Setup 流程**：`db-config.json`、`kuma.db`、`needSetup` 的真实分支与优先级
2. **Setting 机制**：持久化方式、JSON 与非 JSON 值的差异、缓存条件、直接写入路径
3. **聚合迁移**：中断、跳过、恢复场景下的语义与风险
4. **兼容影响**：非标准值在各种读取路径下的行为

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
| 4 | **无论之前什么状态**，`UPTIME_KUMA_DB_TYPE` 存在 | 用环境变量覆盖 `db-config.json` | **强制 `false` |

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

### 3.1 三个读写路径概览

Uptime Kuma 的 setting 表存在**多条独立的读写路径**，这是理解其复杂行为的关键。

#### 3.1.1 三条写入路径

| 路径 | 写入方式 | JSON 序列化 | 使用示例 |
|------|---------|------------|---------|
| **路径 A** | `Settings.set()` | ✅ 是 | `title`、`description`、`checkUpdate` |
| **路径 B** | `Settings.setSettings()` | ✅ 是 | 批量设置某类型的配置 |
| **路径 C** | `initJWTSecret()` | ❌ 否 | `jwtSecret`（特殊历史遗留） |

#### 3.1.2 三条读取路径

| 路径 | 读取方式 | JSON 解析 | 使用缓存 | 使用示例 |
|------|---------|----------|---------|---------|
| **路径 X** | `Settings.get()` | ✅ 是 | ✅ 是（条件） | 大部分单个配置读取 |
| **路径 Y** | `Settings.getSettings()` | ✅ 是 | ❌ 否 | 按类型批量读取 |
| **路径 Z** | `R.findOne()` / `R.getCell()` | ❌ 否 | ❌ 否 | `jwtSecret`、迁移脚本内部 |

### 3.2 标准写入路径：JSON 序列化

#### 路径 A：Settings.set()

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

#### 路径 B：Settings.setSettings()

```javascript
// server/settings.js:117-134
for (let key of keyList) {
    let bean = await R.findOne("setting", " `key` = ? ", [key]);
    // ...
    if (bean.type === type) {
        bean.value = JSON.stringify(data[key]);  // 同样 JSON.stringify
        promiseList.push(R.store(bean));
    }
}
```

**标准写入时的值处理（注意：`JSON.stringify(undefined)` 返回 `undefined` 本身，不是字符串 `"undefined"`）

| 输入值 | JSON.stringify 结果 | `bean.value` 值 | 数据库存储（value 列） | 是否有效落库 |
|--------|---------------------|---------------|----------------------|-----------|
| `"hello"` | `"\"hello\""` | `"\"hello\""` | `"\"hello\""`（带引号的字符串） | ✅ 是 |
| `123` | `"123"` | `"123"` | `"123"`（数字字符串） | ✅ 是 |
| `true` | `"true"` | `"true"` | `"true"`（布尔字符串） | ✅ 是 |
| `{a: 1}` | `"{\"a\":1}"` | `"{\"a\":1}"` | `"{\"a\":1}"`（对象序列化） | ✅ 是 |
| `null` | `"null"` | `"null"` | `"null"`（null 字符串） | ✅ 是 |
| `undefined` | `undefined` | `undefined` | `NULL`（数据库 NULL） | ⚠️ 特殊情况 |

**关键发现**：
- `JSON.stringify(undefined)` 返回 `undefined` 本身（不是字符串 `"undefined"`）
- 所以 `bean.value = undefined` 会导致数据库中存储为 `NULL`
- 读取时 `JSON.parse(null)` 成功解析为 `null`，会被缓存
- 这意味着 `Settings.set(key, undefined)` 实际上会存储 `NULL`，读取时返回 `null`

### 3.3 特殊写入路径：initJWTSecret（绕过 JSON 序列化）

**这是代码库中**唯一**的直接写入路径，不经过 `JSON.stringify()`：

```javascript
// server/util-server.js:42-52
exports.initJWTSecret = async () => {
    let jwtSecretBean = await R.findOne("setting", " `key` = ? ", ["jwtSecret"]);

    if (!jwtSecretBean) {
        jwtSecretBean = R.dispense("setting");
        jwtSecretBean.key = "jwtSecret";
    }

    // 关键：直接设置 value，不经过 JSON.stringify
    jwtSecretBean.value = await passwordHash.generate(genSecret());
    await R.store(jwtSecretBean);
    return jwtSecretBean;
};
```

**jwtSecret 的写入值特点**：

- `passwordHash.generate()` 返回的是**原始字符串**（如 bcrypt hash）
- **没有** `JSON.stringify()`
- 数据库中存储的是**裸字符串**，没有 JSON 引号

**示例对比**：

| 写入方式 | 代码 | 数据库中存储的值 |
|---------|------|-----------------|
| 标准路径（Settings.set） | `bean.value = JSON.stringify("abc")` | `"\"abc\""`（带引号） |
| 特殊路径（initJWTSecret） | `bean.value = "abc"` | `"abc"`（裸字符串） |

### 3.4 历史遗留值的可能性

除了 `initJWTSecret` 这个明确的直接写入路径，数据库中可能还存在其他"非 JSON 值"：

#### 场景 1：1.X 版本的历史数据

在 `patch-setting-value-type.sql` 之前的版本中，setting 表可能有直接插入的原始值。迁移脚本只是重建表结构，**不会**对现有值进行重新编码：

```sql
-- db/old_migrations/patch-setting-value-type.sql
-- 只是复制数据，不做 JSON 转换
insert into setting_dg_tmp(id, key, value, type) select id, key, value, type from setting;
```

#### 场景 2：迁移脚本中的直接操作

某些迁移脚本可能直接使用 SQL INSERT/UPDATE 操作 setting 表，绕过 Settings API。

### 3.5 读取路径 X：Settings.get()（JSON 解析 + 条件缓存）

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
        return v;  // 直接返回缓存值
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

#### 对不同类型值的行为

**标准 JSON 值（通过 Settings.set 写入）：

| 数据库存储 | JSON.parse 结果 | 返回值 | 是否缓存 |
|-----------|----------------|--------|---------|
| `"\"hello\""` | `"hello"`（字符串） | `"hello"` | ✅ 是 |
| `"123"` | `123`（数字） | `123` | ✅ 是 |
| `"true"` | `true`（布尔） | `true` | ✅ 是 |
| `"{\"a\":1}"` | `{a: 1}`（对象） | `{a: 1}` | ✅ 是 |
| `"null"` | `null` | `null` | ✅ 是 |

**特殊非 JSON 值（initJWTSecret 或历史遗留）：

| 数据库存储 | JSON.parse 结果 | 返回值 | 是否缓存 |
|-----------|----------------|--------|---------|
| `"abc123xyz"`（裸字符串） | SyntaxError | `"abc123xyz"`（原始字符串） | ❌ **否** |
| `""`（空字符串） | SyntaxError | `""` | ❌ 否 |
| `undefined`（DB 无此 key） | SyntaxError | `undefined` | ❌ 否 |

**关键发现**：`jwtSecret` 如果用 `Settings.get("jwtSecret")` 读取：
- `JSON.parse()` 会**失败**（因为是裸字符串，不是合法 JSON）
- 返回原始字符串（值是正确的）
- **不会被缓存**

### 3.6 读取路径 Y：Settings.getSettings()（JSON 解析 + 无缓存）

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
            // 解析失败也直接返回原始值
            result[row.key] = row.value;
        }
    }

    return result;
}
```

#### getSettings() 的关键特性：

1. **不使用 `cacheList` 缓存**：每次都查数据库
2. **同样尝试 JSON 解析**：成功返回解析后的值，失败返回原始字符串
3. **按类型筛选**：只返回 `type` 列匹配的记录

#### 对非 JSON 值的行为：

| 值类型 | JSON.parse | 返回值 | 是否缓存 |
|--------|------------|--------|---------|
| 标准 JSON 值 | 成功 | 解析后的值 | ❌ 否（本来就不缓存） |
| 非 JSON 值（如 jwtSecret） | 失败 | 原始字符串 | ❌ 否 |

**jwtSecret 的特殊性：`jwtSecret` 没有 `type` 列的值（`initJWTSecret` 写入时没有设置 `bean.type）

```javascript
// initJWTSecret 中：
jwtSecretBean.key = "jwtSecret";
// 没有设置 jwtSecretBean.type → type 为 null 或 undefined
jwtSecretBean.value = ...
```

所以 `jwtSecret` **不会**被 `getSettings()` 返回，除非查询时 `type` 为 null。

### 3.7 读取路径 Z：直接数据库操作（完全绕过 Settings API

这是 `jwtSecret` 实际使用的读取路径：

```javascript
// server/server.js:1852-1868
// 直接用 R.findOne() 读取，不经过 Settings.get()
let jwtSecretBean = await R.findOne("setting", " `key` = ? ", ["jwtSecret"]);

if (!jwtSecretBean) {
    jwtSecretBean = await initJWTSecret();
}

// 直接访问 .value 属性
server.jwtSecret = jwtSecretBean.value;
```

**路径 Z 的特点：

1. **不解析**：直接使用 `bean.value`，没有 `JSON.parse()`
2. **不缓存**：完全绕过 `Settings.cacheList`
3. **最可靠**：无论值是否为 JSON 格式都正确工作

**三种读取路径对比表

| 特性 | Settings.get() | Settings.getSettings() | R.findOne().value |
|------|----------------|------------------------|-------------------|
| JSON 解析 | ✅ 是 | ✅ 是 | ❌ 否 |
| 使用缓存 | ✅ 是（条件） | ❌ 否 | ❌ 否 |
| 对非 JSON 值 | 返回原始值（不缓存） | 返回原始值 | 返回原始值 |
| 对 JSON 标准值 | 返回解析后的值（缓存） | 返回解析后的值 | 返回 JSON 字符串（**危险！** |

### 3.8 缓存机制详解

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

#### 缓存写入的严格条件

**只有同时满足以下所有条件才会写入缓存**：

1. **通过 `Settings.get(key)` 读取**（`getSettings()` 或 `R.findOne()` 不使用此缓存）
2. **从数据库成功读取到值**（不是 `undefined`）
3. **JSON.parse() 解析成功**（没有抛出异常）

**不会缓存的场景**：

| 场景 | 原因 |
|------|------|
| `jwtSecret` 用 Settings.get() 读取 | JSON.parse 失败（非 JSON 值） |
| 数据库中不存在该 key | `R.getCell()` 返回 `undefined` |
| 通过 Settings.getSettings() 读取 | 方法本身没有缓存逻辑 |
| 通过 R.findOne() 直接读取 | 完全绕过缓存系统 |

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

**注意**：`initJWTSecret()` **不会**调用 `deleteCache()`，因为它绕过了 Settings API。

### 3.9 非标准值的兼容影响深度分析

#### 场景 1：jwtSecret 用不同读取路径的行为

假设数据库中 `jwtSecret` 的值为：`"$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy"`（原始 bcrypt hash，没有 JSON 引号）

| 读取方式 | 代码 | 返回值 | 类型 | 是否可用 |
|---------|------|--------|------|---------|
| 路径 X（Settings.get） | `Settings.get("jwtSecret")` | `"$2a$10$..."` | 字符串 | ✅ 可用（但不缓存） |
| 路径 Y（getSettings） | `getSettings("...")` | `"$2a$10$..."` | 字符串 | ✅ 可用（但 type 为 null） |
| 路径 Z（R.findOne） | `R.findOne(...).value` | `"$2a$10$..."` | 字符串 | ✅ 可用（实际使用路径） |

#### 场景 2：如果有人错误地用 Settings.set("jwtSecret", ...) 写入

```javascript
// 错误用法：
await Settings.set("jwtSecret", "$2a$10$N9qo8uLO...");

// 数据库中存储的是：
"\"$2a$10$N9qo8uLO...\""  // 带 JSON 引号！
```

此时各读取路径的行为：

| 读取方式 | 返回值 | 是否可用 |
|---------|--------|---------|
| 路径 X（Settings.get） | `"$2a$10$..."`（解析后） | ✅ 可用（被缓存） |
| 路径 Y（getSettings） | `"$2a$10$..."`（解析后） | ✅ 可用 |
| 路径 Z（R.findOne.value） | `"\"$2a$10$N9qo8uLO...\""`（带引号的字符串！） | ❌ **不可用！** |

**关键风险**：实际代码使用路径 Z 读取，会得到带引号的字符串，JWT 验证会失败！

#### 场景 3：缓存不一致问题

假设 `jwtSecret` 被 `initJWTSecret()` 更新了（重置密码场景）：

```
时间线：
T1: 某人用 Settings.get("jwtSecret") 读取
    → JSON.parse 失败
    → 返回原始值
    → 不缓存（因为解析失败）

T2: initJWTSecret() 被调用，生成新的 jwtSecret
    → 直接写入数据库
    → 不调用 deleteCache()（但 jwtSecret 本来就没被缓存）

T3: 再用 Settings.get("jwtSecret") 读取
    → 读取到新值
    → 正常（因为没被缓存过）

问题：如果之前有人用另一个 key（如 "checkUpdate" 读取后缓存了，会不会有问题？

T1: Settings.get("checkUpdate") → 读取 → 缓存（解析成功 → 写入缓存

T2: Settings.set("checkUpdate", false) → 写入 DB → deleteCache(["checkUpdate"])

T3: Settings.get("checkUpdate") → 未命中缓存 → 重新读取 → 正确
```

**jwtSecret 特殊情况**：

| 操作 | 缓存状态 |
|------|----------|
| `Settings.get("jwtSecret")` | 未缓存（解析失败） |
| `initJWTSecret()` 被调用 | 不影响缓存（本来就没缓存） |
| 再次 `Settings.get("jwtSecret")` | 读取新值，仍未缓存 |

**结论**：`jwtSecret` 由于解析失败不会被缓存，所以即使 `initJWTSecret()` 不调用 `deleteCache()` 也不会有缓存一致性问题。但这是**巧合**，不是设计。

### 3.10 Setting 机制全景图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         写入路径                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                   │
│  路径 A：Settings.set(key, value, type)                          │
│       └─ JSON.stringify(value)                                     │
│       └─ deleteCache([key])                                      │
│                                                                   │
│  路径 B：Settings.setSettings(type, data)                        │
│       └─ JSON.stringify(data[key])                                  │
│       └─ deleteCache(keyList)                                      │
│                                                                   │
│  路径 C：initJWTSecret()                                         │
│       └─ 直接设置 bean.value = passwordHash.generate(...)               │
│       └─ 不 JSON.stringify                                      │
│       └─ 不 deleteCache                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    setting 表（value 列是 text 类型）              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                   │
│  标准值（路径 A/B 写入）：                                         │
│    key: "title"                                                   │
│    value: "\"My Uptime Kuma\""  (JSON 序列化，带引号)              │
│    type: "general"                                               │
│                                                                   │
│  特殊值（路径 C 写入）：                                          │
│    key: "jwtSecret"                                             │
│    value: "$2a$10$N9qo8uLO..."  (原始字符串，无引号)         │
│    type: null 或 undefined                                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         读取路径                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                   │
│  路径 X：Settings.get(key)                                          │
│       └─ 检查 cacheList（命中则直接返回）                             │
│       └─ SELECT value FROM setting                                    │
│       └─ JSON.parse(value)                                          │
│       ├─ 成功 → 写入 cacheList → 返回解析后的值                       │
│       └─ 失败 → 返回原始值（不缓存）                               │
│                                                                   │
│  路径 Y：Settings.getSettings(type)                              │
│       └─ SELECT key, value FROM setting WHERE type = ?           │
│       └─ JSON.parse(row.value)                                   │
│       ├─ 成功 → result[row.key] = 解析后的值                      │
│       └─ 失败 → result[row.key] = 原始值                        │
│       └─ 不使用 cacheList                                         │
│                                                                   │
│  路径 Z：R.findOne("setting", "key = ?", ...)                    │
│       └─ 直接访问 bean.value                                      │
│       └─ 不 JSON.parse                                            │
│       └─ 不使用 cacheList                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.11 关键发现总结

| 问题 | 答案 |
|------|------|
| 是否存在非 JSON 写入路径？ | ✅ 是，`initJWTSecret()` 是唯一明确的直接写入路径 |
| "全量 JSON 写入"是否正确？ | ❌ 不正确，需要修正为"标准路径 JSON 序列化，特殊路径原始写入" |
| jwtSecret 如何读取？ | 通过路径 Z（`R.findOne().value`），完全绕过 Settings API |
| jwtSecret 用 Settings.get() 读会怎样？ | 返回正确值，但不会缓存（因为 JSON.parse 失败） |
| initJWTSecret() 会清缓存吗？ | ❌ 不会，但 jwtSecret 本来就不被缓存，所以没问题 |
| 历史遗留值存在吗？ | ✅ 可能存在（1.X 数据、迁移脚本直接操作） |

---

## 4. 聚合迁移：中断、跳过、恢复场景的兼容语义与风险

### 4.1 聚合迁移概述

`migrateAggregateTable()` 是 V1 → V2 版本的**数据迁移**（不是 schema 迁移），负责将旧版 `heartbeat` 表的原始数据转换为新版的统计聚合表（`stat_minutely`、`stat_hourly`、`stat_daily`）。

**迁移核心逻辑**（`server/database.js:819-946`）：

```
┌─────────────────────────────────────────────────────────┐
│  migrateAggregateTable() 执行流程                │
├─────────────────────────────────────────────────────────┤
│  1. 检查 SET_MIGRATE_AGGREGATE_TABLE_TO_TRUE 环境变量 │
│     └─ 如设为 1：直接设为 migrated，跳过所有逻辑      │
│                                                   │
│  2. 读取 migrateAggregateTableState 设置           │
│     ├─ "migrated"  ──► 直接返回（已完成）       │
│     ├─ "migrating" ──► throw Error（拒绝启动）   │
│     └─ 其他/空     ──► 继续                      │
│                                                   │
│  3. 检查 stat_* 表是否为空                     │
│     └─ 任一表有数据 ──► 直接返回（不启动迁移）       │
│                                                   │
│  4. 设置状态为 "migrating"                       │
│                                                   │
│  5. 遍历所有 monitor，遍历所有日期，重新计算统计           │
│     └─ 写入 stat_minutely / stat_hourly / stat_daily     │
│                                                   │
│  6. 清理旧的 heartbeat 数据                          │
│                                                   │
│  7. 设置状态为 "migrated"                            │
└─────────────────────────────────────────────────────────┘
```

### 4.2 状态机设计

迁移状态存储在 `setting` 表中，key 为 `migrateAggregateTableState`：

| 状态值 | 含义 | 后续行为 |
|--------|------|---------|
| `undefined` / `""` / 不存在 | 未开始迁移 | 尝试执行迁移 |
| `"migrating"` | 迁移中（或已中断） | **抛出错误，拒绝启动 |
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
|---------|-----------------|---------------|-----------|
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

1. **双写入路径**：
   - 标准路径（`Settings.set()` / `setSettings()`）：全 JSON 序列化
   - 特殊路径（`initJWTSecret()`）：原始字符串，不序列化
2. **三读取路径**：
   - `Settings.get()`：JSON 解析 + 条件缓存
   - `Settings.getSettings()`：JSON 解析 + 无缓存
   - `R.findOne()`：直接读取 + 无解析 + 无缓存
3. **条件缓存**：只有 `Settings.get()` 且 JSON.parse 成功才缓存
4. **懒启动清理**：缓存清理器在第一次读取时启动
5. **历史兼容**：保留非 JSON 值的读取支持

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
| `server/util-server.js` | 旧版 setting API、`initJWTSecret()` 直接写入 |
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
  - 引入 `initJWTSecret()` 作为特殊写入路径

---

**报告生成日期**：2026-05-10
**报告版本**：3.0（重大修订：修正全量 JSON 写入结论，补充直接写入路径分析，深入分析非标准值的兼容影响）
