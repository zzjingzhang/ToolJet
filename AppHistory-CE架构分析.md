# AppHistory 在 CE 代码中的架构分析

## 概述

ToolJet 的 AppHistory（应用历史记录）功能采用 **CE 空桩 + EE 实现** 的架构模式。社区版（CE）中保留了完整的接口定义、hook 调用点和前端 API，但所有核心实现都是空操作或直接抛错；企业版（EE）通过动态导入和类继承机制，在不修改 CE 调用方代码的前提下，提供完整的历史记录功能。

---

## 一、前端可见 API

前端通过 [appHistory.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/appHistory.service.js) 暴露 4 个核心 API，对应后端 4 个 REST 端点：

| API 方法 | HTTP 方法 | 路由 | 功能 |
|---------|----------|------|------|
| `getHistory()` | GET | `/app-history/apps/versions/:versionId` | 分页获取历史记录列表 |
| `restoreToEntry()` | POST | `/app-history/:historyId/restore` | 恢复到某个历史节点 |
| `updateDescription()` | PATCH | `/app-history/:historyId` | 更新历史记录描述 |
| `streamHistoryUpdates()` | SSE GET | `/app-history/apps/versions/:versionId/stream` | SSE 实时推送历史更新 |

### 前端状态管理

前端使用 Zustand store [appHistoryStore.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_stores/appHistoryStore.js) 管理历史状态，包括：
- `historyEntries` - 历史条目列表
- `isLoading` / `isRestoring` - 加载状态
- `updateHistoryFromSSE()` - SSE 消息处理
- `restoreToHistory()` - 触发恢复操作
- `pushHistoryEntry()` - 追加新历史条目

### 前端 UI 组件

前端 UI 组件通过 `withEditionSpecificComponent` HOC 进行版本控制：
- [AppHistoryIcon.jsx](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/LeftSidebar/AppHistory/AppHistoryIcon.jsx) - CE 中返回 `null`
- [index.jsx](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/LeftSidebar/AppHistory/index.jsx) - CE 中返回 `null`

在 [withEditionSpecificComponent.jsx](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/modules/common/helpers/withEditionSpecificComponent.jsx) 中，CE 版本直接使用 BaseComponent（返回 null），EE/Cloud 版本从 `editions` 注册表中加载真实组件。

### 历史恢复与应用刷新

在 [useAppData.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/_hooks/useAppData.js#L741-L892) 中，`restoreTimestamp` 变化时会触发整个应用数据的重新加载，这是历史恢复后的 UI 刷新机制。

---

## 二、后端空实现（Controller / Service / Stream）

### 2.1 Controller 层

[controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/controller.ts) 中 4 个端点全部抛出 `Method not implemented.` 错误：

```typescript
@Controller('app-history')
export class AppHistoryController {
  @Get('apps/versions/:versionId')      // 列表
  async getHistory(...) { throw new Error('Method not implemented.'); }

  @Post(':historyId/restore')           // 恢复
  async restoreHistory(...) { throw new Error('Method not implemented.'); }

  @Patch(':historyId')                  // 更新描述
  async updateHistoryDescription(...) { throw new Error('Method not implemented.'); }

  @Get('apps/versions/:versionId/stream') // SSE 流
  async streamHistoryUpdates(...) { throw new Error('Method not implemented.'); }
}
```

每个端点都使用 `@InitFeature(FEATURE_KEY.XXX)` 装饰器标记功能点，与授权系统联动。

### 2.2 Service 层

[service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/service.ts) 实现了 [IAppHistoryService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/interfaces/IService.ts) 接口，所有方法均为抛错空实现：

| 方法 | 功能 | CE 状态 |
|------|------|---------|
| `captureChange()` | 捕获变更并创建历史记录 | ❌ 抛错 |
| `getHistoryList()` | 获取历史列表 | ❌ 抛错 |
| `restoreToPoint()` | 恢复到历史点 | ❌ 抛错 |
| `updateDescription()` | 更新描述 | ❌ 抛错 |

### 2.3 Stream 服务

[app-history-stream.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/app-history-stream.service.ts) 提供 SSE 流能力，CE 中完全空实现：

```typescript
@Injectable()
export class AppHistoryStreamService {
  getStream(_appVersionId, _getInitialData): Observable<MessageEvent> {
    throw new Error('Method not implemented.');
  }
  emitHistoryUpdate(_appVersionId, _historyEntry): void {
    throw new Error('Method not implemented.');
  }
}
```

### 2.4 其他空桩服务

- **EntityChangeService** ([entity-change.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/services/entity-change.service.ts)) - 构建实体变更 diff，所有方法返回空数组/空对象
- **AppHistoryUtilService** ([util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/util.service.ts)) - 历史工具服务，关键方法说明：
  - `queueHistoryCapture()` - **空操作**（CE 中不做任何事）
  - `captureSettingsUpdateHistory()` - **空操作**（CE 中不做任何事）
  - `isAppHistoryEnabled()` - 返回 `false`
  - 其余方法均抛错

---

## 三、业务服务中的历史 Hook 机制

在 5 个核心业务服务中，每个 CRUD 操作都嵌入了 **before / after hook** 对，形成统一的历史捕获切面。

### 3.1 整体 Hook 模式

每个操作遵循以下模式：

```
beforeXxx() → 实际数据库操作 → afterXxx()
     ↑                                ↑
 捕获旧状态                       捕获新状态
 (CE 返回 null)                (CE 空操作)
```

Hook 调用方式：
- **before hook**：同步调用，返回 context 对象（保存旧状态）
- **after hook**：异步 fire-and-forget 调用（`.catch()` 只打日志不阻塞），传入 context + 新数据

### 3.2 DataQueriesService（数据查询）

[service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts) 包含 3 对 hook：

| Hook 对 | 触发时机 | Context 类型 |
|--------|---------|-------------|
| `beforeQueryCreate` / `afterQueryCreate` | 创建查询 | `QueryCreateContext` |
| `beforeQueryUpdate` / `afterQueryUpdate` | 更新查询 | `QueryUpdateContext` |
| `beforeQueryDelete` / `afterQueryDelete` | 删除查询 | `QueryDeleteContext` |

Context 定义见 [interfaces/IService.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/interfaces/IService.ts#L11-L29)。

### 3.3 VersionService（版本）

[service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts) 包含 3 对 hook：

| Hook 对 | 触发时机 | Context 类型 |
|--------|---------|-------------|
| `beforeVersionCreate` / `afterVersionCreate` | 创建版本 | `VersionCreateContext` |
| `beforeVersionUpdate` / `afterVersionUpdate` | 更新版本（名称/描述/状态） | `VersionUpdateContext` |
| `beforeVersionSettingsUpdate` / `afterVersionSettingsUpdate` | 更新设置（全局设置/页面设置） | `VersionSettingsUpdateContext` |

VersionService 还注入了 `AppHistoryUtilService`，用于设置更新的历史捕获（`captureSettingsUpdateHistory` 方法）。

### 3.4 PageService（页面）

[page.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/services/page.service.ts) 包含 4 对 hook + 2 个 clone 相关 hook：

| Hook 对 | 触发时机 | Context 类型 |
|--------|---------|-------------|
| `beforePageCreate` / `afterPageCreate` | 创建页面 | `PageCreateContext` |
| `beforePageUpdate` / `afterPageUpdate` | 更新页面 | `PageUpdateContext` |
| `beforePageDelete` / `afterPageDelete` | 删除页面 | `PageDeleteContext` |
| `beforePageReorder` / `afterPageReorder` | 页面排序 | `PageReorderContext` |
| `beforePageClone` / `afterPageClone` | 克隆页面 | `PageCreateContext` |

Context 定义见 [IPageService.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/interfaces/services/IPageService.ts#L19-L41)。

### 3.5 ComponentsService（组件）

[component.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/services/component.service.ts) 包含 4 对 hook：

| Hook 对 | 触发时机 | Context 类型 |
|--------|---------|-------------|
| `beforeComponentCreate` / `afterComponentCreate` | 创建组件 | `ComponentCreateContext` |
| `beforeComponentUpdate` / `afterComponentUpdate` | 更新组件 | `ComponentUpdateContext` |
| `beforeComponentDelete` / `afterComponentDelete` | 删除组件 | `ComponentDeleteContext` |
| `beforeComponentLayoutChange` / `afterComponentLayoutChange` | 布局变更 | `ComponentLayoutContext` |

Context 定义见 [IComponentService.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/interfaces/services/IComponentService.ts#L9-L40)。

### 3.6 EventsService（事件）

[event.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/services/event.service.ts) 包含 4 对 hook：

| Hook 对 | 触发时机 | Context 类型 |
|--------|---------|-------------|
| `beforeEventCreate` / `afterEventCreate` | 创建事件 | `EventCreateContext` |
| `beforeBulkEventCreate` / `afterBulkEventCreate` | 批量创建事件 | `EventBulkCreateContext` |
| `beforeEventUpdate` / `afterEventUpdate` | 更新/排序事件 | `EventUpdateContext` |
| `beforeEventDelete` / `afterEventDelete` | 删除事件 | `EventDeleteContext` |

### 3.7 所有 Hook 汇总

| 服务 | before hook 数量 | after hook 数量 |
|------|-----------------|----------------|
| DataQueriesService | 3 | 3 |
| VersionService | 3 | 3 |
| PageService | 5 | 5 |
| ComponentsService | 4 | 4 |
| EventsService | 4 | 4 |
| **总计** | **19** | **19** |

---

## 四、为什么 CE 中不产生历史记录

### 4.1 三层空实现导致无历史

CE 版本中，历史记录的产生需要经过三道关卡，每道都是空的：

```
业务操作 → before hook → 数据库操作 → after hook → queueHistoryCapture →  BullMQ 队列 → 异步写入 DB
              ↓                            ↓              ↓                    ↓                ↓
         返回 null                     空操作         空方法             队列未注册        无processor
```

**关卡 1：Hook 层为空**
- 所有 `beforeXxx()` 返回 `null`
- 所有 `afterXxx()` 是空函数（不做任何事）
- 业务代码照常运行，无任何副作用

**关卡 2：UtilService 为空**
- `AppHistoryUtilService.queueHistoryCapture()` 是空方法
- `AppHistoryUtilService.captureSettingsUpdateHistory()` 是空方法
- `isAppHistoryEnabled()` 返回 `false`

**关卡 3：队列未注册**
- 在 [module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/module.ts#L42-L61) 中，BullMQ 队列 `app-history` 仅在 EE/Cloud 版本注册
- CE 版本 `historyQueue` 为 `null`（因 `@Optional()` 装饰器）
- 队列处理器 `HistoryQueueProcessor` 仅在 EE + WORKER=true 时加载

### 4.2 设计意图

这种"全链路空桩"设计的目的：
1. **接口稳定性**：CE 和 EE 共享完全相同的业务服务接口
2. **零性能损耗**：CE 中历史功能零开销（无队列、无数据库写入、无 SSE）
3. **无缝升级**：从 CE 升级到 EE 时，所有 hook 自动生效，无需修改业务代码
4. **代码清晰**：通过 `// No-op in CE, EE overrides` 注释明确标记扩展点

---

## 五、前端 SSE / Restore / Patch 在 CE 中的失败模式

### 5.1 失败总览

| 前端操作 | 后端端点 | CE 中的失败表现 | 错误类型 |
|---------|---------|---------------|---------|
| 打开历史面板 | GET `/app-history/...` | 直接抛 500 错误 | `Error: Method not implemented.` |
| 点击恢复 | POST `.../restore` | 直接抛 500 错误 | `Error: Method not implemented.` |
| 修改描述 | PATCH `.../:historyId` | 直接抛 500 错误 | `Error: Method not implemented.` |
| SSE 连接 | GET `.../stream` | 直接抛 500 错误，连接断开 | `Error: Method not implemented.` |

### 5.2 前端层面的防护

虽然后端 API 会直接报错，但前端有多层防护，避免用户感知到错误：

**防护 1：UI 组件不渲染**
- CE 版本中 [AppHistoryIcon.jsx](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/LeftSidebar/AppHistory/AppHistoryIcon.jsx) 和 [index.jsx](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/LeftSidebar/AppHistory/index.jsx) 都返回 `null`
- 用户看不到历史记录入口，自然不会触发 API 调用

**防护 2：License / Feature 鉴权**
- 后端使用 `@InitFeature(FEATURE_KEY.LIST_HISTORY)` 等装饰器
- [features.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/constants/features.ts) 定义了 `APP_HISTORY` license 字段
- 无授权时，在能力守卫层就会被拦截，不会执行到 controller 方法

**防护 3：SSE 错误静默处理**
- [appHistory.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/appHistory.service.js#L73-L78) 中 SSE 连接失败只打 `console.error`，不弹窗

### 5.3 潜在风险

如果绕过前端防护（例如直接调用 API），会出现：
- 所有请求返回 **500 Internal Server Error**
- 错误信息为 `Method not implemented.`（可能暴露内部实现细节）
- SSE 连接会反复重连（fetchEventSource 默认重试机制），造成无效请求

---

## 六、EE 覆盖机制及可覆盖点

### 6.1 动态导入覆盖（Module 层）

ToolJet 使用 **基于路径的动态导入** 实现 CE/EE 切换，核心是 [getImportPath()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/index.ts#L11-L35) 函数：

```typescript
export const getImportPath = async (isGetContext?: boolean, edition?: TOOLJET_EDITIONS) => {
  const repoType = edition || getTooljetEdition() || TOOLJET_EDITIONS.CE;
  
  switch (repoType) {
    case TOOLJET_EDITIONS.CE:
      return `${join(process.cwd(), baseDir, 'src/modules')}`;   // CE: src/modules
    case TOOLJET_EDITIONS.EE:
    case TOOLJET_EDITIONS.Cloud:
      return `${join(process.cwd(), baseDir, 'ee')}`;            // EE: ee/
  }
};
```

所有继承 [SubModule](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/sub-module.ts) 的模块都通过 `this.getProviders()` 动态加载服务类：

```typescript
// 以 DataQueriesModule 为例
const { DataQueriesService } = await this.getProviders(
  configs,
  'data-queries',           // 模块名
  ['controller', 'service'] // 路径列表
);
```

### 6.2 两类覆盖方式

**方式 A：文件级覆盖（整体替换）**

适用于整个服务/控制器需要完全重写的场景。EE 在 `ee/` 目录下放置同名文件，完全替换 CE 版本。

适用场景：
- `AppHistoryController` - 完整重写所有端点
- `AppHistoryService` - 完整实现历史逻辑
- `AppHistoryStreamService` - 完整实现 SSE
- `EntityChangeService` - 完整实现 diff 构建
- `AppHistoryUtilService` - 完整实现工具方法

**方式 B：类继承覆盖（部分替换）**

适用于业务服务（DataQuery/Version/Page/Component/Event）。EE 版本继承 CE 的服务类，只覆盖 before/after hook 方法，业务逻辑复用。

```
CE: DataQueriesService (包含完整业务逻辑 + 空hook)
                  ↑
EE: EEDataQueriesService extends DataQueriesService (只覆盖 hook 方法)
```

这种方式的优点：
- 业务逻辑只写一份（在 CE）
- EE 只关注历史捕获逻辑
- CE 的 bugfix 自动同步到 EE

### 6.3 可被 EE 覆盖而不改 Controller 调用方的位置

#### 6.3.1 AppHistory 模块内部（文件级覆盖）

| 类 / 文件 | CE 路径 | 覆盖方式 | 说明 |
|----------|--------|---------|------|
| AppHistoryController | `app-history/controller.ts` | 文件级 | REST API 完整重写 |
| AppHistoryService | `app-history/service.ts` | 文件级 | 历史 CRUD 完整重写 |
| AppHistoryStreamService | `app-history/app-history-stream.service.ts` | 文件级 | SSE 流完整重写 |
| EntityChangeService | `app-history/services/entity-change.service.ts` | 文件级 | diff 构建完整重写 |
| AppHistoryUtilService | `app-history/util.service.ts` | 文件级 | 工具方法完整重写 |
| AppHistoryRepository | `app-history/repository.ts` | 文件级 | 数据库访问完整重写 |
| AppStateRepository | `app-history/repositories/app-state.repository.ts` | 文件级 | 状态存储完整重写 |
| NameResolverRepository | `app-history/repositories/name-resolver.repository.ts` | 文件级 | 名称解析完整重写 |
| HistoryQueueProcessor | `app-history/queue/history-queue.processor.ts` | 文件级 | 异步队列处理器 |
| FeatureAbilityFactory | `app-history/ability/index.ts` | 文件级 | 权限工厂 |
| FeatureAbilityGuard | `app-history/ability/guard.ts` | 文件级 | 权限守卫 |

#### 6.3.2 业务服务（类继承覆盖 - Hook 层）

| 服务 | CE 类 | 覆盖方式 | 可覆盖的 Hook 方法 |
|------|------|---------|------------------|
| 数据查询 | DataQueriesService | 类继承 | beforeQueryCreate, afterQueryCreate, beforeQueryUpdate, afterQueryUpdate, beforeQueryDelete, afterQueryDelete |
| 版本 | VersionService | 类继承 | beforeVersionCreate, afterVersionCreate, beforeVersionUpdate, afterVersionUpdate, beforeVersionSettingsUpdate, afterVersionSettingsUpdate |
| 页面 | PageService | 类继承 | beforePageCreate, afterPageCreate, beforePageUpdate, afterPageUpdate, beforePageDelete, afterPageDelete, beforePageReorder, afterPageReorder, beforePageClone, afterPageClone |
| 组件 | ComponentsService | 类继承 | beforeComponentCreate, afterComponentCreate, beforeComponentUpdate, afterComponentUpdate, beforeComponentDelete, afterComponentDelete, beforeComponentLayoutChange, afterComponentLayoutChange |
| 事件 | EventsService | 类继承 | beforeEventCreate, afterEventCreate, beforeBulkEventCreate, afterBulkEventCreate, beforeEventUpdate, afterEventUpdate, beforeEventDelete, afterEventDelete |

**关键特性**：所有这些覆盖都 **不需要修改 controller 层**。Controller 只依赖服务接口（或服务类），通过 NestJS 的依赖注入和模块动态注册，EE 版本的服务会自动替换 CE 版本。

### 6.4 模块注册中的 EE 特殊逻辑

在 [app-history/module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-history/module.ts) 中，有两处 EE 专属逻辑：

1. **BullMQ 队列注册**（第 42-61 行）
   - 仅 EE/Cloud 版本注册 `app-history` 队列
   - CE 版本不注册，减少依赖和启动时间

2. **队列处理器加载**（第 67-70 行）
   - 仅 EE/Cloud + WORKER=true 时加载 `HistoryQueueProcessor`
   - 符合 ToolJet 的 worker 进程架构

### 6.5 导出的可复用服务

AppHistoryModule 导出两个服务供其他模块使用：

```typescript
exports: [AppHistoryUtilService, EntityChangeService]
```

这意味着：
- 其他模块（Version、Apps 等）可以注入这些服务
- EE 版本替换这些服务后，所有依赖模块自动获得 EE 实现
- **无需修改调用方代码**

---

## 七、数据模型

虽然 CE 不产生历史记录，但数据库实体 [app_history.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_history.entity.ts) 是完整定义的（EE 需要使用）：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | uuid | 主键 |
| app_version_id | uuid | 所属应用版本 |
| user_id | uuid | 操作用户 |
| sequence_number | bigint | 序列号（每个版本内唯一递增） |
| parent_id | uuid | 父历史条目（链式结构） |
| history_type | enum | `snapshot` / `delta`（快照/增量） |
| action_type | varchar | 操作类型（component_add, page_update 等） |
| operation_scope | jsonb | 操作范围（影响哪些实体） |
| description | text | 描述文本 |
| change_payload | jsonb | 变更内容（完整快照或 diff） |
| is_ai_generated | boolean | 是否 AI 生成 |
| created_at / updated_at | timestamp | 时间戳 |

关键设计：
- 快照频率：每 10 条 delta 产生 1 条 snapshot（`SNAPSHOT_FREQUENCY = 10`）
- 保留策略：最多 110 条（`RETENTION_BUFFER_LIMIT = 110`），UI 显示 100 条
- 唯一索引：`(app_version_id, sequence_number)` 联合唯一

---

## 八、架构总结图

```
┌─────────────────────────────────────────────────────────────────────┐
│                            前端 (FE)                                │
│  ┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐  │
│  │ UI 组件     │────▶│ appHistoryStore  │────▶│ appHistory.svc  │  │
│  │ (CE: null)  │     │  (Zustand)       │     │  (4 API 方法)   │  │
│  └─────────────┘     └──────────────────┘     └────────┬────────┘  │
│                                                         │           │
│                    withEditionSpecificComponent        │           │
│                    (CE 用 BaseComponent)               │           │
└─────────────────────────────────────────────────────────│───────────┘
                                                          │ HTTP/SSE
┌─────────────────────────────────────────────────────────│───────────┐
│                          后端 (CE)                       ▼           │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │ AppHistoryController                                     │       │
│  │  - getHistory()  → throw Error                           │       │
│  │  - restoreHistory() → throw Error                        │       │
│  │  - updateDescription() → throw Error                     │       │
│  │  - streamHistoryUpdates() → throw Error                  │       │
│  └────────────────────┬─────────────────────────────────────┘       │
│                       │                                             │
│  ┌────────────────────▼─────────────────────┐    ┌───────────────┐  │
│  │ AppHistoryService                         │    │ AppHistory-   │  │
│  │  - captureChange() → throw Error          │    │ StreamService │  │
│  │  - getHistoryList() → throw Error         │    │  - getStream()│  │
│  │  - restoreToPoint() → throw Error         │    │    → throw    │  │
│  │  - updateDescription() → throw Error      │    │  - emit...()  │  │
│  └───────────────────────────────────────────┘    │    → throw    │  │
│                       ▲                           └───────────────┘  │
│                       │                                              │
│  ┌────────────────────┴──────────────────────┐                       │
│  │ AppHistoryUtilService                      │                       │
│  │  - queueHistoryCapture() → no-op           │                       │
│  │  - captureSettingsUpdateHistory() → no-op  │                       │
│  │  - isAppHistoryEnabled() → false           │                       │
│  └────────────────────────────────────────────┘                       │
│                       ▲                                               │
│                       │ 注入                                          │
│  ┌────────────────────┴──────────────────────┐                       │
│  │ 5 个业务服务 (Service Layer)                │                       │
│  │  - DataQueriesService (3 对 hook)          │                       │
│  │  - VersionService    (3 对 hook)           │                       │
│  │  - PageService       (5 对 hook)           │                       │
│  │  - ComponentsService (4 对 hook)           │                       │
│  │  - EventsService     (4 对 hook)           │                       │
│  │                                             │                       │
│  │  before hook → 业务逻辑 → after hook       │                       │
│  │   (return null)          (no-op)           │                       │
│  └────────────────────────────────────────────┘                       │
│                       ▲                                               │
│                       │ 动态导入                                      │
│  ┌────────────────────┴──────────────────────┐                       │
│  │ SubModule + getImportPath()                │                       │
│  │ CE: src/modules/*                          │                       │
│  │ EE: ee/*                                   │                       │
│  └────────────────────────────────────────────┘                       │
│                                                                      │
│  BullMQ 队列: 未注册 (CE 不启用)                                     │
│  HistoryQueueProcessor: 未加载                                        │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ EE 替换 (整体替换/类继承)
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        后端 (EE - 企业版)                             │
│                                                                      │
│  完整实现所有上述空方法 + 队列处理器 + SSE + 快照/增量算法           │
│  19 个 before/after hook 全部被覆盖 → 捕获历史                        │
│  BullMQ 队列异步处理 → 写入 DB                                       │
│  SSE 实时推送 → 前端更新                                             │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键结论

1. **CE 是完整的"接口壳"**：从前端 API 到后端 controller 到 service 到 hook，全链路定义完整但无实际功能

2. **19 对 hook 是 EE 功能的预埋点**：每个业务操作的 before/after 都留好了扩展位，EE 只需覆盖这些方法即可接入历史记录

3. **两种覆盖机制互补**：
   - **文件级覆盖** 用于 AppHistory 模块本身（完全重写）
   - **类继承覆盖** 用于业务服务（只覆盖 hook，复用业务逻辑）

4. **Controller 层零改动**：EE 版本的所有增强都通过服务层注入和动态模块加载实现，controller 调用方代码完全不需要修改

5. **前端双重防护**：UI 组件版本控制 + License 鉴权，确保 CE 用户不会触达历史功能

6. **零性能开销**：CE 中没有队列、没有额外数据库写入、没有 SSE 连接，历史功能完全"隐形"
