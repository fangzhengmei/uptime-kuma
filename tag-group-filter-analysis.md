# Uptime Kuma 标签、分组和过滤器对监控列表查询状态的影响分析

## 一、概述

本文档详细分析 Uptime Kuma 中两套独立的分组模型、标签系统和过滤器如何影响监控列表的查询状态。

**核心区分：**
- **监控层级分组**：通过 `monitor.parent` 字段实现的树形层级结构，用于监控列表的组织
- **状态页分组**：通过独立的 `group` 表和 `monitor_group` 中间表实现，用于状态页展示

## 二、数据模型详解

### 2.1 数据库表结构

#### 2.1.1 监控表（monitor）
位置：`db/knex_init_db.js:65-122`

关键字段：
- `id`: 主键
- `name`: 监控名称
- `type`: 监控类型（包括 "group" 表示分组类型）
- `parent`: 父监控ID（自引用外键）- **监控层级分组的核心**
- `active`: 是否激活
- `weight`: 排序权重

`parent` 字段定义：`db/knex_init_db.js:496`
```javascript
table.integer("parent").unsigned().references("id").inTable("monitor").onDelete("SET NULL").onUpdate("CASCADE");
```

#### 2.1.2 状态页分组表（group）
位置：`db/knex_init_db.js:25-34`

关键字段：
- `id`: 主键
- `name`: 分组名称
- `public`: 是否公开
- `weight`: 排序权重
- `status_page_id`: 关联的状态页ID

#### 2.1.3 监控-状态页分组关联表（monitor_group）
位置：`db/knex_init_db.js:247-270`

**这是状态页分组专用的中间表，与监控层级分组无关**

字段：
- `monitor_id`: 监控ID
- `group_id`: 状态页分组ID
- `weight`: 在分组内的排序权重
- `send_url`: 是否发送URL
- `custom_url`: 自定义URL

#### 2.1.4 标签表（tag）和监控-标签关联表（monitor_tag）

标签是独立系统，同时服务于监控层级分组和状态页分组。

### 2.2 两套分组模型对比

| 特性 | 监控层级分组 | 状态页分组 |
|------|-------------|-----------|
| 实现方式 | 自引用外键（parent字段） | 独立表 + 中间表 |
| 表名 | monitor 表自身 | group + monitor_group |
| 层级支持 | 支持无限嵌套 | 扁平结构（不支持嵌套） |
| 用途 | 监控列表组织、拖拽排序 | 状态页展示分组 |
| 排序方式 | weight 字段 | weight 字段 |
| 关联方式 | monitor.parent = parent.id | monitor_group 中间表 |
| 状态影响 | 分组有自己的心跳状态（聚合子节点） | 不影响监控状态 |

### 2.3 标签数据结构

#### 标签模型
位置：`server/model/tag.js:1-17`

```javascript
{
    id: this._id,
    name: this._name,
    color: this._color,
}
```

#### 监控-标签关联查询
位置：`server/model/monitor.js:261-266`

```sql
SELECT mt.*, tag.name, tag.color 
FROM monitor_tag mt 
JOIN tag ON mt.tag_id = tag.id 
WHERE mt.monitor_id = ? 
ORDER BY tag.name
```

批量获取监控标签：`server/model/monitor.js:1827-1836`

```sql
SELECT monitor_tag.monitor_id, monitor_tag.tag_id, monitor_tag.value, tag.name, tag.color
FROM monitor_tag
JOIN tag ON monitor_tag.tag_id = tag.id
WHERE monitor_tag.monitor_id IN (?, ?, ...)
```

## 三、后端数据预加载完整链路

### 3.1 触发时机

1. **用户登录后自动推送**：`server/server.js:1808`
   ```javascript
   let monitorList = await server.sendMonitorList(socket);
   ```

2. **前端主动请求**：`server/server.js:971-977`
   ```javascript
   socket.on("getMonitorList", async (callback) => {
       await server.sendMonitorList(socket);
   });
   ```

3. **数据变更后推送**：如代理更新、监控编辑等操作后触发

### 3.2 发送监控列表流程

位置：`server/uptime-kuma-server.js:219-277`

```javascript
async sendMonitorList(socket) {
    let list = await this.getMonitorJSONList(socket.userID);
    this.io.to(socket.userID).emit("monitorList", list);
    return list;
}

async getMonitorJSONList(userID) {
    const monitorList = await R.findAll("monitor", " user_id = ? ORDER BY weight, name ", [userID]);
    
    // 准备预加载数据（防止 N+1 问题）
    const monitorData = monitorList.map((monitor) => ({
        id: monitor.id,
        active: monitor.active,
        name: monitor.name,
    }));
    const preloadData = await Monitor.preparePreloadData(monitorData);
    
    // 序列化每个监控
    const result = {};
    monitorList.forEach((monitor) => (result[monitor.id] = monitor.toJSON(preloadData)));
    return result;
}
```

### 3.3 预加载数据详解

位置：`server/model/monitor.js:1844-1919`

`preparePreloadData` 方法批量获取所有关联数据，避免 N+1 查询问题：

```javascript
static async preparePreloadData(monitorData) {
    const monitorIDs = monitorData.map((monitor) => monitor.id);
    
    // 1. 批量获取通知关联
    const notifications = await Monitor.getMonitorNotification(monitorIDs);
    
    // 2. 批量获取标签（通过 monitor_tag 关联表）
    const tags = await Monitor.getMonitorTag(monitorIDs);
    
    // 3. 批量获取维护状态
    const maintenanceStatuses = await Promise.all(
        monitorData.map((monitor) => Monitor.isUnderMaintenance(monitor.id))
    );
    
    // 4. 批量获取所有子监控ID（用于监控层级分组）
    const childrenIDs = await Promise.all(
        monitorData.map((monitor) => Monitor.getAllChildrenIDs(monitor.id))
    );
    
    // 5. 批量获取激活状态（考虑父节点影响）
    const activeStatuses = await Promise.all(
        monitorData.map((monitor) => Monitor.isActive(monitor.id, monitor.active))
    );
    
    // 6. 批量获取父节点激活状态
    const forceInactiveStatuses = await Promise.all(
        monitorData.map((monitor) => Monitor.isParentActive(monitor.id))
    );
    
    // 7. 批量获取完整路径（用于面包屑导航）
    const paths = await Promise.all(
        monitorData.map((monitor) => Monitor.getAllPath(monitor.id, monitor.name))
    );
    
    // 构建 Map 结构便于快速访问
    return {
        notifications: notificationsMap,      // Map<monitorId, notificationIds>
        tags: tagsMap,                         // Map<monitorId, tag[]>
        maintenanceStatus: maintenanceStatusMap,
        childrenIDs: childrenIDsMap,           // Map<monitorId, childrenId[]>
        activeStatus: activeStatusMap,
        forceInactive: forceInactiveMap,
        paths: pathsMap,
    };
}
```

### 3.4 监控序列化（toJSON）

位置：`server/model/monitor.js:117-254`

```javascript
toJSON(preloadData = {}, includeSensitiveData = true) {
    return {
        id: this.id,
        name: this.name,
        description: this.description,
        path: preloadData.paths.get(this.id) || [],           // 完整路径
        pathName: path.join(" / "),
        parent: this.parent,                                   // 父监控ID（层级分组核心）
        childrenIDs: preloadData.childrenIDs.get(this.id) || [], // 所有子监控ID
        url: this.url,
        weight: this.weight,
        active: preloadData.activeStatus.get(this.id),         // 计算后的激活状态
        forceInactive: preloadData.forceInactive.get(this.id), // 父节点强制不激活
        type: this.type,
        tags: preloadData.tags.get(this.id) || [],              // 标签数组
        maintenance: preloadData.maintenanceStatus.get(this.id),
        // ... 其他字段
    };
}
```

### 3.5 状态页分组数据加载（独立链路）

位置：`server/model/status_page.js:309-340`

```javascript
static async getStatusPageData(statusPage) {
    // 获取状态页配置
    const config = await statusPage.toPublicJSON();
    
    // 获取状态页分组（使用独立的 group 表）
    const publicGroupList = [];
    const list = await R.find("group", " public = 1 AND status_page_id = ? ORDER BY weight ", [statusPage.id]);
    
    // 通过 monitor_group 中间表获取分组内的监控
    for (let groupBean of list) {
        let monitorGroup = await groupBean.toPublicJSON(showTags, config?.showCertificateExpiry);
        publicGroupList.push(monitorGroup);
    }
    
    return {
        config,
        publicGroupList,  // 状态页分组数据
        // ...
    };
}
```

状态页分组的 `toPublicJSON`：`server/model/group.js:13-46`

```javascript
async toPublicJSON(showTags = false, certExpiry = false) {
    // 通过 monitor_group 中间表查询分组内的监控
    let monitorBeanList = await this.getMonitorList();
    // monitor_group.send_url, monitor_group.custom_url 等自定义属性
    
    return {
        id: this.id,
        name: this.name,
        weight: this.weight,
        monitorList,  // 该分组下的监控列表
    };
}
```

## 四、前端数据接收与存储

### 4.1 Socket 事件监听

位置：`src/mixins/socket.js:147-163`

```javascript
socket.on("monitorList", (data) => {
    this.assignMonitorUrlParser(data);
    this.monitorList = data;  // 存储为 Vue 根实例数据
});

socket.on("updateMonitorIntoList", (data) => {
    this.assignMonitorUrlParser(data);
    Object.entries(data).forEach(([monitorID, updatedMonitor]) => {
        this.monitorList[monitorID] = updatedMonitor;
    });
});

socket.on("deleteMonitorFromList", (monitorID) => {
    if (this.monitorList[monitorID]) {
        delete this.monitorList[monitorID];
    }
});
```

### 4.2 数据结构

前端 `$root.monitorList` 结构：

```javascript
{
    "1": {
        id: 1,
        name: "Website",
        type: "http",
        parent: null,           // 根节点
        childrenIDs: [3, 4],    // 子监控ID（通过层级分组）
        active: true,
        tags: [                 // 标签列表
            { tag_id: 1, monitor_id: 1, value: null, name: "prod", color: "#ff0000" },
            { tag_id: 2, monitor_id: 1, value: "v1.0", name: "version", color: "#00ff00" }
        ],
        // ... 其他字段
    },
    "2": {
        id: 2,
        name: "API Group",
        type: "group",          // 分组类型监控
        parent: null,
        childrenIDs: [5, 6, 7],
        active: true,
        tags: [],
    },
    "3": {
        id: 3,
        name: "API Health",
        type: "http",
        parent: 2,              // 属于 ID 为 2 的分组
        childrenIDs: [],
        active: true,
        tags: [],
    }
}
```

## 五、前端过滤完整链路

### 5.1 过滤器状态定义

位置：`src/components/MonitorList.vue:158-163`

```javascript
filterState: {
    status: null,      // 状态过滤数组: [1, 0] 表示显示 UP 和 DOWN
    active: null,      // 激活状态过滤数组: [true] 或 [false]
    tags: null,        // 标签过滤数组: [tagId1, tagId2]
}
```

### 5.2 过滤器UI交互

位置：`src/components/MonitorListFilter.vue`

#### 状态过滤切换：205-220行
```javascript
toggleStatusFilter(status) {
    let newFilter = { ...this.filterState };
    if (newFilter.status == null) {
        newFilter.status = [status];
    } else {
        if (newFilter.status.includes(status)) {
            newFilter.status = newFilter.status.filter((item) => item !== status);
        } else {
            newFilter.status.push(status);
        }
    }
    this.$emit("updateFilter", newFilter);
}
```

#### 标签过滤切换：237-252行
```javascript
toggleTagFilter(tag) {
    let newFilter = { ...this.filterState };
    if (newFilter.tags == null) {
        newFilter.tags = [tag.id];
    } else {
        if (newFilter.tags.includes(tag.id)) {
            newFilter.tags = newFilter.tags.filter((item) => item !== tag.id);
        } else {
            newFilter.tags.push(tag.id);
        }
    }
    this.$emit("updateFilter", newFilter);
}
```

### 5.3 过滤器状态更新

位置：`src/components/MonitorList.vue:346-348`

```javascript
updateFilter(newFilter) {
    this.filterState = newFilter;  // Vue 响应式更新
}
```

### 5.4 计算属性自动更新

位置：`src/components/MonitorList.vue:189-205`

```javascript
sortedMonitorList() {
    // 1. 获取所有监控
    let result = Object.values(this.$root.monitorList);

    // 2. 根级别过滤：只显示没有父节点的监控
    result = result.filter((monitor) => {
        if (monitor.parent !== null) {
            return false;
        }
        return true;
    });

    // 3. 应用过滤函数
    result = result.filter(this.filterFunc);

    // 4. 排序
    result.sort(this.sortFunc);

    return result;
}
```

### 5.5 核心过滤逻辑（filterFunc）

位置：`src/components/MonitorList.vue:528-576`

```javascript
filterFunc(monitor) {
    // ========== 边界场景1：分组特殊处理 ==========
    // 如果是分组类型的监控，只要有子监控匹配，分组就显示
    if (monitor.type === "group") {
        const children = Object.values(this.$root.monitorList).filter(
            (m) => m.parent === monitor.id
        );
        if (children.some((child) => this.filterFunc(child))) {
            return true;
        }
    }

    // ========== 边界场景2：搜索文本匹配 ==========
    // 匹配监控名称、标签名称、标签值
    let searchTextMatch = true;
    if (this.searchText !== "") {
        const loweredSearchText = this.searchText.toLowerCase();
        searchTextMatch =
            monitor.name.toLowerCase().includes(loweredSearchText) ||
            monitor.tags.find(
                (tag) =>
                    tag.name.toLowerCase().includes(loweredSearchText) ||
                    tag.value?.toLowerCase().includes(loweredSearchText)
            );
    }

    // ========== 边界场景3：状态过滤 ==========
    // 依赖 lastHeartbeatList 获取最新心跳状态
    let statusMatch = true;
    if (this.filterState.status != null && this.filterState.status.length > 0) {
        if (monitor.id in this.$root.lastHeartbeatList && this.$root.lastHeartbeatList[monitor.id]) {
            monitor.status = this.$root.lastHeartbeatList[monitor.id].status;
        }
        statusMatch = this.filterState.status.includes(monitor.status);
    }

    // ========== 边界场景4：激活状态过滤 ==========
    let activeMatch = true;
    if (this.filterState.active != null && this.filterState.active.length > 0) {
        activeMatch = this.filterState.active.includes(monitor.active);
    }

    // ========== 边界场景5：标签过滤 ==========
    // 监控标签与过滤器标签取交集，只要有交集就匹配
    let tagsMatch = true;
    if (this.filterState.tags != null && this.filterState.tags.length > 0) {
        tagsMatch =
            monitor.tags
                .map((tag) => tag.tag_id)
                .filter((monitorTagId) => this.filterState.tags.includes(monitorTagId)).length > 0;
    }

    // ========== 最终结果：所有条件都必须满足 ==========
    return searchTextMatch && statusMatch && activeMatch && tagsMatch;
}
```

### 5.6 子项过滤（MonitorListItem）

位置：`src/components/MonitorListItem.vue:144-156`

```javascript
sortedChildMonitorList() {
    let result = Object.values(this.$root.monitorList);

    // 获取当前分组的直接子节点
    result = result.filter((childMonitor) => childMonitor.parent === this.monitor.id);

    // 应用同样的过滤函数
    result = result.filter(this.filterFunc);

    result.sort(this.sortFunc);

    return result;
}
```

## 六、监控层级分组折叠机制

### 6.1 折叠状态存储

位置：`src/components/MonitorListItem.vue:181-200`

```javascript
beforeMount() {
    // 边界场景：直接访问子监控时自动展开父分组
    if (this.monitor.childrenIDs.includes(parseInt(this.$route.params.id))) {
        this.isCollapsed = false;
        return;
    }

    // 从 localStorage 读取折叠状态
    let storage = window.localStorage.getItem("monitorCollapsed");
    if (storage === null) {
        return;  // 默认折叠
    }

    let storageObject = JSON.parse(storage);
    if (storageObject[`monitor_${this.monitor.id}`] == null) {
        return;
    }

    this.isCollapsed = storageObject[`monitor_${this.monitor.id}`];
}
```

### 6.2 全局折叠/展开

位置：`src/components/MonitorList.vue:354-393`

```javascript
toggleCollapseAll() {
    const shouldCollapse = !this.allGroupsCollapsed;

    let storageObject = {};
    const storage = window.localStorage.getItem("monitorCollapsed");
    if (storage !== null) {
        storageObject = JSON.parse(storage);
    }

    // 获取所有有子节点的分组
    this.groupMonitors.forEach((group) => {
        storageObject[`monitor_${group.id}`] = shouldCollapse;
    });

    window.localStorage.setItem("monitorCollapsed", JSON.stringify(storageObject));

    // 边界场景：折叠全部且当前在子分组页面，导航到根父节点
    if (shouldCollapse) {
        const currentMonitorId = parseInt(this.$route.params.id);
        const currentMonitor = this.$root.monitorList[currentMonitorId];

        if (currentMonitor && currentMonitor.parent !== null) {
            // 向上遍历找到根父节点
            let rootParentId = currentMonitor.parent;
            let rootParent = this.$root.monitorList[rootParentId];
            while (rootParent && rootParent.parent !== null) {
                rootParentId = rootParent.parent;
                rootParent = this.$root.monitorList[rootParentId];
            }

            this.$router.push(getMonitorRelativeURL(rootParentId)).finally(() => {
                this.collapseKey++;  // 强制重新渲染
            });
            return;
        }
    }

    this.collapseKey++;  // 触发 Vue 重新计算
}
```

### 6.3 分组检测

位置：`src/components/MonitorList.vue:247-250`

```javascript
groupMonitors() {
    const monitors = Object.values(this.$root.monitorList);
    return monitors.filter(
        (m) => m.type === "group" && monitors.some((child) => child.parent === m.id)
    );
}
```

## 七、排序机制

位置：`src/components/MonitorList.vue:583-605`

```javascript
sortFunc(m1, m2) {
    // 1. 激活状态优先：激活的排在前面
    if (m1.active !== m2.active) {
        if (m1.active === false) return 1;
        if (m2.active === false) return -1;
    }

    // 2. 按权重排序：权重越大越靠前
    if (m1.weight !== m2.weight) {
        if (m1.weight > m2.weight) return -1;
        if (m1.weight < m2.weight) return 1;
    }

    // 3. 按名称字母排序
    return m1.name.localeCompare(m2.name);
}
```

## 八、关键边界场景分析

### 8.1 场景1：分组内子监控匹配但分组本身不匹配

**表现**：分组类型监控本身不满足过滤条件，但其子监控满足

**处理逻辑**：`filterFunc` 530-535行
```javascript
if (monitor.type === "group") {
    const children = Object.values(this.$root.monitorList).filter((m) => m.parent === monitor.id);
    if (children.some((child) => this.filterFunc(child))) {
        return true;  // 分组显示
    }
}
```

**结果**：分组会显示，但其自身不满足的条件会被忽略，只因为有子项匹配

### 8.2 场景2：搜索文本匹配标签名称或值

**表现**：用户搜索 "prod"，不匹配任何监控名称，但匹配某个标签名称

**处理逻辑**：`filterFunc` 540-549行
```javascript
searchTextMatch =
    monitor.name.toLowerCase().includes(loweredSearchText) ||
    monitor.tags.find(
        (tag) =>
            tag.name.toLowerCase().includes(loweredSearchText) ||
            tag.value?.toLowerCase().includes(loweredSearchText)
    );
```

**结果**：带有该标签的监控会被显示

### 8.3 场景3：状态过滤依赖心跳数据

**表现**：过滤器选择 "DOWN" 状态，但某些监控没有心跳数据

**处理逻辑**：`filterFunc` 552-558行
```javascript
if (monitor.id in this.$root.lastHeartbeatList && this.$root.lastHeartbeatList[monitor.id]) {
    monitor.status = this.$root.lastHeartbeatList[monitor.id].status;
}
statusMatch = this.filterState.status.includes(monitor.status);
```

**结果**：没有心跳数据的监控，其 `monitor.status` 为默认值，可能导致不匹配

### 8.4 场景4：标签过滤的 OR 逻辑

**表现**：用户选择了标签 A 和标签 B

**处理逻辑**：`filterFunc` 567-573行
```javascript
tagsMatch =
    monitor.tags
        .map((tag) => tag.tag_id)
        .filter((monitorTagId) => this.filterState.tags.includes(monitorTagId)).length > 0;
```

**结果**：只要监控有任意一个选中的标签就会显示（OR 逻辑），不是必须同时有所有标签

### 8.5 场景5：直接访问子监控时自动展开

**表现**：用户通过 URL 直接访问某个子监控详情页

**处理逻辑**：`MonitorListItem.vue` 183-186行
```javascript
if (this.monitor.childrenIDs.includes(parseInt(this.$route.params.id))) {
    this.isCollapsed = false;
    return;
}
```

**结果**：所有祖先分组都会自动展开，确保用户能看到完整路径

### 8.6 场景6：全局折叠时导航到根父节点

**表现**：用户正在查看嵌套分组内的子监控，点击"全部折叠"

**处理逻辑**：`MonitorList.vue` 370-389行
```javascript
if (shouldCollapse) {
    const currentMonitorId = parseInt(this.$route.params.id);
    const currentMonitor = this.$root.monitorList[currentMonitorId];
    if (currentMonitor && currentMonitor.parent !== null) {
        // 向上遍历找到根父节点并导航
    }
}
```

**结果**：自动导航到最顶层的父分组，避免显示折叠状态下不可见的监控

### 8.7 场景7：空分组的状态

**表现**：分组类型监控没有任何子监控

**处理逻辑**：`server/monitor-types/group.js:15-19`
```javascript
if (children.length === 0) {
    heartbeat.status = PENDING;
    heartbeat.msg = "Group empty";
    return;
}
```

**结果**：空分组的状态为 PENDING，过滤"PENDING"状态时会显示

### 8.8 场景8：标签值为 null 的情况

**表现**：监控的某些标签没有设置 value

**处理逻辑**：`filterFunc` 546-548行
```javascript
tag.value?.toLowerCase().includes(loweredSearchText)
```

**结果**：使用可选链操作符 `?.` 避免 null 错误

## 九、完整数据流图

### 9.1 从后端到前端的完整链路

```
┌─────────────────────────────────────────────────────────────────────┐
│                        后端数据准备阶段                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户登录 / getMonitorList 请求                                      │
│           ↓                                                         │
│  server.sendMonitorList(socket)                                     │
│           ↓                                                         │
│  this.getMonitorJSONList(userID)                                    │
│           ↓                                                         │
│  R.findAll("monitor", ...)  ──→  获取所有监控记录                    │
│           ↓                                                         │
│  Monitor.preparePreloadData(monitorData)                            │
│           ↓                                                         │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  批量查询（避免 N+1）：                                  │        │
│  │  - getMonitorNotification(monitorIDs)                   │        │
│  │  - getMonitorTag(monitorIDs)  ← 监控-标签关联            │        │
│  │  - isUnderMaintenance(monitor.id)                       │        │
│  │  - getAllChildrenIDs(monitor.id)  ← 监控层级分组         │        │
│  │  - isActive(monitor.id, monitor.active)                 │        │
│  │  - isParentActive(monitor.id)                           │        │
│  │  - getAllPath(monitor.id, monitor.name)                 │        │
│  └─────────────────────────────────────────────────────────┘        │
│           ↓                                                         │
│  monitor.toJSON(preloadData)  ──→  序列化每个监控                    │
│           ↓                                                         │
│  Socket.emit("monitorList", list)                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        前端数据接收阶段                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  socket.on("monitorList", (data) => {                               │
│      this.assignMonitorUrlParser(data);                             │
│      this.monitorList = data;  // 存储到 $root.monitorList           │
│  })                                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        前端过滤显示阶段                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户操作过滤器 / 搜索框                                             │
│           ↓                                                         │
│  MonitorListFilter.toggleXxxFilter()                                │
│           ↓                                                         │
│  this.$emit("updateFilter", newFilter)                              │
│           ↓                                                         │
│  MonitorList.updateFilter(newFilter)                                │
│           ↓                                                         │
│  this.filterState = newFilter  ← Vue 响应式更新                     │
│           ↓                                                         │
│  sortedMonitorList 计算属性重新计算                                  │
│           ↓                                                         │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  过滤逻辑：                                              │        │
│  │  1. 根级别过滤：parent === null                          │        │
│  │  2. filterFunc 检查：                                    │        │
│  │     - 分组特殊处理（有子项匹配则显示）                     │        │
│  │     - 搜索文本匹配（名称/标签名/标签值）                   │        │
│  │     - 状态匹配（基于 lastHeartbeatList）                 │        │
│  │     - 激活状态匹配                                       │        │
│  │     - 标签匹配（OR 逻辑，取交集）                         │        │
│  │  3. sortFunc 排序                                        │        │
│  └─────────────────────────────────────────────────────────┘        │
│           ↓                                                         │
│  MonitorListItem 递归渲染子项                                        │
│           ↓                                                         │
│  sortedChildMonitorList 计算属性                                    │
│           ↓                                                         │
│  同样的 filterFunc 应用到子项                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 状态页分组独立链路

```
┌─────────────────────────────────────────────────────────────────────┐
│                    状态页分组数据链路（独立）                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  访问 /status/:slug                                                 │
│           ↓                                                         │
│  StatusPage.renderHTML(indexHTML, statusPage)                       │
│           ↓                                                         │
│  StatusPage.getStatusPageData(statusPage)                           │
│           ↓                                                         │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  状态页分组查询：                                        │        │
│  │  R.find("group", "public = 1 AND status_page_id = ?")   │        │
│  │           ↓                                             │        │
│  │  groupBean.toPublicJSON()                               │        │
│  │           ↓                                             │        │
│  │  groupBean.getMonitorList()  ← monitor_group 中间表     │        │
│  │           ↓                                             │        │
│  │  SELECT monitor.*, monitor_group.send_url, ...         │        │
│  │  FROM monitor, monitor_group                           │        │
│  │  WHERE monitor.id = monitor_group.monitor_id           │        │
│  │  AND group_id = ?                                      │        │
│  └─────────────────────────────────────────────────────────┘        │
│           ↓                                                         │
│  window.preloadData = { config, publicGroupList, ... }              │
│           ↓                                                         │
│  前端 StatusPage.vue 渲染                                           │
│           ↓                                                         │
│  独立逻辑，不使用 MonitorList.vue 的过滤系统                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 十、关键代码位置总结

### 10.1 监控层级分组

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| parent 字段定义 | `db/knex_init_db.js` | 496 |
| 获取父节点 | `server/model/monitor.js` | 1926-1936 |
| 获取子节点 | `server/model/monitor.js` | 1943-1951 |
| 获取所有子节点ID | `server/model/monitor.js` | 1980-1995 |
| 分组监控状态计算 | `server/monitor-types/group.js` | 12-74 |
| 根级别过滤 | `src/components/MonitorList.vue` | 192-198 |
| 子项过滤 | `src/components/MonitorListItem.vue` | 144-156 |
| 分组折叠状态 | `src/components/MonitorListItem.vue` | 181-219 |
| 全局折叠/展开 | `src/components/MonitorList.vue` | 354-393 |

### 10.2 状态页分组

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| group 表定义 | `db/knex_init_db.js` | 25-34 |
| monitor_group 表定义 | `db/knex_init_db.js` | 247-270 |
| Group 模型 | `server/model/group.js` | 1-49 |
| 获取状态页分组 | `server/model/status_page.js` | 309-340 |
| 保存状态页分组 | `server/socket-handlers/status-page-socket-handler.js` | 353-409 |

### 10.3 标签系统

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| Tag 模型 | `server/model/tag.js` | 1-17 |
| 获取监控标签 | `server/model/monitor.js` | 261-266 |
| 批量获取标签 | `server/model/monitor.js` | 1827-1836 |
| 标签过滤逻辑 | `src/components/MonitorList.vue` | 566-573 |
| 标签过滤切换 | `src/components/MonitorListFilter.vue` | 237-252 |
| 获取标签列表 | `src/components/MonitorListFilter.vue` | 258-264 |

### 10.4 过滤器系统

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 过滤器状态定义 | `src/components/MonitorList.vue` | 158-163 |
| 核心过滤逻辑 | `src/components/MonitorList.vue` | 528-576 |
| 排序逻辑 | `src/components/MonitorList.vue` | 583-605 |
| 状态过滤切换 | `src/components/MonitorListFilter.vue` | 205-236 |
| 激活状态过滤切换 | `src/components/MonitorListFilter.vue` | 221-236 |
| 数据预加载 | `server/model/monitor.js` | 1844-1919 |
| 监控序列化 | `server/model/monitor.js` | 117-254 |
| 发送监控列表 | `server/uptime-kuma-server.js` | 219-277 |
| 前端接收数据 | `src/mixins/socket.js` | 147-163 |

## 十一、结论

### 11.1 两套分组模型的本质区别

1. **监控层级分组**：
   - 基于 `monitor.parent` 自引用外键
   - 支持无限嵌套树形结构
   - 分组本身是一种特殊类型的监控（`type: "group"`）
   - 有自己的心跳状态（聚合子节点状态）
   - 用于监控列表的组织和拖拽排序

2. **状态页分组**：
   - 基于独立的 `group` 表和 `monitor_group` 中间表
   - 扁平结构，不支持嵌套
   - 分组不是监控，不参与监控检查
   - 仅用于状态页的展示分组
   - 可以为每个监控设置单独的 URL 显示配置

### 11.2 过滤系统的关键特性

1. **纯前端过滤**：所有过滤逻辑在前端完成，服务器只提供完整数据
2. **Vue 响应式**：通过计算属性 `sortedMonitorList` 自动更新
3. **分组特殊处理**：分组只要有子项匹配就显示，即使分组本身不匹配
4. **标签 OR 逻辑**：多选标签时是 OR 关系，不是 AND
5. **搜索范围广**：支持监控名称、标签名称、标签值

### 11.3 关键边界场景处理

代码中已经处理的边界场景：
- 空分组状态（PENDING）
- 直接访问子监控时自动展开
- 全局折叠时导航到根节点
- 标签值为 null 的情况（可选链）
- 无心跳数据的监控状态处理
- 父节点强制不激活的传播

### 11.4 数据流完整性

从后端预加载到前端过滤的完整链路：
1. 后端：`preparePreloadData` 批量获取所有关联数据
2. 后端：`toJSON` 序列化，包含 `parent`、`childrenIDs`、`tags` 等
3. 前端：Socket 接收并存储到 `$root.monitorList`
4. 前端：用户操作更新 `filterState`
5. 前端：`sortedMonitorList` 计算属性响应式更新
6. 前端：`filterFunc` 递归应用过滤逻辑
7. 前端：`MonitorListItem` 递归渲染子项
