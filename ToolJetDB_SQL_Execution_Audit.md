# ToolJetDB SQL Execution 安全审计报告

## 目录
1. [执行流程总览](#执行流程总览)
2. [各环节详细分析](#各环节详细分析)
3. [安全边界与测试建议](#安全边界与测试建议)
4. [PostgREST Proxy 路径与 SQL 执行路径对比](#postgrest-proxy-路径与-sql-执行路径对比)
5. [总结](#总结)

---

## 执行流程总览

一条用户 SQL 从 `QueryService.run` 进入 `sql_execution` 的完整调用链：

```
用户请求
  ↓
DataQueriesController (API入口)
  ↓
DataQueriesService.runQueryOnBuilder / runQueryForApp
  ↓
DataQueriesUtilService.runQuery
  ↓
service.run()  ← 通过 PluginsServiceSelector 获取 TooljetDbDataOperationsService
  ↓
TooljetDbDataOperationsService.run()  [operation = 'sql_execution']
  ↓
TooljetDbDataOperationsService.sqlExecution()
  ├─ 1. SQL模式开关检查 (isSQLModeDisabled)
  ├─ 2. Workspace配置读取 (Organization + OrganizationTjdbConfigurations)
  ├─ 3. 密码解密 (decryptTooljetDatabasePassword)
  ├─ 4. 数据库连接创建 (createTooljetDatabaseConnection)
  ├─ 5. AST解析 (node-sql-parser)
  ├─ 6. 语句allowlist检查 (checkCommandAllowlist)
  ├─ 7. 表归属检查 (verifyTablesExistInWorkspace)
  ├─ 8. 权限校验 (validateSchemaAndTablePrivileges)
  ├─ 9. 表名改写 (parseTableNameInAST)
  ├─ 10. SQL执行
  └─ 11. 连接销毁 (finally块)
```

---

## 各环节详细分析

### 1. SQL 模式开关检查

**代码位置**: [tooljet-db-data-operations.service.ts:336-337](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L336-L337)

**辅助函数**: [tooljet_db.helper.ts:80-82](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/tooljet_db.helper.ts#L80-L82)

```typescript
export function isSQLModeDisabled(): boolean {
  return process.env.TJDB_SQL_MODE_DISABLE === 'true' || getTooljetEdition() === TOOLJET_EDITIONS.Cloud;
}
```

**分析**:
- 第一层防御：全局功能开关
- 当 `TJDB_SQL_MODE_DISABLE=true` 或版本为 Cloud 时，SQL 执行功能完全禁用
- 禁用状态下直接抛出 `QueryError`，阻止后续所有逻辑
- **注意**: 此检查位于 `sqlExecution` 方法入口，是第一道闸门

---

### 2. Workspace 配置读取

**代码位置**: [tooljet-db-data-operations.service.ts:344-354](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L344-L354)

**流程**:
1. 从 `context.app.organization_id` 获取组织 ID
2. 查询 `Organization` 表验证 workspace 存在性
3. 查询 `OrganizationTjdbConfigurations` 表获取该 workspace 的 TJDB 配置
4. 配置包含 `pgPassword`（加密存储）和 `pgUser`

**实体关系**:
- `Organization`: workspace 主体
- `OrganizationTjdbConfigurations`: 每个 workspace 的 TJDB 专属配置（独立的数据库用户和密码）

**安全意义**:
- 每个 workspace 拥有独立的数据库角色（`user_<orgId>`）
- 每个 workspace 拥有独立的 schema（`workspace_<orgId>`）
- 实现了租户级别的数据隔离

---

### 3. 密码解密

**代码位置**: [tooljet_db.helper.ts:45-57](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/tooljet_db.helper.ts#L45-L57)

**解密服务**: [encryption/service.ts:27-30](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/encryption/service.ts#L27-L30)

```typescript
export async function decryptTooljetDatabasePassword(password: string) {
  if (isSQLModeDisabled()) {
    return process.env.TOOLJET_DB_PASS;
  }
  const encryptionService = new EncryptionService();
  const decryptedvalue = await encryptionService.decryptColumnValue(
    'organization_tjdb_configurations',
    'pg_password',
    password
  );
  return decryptedvalue;
}
```

**加密算法**: AES-256-GCM
- 使用 HKDF 派生每列独立密钥
- 主密钥来自 `LOCKBOX_MASTER_KEY` 环境变量
- 12 字节 nonce + 16 字节 auth tag

**安全意义**:
- 数据库密码不以明文存储
- 每列独立密钥降低密钥泄露影响范围
- SQL 模式禁用时降级使用全局数据库密码（配合 public schema）

---

### 4. 数据库连接创建

**代码位置**: [tooljet_db.helper.ts:21-43](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/tooljet_db.helper.ts#L21-L43)

```typescript
export async function createTooljetDatabaseConnection(
  password: string,
  dbUser: string,
  dbSchema: string
): Promise<{ tooljetDbTenantConnection: DataSource; tooljetDbTenantManager: EntityManager }> {
  const tooljetDbTenantConnection = new DataSource({
    ...tooljetDbOrmconfig,
    schema: dbSchema,
    username: dbUser,
    password: password,
    name: `${dbSchema}_${uuidv4()}`,
    extra: {
      ...tooljetDbOrmconfig.extra,
      idleTimeoutMillis: 10000,
      max: 2,
      allowExitOnIdle: true,
    },
  } as any);
  await tooljetDbTenantConnection.initialize();
  const tooljetDbTenantManager = tooljetDbTenantConnection.createEntityManager();
  return { tooljetDbTenantConnection, tooljetDbTenantManager };
}
```

**关键配置**:
- 使用租户专属用户名和密码登录
- 连接名包含 schema 和 UUID，确保唯一性
- 连接池限制 `max: 2`，防止资源耗尽
- 空闲超时 10 秒
- **注意**: 每个 SQL 执行请求都会创建**新连接**，而非复用连接池

---

### 5. AST 解析

**代码位置**: [tooljet-db-data-operations.service.ts:361-375](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L361-L375)

```typescript
const sqlParser = new Parser();
try {
  const parsedSQL = sqlParser.parse(sqlQuery);
  ast = parsedSQL.ast;
  tableList = parsedSQL.tableList;
} catch (error) {
  return { status: 'failed', errorMessage: 'Syntax error encountered', ... };
}
```

**使用的库**: `node-sql-parser/build/postgresql`

**解析结果**:
- `ast`: SQL 语句的抽象语法树
- `tableList`: SQL 中涉及的所有表名列表，格式为 `<statement_type>::<schema>::<table>`

**安全意义**:
- 语法解析是后续所有安全检查的基础
- 只有语法合法的 SQL 才能进入后续检查
- 解析失败直接返回，不执行任何数据库操作

---

### 6. 语句 Allowlist 检查

**代码位置**: [tooljet-db-data-operations.service.ts:455-468](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L455-L468)

```typescript
protected checkCommandAllowlist(parsedSqlAst: AST[] | AST): boolean {
  const allowList = ['select', 'insert', 'update', 'delete', 'transaction'];
  let isValidCommand = true;
  if (Array.isArray(parsedSqlAst)) {
    parsedSqlAst.forEach((sqlExpression) => {
      if (!allowList.includes(sqlExpression.type)) isValidCommand = false;
    });
  }
  if (!Array.isArray(parsedSqlAst) && parsedSqlAst.type) {
    if (!allowList.includes(parsedSqlAst.type)) isValidCommand = false;
  }
  return isValidCommand;
}
```

**允许的语句类型**:
- `select`: 查询
- `insert`: 插入
- `update`: 更新
- `delete`: 删除
- `transaction`: 事务

**被禁止的语句类型**（隐式禁止）:
- DDL 语句（CREATE/ALTER/DROP TABLE 等）
- DCL 语句（GRANT/REVOKE 等）
- 系统函数调用
- 多语句组合中若有任一不在白名单中，整体拒绝

**安全意义**:
- 白名单机制，只允许数据操作语句
- 防止 DDL/DCL 注入攻击
- 防止通过 SQL 修改数据库结构或权限

---

### 7. 表归属检查

**代码位置**: [tooljet-db-data-operations.service.ts:476-487](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L476-L487)

```typescript
protected async verifyTablesExistInWorkspace(tablesUsedInquery: Array<string>, organizationId: string) {
  const tableDetailsInList = await this.manager.find(InternalTable, {
    where: { organizationId: organizationId, tableName: In(tablesUsedInquery) },
  });
  const tableList = tableDetailsInList.map((table) => table.tableName);
  const tablesNotInOrg = tablesUsedInquery.filter((tableName) => !tableList.includes(tableName));
  if (isEmpty(tablesNotInOrg)) return tableDetailsInList;
  throw new NotFoundException(`Table: ${tablesNotInOrg.join(', ')} not found`);
}
```

**检查逻辑**:
1. 从 AST 解析出所有表名
2. 查询 `InternalTable` 表，验证这些表是否属于当前 workspace
3. 若有任何表不属于当前 workspace，抛出 `NotFoundException`

**InternalTable 实体**:
- 记录 workspace 中所有表的元数据
- 包含 `organizationId` 字段标识归属
- `tableName` 是用户可见的表名
- 实际数据库中的表名是 UUID（`id` 字段）

**安全意义**:
- 防止跨 workspace 表访问
- 即使用户在 SQL 中写了其他 workspace 的表名，也会被拦截
- 这是水平权限隔离的关键环节

---

### 8. 权限校验

**代码位置**: [tooljet-db-data-operations.service.ts:516-560](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L516-L560)

包含两层校验：

#### 8.1 Schema 权限校验
```typescript
protected async validateSchemaPrivileges(tooljetDbManager: EntityManager, pgUser: string, schema: string) {
  const [{ has_schema_privilege }] = await tooljetDbManager.query(
    `SELECT has_schema_privilege('${pgUser}', '${schema}', 'USAGE')`
  );
  if (!has_schema_privilege) throw new Error('You are not authorized to perform actions on some schemas.');
}
```

#### 8.2 表权限校验
```typescript
protected async validateUserHasTablePrivileges(
  internalTableNameToIdMap, tooljetDbManager, pgUser, schema, tableName, tenantSchema
) {
  const queryToExecute = schema
    ? `SELECT has_table_privilege('${pgUser}', '${schema}.${internalTableNameToIdMap[tableName]}', 'SELECT')`
    : `SELECT has_table_privilege('${pgUser}', '${tenantSchema}.${internalTableNameToIdMap[tableName]}', 'SELECT')`;
  const [{ has_table_privilege }] = await tooljetDbManager.query(queryToExecute);
  if (!has_table_privilege) throw new Error('TJDB table permission denied');
}
```

**校验方式**:
- 使用 PostgreSQL 内置的权限检查函数 `has_schema_privilege` 和 `has_table_privilege`
- 以超级用户/管理员身份执行查询（使用 `tooljetDbManager`，即主连接）
- 检查租户用户是否具有相应权限

**安全意义**:
- 数据库层面的权限二次确认
- 即使应用层校验被绕过，数据库层权限仍然生效
- 检查 `USAGE`（schema）和 `SELECT`（table）权限

---

### 9. 表名改写

**代码位置**: [tooljet-db-data-operations.service.ts:435-448](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L435-L448)

```typescript
protected parseTableNameInAST(parsedSql, internalTableNameToIdMap) {
  if (Array.isArray(parsedSql)) {
    parsedSql.forEach((item) => this.parseTableNameInAST(item, internalTableNameToIdMap));
  } else if (typeof parsedSql === 'object' && parsedSql !== null) {
    if (parsedSql['table'] && !isEmpty(internalTableNameToIdMap)) {
      parsedSql.table = internalTableNameToIdMap[parsedSql.table]
        ? internalTableNameToIdMap[parsedSql.table]
        : parsedSql.table;
    }
    Object.keys(parsedSql).forEach((key) => {
      this.parseTableNameInAST(parsedSql[key], internalTableNameToIdMap);
    });
  }
}
```

**改写逻辑**:
1. 递归遍历 AST 所有节点
2. 查找 `table` 属性
3. 将用户可见的表名替换为实际的 UUID 表名
4. 若映射表中不存在，则保留原名（**潜在风险点**）

**为什么需要改写**:
- 数据库中实际表名是 UUID（如 `a1b2c3d4-...`）
- 用户看到的是可读表名（如 `users`）
- 通过 `InternalTable` 表维护映射关系

**安全意义**:
- 防止用户直接猜测/操作真实表名
- 增加了攻击难度（需要同时知道 UUID 表名）
- 配合 schema 隔离，形成多层防御

---

### 10. SQL 执行与连接销毁

**代码位置**: [tooljet-db-data-operations.service.ts:408-427](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L408-L427)

```typescript
this.parseTableNameInAST(ast, internalTableNameToIdMap);
const validSql = await sqlParser.sqlify(ast);
const results = await tooljetDbTenantConnection.query(validSql);
return { status: 'ok', data: { results } };
// ... catch 块 ...
} finally {
  await tooljetDbTenantConnection.destroy();
}
```

**执行流程**:
1. 将改写后的 AST 转回 SQL 字符串（`sqlify`）
2. 使用租户连接执行 SQL
3. 无论成功失败，`finally` 块确保连接销毁

**连接销毁**:
- 放在 `finally` 块中，确保异常时也能释放连接
- 防止连接泄漏导致资源耗尽
- 配合连接池的 `idleTimeoutMillis`，形成双重保障

---

## 安全边界与测试建议

### 边界 1: SQL 注入与 AST 改写绕过

**风险描述**:
`parseTableNameInAST` 方法只替换 AST 中的 `table` 属性。如果攻击者能构造 SQL 使得表名出现在非 `table` 属性中（如通过子查询、CTE、函数参数等），可能绕过表名改写和权限检查。

**相关代码**:
- [parseTableNameInAST](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L435-L448)
- [verifyTablesExistInWorkspace](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L476-L487)

**测试建议**:
1. **子查询注入测试**:
   ```sql
   SELECT * FROM users WHERE id = (SELECT id FROM other_workspace_table LIMIT 1)
   ```
   验证: `other_workspace_table` 是否被检测到并拒绝

2. **CTE (WITH) 注入测试**:
   ```sql
   WITH hacked AS (SELECT * FROM pg_tables) SELECT * FROM hacked
   ```
   验证: 系统表 `pg_tables` 是否被拦截

3. **函数调用中的表名**:
   ```sql
   SELECT * FROM information_schema.tables
   ```
   验证: `information_schema` schema 的访问是否被拒绝

4. **数组/JSON 中的表名**: 测试是否能通过非字符串上下文绕过 AST 解析

---

### 边界 2: 跨租户 Schema 访问

**风险描述**:
虽然默认设置了 `search_path` 到租户 schema，但如果 SQL 中显式指定其他 schema（如 `public`、`information_schema`、`pg_catalog`），可能绕过 schema 隔离。

**相关代码**:
- [validateSchemaPrivileges](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L540-L545)
- [SET search_path](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L385)

**测试建议**:
1. **显式 schema 访问测试**:
   ```sql
   SELECT * FROM public.some_table
   SELECT * FROM pg_catalog.pg_user
   SELECT * FROM information_schema.tables
   ```
   验证: 是否被 `validateSchemaPrivileges` 捕获并拒绝

2. **schema 权限绕过测试**:
   - 测试租户用户是否真的没有其他 schema 的 USAGE 权限（数据库层验证）
   - 验证即使应用层检查被绕过，数据库层权限仍然生效

3. **动态 SQL/函数调用绕过**:
   - 测试是否能通过 `dblink`、`pg_stat_statements` 等扩展绕过隔离
   - 测试 `set search_path` 语句是否在 allowlist 中被禁止

---

### 边界 3: Allowlist 绕过与危险语句

**风险描述**:
`checkCommandAllowlist` 只检查 AST 的顶层 `type`。如果危险操作嵌套在允许的语句内部（如 `SELECT` 中的函数调用有副作用），可能被放行。

**相关代码**:
- [checkCommandAllowlist](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L455-L468)

**测试建议**:
1. **危险函数调用测试**:
   ```sql
   SELECT pg_sleep(10)  -- 资源耗尽攻击
   SELECT version()     -- 信息泄露
   SELECT current_user  -- 信息泄露
   ```
   验证: 这些函数是否应该被禁止？当前实现是允许的（因为顶层是 SELECT）

2. **多语句绕过测试**:
   ```sql
   SELECT 1; DROP TABLE users
   ```
   验证: node-sql-parser 是否能正确解析多语句，且第二条是否被 allowlist 拦截

3. **事务中的危险操作**:
   ```sql
   BEGIN; DROP TABLE users; COMMIT;
   ```
   验证: `transaction` 类型是否会递归检查内部语句

4. **COPY 语句测试**:
   ```sql
   COPY users TO '/tmp/pwned'
   ```
   验证: COPY 语句是否在 allowlist 中

---

### 边界 4 (补充): 连接耗尽与拒绝服务

**风险描述**:
每次 SQL 执行都会创建新的数据库连接（`max: 2`），如果攻击者发起大量并发请求，可能耗尽数据库连接资源。

**相关代码**:
- [createTooljetDatabaseConnection](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/tooljet_db.helper.ts#L21-L43)
- [finally 块中的 destroy](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L425-L427)

**测试建议**:
1. **并发连接测试**: 模拟大量并发 SQL 执行请求，验证是否有速率限制
2. **慢查询连接占用测试**: 执行长耗时 SQL，验证连接占用情况和超时机制
3. **异常路径连接释放测试**: 在各个可能抛出异常的点测试，验证 `finally` 块总能执行

---

## PostgREST Proxy 路径与 SQL 执行路径对比

### 架构对比

| 维度 | PostgREST Proxy 路径 | SQL 执行路径 |
|------|---------------------|-------------|
| **入口** | `POST /api/tooljet-db/proxy/{tableId}` | `QueryService.run` → `sql_execution` |
| **核心服务** | [PostgrestProxyService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts) | [TooljetDbDataOperationsService.sqlExecution](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L335-L428) |
| **执行方式** | HTTP 请求转发给 PostgREST 服务 | 直接通过 TypeORM 连接执行 SQL |
| **认证方式** | JWT (PostgREST 验证) | 数据库用户名/密码 (租户角色) |
| **表名处理** | URL 中直接使用 tableId (UUID) | AST 解析后将可读表名改写为 UUID |

### 安全机制对比

| 安全机制 | PostgREST Proxy | SQL 执行路径 |
|---------|-----------------|-------------|
| **功能开关** | 不直接依赖 `isSQLModeDisabled`，但受 Cloud 模式影响 | `isSQLModeDisabled` 第一关拦截 |
| **语句类型限制** | 受 HTTP 方法限制 (GET/POST/PATCH/DELETE) | AST allowlist 检查 (select/insert/update/delete/transaction) |
| **表归属检查** | 通过 `tableId` 查询 `InternalTable` 验证 | 通过 AST 解析表名，查询 `InternalTable` 验证 |
| **权限校验** | PostgREST 基于角色的权限控制 (JWT 中的 role) | 应用层校验 + 数据库 `has_table_privilege` 双重校验 |
| **Schema 隔离** | 通过 `Accept-Profile` / `Content-Profile` header 指定 schema | `SET search_path` + 权限校验 |
| **SQL 注入防护** | PostgREST 自动参数化（强） | 依赖 AST 解析和表名改写（弱，用户可控 SQL 原文） |
| **连接管理** | PostgREST 连接池复用 | 每次请求新建连接，finally 销毁 |

### 关键差异详解

#### 1. 攻击面差异
- **PostgREST Proxy**: 攻击面较小，操作类型由 HTTP 方法决定，参数化程度高，SQL 注入风险低
- **SQL 执行路径**: 攻击面大，用户可提交任意 SQL，依赖多层校验防御

#### 2. 性能差异
- **PostgREST Proxy**: 连接复用，性能好，适合高频小查询
- **SQL 执行路径**: 每次新建连接，开销大，适合复杂查询

#### 3. 权限模型差异
- **PostgREST Proxy**: 基于 JWT 中的 role 声明，PostgREST 内部切换角色
- **SQL 执行路径**: 使用租户用户名/密码直接登录，权限由数据库角色控制

#### 4. 错误处理差异
- **PostgREST Proxy**: 错误信息经过 `TooljetDatabaseError` 处理，脱敏 schema 名和表 UUID
- **SQL 执行路径**: 同样经过错误处理，但原始 SQL 错误信息可能更详细

#### 5. 功能能力差异
- **PostgREST Proxy**: 支持基本 CRUD、过滤、排序、分页、聚合、外键关联
- **SQL 执行路径**: 支持任意 SELECT/INSERT/UPDATE/DELETE，能力更强但风险更高

### 两条路径的交汇点

两条路径在以下层面共享安全机制:
1. **FeatureAbilityGuard**: 统一的功能权限守卫
2. **OrganizationAuthGuard**: 统一的组织认证
3. **InternalTable**: 共享的表元数据和归属关系
4. **TooljetDatabaseError**: 统一的错误处理和脱敏

---

## 总结

### 核心安全防线（按执行顺序）

1. **功能级**: SQL 模式全局开关 (`isSQLModeDisabled`)
2. **语法级**: AST 解析，只接受语法合法的 SQL
3. **语句类型级**: Allowlist 白名单，只允许 DML 语句
4. **表归属级**: 验证所有表属于当前 workspace
5. **Schema 级**: 租户专属 schema + 权限校验
6. **表级**: 数据库角色表权限双重校验
7. **表名级**: 可读表名 → UUID 表名改写
8. **资源级**: 连接池限制 + finally 确保释放

### 主要风险点

1. **AST 改写不彻底**: 只替换 `table` 属性，可能有遗漏
2. **函数调用风险**: SELECT 语句中的函数调用可能有副作用或信息泄露
3. **连接资源**: 每次新建连接，存在 DoS 风险
4. **SQL 注入**: 用户可控 SQL 原文，虽经多层校验但仍比参数化查询风险高

### 建议优先测试的三个安全边界

1. **AST 表名改写完整性** — 验证所有表引用场景都被正确检测和改写
2. **跨 schema 访问防护** — 验证显式 schema 引用和系统 schema 访问被拦截
3. **Allowlist 绕过** — 验证危险语句无法通过嵌套或伪装进入执行

---

*审计日期: 2026-06-09*
*审计范围: ToolJetDB sql_execution 操作*
