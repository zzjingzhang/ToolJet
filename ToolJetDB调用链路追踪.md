# ToolJetDB 调用链路追踪文档

## 目录

1. [整体架构概述](#整体架构概述)
2. [调用链路总览](#调用链路总览)
3. [各操作详细链路](#各操作详细链路)
   - [proxy 操作](#proxy-操作)
   - [list_rows 操作](#list_rows-操作)
   - [update_rows 操作](#update_rows-操作)
   - [delete_rows 操作](#delete_rows-操作)
   - [join_tables 操作](#join_tables-操作)
   - [bulk 操作](#bulk-操作)
4. [核心技术要点](#核心技术要点)
   - [PostgREST JWT 角色](#postgrest-jwt-角色)
   - [Workspace Schema](#workspace-schema)
   - [tableName 占位符替换](#tablename-占位符替换)
   - [Content-Range 暴露](#content-range-暴露)
   - [null 过滤条件禁止](#null-过滤条件禁止)
   - [limit/offset 校验](#limitoffset-校验)
   - [QueryError 包装](#queryerror-包装)

---

## 整体架构概述

ToolJetDB 是 ToolJet 内置的数据库功能，基于 PostgreSQL + PostgREST 实现。整体采用分层架构：

- **前端层**：提供 UI 交互和查询编辑器
- **Nest Controller 层**：处理 HTTP 请求，路由分发
- **Service 层**：业务逻辑处理
  - `TooljetDbDataOperationsService`：数据操作服务（实现 QueryService 接口）
  - `TooljetDbTableOperationsService`：表结构操作服务
  - `PostgrestProxyService`：PostgREST 代理服务
  - `TooljetDbBulkUploadService`：批量上传服务
- **PostgREST 层**：RESTful API 接口层
- **PostgreSQL 层**：数据存储层

---

## 调用链路总览

ToolJetDB 操作有两种调用路径：

### 路径一：直接 API 调用（表管理操作）

```
前端 (tooljetDatabase.service.js)
  ↓
TooljetDbController (NestJS)
  ↓
TooljetDbTableOperationsService / TooljetDbBulkUploadService
  ↓
PostgreSQL (TypeORM)
```

### 路径二：Data Query 调用（数据操作）

```
前端 Query Editor
  ↓
DataQueriesController (NestJS)
  ↓
DataQueriesService
  ↓
DataQueriesUtilService.runQuery()
  ↓
PluginsServiceSelector.getService() → 返回 TooljetDbDataOperationsService
  ↓
TooljetDbDataOperationsService.run()
  ├─ list_rows/update_rows/delete_rows/create_row → PostgrestProxyService.perform()
  ├─ join_tables → TooljetDbTableOperationsService.perform('join_tables')
  └─ bulk_update/upsert → TooljetDbBulkUploadService
  ↓
PostgREST / PostgreSQL
```

---

## 各操作详细链路

### proxy 操作

#### 入口 Controller

文件：[controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/controller.ts#L61-L66)

```typescript
@All('/proxy/*')
@UseGuards(OrganizationAuthGuard, FeatureAbilityGuard)
async proxy(@Req() req, @Res() res, @Next() next) {
  return this.postgrestProxyService.proxy(req, res, next);
}
```

#### 代理流程

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L28-L86)

1. **提取组织 ID**：从 header `tj-workspace-id` 或 `req.dataQuery.app.organizationId` 获取
2. **确定数据库用户和 Schema**：
   - SQL 模式禁用时：使用 `TOOLJET_DB_USER` 和 `public` schema
   - SQL 模式启用时：使用 `user_${organizationId}` 和 `workspace_${organizationId}`
3. **签名 JWT**：生成 PostgREST JWT token
4. **替换表名占位符**：`${tableName}` → tableId
5. **设置请求头**：Authorization、Prefer、Accept-Profile/Content-Profile
6. **暴露 Content-Range**：设置 `Access-Control-Expose-Headers`
7. **验证 JSONB 输入**：对 PATCH/POST 请求验证 JSONB 列
8. **转发请求**：使用 express-http-proxy 转发到 PostgREST

---

### list_rows 操作

#### 前端调用

文件：[tooljetDatabase.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/tooljetDatabase.service.js#L8-L10)

```javascript
function findOne(headers, tableId, query = '') {
  return tooljetAdapter.get(`/tooljet-db/proxy/${tableId}?${query}`, headers);
}
```

#### Data Query 调用路径

文件：[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L127-L190)

```
TooljetDbDataOperationsService.run()
  ↓ (operation: 'list_rows')
TooljetDbDataOperationsService.listRows()
  ├─ 检查 null 过滤条件
  ├─ 校验 limit/offset 是否为有效整数
  ├─ 查询 InternalTable 获取表信息
  ├─ 构建 PostgREST 查询参数（where、order、aggregates、groupBy）
  ├─ 构建 URL: /api/tooljet-db/proxy/${tableId}?query
  └─ 调用 proxyPostgrest()
       ↓
PostgrestProxyService.perform()
  ├─ 确定 dbUser 和 dbSchema
  ├─ 签名 JWT
  ├─ 替换 URL 路径
  ├─ 设置请求头
  ├─ 使用 got 库发送请求到 PostgREST
  └─ 返回结果或抛出 QueryError
```

关键代码：
```typescript
async listRows(queryOptions, context): Promise<QueryResult> {
  if (hasNullValueInFilters(queryOptions, 'list_rows')) {
    return { status: 'failed', errorMessage: 'Null value comparison not allowed...', data: {} };
  }
  // 校验 limit/offset
  if (limit && isNaN(limit))
    throw new QueryError('An incorrect limit value.', 'Limit should be a valid integer', {});
  // 构建查询并代理
  return await this.proxyPostgrest(maybeSetSubPath(url), 'GET', headers);
}
```

---

### update_rows 操作

#### 前端调用

文件：[tooljetDatabase.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/tooljetDatabase.service.js#L111-L113)

```javascript
function updateRows(headers, tableId, data, query = '') {
  return tooljetAdapter.patch(`/tooljet-db/proxy/${tableId}?${query}`, data, headers);
}
```

#### Data Query 调用路径

文件：[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L204-L228)

```
TooljetDbDataOperationsService.run()
  ↓ (operation: 'update_rows')
TooljetDbDataOperationsService.updateRows()
  ├─ 检查 null 过滤条件
  ├─ 构建 where 查询条件
  ├─ 构建更新数据 body
  ├─ 构建 URL
  └─ 调用 proxyPostgrest() 方法为 PATCH
```

关键代码：
```typescript
async updateRows(queryOptions, context): Promise<QueryResult> {
  if (hasNullValueInFilters(queryOptions, 'update_rows')) {
    return { status: 'failed', errorMessage: 'Null value comparison not allowed...', data: {} };
  }
  // 构建 where 和 body
  const url = maybeSetSubPath(`/api/tooljet-db/proxy/${tableId}?` + query.join('&') + '&order=id');
  return await this.proxyPostgrest(url, 'PATCH', headers, body);
}
```

---

### delete_rows 操作

#### 前端调用

文件：[tooljetDatabase.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/tooljetDatabase.service.js#L123-L125)

```javascript
function deleteRows(headers, tableId, query = '') {
  return tooljetAdapter.delete(`/tooljet-db/proxy/${tableId}?${query}`, headers);
}
```

#### Data Query 调用路径

文件：[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L230-L268)

```
TooljetDbDataOperationsService.run()
  ↓ (operation: 'delete_rows')
TooljetDbDataOperationsService.deleteRows()
  ├─ 检查 null 过滤条件
  ├─ 检查 where 过滤器是否为空（为空则拒绝）
  ├─ 校验 limit 是否为有效整数
  ├─ 构建 where、limit、order 查询
  └─ 调用 proxyPostgrest() 方法为 DELETE
```

关键代码：
```typescript
async deleteRows(queryOptions, context): Promise<QueryResult> {
  if (hasNullValueInFilters(queryOptions, 'delete_rows')) {
    return { status: 'failed', errorMessage: 'Null value comparison not allowed...', data: {} };
  }
  // 必须有 where 条件
  if (isEmpty(whereQuery)) {
    return { status: 'failed', errorMessage: 'Please provide a where filter or a limit to delete rows', data: {} };
  }
  // 校验 limit
  if (limit && isNaN(limit)) {
    throw new QueryError('An incorrect limit value.', 'Limit should be a valid integer', {});
  }
  return await this.proxyPostgrest(url, 'DELETE', headers);
}
```

---

### join_tables 操作

#### 前端调用

文件：[tooljetDatabase.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/tooljetDatabase.service.js#L135-L137)

```javascript
function joinTables(headers, organizationId, data) {
  return tooljetAdapter.post(`tooljet-db/organizations/${organizationId}/join`, data, headers);
}
```

#### Controller 入口

文件：[controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/controller.ts#L165-L178)

```typescript
@Post('/organizations/:organizationId/join')
@UseFilters(new TooljetDbJoinExceptionFilter())
@UseGuards(OrganizationAuthGuard, FeatureAbilityGuard)
async joinTables(@Req() req, @Body() tooljetDbJoinDto: TooljetDbJoinDto, @Param('organizationId') organizationId) {
  const params = {
    joinQueryJson: { ...tooljetDbJoinDto },
    dataQuery: req.dataQuery,
    user: req.user,
  };
  const result = await this.tableOperationsService.perform(organizationId, 'join_tables', params);
  return decamelizeKeys({ result });
}
```

#### Data Query 调用路径

文件：[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L270-L333)

```
TooljetDbDataOperationsService.run()
  ↓ (operation: 'join_tables')
TooljetDbDataOperationsService.joinTables()
  ├─ 验证输入非空
  ├─ 验证必填字段（Select、From）
  ├─ 校验 limit/offset 是否为有效整数
  ├─ 清理空的非必填字段（conditions、group_by、aggregates、order_by）
  └─ 调用 tableOperationsService.perform('join_tables')
       ↓
TooljetDbTableOperationsService.joinTable()
  ├─ 标准化查询 JSON 格式
  ├─ 获取租户数据库配置
  ├─ 收集涉及的表，验证表存在
  ├─ 创建租户数据库连接
  ├─ 使用 TypeORM QueryBuilder 构建 JOIN 查询
  └─ 执行查询并返回结果
```

注意：join_tables 操作 **不经过 PostgREST**，而是直接通过 TypeORM 连接数据库执行 SQL 查询。

---

### bulk 操作

Bulk 操作包含多种批量处理：

1. **bulk-upload**：CSV 文件批量上传（Controller 直接调用）
2. **bulk_update_with_primary_key**：通过主键批量更新
3. **bulk_upsert_with_primary_key**：通过主键批量 upsert

#### bulk-upload（CSV 上传）

文件：[controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/controller.ts#L152-L163)

```
TooljetDbController.bulkUpload()
  ↓
TooljetDbBulkUploadService.perform()
  ↓
TooljetDbBulkUploadService.bulkUploadCsv()
  ├─ 获取表结构信息
  ├─ 解析 CSV 文件
  ├─ 验证表头是表列的子集
  ├─ 验证并转换列数据类型
  ├─ 检查主键重复
  └─ 事务执行批量 upsert
       ↓
TooljetDbBulkUploadService.bulkUpsertRows()
  └─ 使用 INSERT ... ON CONFLICT DO UPDATE 执行
```

#### bulk_update_with_primary_key

文件：[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L85-L125)

```
TooljetDbDataOperationsService.run()
  ↓ (operation: 'bulk_update_with_primary_key')
TooljetDbDataOperationsService.bulkUpdateWithPrimaryKey()
  ├─ 检查 null 过滤条件
  └─ 调用 tooljetDbBulkUploadService.bulkUpdateRowsWithPrimaryKey()
       ↓
TooljetDbBulkUploadService.bulkUpdateRowsWithPrimaryKey()
  ├─ 验证表存在
  ├─ 逐行构建 UPDATE 查询
  └─ 逐行执行更新
```

#### bulk_upsert_with_primary_key

文件：[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L591-L653)

```
TooljetDbDataOperationsService.run()
  ↓ (operation: 'bulk_upsert_with_primary_key')
TooljetDbDataOperationsService.bulkUpsertUsingPrimaryKey()
  ├─ 检查 null 过滤条件
  ├─ 验证输入
  └─ 调用 tooljetDbBulkUploadService.bulkUpsertRowsWithPrimaryKey()
       ↓
TooljetDbBulkUploadService.bulkUpsertRowsWithPrimaryKey()
  ├─ 验证表存在
  ├─ 验证主键列存在且是主键
  ├─ 按列分组处理
  └─ 使用 INSERT ... ON CONFLICT DO UPDATE 执行
```

---

## 核心技术要点

### PostgREST JWT 角色

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L216-L225)

#### 实现方式

```typescript
protected signJwtPayload(role: string) {
  const payload = { role };
  const secretKey = this.configService.get<string>('PGRST_JWT_SECRET');
  const token = jwt.sign(payload, secretKey, {
    algorithm: 'HS256',
    expiresIn: '1m',
  });
  return token;
}
```

#### 角色确定逻辑

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L31-L39)

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

#### 关键点

- **JWT 算法**：HS256
- **过期时间**：1 分钟
- **Payload**：仅包含 `role` 字段
- **角色命名**：`user_${organizationId}`（SQL 模式启用时）
- **Secret**：从环境变量 `PGRST_JWT_SECRET` 获取

---

### Workspace Schema

#### Schema 命名规则

每个工作区（organization）有独立的数据库 schema，命名格式为：

```
workspace_${organizationId}
```

#### Schema 确定位置

1. **PostgrestProxyService.proxy()**：[postgrest-proxy.service.ts#L31-L39](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L31-L39)
2. **PostgrestProxyService.perform()**：[postgrest-proxy.service.ts#L103-L111](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L103-L111)
3. **findTenantSchema()** 工具函数

#### PostgREST 中的 Schema 设置

通过 HTTP Header 设置 schema：

- **GET/HEAD 请求**：`Accept-Profile: ${dbSchema}`
- **POST/PATCH/PUT/DELETE 请求**：`Content-Profile: ${dbSchema}`

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L48-L49)

```typescript
if (['GET', 'HEAD'].includes(req.method)) req.headers['Accept-Profile'] = dbSchema;
if (['POST', 'PATCH', 'PUT', 'DELETE'].includes(req.method)) req.headers['Content-Profile'] = dbSchema;
```

---

### tableName 占位符替换

#### 占位符格式

URL 中使用 `${tableName}` 作为表名占位符，最终会被替换为实际的 table ID（UUID）。

示例：
```
输入: /proxy/${actors}?select=first_name,last_name,${films}(title)
输出: /proxy/table-id-1?select=first_name,last_name,table-id-2(title)
```

#### 替换流程

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L232-L248)

```
replaceTableNamesAtPlaceholder(url, organizationId)
  ├─ 用正则匹配所有 ${...} 占位符
  ├─ 提取表名列表
  ├─ 查询 InternalTable 验证表存在
  ├─ 构建 tableName → tableId 映射
  └─ 替换所有占位符
```

#### 关键代码

```typescript
async replaceTableNamesAtPlaceholder(url: string, organizationId: string) {
  const urlToReplace = decodeURIComponent(url);
  const placeHolders = urlToReplace.match(/\$\{.+\}/g);
  if (isEmpty(placeHolders)) return url;
  
  const requestedtableNames = placeHolders.map((placeHolder) => placeHolder.slice(2, -1));
  const internalTables = await this.findOrFailAllInternalTableFromTableNames(requestedtableNames, organizationId);
  // ... 构建映射并替换
}
```

---

### Content-Range 暴露

#### 实现位置

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L51)

```typescript
res.set('Access-Control-Expose-Headers', 'Content-Range');
```

#### 作用

- PostgREST 在响应中返回 `Content-Range` header，用于分页（显示总记录数）
- 由于 CORS 策略，浏览器默认无法访问自定义 header
- 通过设置 `Access-Control-Expose-Headers: Content-Range`，允许前端读取该 header

#### Prefer Header

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L47)

```typescript
req.headers['Prefer'] = `count=exact, return=representation`;
```

- `count=exact`：要求 PostgREST 返回精确的总记录数（在 Content-Range 中）
- `return=representation`：要求 POST/PATCH/PUT 操作返回修改后的行数据

---

### null 过滤条件禁止

#### 目的

防止使用 `=` 等操作符与 `null` 值比较（在 SQL 中应该使用 `IS NULL` 或 `IS NOT NULL`）

#### 实现位置

文件：[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L656-L668)

```typescript
function hasNullValueInFilters(queryOptions, operation) {
  const filters = queryOptions.operation?.where_filters;
  if (filters) {
    const filterKeys = Object.keys(filters);
    for (let i = 0; i < filterKeys.length; i++) {
      const filter = filters[filterKeys[i]];
      if (filter.operator !== 'is' && filter.value === null) {
        return true;
      }
    }
  }
  return false;
}
```

#### 应用的操作

以下操作都会检查 null 过滤条件：

- `list_rows`
- `update_rows`
- `delete_rows`
- `bulk_update_with_primary_key`
- `bulk_upsert_with_primary_key`

#### 错误信息

```
Null value comparison not allowed, To check null values Please use IS operator instead.
```

---

### limit/offset 校验

#### 目的

确保 limit 和 offset 参数是有效的整数，防止无效值传入 PostgREST 导致错误。

#### 实现位置

**list_rows 中**：
[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L150-L153)

```typescript
if (limit && isNaN(limit))
  throw new QueryError('An incorrect limit value.', 'Limit should be a valid integer', {});
if (offset && isNaN(offset))
  throw new QueryError('An incorrect offset value.', 'Offset should be a valid integer', {});
```

**delete_rows 中**：
[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L252-L254)

```typescript
if (limit && isNaN(limit)) {
  throw new QueryError('An incorrect limit value.', 'Limit should be a valid integer', {});
}
```

**join_tables 中**：
[tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts#L299-L305)

```typescript
if (sanitizedJoinTableJson.limit && isNaN(sanitizedJoinTableJson.limit)) {
  throw new QueryError('An incorrect limit value.', 'Limit should be a valid integer', {});
}
if (sanitizedJoinTableJson.offset && isNaN(sanitizedJoinTableJson.offset)) {
  throw new QueryError('An incorrect offset value.', 'Offset should be a valid integer', {});
}
```

---

### QueryError 包装

#### 错误包装层级

```
PostgreSQL 原始错误
  ↓
PostgrestError (PostgREST 返回的错误)
  ↓
TooljetDatabaseError (封装上下文信息)
  ↓
QueryError (插件标准错误格式)
```

#### TooljetDatabaseError

文件：[types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/types.ts#L133-L252)

**功能**：
- 保存错误上下文（origin、internalTables）
- 根据错误代码映射用户友好的错误消息
- 替换错误消息中的占位符（`{{table}}`, `{{column}}`, `{{value}}`）
- 替换 UUID 为表名
- 屏蔽 workspache schema 名称

**错误代码映射**：
| 错误代码 | 操作 | 消息模板 |
|---------|------|---------|
| 23502 (NotNullViolation) | proxy_postgrest | Not null constraint violated for {{table}}.{{column}} |
| 23505 (UniqueViolation) | proxy_postgrest | Unique constraint violated as {{value}} already exists in {{table}}.{{column}} |
| 42P01 (UndefinedTable) | default | Could not find the table {{table}}. |
| 23503 (ForeignKeyViolation) | proxy_postgrest | Update or delete on {{table}}.{{column}} with {{value}} violates foreign key constraint |
| 42501 (PermissionDenied) | default | Insufficient privilege |

#### PostgrestProxyService 中的错误处理

文件：[postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts#L162-L182)

```typescript
catch (error) {
  if (!isEmpty(error.response.rawBody) && ...) {
    const postgrestResponse = JSON.parse(error.response.rawBody...);
    const errorMessage = postgrestResponse.message;
    const errorContext = {
      origin: 'proxy_postgrest',
      internalTables: ... // 从 header 的 tableinfo 构建
    };
    const errorObj = new QueryFailedError(postgrestResponse, [], new PostgrestError(postgrestResponse));
    const tooljetDbError = new TooljetDatabaseError(errorMessage, errorContext, errorObj);
    throw new QueryError(tooljetDbError.toString(), { code: tooljetDbError.code }, {});
  }
  throw new QueryError('Query could not be completed', error.message, { message: error.message });
}
```

#### 异常过滤器

文件：[tooljetdb-exception-filter.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/filters/tooljetdb-exception-filter.ts)

```typescript
@Catch()
export class TooljetDbExceptionFilter implements ExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const next = ctx.getNext();
    if (exception instanceof TooljetDatabaseError) {
      next(exception);
    } else {
      // 处理验证错误等，简化错误消息
      if (Array.isArray(exception?.response?.message)) {
        // 只显示第一个错误
        exception.response.message = strippedErrorMessage;
      }
      next(exception);
    }
  }
}
```

---

## 相关文件索引

| 文件 | 说明 |
|------|------|
| [controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/controller.ts) | ToolJetDB NestJS 控制器 |
| [tooljet-db-data-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-data-operations.service.ts) | 数据操作服务（QueryService 实现） |
| [postgrest-proxy.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/postgrest-proxy.service.ts) | PostgREST 代理服务 |
| [tooljet-db-table-operations.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-table-operations.service.ts) | 表结构操作服务 |
| [tooljet-db-bulk-upload.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/services/tooljet-db-bulk-upload.service.ts) | 批量上传服务 |
| [types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/types.ts) | 类型定义和错误类 |
| [tooljetdb-exception-filter.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/tooljet-db/filters/tooljetdb-exception-filter.ts) | 异常过滤器 |
| [tooljetDatabase.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/tooljetDatabase.service.js) | 前端 service 层 |
| [plugin-selector.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/services/plugin-selector.service.ts) | 插件选择器服务 |
| [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts) | Data Query 工具服务 |
