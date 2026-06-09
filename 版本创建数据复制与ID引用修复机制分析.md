# ToolJet 版本创建数据复制与 ID 引用修复机制分析

## 目录

1. [概述](#概述)
2. [版本创建入口流程](#版本创建入口流程)
3. [各实体数据复制与 ID 修复详解](#各实体数据复制与-id-修复详解)
4. [关键字段概念解释](#关键字段概念解释)
5. [version-specific globalDS 旧逻辑移除后的影响](#version-specific-globalds-旧逻辑移除后的影响)
6. [引用替换遗漏可能导致的实际问题](#引用替换遗漏可能导致的实际问题)
7. [总结](#总结)

---

## 概述

当 ToolJet 创建新的 **AppVersion**（正式版本）或 **draftVersion**（草稿版本）时，系统会从源版本复制全部应用数据，包括：

- `globalSettings`（全局设置）
- `pageSettings`（页面设置）
- `local/global datasource`（本地/全局数据源）
- `data query`（数据查询）
- `query events`（查询事件）
- `page`（页面）
- `component`（组件）
- `event handler`（事件处理器）
- `workflow bundle`（工作流构建包）

由于每个新实体都会生成新的 UUID，所有引用旧 ID 的地方都必须被替换为新 ID，否则会导致引用悬空。

核心入口：[VersionsCreateService.setupNewVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L43-L98)

---

## 版本创建入口流程

### 1. 创建请求入口

- **API 层**：[VersionService.createVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L141-L146)
- **实际创建**：[VersionUtilService.createVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts#L153-L232)

### 2. 初始 AppVersion 实体创建

在 `VersionUtilService.createVersion()` 中，首先创建新的 `AppVersion` 记录：

```typescript
const appVersion = await manager.save(
  AppVersion,
  manager.create(AppVersion, {
    name: versionName,
    appId: app.id,
    definition: versionFrom?.definition,     // 先复制 definition
    currentEnvironmentId: firstPriorityEnv?.id,
    status: AppVersionStatus.DRAFT,
    parentVersionId: versionCreateDto.versionFromId ? versionFromId : null,
    versionType: versionType ? versionType : AppVersionType.VERSION,
    createdBy: user.id,
    co_relation_id: app.co_relation_id,
    ...(app.type === APP_TYPES.MODULE && { moduleReferenceId: uuid() }),
    ...(branchId && { branchId }),
  })
);
```

**关键点**：
- `parentVersionId`：记录版本派生关系
- `co_relation_id`：继承自 App，用于跨实例/跨分支稳定标识
- `moduleReferenceId`：仅 module 类型应用生成，用于外部稳定引用
- `branchId`：git 分支版本专用

### 3. 数据复制与引用修复主流程

调用 [createVersionService.setupNewVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L43-L98) 执行完整的数据复制和引用修复：

```
setupNewVersion()
├── 复制 globalSettings / pageSettings / showViewerNavigation
├── createNewDataSourcesAndQueriesForVersion()  // 数据源 + 查询 + 查询事件
│   ├── 复制 local data sources
│   ├── 复制 global data queries（共享 global DS）
│   ├── 复制所有 data query events
│   ├── 修复 data query options 中的 query ID 引用
│   ├── 修复 definition 中的 query ID 引用
│   └── 为 local DS 创建 DSV + DSVO + 凭证
│
├── createNewPagesAndComponentsForVersion()    // 页面 + 组件 + 布局 + 页面/组件事件
│   ├── 复制 pages
│   ├── 复制 page events
│   ├── 复制 components（含 parent ID 重映射）
│   ├── 复制 layouts
│   └── 复制 component events
│
├── updateEntityReferencesForNewVersion()      // 深度遍历修复组件/查询属性中的引用
│   ├── 修复 component.properties/styles/... 中的 JS 表达式引用
│   └── 修复 data query.options 中的 JS 表达式引用
│
├── 修复 globalSettings 中的实体引用
│
├── updateEventActionsForNewVersionWithNewMappingIds()  // 事件动作的 ID 修复
│   ├── run-query → queryId
│   ├── control-component → componentId
│   ├── switch-page → pageId
│   ├── show-modal/close-modal → modal
│   └── set-table-page → table
│
└── copyWorkflowBundlesForVersion()  // 工作流 bundle（仅 workflow 类型应用）
```

---

## 各实体数据复制与 ID 修复详解

### 1. globalSettings & pageSettings

**位置**：[create.service.ts#L50-L53](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L50-L53)

```typescript
appVersion.showViewerNavigation = versionFrom.showViewerNavigation;
appVersion.globalSettings = versionFrom.globalSettings;
appVersion.pageSettings = versionFrom.pageSettings;
await manager.save(appVersion);
```

**globalSettings 的引用修复**（在后面单独处理）：

**位置**：[create.service.ts#L76-L83](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L76-L83)

```typescript
if (appVersion.globalSettings) {
  const globalSettings = appVersion.globalSettings;
  const updatedGlobalSettings = updateEntityReferences(globalSettings, {
    ...oldDataQueryToNewMapping,
    ...oldComponentToNewComponentMapping,
  });
  await manager.update(AppVersion, { id: appVersion.id }, { globalSettings: updatedGlobalSettings });
}
```

> **⚠️ 注意**：`pageSettings` 仅被直接复制，**没有**经过 `updateEntityReferences()` 进行引用替换。如果 `pageSettings` 中包含对组件或查询的引用（如 JS 表达式），将导致引用仍然指向旧版本的 ID。

---

### 2. DataSource（数据源）

数据源分为两种作用域：

- **`local`**：每个版本各自拥有，独立复制
- **`global`**：跨版本共享，不复制实体本身

**位置**：[create.service.ts#L100-L290](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L100-L290)

#### Local DataSource 复制

```typescript
const dataSources = versionFrom?.dataSources.filter(
  (ds) => ds.scope == DataSourceScopes.LOCAL
);

for (const dataSource of dataSources) {
  const dataSourceParams: Partial<DataSource> = {
    name: dataSource.name,
    kind: dataSource.kind,
    type: dataSource.type,
    co_relation_id: dataSource?.co_relation_id,  // 保留稳定标识
    appVersionId: appVersion.id,                   // 指向新版本
  };
  const newDataSource = await manager.save(manager.create(DataSource, dataSourceParams));
  dataSourceMapping[dataSource.id] = newDataSource.id;
  // ... 继续复制该数据源下的 data queries
}
```

**关键字段**：
- `co_relation_id`：保留源版本的 `co_relation_id`，用于跨版本/跨分支匹配
- `appVersionId`：更新为新版本 ID

#### Global DataSource 处理

```typescript
const globalQueries: DataQuery[] = await this.dataQueryRepository.getQueriesByVersionId(
  versionFrom.id,
  DataSourceScopes.GLOBAL,
  manager
);
const globalDataSources = [...new Map(globalQueries.map((gq) => [gq.dataSource.id, gq.dataSource])).values()];
```

Global 数据源**不复制实体**，新版本的 global data query 直接共享同一个 `dataSourceId`。

> **注意**：旧版本中曾有针对 global DS 的 per-version DataSourceVersion 机制，现已移除。详见 [version-specific globalDS 旧逻辑移除后的影响](#version-specific-globalds-旧逻辑移除后的影响)。

---

### 3. DataQuery（数据查询）

**位置**：[create.service.ts#L143-L172](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L143-L172)（local DS 的查询）

**位置**：[create.service.ts#L176-L206](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L176-L206)（global DS 的查询）

无论是 local 还是 global data source，**每个 data query 都会被复制**到新版本：

```typescript
// Local DS 查询
const dataQueryParams = {
  name: dataQuery.name,
  options: dataQuery.options,
  dataSourceId: newDataSource.id,    // local DS：指向新数据源
  appVersionId: appVersion.id,
  co_relation_id: dataQuery?.co_relation_id,
};

// Global DS 查询
const dataQueryParams = {
  name: globalQuery.name,
  options: globalQuery.options,
  dataSourceId: globalQuery.dataSourceId,  // global DS：共享同一个 DS
  appVersionId: appVersion.id,
  co_relation_id: globalQuery?.co_relation_id,
};
```

**生成新旧 ID 映射**：
```typescript
oldDataQueryToNewMapping[dataQuery.id] = newQuery.id;
```

#### DataQuery Options 内部引用修复

**位置**：[create.service.ts#L208-L213](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L208-L213)

```typescript
for (const newQuery of newDataQueries) {
  const newOptions = this.replaceDataQueryOptionsWithNewDataQueryIds(
    newQuery.options,
    oldDataQueryToNewMapping
  );
  newQuery.options = newOptions;
  await manager.save(newQuery);
}
```

**位置**：[create.service.ts#L292-L303](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L292-L303)

```typescript
protected replaceDataQueryOptionsWithNewDataQueryIds(options, dataQueryMapping) {
  if (options && options.events) {
    const replacedEvents = options.events.map((event) => {
      if (event.queryId) {
        event.queryId = dataQueryMapping[event.queryId];
      }
      return event;
    });
    options.events = replacedEvents;
  }
  return options;
}
```

> 仅修复 `options.events` 数组中的 `queryId`。更深度的引用（如 JS 表达式中的查询引用）由后续的 `updateEntityReferencesForNewVersion()` 处理。

---

### 4. QueryEvents（查询事件 / EventHandler）

**位置**：[create.service.ts#L153-L168](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L153-L168)（local DS 查询事件）

**位置**：[create.service.ts#L187-L202](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L187-L202)（global DS 查询事件）

每个 data query 关联的 `EventHandler`（`target = data_query`）都会被复制：

```typescript
const dataQueryEvents = allEvents.filter((event) => event.sourceId === dataQuery.id);

dataQueryEvents.forEach(async (event, index) => {
  const newEvent = new EventHandler();
  newEvent.id = uuid.v4();
  newEvent.name = event.name;
  newEvent.sourceId = newQuery.id;           // 指向新查询
  newEvent.target = event.target;             // Target.dataQuery
  newEvent.event = event.event;               // ⚠️ 原始 event JSON 直接复制
  newEvent.index = event.index ?? index;
  newEvent.appVersionId = appVersion.id;
  newEvent.co_relation_id = event?.co_relation_id;
  await manager.save(newEvent);
});
```

> **⚠️ 注意**：此处复制时 `event` 字段（事件动作定义）是直接复制的，里面可能包含对其他查询/组件的引用。这些引用会在后续的 `updateEventActionsForNewVersionWithNewMappingIds()` 中统一修复。

---

### 5. definition 中的查询引用修复

**位置**：[create.service.ts#L215-L216](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L215-L216)

```typescript
appVersion.definition = this.replaceDataQueryIdWithinDefinitions(
  appVersion.definition,
  oldDataQueryToNewMapping
);
```

**位置**：[create.service.ts#L305-L379](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L305-L379)

该方法遍历 `definition.pages` 下所有页面和组件，修复：
- 页面事件（`definition.pages[pageId].events`）中的 `queryId`
- 组件定义事件（`component.definition.events`）中的 `queryId`
- 组件动作属性（`component.definition.properties.actions.value`）中的 `queryId`
- Table 组件列事件（`column.events`）中的 `queryId`
- 工作流定义查询（`definition.queries`）中的 `id`

> **注意**：这只是第一层的 `queryId` 显式字段替换。嵌套在 JS 表达式（`{{...}}`）中的引用由 `updateEntityReferences()` 处理。

---

### 6. Page（页面）

**位置**：[create.service.ts#L467-L488](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L467-L488)

```typescript
const savedPage = await manager.save(
  manager.create(Page, {
    name: page.name,
    handle: page.handle,
    index: page.index,
    disabled: page.disabled,
    hidden: page.hidden,
    pageHeader: page.pageHeader,
    pageFooter: page.pageFooter,
    icon: page.icon,
    type: page.type,
    openIn: page.openIn,
    appId: page.appId,
    url: page.url,
    pageGroupId: page.pageGroupId,
    pageGroupIndex: page.pageGroupIndex,
    isPageGroup: page.isPageGroup,
    appVersionId: appVersion.id,      // 指向新版本
    co_relation_id: page.co_relation_id, // 保留稳定标识
  })
);
oldPageToNewPageMapping[page.id] = savedPage.id;
```

**主页 ID 更新**：
```typescript
if (page.id === prevHomePagePage) {
  homePageId = savedPage.id;
}
// ...
await manager.update(AppVersion, { id: appVersion.id }, { homePageId });
```

**页面事件复制**：
```typescript
const pageEvents = allEvents.filter((event) => event.sourceId === page.id);
// 逻辑与查询事件类似，sourceId 更新为新页面 ID
```

---

### 7. Component（组件）

**位置**：[create.service.ts#L511-L625](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L511-L625)

组件复制是最复杂的部分，因为涉及 `parent` 引用的重映射。

#### 7.1 跳过损坏组件

```typescript
const originalPageComponents = page.components.filter(
  (component) => !shouldSkipComponentOnVersionCopy(component)
);
```

**位置**：[version-copy-parent.helper.ts#L27-L29](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/helpers/version-copy-parent.helper.ts#L27-L29)

```typescript
export function shouldSkipComponentOnVersionCopy(component) {
  return !!component.parent?.startsWith('undefined');
}
```

跳过 `parent` 以 `undefined` 开头的损坏组件。

#### 7.2 预生成所有新组件 ID

```typescript
const tempNewComponents: Component[] = [];
for (const component of originalPageComponents) {
  const newComponent = new Component();
  newComponent.id = uuid.v4();
  newComponent.co_relation_id = component?.co_relation_id;
  oldComponentToNewComponentMapping[component.id] = newComponent.id;
  tempNewComponents.push(newComponent);
}
```

先建立完整的新旧 ID 映射，再逐个填充属性，这样处理 parent 引用时可以查到所有新 ID。

#### 7.3 Parent ID 重映射

**位置**：[version-copy-parent.helper.ts#L47-L65](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/helpers/version-copy-parent.helper.ts#L47-L65)

```typescript
export function remapParentIdForVersionCopy(
  parentId: string,
  idMap: Record<string, string>,
  validComponentIds: Set<string>,
  suffix?: string
): string | null {
  // 复合 ID（如 tabsId-t2）：重映射 baseId，保留 suffix
  if (compositeSuffix && baseId) {
    const newBaseId = idMap[baseId];
    if (newBaseId) return `${newBaseId}-${compositeSuffix}`;
    return isGhostParent(parentId, validComponentIds) ? parentId : null;
  }
  // 普通 ID：直接映射，找不到则判断是否为 ghost parent
  return idMap[parentId] ?? (isGhostParent(parentId, validComponentIds) ? parentId : null);
}
```

**Parent 处理规则**：

| 情况 | 处理方式 |
|------|---------|
| `canvas-header` / `canvas-footer` | 保留原样（虚拟容器，不是 UUID） |
| Tabs/Calendar 子项（带 suffix） | 重映射 baseId，保留 suffix |
| Kanban Modal 子项 | 重映射 baseId，保留 `modal` suffix |
| 普通组件 parent | 在 ID map 中查找替换 |
| Ghost parent（不在本页的 UUID） | 保留原始值（兼容历史数据） |
| 其他情况 | 设为 `null`（主画布） |

#### 7.4 buttonToSubmit 属性引用修复

```typescript
if (newComponent?.properties?.buttonToSubmit?.value) {
  const oldButtonId = newComponent.properties.buttonToSubmit.value;
  if (oldButtonId && oldComponentToNewComponentMapping[oldButtonId]) {
    set(newComponent, 'properties.buttonToSubmit.value', oldComponentToNewComponentMapping[oldButtonId]);
  }
}
```

显式修复 `buttonToSubmit` 属性中的组件 ID 引用。

---

### 8. Layout（布局）

**位置**：[create.service.ts#L592-L607](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L592-L607)

```typescript
originalComponent.layouts.forEach((layout) => {
  const newLayout = new Layout();
  newLayout.id = uuid.v4();
  newLayout.co_relation_id = layout?.co_relation_id;
  newLayout.type = layout.type;
  newLayout.top = layout.top;
  newLayout.left = layout.left;
  newLayout.width = layout.width;
  newLayout.height = layout.height;
  newLayout.componentId = newComponent.id;  // 指向新组件
  newLayout.dimensionUnit = LayoutDimensionUnits.COUNT;
  newComponentLayouts.push(newLayout);
});
```

Layout 的 `componentId` 更新为新组件 ID。

---

### 9. ComponentEvents（组件事件）

**位置**：[create.service.ts#L609-L622](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L609-L622)

与查询事件类似，组件的 `EventHandler` 会被复制，`sourceId` 指向新组件。

```typescript
const componentEvents = allEvents.filter((event) => event.sourceId === originalComponent.id);
componentEvents.forEach(async (event, index) => {
  const newEvent = new EventHandler();
  newEvent.id = uuid.v4();
  newEvent.co_relation_id = event?.co_relation_id;
  newEvent.name = event.name;
  newEvent.sourceId = newComponent.id;  // 新组件 ID
  newEvent.target = event.target;
  newEvent.event = event.event;         // ⚠️ 原始 event JSON 直接复制
  newEvent.index = event.index ?? index;
  newEvent.appVersionId = appVersion.id;
  await manager.save(newEvent);
});
```

> `event` 字段中的引用由后续统一修复。

---

### 10. updateEntityReferencesForNewVersion（深度引用修复）

**位置**：[create.service.ts#L636-L680](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L636-L680)

```typescript
protected async updateEntityReferencesForNewVersion(manager, resourceMapping) {
  const mappings = { ...resourceMapping.componentsMapping, ...resourceMapping.dataQueryMapping };
  
  // 修复组件属性中的引用
  if (newComponentIds.length > 0) {
    const components = await manager
      .createQueryBuilder(Component, 'components')
      .where('components.id IN(:...componentIds)', { componentIds: newComponentIds })
      .select([
        'components.id',
        'components.properties',
        'components.styles',
        'components.general',
        'components.validation',
        'components.generalStyles',
        'components.displayPreferences',
      ])
      .getMany();

    const toUpdateComponents = components.filter((component) => {
      return updateEntityReferences(component, mappings);
    });
    if (!isEmpty(toUpdateComponents)) await manager.save(toUpdateComponents);
  }

  // 修复查询选项中的引用
  if (newQueriesIds.length > 0) {
    // ... 类似逻辑，修复 dataQuery.options
  }
}
```

核心是调用 `updateEntityReferences()` 函数深度遍历 JSON 对象，替换所有 JS 表达式中的实体 ID。

#### updateEntityReferences 原理

**位置**：[import_export.helpers.ts#L32-L45](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/import_export.helpers.ts#L32-L45)

```typescript
export function updateEntityReferences(node, resourceMapping = {}) {
  if (typeof node === 'object') {
    for (const key in node) {
      let value = node[key];
      if (typeof value === 'string' && value.includes('{{') && value.includes('}}')) {
        node[key] = extractAndReplaceReferencesFromString(value, resourceMapping, resourceMapping)?.valueWithId;
      } else if (typeof value === 'object') {
        value = updateEntityReferences(value, resourceMapping);
      }
    }
  }
  return node;
}
```

工作原理：
1. 递归遍历对象的所有属性
2. 遇到包含 `{{...}}` 的字符串，调用 `extractAndReplaceReferencesFromString()`
3. 使用 acorn 解析 JS 表达式 AST
4. 查找 `components.xxx` 和 `queries.xxx` 形式的成员访问
5. 如果 `xxx` 在映射表中，替换为新 ID

**位置**：[import_export.helpers.ts#L247-L323](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/import_export.helpers.ts#L247-L323)

```typescript
walk.simple(ast, {
  MemberExpression(node) {
    if (node.object.type === 'Identifier' &&
        (node.object.name === 'components' || node.object.name === 'queries')) {
      // 查找映射，替换 ID
    }
  }
});
```

---

### 11. updateEventActionsForNewVersionWithNewMappingIds（事件动作修复）

**位置**：[create.service.ts#L682-L720](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L682-L720)

```typescript
protected async updateEventActionsForNewVersionWithNewMappingIds(
  manager, versionId, oldDataQueryToNewMapping, oldComponentToNewComponentMapping, oldPageToNewPageMapping
) {
  const allEvents = await manager.find(EventHandler, { where: { appVersionId: versionId } });
  const mappings = { ...oldDataQueryToNewMapping, ...oldComponentToNewComponentMapping };

  for (const event of allEvents) {
    // 第一步：通过 updateEntityReferences 修复 JS 表达式中的引用
    const eventDefinition = updateEntityReferences(event.event, mappings);

    // 第二步：显式修复各 action 类型的特定 ID 字段
    if (eventDefinition?.actionId === 'run-query') {
      eventDefinition.queryId = oldDataQueryToNewMapping[eventDefinition.queryId];
    }
    if (eventDefinition?.actionId === 'control-component') {
      eventDefinition.componentId = oldComponentToNewComponentMapping[eventDefinition.componentId];
    }
    if (eventDefinition?.actionId === 'switch-page') {
      eventDefinition.pageId = oldPageToNewPageMapping[eventDefinition.pageId];
    }
    if (eventDefinition?.actionId == 'show-modal' || eventDefinition?.actionId === 'close-modal') {
      eventDefinition.modal = oldComponentToNewComponentMapping[eventDefinition.modal];
    }
    if (eventDefinition?.actionId === 'set-table-page') {
      eventDefinition.table = oldComponentToNewComponentMapping[eventDefinition.table];
    }

    event.event = eventDefinition;
    await manager.save(event);
  }
}
```

**修复的事件动作类型**：

| Action ID | 修复字段 | 映射表 |
|-----------|---------|--------|
| `run-query` | `queryId` | 数据查询映射 |
| `control-component` | `componentId` | 组件映射 |
| `switch-page` | `pageId` | 页面映射 |
| `show-modal` / `close-modal` | `modal` | 组件映射 |
| `set-table-page` | `table` | 组件映射 |

此外，`event.event` 中所有 JS 表达式内的引用也会通过 `updateEntityReferences()` 修复。

---

### 12. WorkflowBundle（工作流构建包）

**位置**：[create.service.ts#L722-L747](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L722-L747)

仅当应用类型为 `workflow` 时执行：

```typescript
if (app?.type === APP_TYPES.WORKFLOW) {
  await this.copyWorkflowBundlesForVersion(manager, appVersion.id, versionFrom.id);
}

protected async copyWorkflowBundlesForVersion(manager, newAppVersionId, sourceAppVersionId) {
  const sourceBundles = await manager.find(WorkflowBundle, {
    where: { appVersionId: sourceAppVersionId },
  });

  for (const bundle of sourceBundles) {
    await manager.save(
      manager.create(WorkflowBundle, {
        appVersionId: newAppVersionId,
        language: bundle.language,
        runtimeVersion: bundle.runtimeVersion,
        dependencies: bundle.dependencies,
        bundleBinary: bundle.bundleBinary,
        bundleSize: bundle.bundleSize,
        bundleSha: bundle.bundleSha,
        status: bundle.status,
        generationTimeMs: bundle.generationTimeMs,
        error: bundle.error,
      })
    );
  }
}
```

**注意**：WorkflowBundle 是纯运行时资产，不包含对其他版本实体的 ID 引用，因此不需要 ID 修复，直接复制即可。

---

## 关键字段概念解释

### moduleReferenceId

**定义位置**：[app_version.entity.ts#L115-L116](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts#L115-L116)

```typescript
@Column({ name: 'module_reference_id', type: 'uuid', nullable: true })
moduleReferenceId: string;
```

**作用**：

`moduleReferenceId` 是**模块版本**的稳定外部引用标识，主要用于 `ModuleViewer` 组件引用模块版本。

**生成时机**：仅当 App 类型为 `module` 时，在创建版本时自动生成 UUID。

**位置**：[util.service.ts#L206](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts#L206)

```typescript
...(app.type === APP_TYPES.MODULE && { moduleReferenceId: uuid() }),
```

**使用场景**：

ModuleViewer 组件在其 `properties` 中存储两个值：
- `moduleAppId`：模块 App 的 `co_relation_id`
- `moduleVersionId`：模块版本的 `module_reference_id`

**解析逻辑**：[module-ref.util.ts#L101-L143](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L101-L143)

```
解析规则：
1. moduleReferenceId 存在且匹配 → 固定引用（pinned）
   - 优先匹配消费者分支上的版本
   - 否则匹配默认分支上的版本
2. moduleReferenceId 为空 → 未固定（unpinned）
   - 使用消费者分支上的最新版本
3. moduleReferenceId 存在但无匹配 → 孤立（orphaned）
   - 回退到默认分支的最新版本
```

**为什么需要这个字段？**

- 数据库主键 `id` 是实例本地的，跨实例（如 git clone 后的工作区）不相同
- `moduleReferenceId` 是稳定的 UUID，可以在 git push/pull、zip 导入导出时保持一致
- 支持 ModuleViewer 在不同工作区、不同分支间正确解析模块版本

---

### co_relation_id

**定义位置**：几乎所有版本相关实体都有此字段（App、AppVersion、DataSource、DataQuery、Page、Component、EventHandler、Layout 等）

```typescript
@Column({ name: 'co_relation_id', nullable: true })
co_relation_id: string;
```

**作用**：

`co_relation_id`（correlation ID）是**跨版本/跨实例/跨分支的稳定标识**，用于在不同数据库实例、不同版本、不同分支之间识别"同一个"实体。

**在版本复制中的表现**：

所有被复制的实体都会**保留源实体的 `co_relation_id`**，例如：

- **DataSource**：[create.service.ts#L135](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L135)
- **DataQuery**：[create.service.ts#L149](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L149)
- **Page**：[create.service.ts#L486](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L486)
- **Component**：[create.service.ts#L520](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L520)
- **EventHandler**：[create.service.ts#L165](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L165)
- **Layout**：[create.service.ts#L595](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L595)

**典型应用场景**：

1. **Git Sync 分支协调**：通过 `co_relation_id` 匹配不同分支上的同一实体，实现跨分支数据同步
2. **Import/Export**：导入导出时通过 `co_relation_id` 识别已存在的实体，避免重复创建
3. **模块版本解析**：ModuleViewer 通过 `co_relation_id` 找到模块 App
4. **模块 viewer pins 协调**：[reconcileModuleViewerPinsFromDefault()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L159-L233) 通过 `co_relation_id` 在默认分支和特性分支间匹配 ModuleViewer 组件

---

### parentVersionId

**定义位置**：[app_version.entity.ts#L76-L77](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts#L76-L77)

```typescript
@Column({ name: 'parent_version_id', type: 'uuid', nullable: true })
parentVersionId: string;
```

**作用**：记录版本的**直接父版本**，即新版本是从哪个版本派生/复制而来。

**设置时机**：[util.service.ts#L201](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts#L201)

```typescript
parentVersionId: versionCreateDto.versionFromId ? versionFromId : null,
```

**用途**：

1. 版本谱系追踪：可以追溯版本的演变历史
2. 查找子版本：[findParentVersionApps()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/repository.ts#L280-L287) 可查找某个版本的所有子版本
3. 草稿版本命名：创建草稿版本时，根据父版本的子版本数量自动命名

---

### branchId

**定义位置**：[app_version.entity.ts#L108-L113](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts#L108-L113)

```typescript
@Column({ name: 'branch_id', nullable: true })
branchId: string;

@ManyToOne(() => WorkspaceBranch, { nullable: true, onDelete: 'SET NULL' })
@JoinColumn({ name: 'branch_id' })
branch: WorkspaceBranch;
```

**作用**：标识版本所属的 **Git 分支**（当启用 Git Sync 分支功能时）。

**与 versionType 的关系**：

- `versionType = 'version'` + `branchId = null`：普通版本（非 git 分支环境）
- `versionType = 'version'` + `branchId = defaultBranchId`：默认分支上的版本
- `versionType = 'branch'` + `branchId = featureBranchId`：特性分支草稿

**位置**：[util.service.ts#L203-L207](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts#L203-L207)

```typescript
versionType: versionType ? versionType : AppVersionType.VERSION,
...(branchId && { branchId }),
```

**应用场景**：

1. **模块版本解析**：根据消费者的 `branchId` 解析对应的模块版本
2. **版本列表过滤**：按分支筛选版本列表
3. **草稿版本约束**：启用分支时，每个分支只允许一个 draft 版本
4. **全局数据源版本**：DataSourceVersion 也有 `branchId`，每个分支有自己的 DSV

---

### versionType

**定义位置**：[app_version.entity.ts#L28-L31](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts#L28-L31)

```typescript
export enum AppVersionType {
  VERSION = 'version',
  BRANCH = 'branch',
}
```

**作用**：区分版本的类型——是正式版本还是分支草稿。

**两者区别**：

| 特性 | `version` 类型 | `branch` 类型 |
|------|-------------|-------------|
| 用途 | 保存/发布的正式版本 | Git 分支上的编辑草稿 |
| 状态流转 | DRAFT → PUBLISHED → RELEASED | 通常保持 DRAFT |
| 命名 | 用户自定义 | 通常与分支同名 |
| 模块解析 | 可被固定引用（pin） | 仅在对应分支上可见 |
| 数量限制 | 无限制（但发布版有限制） | 每个分支最多一个 draft |

---

## version-specific globalDS 旧逻辑移除后的影响

### 旧逻辑是什么？

在 [create.service.ts#L267-L286](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L267-L286) 的注释中保留了旧逻辑的痕迹：

```typescript
// Removed: version-specific DSVs (app_version_id) are no longer created.
// Released versions now read from the main-branch default DSV (is_default = true).
// for (const globalDs of globalDataSources) {
//   const dsvName = globalDs.name || 'v1';
//   const existingDsv = await manager.findOne(DataSourceVersion, { ... });
//   if (existingDsv) { continue; }
//   let sourceDsv = await manager.findOne(DataSourceVersion, {
//     where: { dataSourceId: globalDs.id, appVersionId: versionFrom.id },
//   });
//   ... 创建新的 per-version DSV ...
// }
```

### 旧逻辑架构

```
旧模式：
  Global DataSource
    ├── DataSourceVersion (app_version_id = v1)  ← 版本 v1 专用
    │   └── DataSourceVersionOptions (各环境)
    ├── DataSourceVersion (app_version_id = v2)  ← 版本 v2 专用
    │   └── DataSourceVersionOptions (各环境)
    └── DataSourceVersion (is_default = true)    ← 默认版本
        └── DataSourceVersionOptions (各环境)
```

每个 AppVersion 都有自己专属的 global DS 配置（通过 `app_version_id` 关联的 DataSourceVersion）。

### 新模式架构

```
新模式：
  Global DataSource
    └── DataSourceVersion (is_default = true, branch_id = xxx)  ← 分支默认
        └── DataSourceVersionOptions (各环境)
```

所有版本共享同一个**默认 DataSourceVersion**（`is_default = true`），按 `branchId` 区分分支。

### 移除原因

1. **减少数据冗余**：每个版本都复制一份 global DS 配置是不必要的
2. **简化架构**：global DS 的核心特性就是"全局共享"，per-version 违背了这一设计
3. **与分支模型整合**：新模型以 `branchId` 为维度隔离，而不是 `appVersionId`
4. **降低维护成本**：数据源配置变更时不需要同步到每个版本

### 潜在影响

| 方面 | 影响 |
|------|------|
| **版本隔离性** | 不同版本的 global DS 配置不再独立。修改 global DS 配置会影响所有使用该 DS 的版本 |
| **版本回溯** | 无法通过旧版本查看当时的数据源配置（因为配置是共享的） |
| **发布安全** | 发布版本不再有"冻结"的数据源配置快照 |
| **分支隔离** | 不同分支仍有各自的 DSV（通过 branchId 区分），分支级隔离保留 |
| **Local DS** | local data source 不受影响，仍然是 per-version 的 |

> **注意**：这是一个有意的架构决策——global DS 本意就是全局共享，per-version 版本实际上混淆了 global 和 local 的界限。

---

## 引用替换遗漏可能导致的实际问题

### 潜在遗漏点：pageSettings 的引用未修复

**问题描述**：

在 [setupNewVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L50-L83) 中：

```typescript
appVersion.globalSettings = versionFrom.globalSettings;
appVersion.pageSettings = versionFrom.pageSettings;
// ...
// globalSettings 有引用修复：
if (appVersion.globalSettings) {
  const updatedGlobalSettings = updateEntityReferences(globalSettings, { ... });
  await manager.update(AppVersion, { id: appVersion.id }, { globalSettings: updatedGlobalSettings });
}
// ⚠️ pageSettings 没有引用修复！
```

`pageSettings` 被直接复制到新版本，但**没有**经过 `updateEntityReferences()` 进行 ID 引用替换。

### 可能导致的实际问题

假设 `pageSettings` 中存储了类似这样的配置（实际结构需要进一步确认）：

```javascript
{
  "defaultPageSize": "{{queries.abc123-def456.data.length}}",
  "pageTitleTemplate": "详情 - {{components.xyz789-abc123.text}}"
}
```

如果 `pageSettings` 包含引用组件或查询的 JS 表达式，创建新版本后：

1. **表达式指向旧版本实体**：`{{queries.old-uuid.data}}` 仍然指向旧版本的查询 ID
2. **运行时查询失败**：当页面加载时，JS 表达式尝试访问旧查询的数据，但旧查询可能不存在或数据不同
3. **组件引用失效**：引用的组件 ID 在新版本中不存在，导致 `undefined` 错误
4. **难以调试**：错误表现为页面行为异常或数据不显示，但不会直接报"ID 不存在"的明显错误
5. **跨版本干扰**：如果旧版本被删除或修改，新版本的 pageSettings 行为会不可预测

### 验证方式

需要确认 `pageSettings` 的实际内容结构。如果其中包含：
- JS 表达式（`{{...}}`）
- 对 `components` 或 `queries` 的引用
- 任何基于 ID 的引用

那么这就是一个真实的 Bug。

### 其他可能的遗漏点

| 位置 | 是否处理 | 风险 |
|------|---------|------|
| `globalSettings` | ✅ 已处理（updateEntityReferences） | 低 |
| `pageSettings` | ❌ 未处理 | 中/高（取决于实际内容） |
| `definition` 中的 query IDs | ✅ 已处理（replaceDataQueryIdWithinDefinitions） | 低 |
| `definition` 中的 JS 表达式 | ❓ 未明确处理（可能依赖前端） | 中 |
| DataSource.options | ❌ 未处理（但 DS 配置通常不包含查询引用） | 低 |
| WorkflowBundle | ✅ 不需要（无内部引用） | 低 |
| DataQueryFolder / FolderMapping | ⚠️ 未在复制流程中看到 | 中 |

---

## 总结

### 版本创建数据复制总览

```
AppVersion (新版本)
├── 基本属性复制（name, status 等）
├── globalSettings → 经 updateEntityReferences 修复引用 ✅
├── pageSettings → 直接复制，未修复引用 ⚠️
├── definition → 经 replaceDataQueryIdWithinDefinitions 修复 queryId
├── DataSources (local scope) → 新建实体，更新 appVersionId
│   └── DataQueries → 新建实体，更新 dataSourceId + appVersionId
│       └── EventHandlers (query events) → 新建实体，更新 sourceId
├── DataQueries (global scope) → 新建实体，共享 dataSourceId
│   └── EventHandlers (query events) → 新建实体，更新 sourceId
├── Pages → 新建实体，更新 appVersionId
│   └── EventHandlers (page events) → 新建实体，更新 sourceId
│   └── Components → 新建实体，更新 pageId + parent 重映射
│       ├── Layouts → 新建实体，更新 componentId
│       └── EventHandlers (component events) → 新建实体，更新 sourceId
├── 所有 EventHandlers 的 event 字段 → 经 updateEventActions 修复动作 ID
├── Components properties/styles/etc → 经 updateEntityReferences 修复 JS 表达式引用
├── DataQueries options → 经 updateEntityReferences 修复 JS 表达式引用
└── WorkflowBundles (仅 workflow 应用) → 直接复制
```

### 关键 ID 概念对比

| 字段 | 作用域 | 稳定性 | 主要用途 |
|------|--------|--------|---------|
| `id` (主键) | 表内唯一 | 每个版本新建 | 数据库关联 |
| `co_relation_id` | 跨版本/跨实例 | 复制时保留 | 实体匹配、git sync、导入导出 |
| `moduleReferenceId` | 跨分支/跨实例 | 仅 module 版本生成，版本间不同 | ModuleViewer 稳定引用 |
| `parentVersionId` | 同 App 内 | 创建时确定 | 版本谱系追踪 |
| `branchId` | 同组织内 | 分支级 | Git 分支隔离 |

### 引用修复的三层机制

1. **显式字段修复**：直接替换已知字段（如 `event.queryId`、`component.parent`、`buttonToSubmit`）
2. **JS 表达式深度扫描**：`updateEntityReferences()` 递归遍历 JSON，用 AST 解析替换 `{{...}}` 中的实体 ID
3. **事件动作专项修复**：`updateEventActionsForNewVersionWithNewMappingIds()` 针对 EventHandler 的 action 字段做类型特定的 ID 替换

这三层机制相互补充，确保绝大多数引用能被正确替换。
