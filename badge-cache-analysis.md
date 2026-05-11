# Uptime Kuma 状态徽章系统分析报告

## 1. 概述

Uptime Kuma 提供了一个功能丰富的状态徽章系统，允许用户生成可嵌入的 SVG 徽章来展示监控状态。本文档详细分析了徽章的生成机制、缓存策略以及公开展示时的信息裁剪措施。

## 2. 徽章类型和路由

### 2.1 徽章分类

Uptime Kuma 支持两种主要类型的徽章：

#### 2.1.1 状态页面整体徽章
- **路由**: `/api/status-page/:slug/badge`
- **文件**: `server/routers/status-page-router.js:170`
- **功能**: 展示整个状态页面的综合状态
- **缓存时间**: 5 分钟

#### 2.1.2 单个监控器徽章
在 `server/routers/api-router.js` 中定义了多个徽章端点：

1. **状态徽章**: `/api/badge/:id/status` (第 148 行)
   - 显示监控器的当前状态（Up/Down/Pending/Maintenance）

2. **正常运行时间徽章**: `/api/badge/:id/uptime/:duration?` (第 221 行)
   - 显示指定时间段内的正常运行时间百分比
   - 默认 24 小时，支持自定义时长

3. **Ping 时间徽章**: `/api/badge/:id/ping/:duration?` (第 285 行)
   - 显示平均延迟时间
   - 默认 24 小时，支持自定义时长

4. **平均响应时间徽章**: `/api/badge/:id/avg-response/:duration?` (第 351 行)
   - 显示平均响应时间

5. **证书过期徽章**: `/api/badge/:id/cert-exp` (第 447 行)
   - 显示 SSL/TLS 证书剩余天数

6. **响应状态码徽章**: `/api/badge/:id/response`
   - 显示当前响应状态码

## 3. 徽章生成机制

### 3.1 核心依赖

徽章系统使用以下核心依赖：

1. **badge-maker 库**: 
   - 用于生成 SVG 格式的徽章
   - 导入位置: `server/routers/status-page-router.js:8` 和 `server/routers/api-router.js:16`

2. **badgeConstants 常量**:
   - 定义默认颜色、样式等配置
   - 位置: `src/util.ts:143-161`
   - 包含默认颜色配置：
     - `defaultUpColor`: "#66c20a" (绿色)
     - `defaultDownColor`: "#c2290a" (红色)
     - `defaultPendingColor`: "#f8a306" (橙色)
     - `defaultMaintenanceColor`: "#1747f5" (蓝色)
     - `naColor`: "#999" (灰色，用于不可用状态)
     - `defaultStyle`: "flat"

### 3.2 状态页面徽章生成流程

状态页面徽章的生成逻辑在 `status-page-router.js:170-262` 中实现：

1. **获取状态页面 ID**: 通过 slug 转换为 ID
2. **查询公开监控器**: 
   - SQL 查询获取该状态页面下所有公开监控器的 ID
   - 查询条件: `public = 1`
3. **状态聚合**:
   - 遍历所有监控器，获取最新心跳记录
   - 检查每个监控器的状态：
     - 状态 3: 维护模式 (Maintenance)
     - 状态 2: 忽略 (Ignored)
     - 状态 1: 正常 (Up)
     - 状态 0: 故障 (Down)
4. **徽章状态决策**:
   - 所有正常且无维护: 显示 "Up"
   - 有故障但有正常: 显示 "Degraded"
   - 全部故障: 显示 "Down"
   - 有维护: 显示 "Maintenance"
   - 无公开监控器: 显示 "N/A"
5. **生成 SVG**: 使用 `makeBadge()` 函数生成 SVG 图像

### 3.3 单个监控器徽章生成流程

单个监控器徽章的生成逻辑在 `api-router.js` 中实现，以状态徽章为例 (`api-router.js:148-219`)：

1. **参数解析**: 从查询字符串解析自定义参数（颜色、标签、样式等）
2. **公开性检查**: 调用 `isMonitorPublic()` 检查监控器是否公开
3. **获取状态**: 
   - 调用 `Monitor.getPreviousHeartbeat()` 获取最新心跳记录
   - 支持通过 `value` 参数覆盖状态（仅用于演示）
4. **构建徽章值**:
   - 根据状态设置颜色和消息
   - 支持自定义标签
5. **生成 SVG**: 使用 `makeBadge()` 函数生成 SVG 图像

## 4. 缓存机制

### 4.1 缓存系统架构

Uptime Kuma 使用自定义的 `apicache` 模块来实现徽章缓存：

#### 4.1.1 缓存模块结构
- **主文件**: `server/modules/apicache/index.js`
- **核心实现**: `server/modules/apicache/apicache.js`
- **内存缓存**: `server/modules/apicache/memory-cache.js`

#### 4.1.2 缓存配置
在 `server/modules/apicache/index.js` 中配置了缓存选项：

```javascript
apicache.options({
    headerBlacklist: ["cache-control"],
    headers: {
        // 禁用客户端缓存，仅使用服务器端缓存
        "cache-control": "no-cache",
    },
});
```

关键配置：
- 黑名单 `cache-control` 头
- 强制设置 `cache-control: no-cache` 响应头
- 这意味着浏览器不会缓存徽章，但服务器端会缓存

### 4.2 缓存策略

#### 4.2.1 缓存持续时间

不同端点的缓存时间：
- **状态页面徽章**: 5 分钟
- **单个监控器徽章**: 5 分钟
- **状态页面 API**: 5 分钟
- **心跳数据 API**: 1 分钟
- **Manifest.json**: 1440 分钟 (24 小时)

#### 4.2.2 缓存实现细节

在 `apicache.js` 中实现的缓存机制：

1. **内存存储**: 使用 MemoryCache 类存储缓存数据
2. **缓存键**: 基于请求 URL 和参数生成
3. **缓存对象结构**:
   - `status`: HTTP 状态码
   - `headers`: 响应头（过滤黑名单）
   - `data`: 响应数据
   - `encoding`: 编码类型
   - `timestamp`: 时间戳

4. **缓存过滤**:
   - 根据状态码决定是否缓存
   - 支持自定义过滤函数
   - 过滤黑名单中的响应头

### 4.3 缓存工作流程

1. **请求到达**: 客户端请求徽章
2. **缓存检查**: apicache 中间件检查是否有缓存
3. **缓存命中**: 
   - 如果有有效缓存，直接返回缓存的响应
   - 不执行后续路由处理
4. **缓存未命中**:
   - 执行路由处理函数生成徽章
   - 存储响应到缓存
   - 返回响应给客户端

## 5. 公开展示时的信息裁剪

Uptime Kuma 在设计徽章系统时充分考虑了隐私和安全，实施了多层信息裁剪措施：

### 5.1 监控器公开性检查

所有徽章端点都首先检查监控器是否公开：

#### 5.1.1 检查函数
函数 `isMonitorPublic()` 在 `api-router.js:626-637` 中定义：

```javascript
async function isMonitorPublic(monitorID) {
    let publicMonitor = await R.getRow(
        `
            SELECT monitor_group.monitor_id FROM monitor_group, \`group\`
            WHERE monitor_group.group_id = \`group\`.id
            AND monitor_group.monitor_id = ?
            AND public = 1
        `,
        [monitorID]
    );
    return !!publicMonitor;
}
```

检查逻辑：
- 通过 `monitor_group` 和 `group` 表关联查询
- 检查 `public` 字段是否为 1
- 返回布尔值表示监控器是否公开

#### 5.1.2 非公开监控器的处理

对于非公开监控器：
- 返回灰色的 "N/A" 徽章
- 颜色: `badgeConstants.naColor` (#999)
- 不暴露任何状态信息

### 5.2 信息暴露限制

### 5.2.1 仅聚合数据

徽章系统只返回聚合数据，不暴露：

1. **不暴露详细历史**: 
   - 正常运行时间只显示百分比，不显示具体故障时间
   - Ping 时间只显示平均值，不显示波动详情

2. **不暴露敏感配置**:
   - 不显示监控器 URL、端口等配置信息
   - 不显示认证信息
   - 不显示监控间隔等参数

3. **不暴露详细错误信息**:
   - 故障时只显示 "Down"，不显示具体错误原因
   - 不暴露错误日志或堆栈跟踪

### 5.2.2 状态页面徽章的额外裁剪

状态页面徽章 (`status-page-router.js:170-262`) 实施了更严格的裁剪：

1. **只包含公开监控器**:
   - SQL 查询明确过滤 `public = 1`
   - 私有监控器完全不参与状态计算

2. **简化状态表示**:
   - 只显示 5 种状态: Up、Down、Degraded、Maintenance、N/A
   - 不显示具体哪个监控器故障
   - 不显示故障数量

### 5.3 数据精度控制

#### 5.3.1 正常运行时间精度
在 `api-router.js:261` 中：
```javascript
const cleanUptime = (uptime * 100).toPrecision(4);
```
- 限制显示 4 位有效数字
- 例如: 99.99% 而不是 99.987654321%

#### 5.3.2 Ping 时间精度
在 `api-router.js:328` 中：
```javascript
const avgPingValue = parseInt(overrideValue ?? avgPing);
```
- 取整为整数毫秒
- 不显示小数部分

### 5.4 CORS 策略

徽章端点使用 `allowAllOrigin()` 或 `allowDevAllOrigin()` 函数：

- **生产环境**: 允许所有来源 (`Access-Control-Allow-Origin: *`)
- **开发环境**: 额外的 CORS 配置

这使得徽章可以嵌入到任何网站，但只暴露徽章数据本身。

## 6. 安全设计亮点

### 6.1 演示模式
所有徽章端点都支持 `value` 参数：
- 仅用于演示和测试
- 不影响实际监控数据
- 方便用户预览徽章样式

### 6.2 输入验证
所有徽章端点都进行输入验证：
- 监控器 ID 必须是有效整数
- 持续时间参数格式验证
- 防止 SQL 注入和 XSS 攻击

### 6.3 分层设计
徽章系统采用分层架构：
1. **缓存层**: apicache 中间件
2. **路由层**: Express 路由处理
3. **业务逻辑层**: 状态计算和聚合
4. **数据访问层**: 数据库查询

这种设计确保了安全性和可维护性。

## 7. 总结

Uptime Kuma 的状态徽章系统设计精良，具有以下特点：

1. **功能丰富**: 支持多种徽章类型和自定义选项
2. **性能优化**: 5 分钟服务器端缓存减少数据库查询
3. **安全可靠**: 严格的公开性检查和信息裁剪
4. **隐私保护**: 只暴露必要的聚合数据，保护敏感信息
5. **易于集成**: 允许跨域请求，方便嵌入到任何网站

通过这些设计，Uptime Kuma 提供了一个既实用又安全的状态徽章系统，平衡了展示需求和隐私保护。
