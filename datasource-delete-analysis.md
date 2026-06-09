# ToolJet DataSource 删除机制深度分析

## 一、概述

本文档深入分析 ToolJet 中 DataSource（数据源）的删除机制，重点探讨删除操作如何防止破坏已有的 DataQuery（数据查询），并对四种不同类型数据源的删除路径进行详细比较。

**核心删除入口**：[service.ts 中的 `delete` 方法](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts#L205-L251)

---

## 二、删除流程总览

DataSource 的删除操作遵循严格的校验和执行流程：

```
删除请求
    ↓
1. 数据源存在性校验
    ↓
2. Sample 类型检查（Sample 数据源不可删除）
    ↓
3. Dependent Queries 依赖检查（有关联查询则拒绝删除）
    ↓
4. 根据 scope + branch 判断删除策略
    ├─ 全局数据源 + 有 branch → 软删除（DataSourceVersion.isActive = false）
    └─ 其他情况 → 硬删除（repository.delete）
    ↓
5. 记录审计日志上下文
```

---

## 三、Dependent Queries 检查机制

### 3.1 检查目的

防止删除操作作为删除前的第一道安全防线，确保数据源被删除前检查是否存在依赖该数据源的 DataQuery，从根本上避免删除后导致数据查询失效。

### 3.2 核心实现

检查逻辑位于 [service.ts 中的 `findQueriesLinkedToDatasource` 方法](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts#L314-L325)：

```typescript
async findQueriesLinkedToDatasource(datasourceId: string, organizationId?: string, branchId?: string) {
  const dataSourceDetails = await this.dataSourcesRepository.getQueriesByDatasourceId(datasourceId, branchId || null);
  if (dataSourceDetails.length == 0) return { datasources: 0, dependent_queries: 0 };

  const queries = [];
  dataSourceDetails.forEach((datasourceDetail) => {
    const { dataQueries = [] } = datasourceDetail;
    if (dataQueries.length) queries.push(...dataQueries);
  });

  return { datasources: dataSourceDetails.length, dependent_queries: queries.length };
}
```

### 3.3 Repository 查询实现

Repository 层的查询在 [repository.ts 中的 `getQueriesByDatasourceId` 方法](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/repository.ts#L330-L350)，支持 branch 感知：

```typescript
getQueriesByDatasourceId(datasourceId: string, branchId?: string | null) {
  return dbTransactionWrap(async (manager: EntityManager) => {
    if (branchId) {
      return manager
        .createQueryBuilder(DataSource, 'ds')
        .leftJoinAndSelect(
          'ds.dataQueries',
          'dq',
          'dq.app_version_id IN (SELECT av.id FROM app_versions av WHERE av.branch_id = :branchId)',
          { branchId }
        )
        .where('ds.id = :datasourceId', { datasourceId })
        .getMany();
    }

    return manager.find(DataSource, {
      where: { id: datasourceId },
      relations: ['dataQueries'],
    });
  });
}
```

### 3.4 删除中的校验逻辑

在删除方法中的应用：

```typescript
const result = await this.findQueriesLinkedToDatasource(dataSourceId, user.organizationId, branchId);
if (result.dependent_queries) {
  throw new BadRequestException(`Datasource can't be deleted, queries are in use');
}
```

**关键点：
- 有 branchId 时，仅检查该 branch 下 app versions 关联的 queries
- 无 branchId 时，检查所有关联的 queries
- 只要存在至少一个依赖查询，就抛出异常阻止删除

---

## 四、DataSourceVersion.isActive 软删除机制

### 4.1 软删除的使用场景

当删除全局数据源且指定了 `branchId` 时，采用软删除策略：

```typescript
const effectiveBranchId = dataSource.scope === DataSourceScopes.GLOBAL ? branchId || null : null;

if (effectiveBranchId) {
  await dbTransactionWrap(async (manager: EntityManager) => {
    await manager.update(
      DataSourceVersion,
      { dataSourceId, branchId: effectiveBranchId },
      { isActive: false, updatedAt: new Date() }
    );
  });
}
```

### 4.2 DataSourceVersion 实体结构

[DataSourceVersion 实体](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source_version.entity.ts) 关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `dataSourceId | string | 关联的数据源 ID |
| `branchId` | string | 关联的分支 ID（可为 null） |
| `isActive` | boolean | 是否激活（软删除标记），默认 true |
| `isDefault` | boolean | 是否为默认版本 |
| `name` | string | 版本名称 |

### 4.3 软删除的影响范围

软删除**仅影响特定 branch 下的 DataSourceVersion**，不会影响：
- 主分支（默认版本）
- 其他分支的版本
- DataSource 主记录本身
- 其他环境的配置

### 4.4 软删除后的查询过滤

在查询数据源列表时，会通过 `is_active = true` 条件过滤掉已软删除的版本。例如在 [repository.ts 的 allGlobalDS 方法](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/repository.ts#L61-L68)：

```typescript
query.leftJoin(
  'data_source_versions',
  'dsv',
  `dsv.data_source_id = data_source.id
   AND dsv.branch_id = :branchId
   AND dsv.is_active = true`,
  { branchId }
);
```

---

## 五、repository.delete 硬删除逻辑

### 5.1 硬删除的使用场景

以下情况会触发硬删除：

1. **全局数据源 + 无 branchId（直接删除主记录
2. **本地数据源（scope = local）的删除

### 5.2 硬删除实现

硬删除通过 TypeORM Repository 的 `delete` 方法：

```typescript
await this.dataSourcesRepository.delete(dataSourceId);
```

`DataSourcesRepository` 继承自 TypeORM 的 `Repository<DataSource>`，因此 `delete` 方法是 TypeORM 内置的，直接从数据库中物理删除记录。

### 5.3 级联删除影响

由于实体关系定义了级联删除策略，硬删除会触发级联删除：

**DataSource 实体中的关系**：

1. **与 DataQuery 的关系**：
   - [data_source.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source.entity.ts#L118-L119) 中定义了 `@OneToMany(() => DataQuery, (dq) => dq.dataSource)`
   - [data_query.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_query.entity.ts#L47-L49) 中定义了 `@ManyToOne(() => DataSource, (dataSource) => dataSource.id, { onDelete: 'CASCADE' })`
   - **影响：删除 DataSource 会级联删除所有关联的 DataQuery

2. **与 DataSourceVersion 的关系**：
   - `@OneToMany(() => DataSourceVersion, (dsv) => dsv.dataSource)`
   - DataSourceVersion 侧有 `{ onDelete: 'CASCADE' }`
   - **影响**：删除 DataSource 会级联删除所有关联的 DataSourceVersion

3. **与其他级联关系**：
   - DataSourceGroupPermission：级联删除
   - GroupDataSources：级联删除
   - Plugin：多对一级联删除（反向）

> 重要**：由于有 dependent queries 检查在硬删除之前执行，正常情况下不会走到级联删除 DataQuery 的这一步。dependent queries 检查是业务层面的保护机制，而级联删除是数据库层面的兜底机制。

---

## 六、不同类型数据源的删除路径比较

### 6.1 四种数据源类型概览

| 类型 | scope | type | 说明 |
|------|-------|------|------|
| Sample 数据源 | global | sample | 示例数据源，系统预置 |
| 全局数据源（主分支 | global | default/static | 全局共享数据源 |
| Branch 中的全局数据源 | global | default/static | 分支版本化的全局数据源 |
| 普通本地数据源 | local | default/static | 应用内私有数据源 |

### 6.2 详细删除路径对比

#### 6.2.1 Sample 数据源

**删除路径**：直接拒绝

**代码位置**：[service.ts L210-L212](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts#L210-L212)

```typescript
if (dataSource.type === DataSourceTypes.SAMPLE) {
  throw new BadRequestException('Cannot delete sample data source');
}
```

**特点**：
- 完全不允许删除
- 无需进行 dependent queries 检查
- 不会执行任何删除操作
- 直接抛出异常终止

---

#### 6.2.2 全局数据源（无 branch）

**删除路径**：硬删除

**执行步骤：

1. **存在性校验
2. **Sample 类型检查（通过）
3. **Dependent queries 检查**（如有则拒绝）
4. **硬删除 DataSource 主记录
5. **级联删除所有关联的 DataSourceVersion、DataQuery 等
6. **记录审计日志

**关键代码**：

```typescript
const effectiveBranchId = dataSource.scope === DataSourceScopes.GLOBAL ? branchId || null : null;

if (effectiveBranchId) {
  // 软删除路径
} else {
  await this.dataSourcesRepository.delete(dataSourceId);
}
```

**影响范围**：
- 删除 DataSource 主记录
- 级联删除所有关联的 DataSourceVersion（所有版本）
- 级联删除所有关联的 DataQuery
- 级联删除所有关联的权限记录
- 影响所有使用该数据源的应用

---

#### 6.2.3 Branch 中的全局数据源

**删除路径**：软删除

**执行步骤**：

1. 存在性校验
2. Sample 类型检查（通过）
3. **Dependent queries 检查**（仅检查该 branch 下的 queries，如有则拒绝）
4. **软删除**：设置 `DataSourceVersion.isActive = false`
5. 记录审计日志

**关键代码**：

```typescript
const effectiveBranchId = dataSource.scope === DataSourceScopes.GLOBAL ? branchId || null : null;

if (effectiveBranchId) {
  await dbTransactionWrap(async (manager: EntityManager) => {
    await manager.update(
      DataSourceVersion,
      { dataSourceId, branchId: effectiveBranchId },
      { isActive: false, updatedAt: new Date() }
    );
  });
}
```

**特点**：
- 仅影响当前分支的版本
- DataSource 主记录仍然存在
- 主分支（默认版本）不受影响
- 其他分支不受影响
- 关联的查询在该分支下不可用

**查询过滤查询时通过 `is_active = true` 过滤

---

#### 6.2.4 普通本地数据源

**删除路径**：硬删除（级联删除）

**本地数据源特点**：
- `scope = 'local'`
- 与特定 `appVersionId` 关联
- 属于特定应用版本私有

**删除方式**：

1. **通过 AppVersion 级联删除**：
   - DataSource 实体中 `appVersion` 关系有 `{ onDelete: 'CASCADE' }`
   - 当删除 AppVersion 时，会级联删除所有关联的本地数据源
   - 进而级联删除本地数据源关联的 DataQuery

2. **直接删除**：
   - 也可通过 data-sources API 直接删除
   - 同样遵循相同的删除逻辑

**关键实体关系**：

[DataSource 实体](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source.entity.ts#L69-L73)：

```typescript
@ManyToOne(() => AppVersion, (appVersion) => appVersion.id, {
  onDelete: 'CASCADE',
})
@JoinColumn({ name: 'app_version_id' })
appVersion: AppVersion;
```

**版本复制行为**：

在创建新版本时，本地数据源会被复制到新版本中，参见 [versions/services/create.service.ts L119](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts#L119)：

```typescript
const dataSources = versionFrom?.dataSources.filter((ds) => ds.scope == DataSourceScopes.LOCAL); //Local data sources
```

---

### 6.3 删除路径对比表

| 维度 | Sample 数据源 | 全局数据源（主分支） | Branch 中的全局数据源 | 普通本地数据源 |
|------|-------------|----------------------|-------------------|-------------|
| **删除方式** | 禁止删除 | 硬删除 | 软删除 | 硬删除（级联） |
| **Dependent Queries 检查** | 不需要 | 需要（全量） | 需要（仅该 branch） | 需要 |
| **DataSource 主记录** | 保留 | 删除 | 保留 | 删除 |
| **DataSourceVersion** | - | 全部删除 | 仅该 branch 版本标记 isActive=false | 全部删除 |
| **关联 DataQuery** | - | 全部级联删除 | 不受影响（但该 branch 查询不可用） | 全部级联删除 |
| **影响范围** | 无 | 全局所有应用 | 仅当前 branch | 仅当前应用版本 |
| **可恢复性** | - | 不可恢复 | 可恢复（重新设置 isActive=true | 不可恢复 |
| **审计日志** | - | 记录 | 记录 | 记录 |

---

## 七、审计上下文

### 7.1 审计日志机制

删除操作完成后，会设置审计日志上下文，用于记录删除操作的详细信息。审计上下文通过 `RequestContext.setLocals` 设置，键为 `AUDIT_LOGS_REQUEST_CONTEXT_KEY`。

### 7.2 删除操作的审计数据

[service.ts L234-L249](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts#L234-L249)：

```typescript
RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, {
  userId: user.id,
  organizationId: user.organizationId,
  resourceId: dataSourceId,
  resourceName: dataSource.name,
  resourceData: {
    dataSourceKind: dataSource?.kind,
    dataSourceScope: dataSource?.scope,
    appId: dataSource?.app?.id || null,
    appVersionId: dataSource?.appVersionId,
  },
  metadata: {
    deletedAt: new Date().toISOString(),
  },
});
```

### 7.3 审计字段说明

| 字段 | 说明 |
|------|------|
| `userId` | 执行删除操作的用户 ID |
| `organizationId` | 组织 ID |
| `resourceId` | 被删除的数据源 ID |
| `resourceName` | 被删除的数据源名称 |
| `resourceData.dataSourceKind` | 数据源类型（如 postgresql、restapi 等） |
| `resourceData.dataSourceScope` | 数据源范围（local/global） |
| `resourceData.appId` | 关联的应用 ID（本地数据源有值） |
| `resourceData.appVersionId` | 关联的应用版本 ID（本地数据源有值） |
| `metadata.deletedAt` | 删除时间 |

---

## 八、DataQuery 实体关系与级联删除影响

### 8.1 实体关系图

```
DataSource (1) ──── (N) DataQuery
     │                      │
     │ onDelete: CASCADE    │
     ▼                      ▼
DataSourceVersion (1) ── (N) DataSourceVersionOptions
     │
     │ onDelete: CASCADE
     ▼
DataSourceGroupPermission / GroupDataSources
```

### 8.2 关键关系详解

**DataSource → DataQuery 关系：

- DataSource 侧：`@OneToMany(() => DataQuery, (dq) => dq.dataSource)`
- DataQuery 侧：`@ManyToOne(() => DataSource, (dataSource) => dataSource.id, { onDelete: 'CASCADE' })`
- **删除影响**：DataSource 被删除时，所有关联的 DataQuery 都会被级联删除

**DataSource → DataSourceVersion 关系**：

- DataSource 侧：`@OneToMany(() => DataSourceVersion, (dsv) => dsv.dataSource)`
- DataSourceVersion 侧：`@ManyToOne(() => DataSource, (ds) => ds.dataSourceVersions, { onDelete: 'CASCADE' })`
- **删除影响**：DataSource 被删除时，所有关联的 DataSourceVersion 都会被级联删除

**DataSourceVersion → DataSourceVersionOptions 关系**：

- DataSourceVersion 侧：`@OneToMany(() => DataSourceVersionOptions, (dsvo) => dsvo.dataSourceVersion)`
- **删除影响**：DataSourceVersion 被删除时，会级联删除对应的选项

### 8.3 保护机制层级

为了防止 DataQuery 被误删，系统设置了多层保护：

1. **第一层：业务层保护（Dependent Queries 检查）
   - 删除前主动检查是否有关联查询
   - 有则直接拒绝，用户明确的错误信息
   - 这是主要的保护机制

2. **第二层：数据库层保护（级联删除）
   - 作为兜底机制
   - 确保数据一致性
   - 正常情况下不会触发

---

## 九、最终状态总结

### 9.1 防止破坏 DataQuery 的保护机制总结

| 保护层级 | 机制 | 效果 |
|---------|------|------|
| 业务层 | Dependent Queries 检查 | 删除前检查，有依赖则拒绝删除 |
| 分支隔离 | 软删除（isActive） | 分支删除不影响主分支和其他分支 |
| 数据库层 | 级联删除（onDelete: CASCADE | 确保数据一致性 |

### 9.2 四种数据源删除后的最终状态

| 数据源类型 | 删除后 DataSource 状态 | 删除后 DataSourceVersion 状态 | 删除后 DataQuery 状态 |
|-----------|---------------------|----------------------------|---------------------|
| **Sample** | 不变 | 不变 | 不变 |
| **全局（主分支） | 已删除 | 全部删除 | 全部级联删除 |
| **全局（branch） | 不变 | 该 branch 版本 isActive=false | 不变（但该 branch 下不可用） |
| **本地** | 已删除 | 全部删除 | 全部级联删除 |

### 9.3 关键注意事项

1. **Sample 数据源是系统预置的，永远不能删除
2. **删除全局数据源（主分支）是破坏性操作，会级联删除所有关联的查询和版本
3. **Branch 中的删除是安全的，只会影响当前分支，不会影响主分支
4. **本地数据源随应用版本的生命周期管理，删除应用版本时会级联删除
5. **Dependent Queries 检查是第一道防线，通常不会走到级联删除这一步
6. **审计日志记录了所有删除操作的详细信息，可用于追踪和审计
