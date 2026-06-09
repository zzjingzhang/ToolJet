# ModuleViewer 组件模块版本解析深度分析

## 1. 核心概念概述

ModuleViewer 组件用于在应用中嵌入模块，其通过两个关键属性引用模块版本：

- **moduleAppId**：模块的 `co_relation_id`，在分支和 Git 克隆工作区中保持稳定
- **moduleVersionId**：版本的 `module_reference_id`（UUID），在分支和实例间保持稳定，可通过 Git push/pull 和 zip 导出/导入进行往返

模块版本解析的核心代码位于 [module-ref.util.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts)。

---

## 2. 模块版本解析规则详解

### 2.1 解析优先级总览

`resolveAllModuleViewersForVersion` 函数实现了完整的解析逻辑，优先级从高到低为：

| 优先级 | 匹配类型 | 说明 | matchKind |
|--------|----------|------|-----------|
| 1 | UUID pin → 消费者分支的 `module_reference_id` | 优先匹配当前分支上的版本 | `pin-hit` |
| 2 | UUID pin → 默认分支的 `module_reference_id` | 当前分支无匹配时回退到默认分支 | `pin-hit` |
| 3 | Legacy 版本名 → `version_type='version'` 行 | 通过版本名称匹配 | `pin-hit` |
| 4 | branch_name 简写 → 默认分支 version 行 | pin 值是分支名时，匹配该分支默认版本 | `pin-hit` |
| 5 | Orphan（非空但无匹配）→ 消费者分支最新非 stub 版本，再回退到默认分支 | pin 值存在但找不到匹配 | `orphan-fallback` |
| 6 | Unpinned（空值）→ 同上 | 未指定 pin 值 | `unpinned-fallback` |

### 2.2 moduleAppId 的解析

`moduleAppId` 存储的是模块应用的 `co_relation_id`，而非数据库主键 `id`。解析时：

1. 从组件 `properties` 中提取 `moduleAppId.value`
2. 通过 `co_relation_id` 在 `apps` 表中查找类型为 `MODULE` 的应用
3. 当同一 `co_relation_id` 有多个匹配时（如 Git 克隆产生的重复），取 `created_at ASC`（最早创建的）

相关代码片段（[module-ref.util.ts#L314-L324](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L314-L324)）：

```sql
SELECT DISTINCT ON (co_relation_id)
        id, co_relation_id AS "coRel", name, current_version_id AS "currentVersionId"
 FROM apps
 WHERE co_relation_id::text = ANY($1)
   AND type = $2
   AND organization_id = $3
 ORDER BY co_relation_id, created_at ASC
```

### 2.3 moduleVersionId 的各种情况解析

#### 2.3.1 UUID Pin（module_reference_id 匹配）

当 `moduleVersionId` 是合法 UUID 时：

1. **消费者分支优先**：先在消费者分支（`branchId` 来自请求头 `x-branch-id`）上查找匹配 `module_reference_id` 的非 stub 版本
2. **默认分支回退**：消费者分支无匹配时，在默认分支上查找

相关代码（[module-ref.util.ts#L431-L453](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L431-L453)）：

```typescript
if (pin && UUID_RE.test(pin)) {
  const byId = candidates.find((r) => r.id === pin);
  if (byId) return pickKind(byId, 'pin-hit');
  
  const onConsumer = parent.branchId
    ? candidates.find((r) => r.moduleReferenceId === pin && r.branchId === parent.branchId)
    : undefined;
  if (onConsumer) return pickKind(onConsumer, 'pin-hit');
  
  if (defaultBranch) {
    const onDefault = candidates.find(
      (r) => r.moduleReferenceId === pin && r.branchId === defaultBranch.id
    );
    if (onDefault) return pickKind(onDefault, 'pin-hit');
  }
}
```

#### 2.3.2 Legacy Version Name

当 `moduleVersionId` 非空但不是 UUID 时，尝试按版本名称匹配 `version_type='version'` 的行：

```typescript
const byName = candidates.find((r) => r.name === pin && r.versionType === AppVersionType.VERSION);
if (byName) return pickKind(byName, 'pin-hit');
```

#### 2.3.3 branch_name Shorthand

当 pin 值是一个有效的分支名称时，匹配该默认分支上最新的 `version_type='version'` 行：

```typescript
if (branchNameSet.has(pin) && defaultBranch) {
  const onDefault = candidates.find(
    (r) => r.branchId === defaultBranch.id && r.versionType === AppVersionType.VERSION
  );
  if (onDefault) return pickKind(onDefault, 'pin-hit');
}
```

#### 2.3.4 Orphan（孤立 Pin）

当 pin 值非空但找不到任何匹配时，称为 orphan。此时回退到：

1. 消费者分支上最新的非 stub 版本
2. 若无，回退到默认分支上最新的非 stub 版本

matchKind 为 `orphan-fallback`。

#### 2.3.5 Unpinned（未固定）

当 `moduleVersionId` 为空或缺失时，称为 unpinned。回退逻辑与 orphan 相同，但 matchKind 为 `unpinned-fallback`。

#### 2.3.6 No-row（无可用行）

当模块应用不存在，或没有任何可用的版本行时，matchKind 为 `no-row`，模块无法渲染。

### 2.4 分支与默认分支的作用

- **消费者分支（Consumer Branch）**：由请求头 `x-branch-id` 指定，是用户当前正在操作的分支
- **默认分支（Default Branch）**：组织的默认分支，作为回退目标
- **解析范围**：只考虑消费者分支和默认分支上的非 stub 版本行

---

## 3. matchKind 与发布/提升阻断机制

### 3.1 matchKind 四种类型

`ModuleResolutionMatch` 类型定义了四种解析结果（[module-ref.util.ts#L245-L249](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L245-L249)）：

```typescript
export type ModuleResolutionMatch =
  | 'pin-hit'           // pin 匹配成功
  | 'orphan-fallback'   // pin 非空但无匹配，回退到最新版本
  | 'unpinned-fallback' // pin 为空，回退到最新版本
  | 'no-row';           // 无模块应用或无可用版本
```

### 3.2 checkDraftModulesInApp — 发布阻断

`checkDraftModulesInApp` 函数在保存/发布版本时被调用，用于检查所有引用的模块是否处于可发布状态。

**调用位置**：[service.ts#L305-L307](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L305-L307)

```typescript
if (appVersionUpdateDto?.status === AppVersionStatus.PUBLISHED && app.type !== 'module') {
  await this.versionsUtilService.checkDraftModulesInApp(appVersion.id, user.organizationId, manager);
}
```

**阻断规则**（[util.service.ts#L278-L284](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts#L278-L284)）：

以下情况会阻断发布：

| matchKind | 阻断原因 |
|-----------|----------|
| `no-row` | 模块无可用版本，无法使用 |
| `orphan-fallback` | Pin 无效，运行时会漂移到当前编辑的草稿 |
| `unpinned-fallback` | 未固定版本，运行时会漂移到当前编辑的草稿 |
| `pin-hit` + DRAFT 状态 | Pin 直接指向草稿版本 |

**设计意图**：确保发布的应用引用的是稳定的、已保存的模块版本，而不是可能随时变化的草稿。

### 3.3 checkModulesPromotableToEnvironment — 提升阻断

`checkModulesPromotableToEnvironment` 函数用于在提升环境前检查模块版本是否已提升到目标环境。

**阻断规则**（[util.service.ts#L338-L349](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts#L338-L349)）：

以下情况会阻断提升：

| 条件 | 阻断原因 |
|------|----------|
| `no-row` | 模块无可用版本 |
| `orphan-fallback` | Pin 无效，回退不是稳定的提升目标 |
| `unpinned-fallback` | 未固定版本，回退不是稳定的提升目标 |
| `pin-hit` + envPriority < targetPriority | 模块版本尚未提升到目标环境 |

**设计意图**：确保提升时，所有依赖的模块版本都已经在目标环境可用，保证环境一致性。

---

## 4. reconcileModuleViewerPinsFromDefault 深度解析

### 4.1 功能定位

`reconcileModuleViewerPinsFromDefault` 函数用于在特性分支版本水化（hydrate）后，从默认分支的对应组件继承固定的 `moduleVersionId` 值。

### 4.2 为什么按 co_relation_id 匹配

#### 4.2.1 co_relation_id 的本质

`co_relation_id` 是组件的关联标识符，具有以下特性：

1. **跨分支稳定**：同一个组件在不同分支上具有相同的 `co_relation_id`
2. **Git 友好**：提交到 Git 的组件 JSON 中包含 `co_relation_id`
3. **跨实例稳定**：通过 Git push/pull 同步后，同一组件的 `co_relation_id` 保持不变

#### 4.2.2 设计原因

按 `co_relation_id` 匹配的核心原因：

**原因 1：组件 JSON 可能过时**

特性分支创建时，组件 JSON 中保存的 `moduleVersionId` 可能是分支创建时的旧值。随着默认分支上模块版本的更新，特性分支的 pin 值会变得陈旧。

**原因 2：保持版本一致性**

当用户切换到特性分支时，期望看到的是与默认分支一致的模块 pin 配置，而不是分支创建时的历史快照。

**原因 3：co_relation_id 是唯一可靠的跨分支匹配键**

- 组件的数据库 `id` 在不同分支上是不同的（每个分支有独立的组件行）
- 组件名称可能被修改
- 只有 `co_relation_id` 在分支间保持稳定对应关系

相关代码（[module-ref.util.ts#L204-L210](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L204-L210)）：

```typescript
const defaultViewers = await readViewers(defaultVersion.id);
const defaultByCoRel = new Map(
  defaultViewers.filter((v) => v.co_relation_id).map((v) => [v.co_relation_id as string, v])
);
```

### 4.3 继承策略

继承遵循以下策略（[module-ref.util.ts#L154-L157](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L154-L157)）：

1. **仅当默认分支有值时才复制**：默认分支的 pin 是空的则跳过
2. **仅当值不同时才复制**：特性分支已有相同值则保持不变
3. **验证模块应用存在**：确保 `moduleAppId` 对应的模块应用实际存在

相关代码（[module-ref.util.ts#L212-L232](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts#L212-L232)）：

```typescript
for (const feat of featureViewers) {
  if (!feat.co_relation_id) continue;
  const def = defaultByCoRel.get(feat.co_relation_id);
  if (!def?.moduleVersionId) continue;      // nothing to inherit
  if (feat.moduleVersionId === def.moduleVersionId) continue; // already matches

  const moduleApp = def.moduleAppId
    ? await manager.findOne(App, {
        where: { co_relation_id: def.moduleAppId, type: APP_TYPES.MODULE, organizationId },
        order: { createdAt: 'ASC' },
      })
    : null;
  if (!moduleApp) continue;

  await manager.query(
    `UPDATE components
     SET properties = jsonb_set(properties::jsonb, '{moduleVersionId,value}', to_jsonb($1::text))
     WHERE id = $2`,
    [def.moduleVersionId, feat.id]
  );
}
```

### 4.4 调用时机

此函数在特性分支的版本水化后调用，确保用户打开特性分支时看到的模块 pin 是最新的（与默认分支同步），而不是分支创建时的旧值。

---

## 5. 关键数据结构关系图

```
ModuleViewer 组件
├── properties.moduleAppId.value  →  模块 app 的 co_relation_id
└── properties.moduleVersionId.value  →  module_reference_id (UUID)
                                        或版本名 (legacy)
                                        或分支名 (shorthand)
                                        或空 (unpinned)

AppVersion (模块版本)
├── id (UUID, 主键)
├── co_relation_id (跨分支稳定)
├── module_reference_id (跨实例稳定, UUID)
├── branch_id (所属分支)
├── version_type (version / branch)
├── status (DRAFT / PUBLISHED / RELEASED)
└── is_stub (是否为占位行)

App (模块应用)
├── id (UUID, 主键)
├── co_relation_id (跨分支/实例稳定)
└── current_version_id (当前发布版本)
```

---

## 6. 关键文件索引

| 文件 | 作用 |
|------|------|
| [module-ref.util.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts) | 模块版本解析核心逻辑 |
| [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts) | 发布/提升检查逻辑 |
| [service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts) | 版本服务入口 |
| [app_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts) | AppVersion 实体定义 |
| [ModuleViewer.jsx](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/modules/Modules/components/ModuleViewer/ModuleViewer.jsx) | 前端 ModuleViewer 组件入口 |
| [moduleViewer.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/WidgetManager/widgets/moduleViewer.js) | 前端 widget 配置 |
