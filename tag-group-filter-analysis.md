# Uptime Kuma 标签、分组和过滤器对监控列表查询状态的影响分析

## 一、概述

本文档详细分析 Uptime Kuma 中标签（Tag）、分组（Group）和过滤器（Filter）如何影响监控列表（Monitor List）的查询状态。分析涵盖前端实现和后端数据模型两个层面。

## 二、核心数据结构

### 2.1 标签（Tag）

#### 数据模型
- 位置：`server/model/tag.js:1-17`
- 属性：
  - `id`: 标签唯一标识
  - `name`: 标签名称
  - `color`: 标签颜色

#### 关联关系
- 监控与标签通过中间表 `monitor_tag` 关联
- 查询 SQL：`server/model/monitor.js:261-266`
  ```sql
  SELECT mt.*, tag.name, tag.color 
  FROM monitor_tag mt 
  JOIN tag ON mt.tag_id = tag.id 
  WHERE mt.monitor_id = ? 
  ORDER BY tag.name
  ```

### 2.2 分组（Group）

#### 数据模型
- 位置：`server/model/group.js:1-49`
- 特性：
  - 分组是一种特殊类型的监控器（`type: "group"`）
  - 通过 `parent` 字段建立层级关系
  - 支持嵌套分组

#### 关联关系
- 子监控通过 `parent` 字段指向父分组
- 分组可以包含普通监控和其他分组
- 位置：`server/model/monitor.js:133`
  - `parent`: 父监控ID
  - `childrenIDs`: 子监控ID列表

### 2.3 监控器（Monitor）

#### 完整结构
- 位置：`server/model/monitor.js:117-254`
- 关键属性：
  - `id`: 监控唯一标识
  - `name`: 监控名称
  - `type`: 监控类型（包括 "group"）
  - `parent`: 父监控ID
  - `active`: 是否激活
  - `weight`: 排序权重
  - `tags`: 标签数组（来自 preloadData）

## 三、过滤器实现机制

### 3.1 过滤器状态定义

位置：`src/components/MonitorList.vue:158-163`

```javascript
filterState: {
    status: null,      // 按状态过滤
    active: null,      // 按激活状态过滤
    tags: null,        // 按标签过滤
}
```

### 3.2 过滤器组件

#### 组件结构
- `MonitorList.vue`: 主列表组件
- `MonitorListFilter.vue`: 过滤器UI组件
- `MonitorListFilterDropdown.vue`: 下拉菜单组件

#### 过滤器更新流程

1. **状态切换过滤**：`src/components/MonitorListFilter.vue:205-220`
   - 方法：`toggleStatusFilter(status)`
   - 支持状态：UP(1), DOWN(0), PENDING(2), MAINTENANCE(3)

2. **激活状态过滤**：`src/components/MonitorListFilter.vue:221-236`
   - 方法：`toggleActiveFilter(active)`
   - 支持值：`true` (运行中), `false` (已暂停)

3. **标签过滤**：`src/components/MonitorListFilter.vue:237-252`
   - 方法：`toggleTagFilter(tag)`
   - 通过 `tag.id` 进行过滤
   - 支持多选标签

### 3.3 过滤逻辑核心实现

位置：`src/components/MonitorList.vue:528-576`

```javascript
filterFunc(monitor) {
    // 1. 分组特殊处理：如果子监控匹配，分组也显示
    if (monitor.type === "group") {
        const children = Object.values(this.$root.monitorList).filter((m) => m.parent === monitor.id);
        if (children.some((child) => this.filterFunc(child))) {
            return true;
        }
    }

    // 2. 搜索文本匹配
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

    // 3. 状态过滤
    let statusMatch = true;
    if (this.filterState.status != null && this.filterState.status.length > 0) {
        if (monitor.id in this.$root.lastHeartbeatList && this.$root.lastHeartbeatList[monitor.id]) {
            monitor.status = this.$root.lastHeartbeatList[monitor.id].status;
        }
        statusMatch = this.filterState.status.includes(monitor.status);
    }

    // 4. 激活状态过滤
    let activeMatch = true;
    if (this.filterState.active != null && this.filterState.active.length > 0) {
        activeMatch = this.filterState.active.includes(monitor.active);
    }

    // 5. 标签过滤
    let tagsMatch = true;
    if (this.filterState.tags != null && this.filterState.tags.length > 0) {
        tagsMatch =
            monitor.tags
                .map((tag) => tag.tag_id)
                .filter((monitorTagId) => this.filterState.tags.includes(monitorTagId)).length > 0;
    }

    return searchTextMatch && statusMatch && activeMatch && tagsMatch;
}
```

## 四、分组影响机制

### 4.1 分组展示逻辑

#### 根级别过滤
位置：`src/components/MonitorList.vue:192-205`

```javascript
sortedMonitorList() {
    let result = Object.values(this.$root.monitorList);

    // 根列表不显示子项
    result = result.filter((monitor) => {
        if (monitor.parent !== null) {
            return false;
        }
        return true;
    });

    result = result.filter(this.filterFunc);
    result.sort(this.sortFunc);

    return result;
}
```

#### 子项过滤
位置：`src/components/MonitorListItem.vue:144-156`

```javascript
sortedChildMonitorList() {
    let result = Object.values(this.$root.monitorList);

    // 获取子项
    result = result.filter((childMonitor) => childMonitor.parent === this.monitor.id);

    // 对子项应用过滤
    result = result.filter(this.filterFunc);

    result.sort(this.sortFunc);

    return result;
}
```

### 4.2 分组折叠功能

#### 折叠状态存储
位置：`src/components/MonitorListItem.vue:189-200`

```javascript
beforeMount() {
    // 直接访问监控时自动展开
    if (this.monitor.childrenIDs.includes(parseInt(this.$route.params.id))) {
        this.isCollapsed = false;
        return;
    }

    // 从 localStorage 读取折叠状态
    let storage = window.localStorage.getItem("monitorCollapsed");
    if (storage === null) {
        return;
    }

    let storageObject = JSON.parse(storage);
    if (storageObject[`monitor_${this.monitor.id}`] == null) {
        return;
    }

    this.isCollapsed = storageObject[`monitor_${this.monitor.id}`];
}
```

#### 全局折叠/展开
位置：`src/components/MonitorList.vue:354-393`

```javascript
toggleCollapseAll() {
    const shouldCollapse = !this.allGroupsCollapsed;

    let storageObject = {};
    const storage = window.localStorage.getItem("monitorCollapsed");
    if (storage !== null) {
        storageObject = JSON.parse(storage);
    }

    this.groupMonitors.forEach((group) => {
        storageObject[`monitor_${group.id}`] = shouldCollapse;
    });

    window.localStorage.setItem("monitorCollapsed", JSON.stringify(storageObject));
    this.collapseKey++;  // 强制重新渲染
}
```

### 4.3 分组检测

位置：`src/components/MonitorList.vue:247-250`

```javascript
groupMonitors() {
    const monitors = Object.values(this.$root.monitorList);
    return monitors.filter((m) => m.type === "group" && monitors.some((child) => child.parent === m.id));
}
```

## 五、排序机制

位置：`src/components/MonitorList.vue:583-605`

```javascript
sortFunc(m1, m2) {
    // 1. 激活状态优先
    if (m1.active !== m2.active) {
        if (m1.active === false) {
            return 1;
        }
        if (m2.active === false) {
            return -1;
        }
    }

    // 2. 按权重排序
    if (m1.weight !== m2.weight) {
        if (m1.weight > m2.weight) {
            return -1;
        }
        if (m1.weight < m2.weight) {
            return 1;
        }
    }

    // 3. 按名称排序
    return m1.name.localeCompare(m2.name);
}
```

## 六、过滤状态对查询的影响

### 6.1 标签过滤影响

- **过滤逻辑**：`src/components/MonitorList.vue:566-573`
- **机制**：检查监控的标签ID数组与过滤器选择的标签ID数组是否有交集
- **特点**：
  - 支持多选标签
  - 使用 OR 逻辑（只要匹配任意选中的标签即可显示）
  - 分组如果包含匹配的子监控，分组自身也会显示

### 6.2 状态过滤影响

- **过滤逻辑**：`src/components/MonitorList.vue:551-558`
- **依赖数据**：`this.$root.lastHeartbeatList`（最新心跳数据）
- **支持状态**：
  - `0`: DOWN
  - `1`: UP
  - `2`: PENDING
  - `3`: MAINTENANCE

### 6.3 激活状态过滤影响

- **过滤逻辑**：`src/components/MonitorList.vue:560-564`
- **检查字段**：`monitor.active`
- **支持值**：`true` (运行中), `false` (已暂停)

### 6.4 搜索文本影响

- **过滤逻辑**：`src/components/MonitorList.vue:537-549`
- **匹配范围**：
  - 监控名称
  - 标签名称
  - 标签值

## 七、关键交互流程

### 7.1 过滤器应用流程

```
用户操作 (点击状态/标签/搜索)
    ↓
MonitorListFilter.vue 中的 toggleXxxFilter()
    ↓
触发 updateFilter 事件
    ↓
MonitorList.vue:346-348 updateFilter(newFilter)
    ↓
更新 filterState
    ↓
Vue 响应式更新 sortedMonitorList 计算属性
    ↓
重新应用 filterFunc 过滤
    ↓
UI 刷新显示过滤后的列表
```

### 7.2 分组折叠流程

```
用户点击折叠按钮
    ↓
MonitorListItem.vue:207-219 changeCollapsed()
    ↓
切换 isCollapsed 状态
    ↓
保存到 localStorage (monitorCollapsed)
    ↓
Vue 响应式更新 sortedChildMonitorList 显示/隐藏
```

### 7.3 标签获取流程

1. 前端通过 Socket 发送请求：`src/components/MonitorListFilter.vue:258-264`
   ```javascript
   getExistingTags() {
       this.$root.getSocket().emit("getTags", (res) => {
           if (res.ok) {
               this.tagsList = res.tags;
           }
       });
   }
   ```

2. 后端查询数据库并返回标签列表

## 八、关键代码位置总结

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 过滤器状态定义 | `src/components/MonitorList.vue` | 158-163 |
| 过滤逻辑实现 | `src/components/MonitorList.vue` | 528-576 |
| 排序逻辑 | `src/components/MonitorList.vue` | 583-605 |
| 根列表过滤 | `src/components/MonitorList.vue` | 189-205 |
| 子项过滤 | `src/components/MonitorListItem.vue` | 144-156 |
| 折叠状态管理 | `src/components/MonitorListItem.vue` | 181-219 |
| 全局折叠/展开 | `src/components/MonitorList.vue` | 354-393 |
| 标签模型 | `server/model/tag.js` | 1-17 |
| 分组模型 | `server/model/group.js` | 1-49 |
| 监控标签查询 | `server/model/monitor.js` | 261-266 |
| 标签过滤切换 | `src/components/MonitorListFilter.vue` | 237-252 |
| 状态过滤切换 | `src/components/MonitorListFilter.vue` | 205-236 |

## 九、结论

Uptime Kuma 的标签、分组和过滤器系统通过以下方式影响监控列表查询状态：

1. **标签过滤**：基于标签ID的数组交集匹配，支持多选，分组会因为包含匹配子项而显示
2. **分组机制**：通过 `parent` 字段建立层级结构，过滤逻辑会递归应用到所有层级
3. **状态过滤**：基于最新心跳状态，支持UP/DOWN/PENDING/MAINTENANCE四种状态
4. **激活状态过滤**：基于 `monitor.active` 字段，区分运行中和已暂停
5. **搜索功能**：模糊匹配监控名称、标签名称和标签值
6. **排序**：激活状态优先，然后按权重，最后按名称

整个过滤系统是客户端实现的，通过 Vue 的响应式计算属性实时更新列表显示，所有过滤逻辑都在前端完成，服务器只负责提供完整的监控列表数据。
