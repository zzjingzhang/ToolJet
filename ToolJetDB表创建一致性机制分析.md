# ToolJetDB 表创建一致性机制分析

## 概述

ToolJetDB 在创建表时需要同时维护四层数据的一致性：

1. **应用数据库的 InternalTable 记录** — 存储表的元数据（表名、列配置、组织关联等）
2. **租户 schema 中的物理表** — PostgreSQL 中实际存储数据的表
3. **主键/外键约束配置** — 数据库层面的完整性约束
4. **PostgREST schema 缓存** — PostgREST 的 schema 缓存需要刷新以识别新表

为了保证这四层数据的一致性，系统采用了**双 queryRunner 事务管理**、**前置校验**、**失败回滚**、**数量守卫**等机制。

---

## 一、为什么 createTable 需要同时管理两个 queryRunner

### 1.1 架构背景

ToolJetDB 采用**双数据库连接**架构：

| 连接 | 用途 | 对应 EntityManager |
|------|------|-------------------|
| 应用数据库连接 | 存储 ToolJet 平台的业务数据（用户、组织、InternalTable 元数据等） | `this.manager` (appManager) |
| ToolJetDB 连接 | 管理实际的 ToolJet 数据库，执行 DDL/DML 操作 | `this.tooljetDbManager` (tjdbManager) |

### 1.2 双 queryRunner 的必要性

在 [createTable](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L290-L420) 方法中，需要同时对两个数据库进行写操作：

```typescript
const queryRunner = appManager?.queryRunner || appManager.connection.createQueryRunner();
const tjdbQueryRunner = tjdbManager?.queryRunner || tjdbManager.connection.createQueryRunner();

await queryRunner.connect();
await queryRunner.startTransaction();
await tjdbQueryRunner.connect();
await tjdbQueryRunner.startTransaction();
```

**两个 queryRunner 分别负责：**

- **queryRunner (app 侧)**：
  - 创建 `InternalTable` 实体记录，写入应用数据库的 `internal_tables` 表
  - 维护列名到 UUID 的映射关系（`configurations.columns.column_names`）
  - 存储列的额外配置（`configurations.columns.configurations`）

- **tjdbQueryRunner (TJDB 侧)**：
  - 在租户 schema 下创建物理表（`CREATE TABLE`）
  - 创建主键约束（`CREATE PRIMARY KEY`）
  - 创建外键约束（如果有）

### 1.3 为何不用分布式事务

两个数据库连接是独立的 PostgreSQL 连接（可能是同一个 PG 实例的不同 database，也可能是不同实例），无法使用单个数据库事务保证原子性。系统通过**两阶段提交的简化版本**来尽可能保证一致性：

1. 先启动两个事务
2. 两个事务内的操作都成功后，**依次提交**
3. 任一事务内操作失败，则**两个都回滚**

> ⚠️ **注意**：这种方案存在理论上的不一致窗口 — 如果第一个事务提交成功但第二个事务提交失败，会出现数据不一致。但在实际场景中，第二个提交失败的概率极低（通常只有网络中断或数据库崩溃才会发生）。

---

## 二、写入前的校验机制

在启动事务之前，`createTable` 方法会执行一系列前置校验，确保请求合法。这些校验全部发生在**事务启动之前**，避免不必要的事务开销。

### 2.1 主键必填校验

```typescript
const primaryKeyColumnList = params.columns
  .filter((column) => column.constraints_type.is_primary_key)
  .map((column) => column.column_name);

if (isEmpty(primaryKeyColumnList)) throw new BadRequestException('Primary key is mandatory');
```

- **位置**：[tooljet-db-table-operations.service.ts#L299-L303](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L299-L303)
- **目的**：确保每张表至少有一个主键列

### 2.2 表名重复校验

```typescript
const tableWithSameName = await appManager.findOne(InternalTable, {
  where: { tableName, organizationId },
});

if (!isEmpty(tableWithSameName)) throw new ConflictException(`Table with name "${tableName}" already exists`);
```

- **位置**：[tooljet-db-table-operations.service.ts#L307-L314](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L307-L314)
- **目的**：同一组织内表名唯一

### 2.3 外键引用表存在性校验

```typescript
if (foreign_keys.length) {
  const referenced_table_list = foreign_keys.map((fk) => fk.referenced_table_name);
  referenced_tables_info = await this.fetchAndCheckIfValidForeignKeyTables(
    referenced_table_list, organizationId, 'TABLENAME', appManager
  );
}
```

- **位置**：[tooljet-db-table-operations.service.ts#L317-L325](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L317-L325)
- **调用**：[fetchAndCheckIfValidForeignKeyTables](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L1365-L1407)
- **目的**：确保外键引用的表在当前组织内真实存在

### 2.4 复合主键外键引用校验

```typescript
const isFKfromCompositePK = await this.checkIfForeignKeyReferencedColumnsAreFromCompositePrimaryKey(
  foreign_keys, organizationId, connectionManagers
);

if (isFKfromCompositePK) throw new ConflictException(
  'Foreign key cannot be created as the referenced column is in the composite primary key.'
);
```

- **位置**：[tooljet-db-table-operations.service.ts#L327-L336](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L327-L336)
- **调用**：[checkIfForeignKeyReferencedColumnsAreFromCompositePrimaryKey](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L1599-L1640)
- **目的**：ToolJetDB 不支持引用复合主键中的单个列作为外键

### 2.5 表数量限制校验（TableCountGuard）

在校验之前，Controller 层会先通过 `TableCountGuard` 进行许可数量校验。详见第四节。

---

## 三、失败时的回滚机制

### 3.1 整体流程

```
        ┌─────────────────────────┐
        │   启动 app 事务         │
        │   启动 tjdb 事务        │
        └───────────┬─────────────┘
                    │
        ┌───────────▼─────────────┐
        │  写入 InternalTable     │  ◄── app 侧操作
        └───────────┬─────────────┘
                    │
        ┌───────────▼─────────────┐
        │  创建物理表 + 主键/外键  │  ◄── tjdb 侧操作
        └───────────┬─────────────┘
                    │
        ┌───────────▼─────────────┐
        │  提交 app 事务          │
        └───────────┬─────────────┘
                    │
        ┌───────────▼─────────────┐
        │  提交 tjdb 事务         │
        └───────────┬─────────────┘
                    │
        ┌───────────▼─────────────┐
        │  NOTIFY pgrst 刷新      │
        └─────────────────────────┘
```

### 3.2 回滚实现

在 [createTable](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L397-L419) 的 catch 块中：

```typescript
catch (err) {
  await queryRunner.rollbackTransaction();
  await tjdbQueryRunner.rollbackTransaction();
  await queryRunner.release();
  await tjdbQueryRunner.release();
  // ... 错误包装和抛出
}
```

**回滚保证：**
- 两个事务**分别独立回滚**
- 即使某个回滚操作本身抛出异常（极端情况），也会确保连接被释放
- 错误会被包装为 `TooljetDatabaseError`，包含相关的 `internalTables` 信息以便调试

### 3.3 各操作的回滚行为

| 操作 | 回滚后状态 |
|------|-----------|
| InternalTable 写入 | 回滚后记录消失，表名可重新使用 |
| 物理表创建 | 回滚后表不存在 |
| 主键约束 | 随表回滚而消失 |
| 外键约束 | 随表回滚而消失 |
| PostgREST schema 刷新 | 在事务提交后才执行，失败时不会执行 |

> 💡 **关键点**：PostgREST 的 schema 刷新（`NOTIFY pgrst, 'reload schema'`）是在**两个事务都提交成功之后**才执行的，所以不会出现"刷新了但表没创建"的情况。

---

## 四、TableCountGuard 如何阻止超限

### 4.1 Guard 位置与触发时机

`TableCountGuard` 是一个 NestJS `CanActivate` 守卫，在 Controller 层通过 `@UseGuards` 装饰器应用于 `createTable` 接口：

```typescript
@Post('/organizations/:organizationId/table')
@UseGuards(JwtAuthGuard, TableCountGuard, FeatureAbilityGuard)
async createTable(...) { ... }
```

- **位置**：[controller.ts#L94-L100](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/controller.ts#L94-L100)
- **执行时机**：在请求进入 `createTable` 方法**之前**执行

### 4.2 实现逻辑

在 [table.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/table.guard.ts) 中：

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  const request = context.switchToHttp().getRequest();
  const tablesCount = await this.licenseTermsService.getLicenseTerms(
    LICENSE_FIELD.TABLE_COUNT,
    request?.user?.organizationId
  );
  
  // 无限许可直接放行
  if (tablesCount === LICENSE_LIMIT.UNLIMITED) {
    return true;
  }

  return dbTransactionWrap(async (manager) => {
    if ((await this.fetchTotalTablesCount(manager, request?.user?.organizationId)) >= tablesCount) {
      throw new HttpException('You have reached your maximum limit for apps.', 451);
    }
    return true;
  });
}
```

### 4.3 计数方式

```typescript
async fetchTotalTablesCount(manager: EntityManager, organizationId: string): Promise<number> {
  const tables = await manager.find(InternalTable, {
    where: { organizationId },
  });
  return tables.length;
}
```

- **计数依据**：通过查询 `InternalTable` 实体的数量来统计
- **计数范围**：按组织（`organizationId`）隔离
- **Cloud 版差异**：在 [getTablesLimit](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts#L866-L897) 方法中可以看到，Cloud 版按组织计数，其他版本可能是全局计数

### 4.4 注意事项

- **竞态条件**：Guard 的检查和实际创建表不是在同一个事务中，因此在高并发下可能存在"检查通过但创建时已超限"的理论可能性
- **错误码**：超限返回 HTTP 451 状态码
- **错误信息**：错误信息写的是 "apps" 而不是 "tables"，可能是历史遗留问题

---

## 五、SQL 模式开关如何改变 PostgREST 预配置

### 5.1 SQL 模式开关是什么

SQL 模式开关由 `isSQLModeDisabled()` 函数控制：

```typescript
export function isSQLModeDisabled(): boolean {
  return process.env.TJDB_SQL_MODE_DISABLE === 'true' || getTooljetEdition() === TOOLJET_EDITIONS.Cloud;
}
```

- **位置**：[tooljet_db.helper.ts#L80-L82](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/tooljet_db.helper.ts#L80-L82)
- **触发条件**：环境变量 `TJDB_SQL_MODE_DISABLE=true` 或 Cloud 版本

### 5.2 两种模式下的 schema 策略

| 维度 | SQL 模式启用（默认） | SQL 模式禁用（Cloud） |
|------|---------------------|----------------------|
| 租户 schema 名称 | `workspace_<organizationId>` | `public` |
| schema 数量 | 每个组织一个 schema | 所有组织共用 public schema |
| 数据库用户 | `user_<organizationId>` | 共用 `TOOLJET_DB_USER` |
| PostgREST schema 列表 | 动态加载所有 `workspace_*` schema | 固定为 `public` |

**函数调用示例（findTenantSchema）：**

```typescript
export function findTenantSchema(organisationId: string): string {
  if (isSQLModeDisabled()) {
    return 'public';
  }
  return `workspace_${organisationId}`;
}
```

### 5.3 两套 PostgREST pre_config 函数

对应两种模式，系统提供了两套 `postgrest.pre_config()` 函数的实现：

#### 模式一：SQL 模式启用（含 Schema 同步）

函数名：`reconfigurePostgrest`

- **位置**：[helper.ts#L3-L76](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/helper.ts#L3-L76)
- **pre_config 内容**：

```sql
create or replace function postgrest.pre_config()
returns void as $$
  select
    set_config('pgrst.db_aggregates_enabled', 'true/false', false);
  select
    set_config('pgrst.db_schemas', string_agg(nspname, ','), true)
  from pg_namespace
  where nspname like 'workspace_%';
$$ language sql;
```

**核心逻辑**：
- 动态从 `pg_namespace` 中查询所有 `workspace_` 开头的 schema
- 将它们设置为 PostgREST 可见的 schema 列表
- 每次新组织创建时需要调用此函数刷新

#### 模式二：SQL 模式禁用（无 Schema 同步）

函数名：`reconfigurePostgrestWithoutSchemaSync`

- **位置**：[helper.ts#L82-L152](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/helper.ts#L82-L152)
- **pre_config 内容**：

```sql
create or replace function postgrest.pre_config()
returns void as $$
  select
    set_config('pgrst.db_aggregates_enabled', 'true/false', false);
$$ language sql;
```

**核心逻辑**：
- 只设置聚合功能开关
- **不设置 `pgrst.db_schemas`**，即使用 PostgREST 默认配置（通常是 `public` schema）
- 原因：Cloud 版有大量租户，如果每个租户一个 schema 全部加载到 PostgREST 内存中，性能不可接受

### 5.4 PostgREST 代理中的模式差异

在 [PostgrestProxyService.proxy](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L28-L86) 方法中：

```typescript
const { dbUser, dbSchema } = isSQLModeDisabled()
  ? {
      dbUser: this.configService.get<string>('TOOLJET_DB_USER'),
      dbSchema: 'public',
    }
  : {
      dbUser: `user_${organizationId}`,
      dbSchema: `workspace_${organizationId}`,
    };
```

请求会根据模式设置不同的：
- **dbUser**：JWT token 中的 role
- **dbSchema**：通过 `Accept-Profile` / `Content-Profile` header 指定的 schema

### 5.5 为什么 Cloud 版要禁用 SQL 模式

代码注释给出了明确解释：

```
// TODO: Cloud TJDB SQL mode is disabled: Use public schema for cloud edition
// This is because Postgrest doesn't handle loading large amount of schemas in memory
// We need to migrate to use Table based access control instead of schema based access control
```

简而言之：**PostgREST 在内存中加载大量 schema 性能不佳**，所以 Cloud 版退化为单 schema + 表级隔离的模式。

---

## 六、完整流程图

```
HTTP 请求创建表
     │
     ▼
TableCountGuard 校验表数量
     │
     ├─ 超限 → 返回 451 错误
     │
     ▼
createTable 方法开始
     │
     ├─ 1. 校验主键是否存在
     ├─ 2. 校验表名是否重复
     ├─ 3. 校验外键引用表是否存在
     └─ 4. 校验外键是否引用复合主键
     │
     ▼
创建双 queryRunner，启动双事务
     │
     ▼
app 侧：创建 InternalTable 记录
     │
     ▼
tjdb 侧：创建物理表
     │
     ▼
tjdb 侧：创建主键约束
     │
     ▼
tjdb 侧：创建外键约束（如有）
     │
     ▼
提交 app 事务
     │
     ▼
提交 tjdb 事务
     │
     ▼
NOTIFY pgrst, 'reload schema'  ← 刷新 PostgREST 缓存
     │
     ▼
释放连接，返回结果
```

**失败路径（任一环节失败）：**

```
任一操作抛出异常
     │
     ▼
catch 块捕获
     │
     ├─ 回滚 app 事务
     ├─ 回滚 tjdb 事务
     ├─ 释放两个连接
     └─ 包装错误信息后抛出
```

---

## 七、一致性风险点

### 7.1 双事务提交间隙

两个事务是**顺序提交**的，不是原子提交。如果 app 事务提交成功、tjdb 事务提交失败：
- InternalTable 记录存在
- 物理表不存在
- 会出现"幽灵表" — 在列表中能看到但无法操作

**缓解措施**：创建表时通过 InternalTable 查询，实际操作物理表时如果不存在会抛出异常。

### 7.2 PostgREST 刷新失败

`NOTIFY pgrst, 'reload schema'` 是一个异步通知，不保证成功。如果 PostgREST 没有收到通知或刷新失败：
- 新创建的表在 PostgREST 中不可见
- 但物理表和 InternalTable 都是一致的

**缓解措施**：PostgREST 会定期自动刷新 schema 缓存；用户也可以通过其他操作触发刷新。

### 7.3 TableCountGuard 竞态条件

Guard 检查和实际创建表不在同一个事务中，高并发下可能超限。

**缓解措施**：对于大多数场景，并发创建表的概率很低，这个风险是可接受的。

---

## 八、关键文件索引

| 文件 | 核心内容 |
|------|---------|
| [tooljet-db-table-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts) | 表操作核心逻辑，createTable 实现 |
| [table.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/table.guard.ts) | 表数量限制守卫 |
| [postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts) | PostgREST 代理服务 |
| [tooljet_db.helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/tooljet_db.helper.ts) | ToolJetDB 辅助函数（isSQLModeDisabled、findTenantSchema 等） |
| [helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/helper.ts) | PostgREST 预配置函数（两套 pre_config） |
| [internal_table.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/internal_table.entity.ts) | InternalTable 实体定义 |
| [controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/controller.ts) | ToolJetDB Controller，Guard 挂载点 |
