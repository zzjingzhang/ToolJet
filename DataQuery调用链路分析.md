# DataQuery 调用链路分析：Builder 模式 vs Viewer 模式

## 概述

本文档详细追踪 ToolJet 中 DataQuery 在 Builder 模式（编辑态）和 Viewer 模式（查看态）下从 HTTP 路由入口到插件 `service.run` 的完整调用链路，并对比两者在权限、options 持久化、公共应用、模块回退等方面的差异。

---

## 一、Builder 模式完整链路（runQueryOnBuilder）

### 1. HTTP 路由入口

**路由**: `POST /data-queries/:id/versions/:versionId/run/:environmentId`

**控制器**: [DataQueriesController.runQueryOnBuilder](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/controller.ts#L117-L140)

**装饰器/守卫顺序**（执行顺序从上到下）：
- `@InitFeature(FEATURE_KEY.RUN_EDITOR)` - 功能初始化
- `@UseGuards(JwtAuthGuard)` - JWT 认证
- `@UseGuards(ValidateAppVersionGuard)` - 应用版本验证
- `@UseGuards(ValidateQueryAppGuard)` - 查询所属应用验证
- `@UseGuards(AppFeatureAbilityGuard)` - 应用级能力权限检查
- `@UseGuards(ValidateQuerySourceGuard)` - 数据源验证（附加 dataSource 到 request）
- `@UseGuards(DataSourceFeatureAbilityGuard)` - 数据源级能力权限检查

### 2. 关键守卫详解

#### (1) ValidateQueryAppGuard

[validate-query-app.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/validate-query-app.guard.ts)

- 通过 `id`（queryId）或 `versionId` 或 `appId` 查找关联的 App
- 必须有已认证用户（`user` 不能为空，否则抛 `ForbiddenException`）
- 找到 App 后挂载到 `request.tj_app`
- 用于后续权限验证

#### (2) ValidateQuerySourceGuard

[validate-query-source.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/validate-query-source.guard.ts)

- 通过 `id`（queryId）或 `dataSourceId` 查找 DataSource
- 必须有已认证用户（`user` 不能为空，否则抛 `ForbiddenException`）
- 找到 DataSource 后挂载到 `request.tj_data_source`
- **Builder 模式特有：在 Controller 层就完成了数据源的附加**

### 3. Service 层：runQueryOnBuilder

[service.ts#L253-L277](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts#L253-L277)

```typescript
async runQueryOnBuilder(
  user: User,
  dataQueryId: string,
  environmentId: string,
  updateDataQueryDto: UpdateDataQueryDto,
  ability: AppAbility,
  dataSource: DataSource,
  response: Response,
  mode?: string,
  app?: App
) {
  const { options, resolvedOptions } = updateDataQueryDto;

  const dataQuery = await this.dataQueryRepository.getOneById(dataQueryId, {
    dataSource: true,
    appVersion: true,
  });

  // 关键：有权限且有 options 时，持久化到数据库
  if (ability.can(FEATURE_KEY.UPDATE_ONE, DataSource, dataSource.id) && !isEmpty(options)) {
    await this.dataQueryRepository.updateOne(dataQueryId, { options });
    dataQuery['options'] = options;
  }

  return this.runAndGetResult(user, dataQuery, resolvedOptions, response, environmentId, mode, app);
}
```

**核心特点**：
- 接收 `environmentId` 参数（用于选择环境）
- 接收 `options` 和 `resolvedOptions`（未解析和已解析的选项）
- **有条件持久化 options**：检查用户是否有 `UPDATE_ONE` 权限，有则保存
- 传递 `mode` 参数（来自 query string）

### 4. 通用执行层：runAndGetResult

[service.ts#L300-L343](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts#L300-L343)

```typescript
protected async runAndGetResult(
  user: User,
  dataQuery: DataQuery,
  resolvedOptions: object,
  response: Response,
  environmentId?: string,
  mode?: string,
  app?: App
): Promise<object> {
  let result = {};
  try {
    result = await this.dataQueryUtilService.runQuery(
      user,
      dataQuery,
      resolvedOptions,
      response,
      environmentId,
      mode,
      app,
      undefined
    );
  } catch (error) {
    if (error.constructor.name === 'QueryError') {
      result = {
        status: 'failed',
        message: error?.message,
        description: error?.description,
        data: error?.data,
        metadata: error?.metadata,
      };
    } else {
      console.error(error);
      result = {
        status: 'failed',
        message: error?.message || 'Internal server error',
        description: error?.message,
        data: error?.data || {},
      };
    }
  }
  return result;
}
```

### 5. 工具服务层：runQuery

[util.service.ts#L66-L386](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L66-L386)

核心流程：
1. 从 dataQuery 中获取 dataSource 和 app
2. 懒加载 appVersion 用于 BRANCH 类型版本的 branchId 解析
3. 通过 `appEnvironmentUtilService.getOptions` 获取环境相关的数据源选项
4. 调用 `fetchServiceAndParsedParams` 获取插件服务实例和解析后的参数
5. 处理查询超时（AbortController）
6. 公共应用 + multi-auth 检查（公共应用不支持多认证）
7. 对于 restapi 类型，添加 X-Forwarded-For 和 cookie 转发
8. 调用 `service.run()` 执行插件查询
9. OAuth token 刷新逻辑（如果遇到 `OAuthUnauthorizedClientError`）
10. 审计日志记录

### 6. 插件服务获取与参数解析：fetchServiceAndParsedParams

[util.service.ts#L432-L459](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L432-L459)

```typescript
async fetchServiceAndParsedParams(
  dataSource,
  dataQuery,
  queryOptions,
  organization_id,
  environmentId = undefined,
  user = undefined,
  opts?: DataQueryExecutionOptions
) {
  const sourceOptions = await this.dataSourceUtilService.parseSourceOptions(
    dataSource.options,
    organization_id,
    environmentId,
    user
  );

  const parsedQueryOptions = await this.parseQueryOptions(
    dataQuery.options,
    queryOptions,
    organization_id,
    environmentId,
    user,
    opts
  );
  const service = await this.pluginsSelectorService.getService(dataSource.pluginId, dataSource.kind);

  return { service, sourceOptions, parsedQueryOptions };
}
```

### 7. PluginSelector 插件选择器

[plugin-selector.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/services/plugin-selector.service.ts)

`PluginsServiceSelector.getService(pluginId, kind)` 逻辑：

| 类型 | 判断条件 | 返回 |
|------|----------|------|
| ToolJet Database | `kind === 'tooljetdb'` | `tooljetDbDataOperationsService`（单例） |
| Marketplace 插件 | `!!pluginId` | 从数据库加载并通过 `vm.runInNewContext` 沙箱执行后实例化 |
| 内置插件 | 其他 | `new allPlugins[kind]()`（直接 new 实例） |

**注意**：每次调用 `getService` 都会新建一个插件服务实例（除了 tooljetdb 是单例服务）。

### 8. 插件执行：service.run

以 `restapi` 插件为例：[restapi/lib/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/restapi/lib/index.ts#L48-L91)

```typescript
async run(
  sourceOptions: any,
  queryOptions: any,
  dataSourceId: string,
  dataSourceUpdatedAt: string,
  context?: { user?: User; app?: App }
): Promise<RestAPIResult> {
  // 1. 构造 URL
  // 2. SSRF 防护验证
  // 3. 构造请求选项（headers, method, body 等）
  // 4. 验证并设置认证类型
  // 5. 发送请求
  // 6. 返回 { status: 'ok', data: result, metadata: {...} }
  // 失败时抛出 QueryError 或 OAuthUnauthorizedClientError
}
```

---

## 二、Viewer 模式完整链路（runQueryForApp）

### 1. HTTP 路由入口

**路由**: `POST /data-queries/:id/run`

**控制器**: [DataQueriesController.runQuery](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/controller.ts#L142-L153)

**装饰器/守卫顺序**：
- `@InitFeature(FEATURE_KEY.RUN_VIEWER)` - 功能初始化
- `@UseGuards(QueryAuthGuard)` - 查询认证（支持公共应用）
- `@UseGuards(AppFeatureAbilityGuard)` - 应用级能力权限检查

### 2. 关键守卫：QueryAuthGuard

[query-auth.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/query-auth.guard.ts)

这是 Viewer 模式特有的认证守卫，核心逻辑：

```typescript
async canActivate(context: ExecutionContext): Promise<any> {
  const request = context.switchToHttp().getRequest();
  const { id } = request.params;  // dataQueryId

  // 1. 通过 dataQueryId 查找所属 app
  const app = await this.appRepository.findByDataQuery(id);
  
  // 2. 检查组织状态
  const organization = await this.organizationRepository.getSingleOrganizationWithId(app?.organizationId);
  
  // 3. 挂载 app 到 request.tj_app
  request.tj_app = app;

  // 4. 如果是公共应用，直接放行（无需用户认证）
  if (app.isPublic === true) {
    this.organizationRepository.touchLastAccessedAt(app.organizationId);
    return true;
  }

  // 5. 如果是模块（Module）类型应用
  if (app.type === APP_TYPES.MODULE) {
    try {
      // 先尝试 JWT 认证
      return await super.canActivate(context);
    } catch {
      // JWT 失败时，检查该模块是否被公共父应用引用
      const parentApp = await this.dataQueryRepository.findPublicParentAppForModuleQuery(app.id, id);
      if (parentApp) {
        app.isPublic = true;  // 标记为公共
        return true;
      }
      throw new UnauthorizedException(...);
    }
  }

  // 6. 普通应用：必须 JWT 认证通过
  try {
    return await super.canActivate(context);
  } catch {
    throw new UnauthorizedException(...);
  }
}
```

### 3. 模块父应用回退查询

[repository.ts#L42-L63](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/repository.ts#L42-L63)

`findPublicParentAppForModuleQuery(moduleAppId, dataQueryId)` 通过复杂的多表关联查询：

- `App` → `AppVersion` → `Page` → `Component`（ModuleViewer 类型）
- 关联 `DataQuery` 和模块的 `AppVersion`、`App`
- 条件：
  - 组件类型为 `ModuleViewer`
  - 父应用 `is_public = true`
  - 模块版本 ID 匹配
  - 组件属性中的 `moduleAppId` 和 `moduleVersionId` 匹配

**作用**：当模块本身不是公共应用时，如果它被一个公共的父应用通过 ModuleViewer 组件引用，那么该查询也可以被匿名访问。

### 4. Service 层：runQueryForApp

[service.ts#L279-L294](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts#L279-L294)

```typescript
async runQueryForApp(
  user: User,
  dataQueryId: string,
  updateDataQueryDto: UpdateDataQueryDto,
  response: Response,
  app?: App
) {
  const { resolvedOptions } = updateDataQueryDto;

  const dataQuery = await this.dataQueryRepository.getOneById(dataQueryId, {
    dataSource: true,
    appVersion: true,
  });

  return this.runAndGetResult(user, dataQuery, resolvedOptions, response, undefined, 'view', app);
}
```

**核心特点**：
- **没有** `environmentId` 参数（使用默认环境）
- **只接收** `resolvedOptions`（不传 `options`）
- **不持久化 options**：Viewer 模式下运行查询不会保存配置
- `mode` 硬编码为 `'view'`

### 5. 后续链路

后续的 `runAndGetResult` → `dataQueryUtilService.runQuery` → `fetchServiceAndParsedParams` → `service.run` 流程与 Builder 模式完全一致。

---

## 三、详细差异对比

### 1. 权限/认证对比

| 维度 | Builder 模式 (runQueryOnBuilder) | Viewer 模式 (runQueryForApp) |
|------|----------------------------------|-------------------------------|
| **认证守卫** | `JwtAuthGuard` - 强制 JWT 认证 | `QueryAuthGuard` - 支持公共应用匿名访问 |
| **应用验证** | `ValidateQueryAppGuard` - 通过 versionId/queryId 查找，必须有用户 | QueryAuthGuard 内部完成 - 通过 queryId 查找 |
| **数据源验证** | `ValidateQuerySourceGuard` - 附加 dataSource 到 request | **无** - 不在 guard 层验证数据源 |
| **能力守卫** | `AppFeatureAbilityGuard` + `DataSourceFeatureAbilityGuard`（双重） | `AppFeatureAbilityGuard`（仅应用级） |
| **Feature Key** | `RUN_EDITOR` | `RUN_VIEWER` |
| **用户要求** | 必须有已认证用户 | 公共应用可无用户；非公共应用必须认证 |
| **审计日志** | `shouldNotSkipPublicApp: true`（公共应用也记录） | 无此标记 |

**关键代码引用**：
- [constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/constants/feature.ts)
- [controller.ts#L109-L140](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/controller.ts#L109-L140)（Builder 守卫）
- [controller.ts#L142-L153](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/controller.ts#L142-L153)（Viewer 守卫）

### 2. Options 持久化对比

| 维度 | Builder 模式 | Viewer 模式 |
|------|-------------|------------|
| **是否保存 options** | ✅ 有条件保存 | ❌ 不保存 |
| **保存条件** | 1. 用户有 `UPDATE_ONE` 权限<br>2. `options` 不为空 | - |
| **保存方法** | `dataQueryRepository.updateOne(dataQueryId, { options })` | - |
| **传入参数** | `options` + `resolvedOptions` | 仅 `resolvedOptions` |
| **对数据库的影响** | 每次运行（有权限时）都会更新 query.options | 只读，不修改数据库 |

**关键代码引用**：
- [service.ts#L271-L274](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts#L271-L274)（Builder 保存逻辑）
- [service.ts#L286-L286](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts#L286-L286)（Viewer 只取 resolvedOptions）

### 3. 公共应用（Public App）支持对比

| 维度 | Builder 模式 | Viewer 模式 |
|------|-------------|------------|
| **是否支持公共应用访问** | ❌ 不支持（必须 JWT 认证） | ✅ 支持 |
| **认证方式** | 强制 JWT | 公共应用跳过认证 |
| **多认证兼容** | 不涉及 | 公共应用不支持 multi-auth（会抛 QueryError） |
| **用户信息** | 总有 user | 公共应用时 user 可能为 undefined |
| **组织 ID 获取** | `user.organizationId` | 优先 `user.organizationId`，否则 `app.organizationId` |

**公共应用 + 多认证检查**（在 `runQuery` 中）：
```typescript
// multi-auth will not work with public apps
if (appToUse?.isPublic && sourceOptions['multiple_auth_enabled']) {
  throw new QueryError(
    'Authentication required for all users should be turned off since the app is public',
    '',
    {}
  );
}
```

**关键代码引用**：
- [query-auth.guard.ts#L46-L50](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/query-auth.guard.ts#L46-L50)（公共应用放行）
- [util.service.ts#L144-L150](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L144-L150)（公共应用 + 多认证检查）

### 4. 模块内 ModuleViewer 父应用回退

| 维度 | Builder 模式 | Viewer 模式 |
|------|-------------|------------|
| **是否支持父应用回退** | ❌ 不支持 | ✅ 支持 |
| **回退逻辑位置** | - | `QueryAuthGuard` 中 |
| **触发条件** | - | 模块类型应用 + JWT 认证失败 |
| **查询方法** | - | `dataQueryRepository.findPublicParentAppForModuleQuery()` |
| **效果** | - | 如果被公共父应用引用，则标记模块为公共并放行 |

**回退逻辑详解**：
1. 首先尝试 JWT 认证
2. 若失败，查询是否有公共的父应用通过 ModuleViewer 组件引用了此模块的此查询
3. 若有，则设置 `app.isPublic = true` 并放行
4. 若无，抛出 `UnauthorizedException`

**关键代码引用**：
- [query-auth.guard.ts#L52-L70](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/query-auth.guard.ts#L52-L70)（模块回退逻辑）
- [repository.ts#L42-L63](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/repository.ts#L42-L63)（父应用查询 SQL）

### 5. ValidateQuerySourceGuard 附加数据源

| 维度 | Builder 模式 | Viewer 模式 |
|------|-------------|------------|
| **是否使用 ValidateQuerySourceGuard** | ✅ 使用 | ❌ 不使用 |
| **数据源附加时机** | Guard 阶段（Controller 执行前） | 运行时按需加载 |
| **数据源来源** | 通过 queryId 查询并验证组织权限 | 从 dataQuery.dataSource 关联获取 |
| **权限校验** | Guard 中检查 `user.organizationId` | 无额外权限校验 |
| **对 Controller 的影响** | 可通过 `@DataSource()` 装饰器直接注入 | 不注入 dataSource 参数 |

**Builder 模式 Controller 签名**（有 dataSource 参数）：
```typescript
runQueryOnBuilder(
  @User() user: UserEntity,
  @AppDecorator() app: App,
  @Param('id') dataQueryId,
  @Param('environmentId') environmentId,
  @Body() updateDataQueryDto: UpdateDataQueryDto,
  @Ability() ability: AppAbility,
  @DataSource() dataSource: DataSourceEntity,  // <-- 有这个
  @Res({ passthrough: true }) response: Response,
  @Query('mode') mode?: string
)
```

**Viewer 模式 Controller 签名**（无 dataSource 参数）：
```typescript
async runQuery(
  @User() user: UserEntity,
  @AppDecorator() app: App,
  @Param('id') dataQueryId,
  @Body() updateDataQueryDto: UpdateDataQueryDto,
  @Res({ passthrough: true }) response: Response
)
```

**关键代码引用**：
- [validate-query-source.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/validate-query-source.guard.ts)
- [data-source.decorator.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/decorators/data-source.decorator.ts)

### 6. PluginSelector 实例化对比

两种模式在插件实例化方面**完全相同**，都通过同一个 `PluginsServiceSelector.getService()` 方法获取实例。

但存在以下间接差异：

| 维度 | Builder 模式 | Viewer 模式 |
|------|-------------|------------|
| **实例化路径** | 相同 | 相同 |
| **调用次数** | 每次请求新建实例 | 每次请求新建实例 |
| **环境变量影响** | 可通过 `environmentId` 指定环境 | 使用默认环境 |
| **数据源选项** | 根据指定环境解析 | 根据默认环境解析 |

**PluginSelector 实例化逻辑**：
- `tooljetdb` → 返回单例服务（`TooljetDbDataOperationsService`）
- MarketPlace 插件（有 `pluginId`）→ 从数据库加载代码 → `vm.runInNewContext` 沙箱执行 → `new` 实例
- 内置插件（无 `pluginId`）→ 直接 `new allPlugins[kind]()`

**关键代码引用**：
- [plugin-selector.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/services/plugin-selector.service.ts)

### 7. 环境（Environment）处理对比

| 维度 | Builder 模式 | Viewer 模式 |
|------|-------------|------------|
| **environmentId 参数** | ✅ URL 路径参数 `:environmentId` | ❌ 无此参数 |
| **环境选择** | 用户指定环境 | 使用默认/当前环境 |
| **分支版本支持** | 支持（通过 appVersion 推导 branchId） | 支持（通过 appVersion 推导 branchId） |
| **数据源选项获取** | `getOptions(dataSourceId, orgId, envId, branchId)` | `getOptions(dataSourceId, orgId, undefined, branchId)` |

---

## 四、错误处理与 failed 状态包装

### 1. 错误包装层级

错误在以下层级被捕获并包装成 `{ status: 'failed', ... }` 格式返回：

#### 第一层：service.runAndGetResult

[service.ts#L311-L340](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts#L311-L340)

```typescript
try {
  result = await this.dataQueryUtilService.runQuery(...);
} catch (error) {
  if (error.constructor.name === 'QueryError') {
    result = {
      status: 'failed',
      message: error?.message,
      description: error?.description,
      data: error?.data,
      metadata: error?.metadata,  // QueryError 特有
    };
  } else {
    console.error(error);
    result = {
      status: 'failed',
      message: error?.message || 'Internal server error',
      description: error?.message,
      data: error?.data || {},
    };
  }
}
```

#### 第二层：listTablesForApp 也有相同模式

[service.ts#L353-L373](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts#L353-L373)

### 2. 会被包装成 failed 状态的错误类型

#### (1) QueryError（插件抛出的查询错误）

**来源**：插件内部 `throw new QueryError(...)`

**包含字段**：
- `message`: 错误消息
- `description`: 错误描述
- `data`: 错误数据（如 requestObject, responseObject 等）
- `metadata`: 元数据（QueryError 特有，如 request/response 详情）

**常见触发场景**：
- REST API 请求失败（非 2xx 状态码）
- 数据库查询错误
- 认证失败（非 OAuth 相关）
- SSRF 防护拦截
- 插件执行中的业务错误

**关键代码引用**：
- [query.error.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/query.error.ts)

#### (2) 其他所有错误（非 QueryError）

**包含字段**：
- `message`: `error.message || 'Internal server error'`
- `description`: `error.message`
- `data`: `error.data || {}`

**额外行为**：
- 控制台打印 `console.error(error)`
- 作为"内部服务器错误"处理

**常见触发场景**：
- 数据库连接错误
- 代码运行时异常
- 空指针/未定义属性访问
- 类型错误
- 网络超时等

### 3. 特殊错误处理（不包装成 failed，直接返回特殊状态）

以下错误**不会**被包装成 `failed`，而是返回特殊状态码：

#### (1) OAuthUnauthorizedClientError → `needs_oauth`

在 `dataQueryUtilService.runQuery` 内部处理，不向外抛出 failed：

```typescript
if (api_error.constructor.name === 'OAuthUnauthorizedClientError') {
  // 尝试 refresh token
  // 如果刷新成功，重新执行查询
  // 如果刷新失败或无需刷新，返回 needs_oauth 状态
  return {
    status: 'needs_oauth',
    data: {
      auth_url: result.url,
      // 或包含 kind, options 等信息
    },
  };
}
```

**返回状态**: `{ status: 'needs_oauth', data: {...} }`

**关键代码引用**：
- [util.service.ts#L206-L337](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L206-L337)

### 4. Guard 层抛出的错误（不会被包装成 failed）

以下错误在 Guard 层抛出，会直接作为 HTTP 错误响应（4xx/5xx 状态码），**不会**被包装成 `{ status: 'failed' }` 格式：

| 错误类型 | 状态码 | 触发场景 |
|----------|--------|----------|
| `BadRequestException` | 400 | 缺少必要参数、组织已归档 |
| `ForbiddenException` | 403 | 用户未认证（Builder 模式） |
| `NotFoundException` | 404 | 应用/数据源不存在 |
| `UnauthorizedException` | 401 | JWT 认证失败且非公共应用 |
| `ConflictException` | 409 | 查询名称重复（创建/更新时） |

### 5. 错误处理完整流程图

```
HTTP 请求
   │
   ▼
Guard 层校验
   │
   ├─ 校验失败 → 抛出 HTTP 异常（4xx/5xx）→ 直接返回，不包装成 failed
   │
   ▼
Controller → Service
   │
   ▼
runAndGetResult
   │
   ├─ try 块
   │    │
   │    ▼
   │  dataQueryUtilService.runQuery
   │    │
   │    ├─ 插件 service.run 成功 → 返回 { status: 'ok', data: ... }
   │    │
   │    ├─ 插件抛出 QueryError → 被 catch → 包装成 { status: 'failed', ... }
   │    │
   │    ├─ 插件抛出 OAuthUnauthorizedClientError
   │    │    │
   │    │    ├─ 可刷新 token → 刷新后重试 → 成功返回 ok
   │    │    └─ 不可刷新/刷新失败 → 返回 { status: 'needs_oauth', ... }
   │    │
   │    └─ 其他运行时错误 → 抛出 → 被外层 catch
   │
   └─ catch 块（runAndGetResult 中）
        │
        ├─ QueryError → { status: 'failed', message, description, data, metadata }
        │
        └─ 其他错误 → { status: 'failed', message: 'Internal server error', ... }
```

---

## 五、完整链路对比总表

| 阶段 | Builder 模式 (runQueryOnBuilder) | Viewer 模式 (runQueryForApp) |
|------|--------------------------------|------------------------------|
| **HTTP 路由** | `POST /data-queries/:id/versions/:versionId/run/:environmentId` | `POST /data-queries/:id/run` |
| **Feature Key** | `RUN_EDITOR` | `RUN_VIEWER` |
| **认证守卫** | JwtAuthGuard（强制认证） | QueryAuthGuard（支持公共应用） |
| **应用验证** | ValidateQueryAppGuard | QueryAuthGuard 内部完成 |
| **数据源验证** | ValidateQuerySourceGuard | 无 |
| **能力守卫** | App + DataSource（双重） | 仅 App |
| **公共应用支持** | ❌ 不支持 | ✅ 支持 |
| **模块父应用回退** | ❌ 不支持 | ✅ 支持 |
| **环境选择** | 通过 environmentId 指定 | 默认环境 |
| **Options 持久化** | 有权限时保存 options | 不保存 |
| **传入 options** | options + resolvedOptions | 仅 resolvedOptions |
| **mode 参数** | 来自 query string | 硬编码 'view' |
| **插件实例化** | 每次新建实例 | 每次新建实例 |
| **错误包装** | runAndGetResult 中包装 | runAndGetResult 中包装 |

---

## 六、关键文件索引

| 文件路径 | 作用 |
|----------|------|
| [data-queries/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/controller.ts) | 控制器：路由定义和守卫配置 |
| [data-queries/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts) | 服务层：runQueryOnBuilder / runQueryForApp / runAndGetResult |
| [data-queries/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts) | 工具服务：runQuery 核心执行逻辑 |
| [data-queries/guards/query-auth.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/query-auth.guard.ts) | Viewer 模式认证守卫（含公共应用和模块回退） |
| [data-queries/guards/validate-query-app.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/validate-query-app.guard.ts) | Builder 模式应用验证守卫 |
| [data-queries/guards/validate-query-source.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/guards/validate-query-source.guard.ts) | 数据源验证守卫（Builder 模式使用） |
| [data-queries/repository.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/repository.ts) | 数据访问层（含父应用查询） |
| [data-sources/services/plugin-selector.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/services/plugin-selector.service.ts) | 插件选择器：获取插件服务实例 |
| [plugins/packages/common/lib/query.error.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/query.error.ts) | QueryError 定义 |
| [plugins/packages/restapi/lib/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/restapi/lib/index.ts) | 示例插件：restapi 的 run 方法 |
