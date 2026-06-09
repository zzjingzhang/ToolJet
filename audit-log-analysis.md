# ToolJet 审计日志机制分析

## 一、整体流程概述

审计日志的产生遵循 **"业务服务写入 → RequestContext/response.locals 暂存 → ResponseInterceptor 转换 → auditLogEntry 事件发射"** 的链路。

```
┌──────────────────┐     ┌─────────────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│   业务服务层      │────▶│  RequestContext /        │────▶│  ResponseInterceptor │────▶│ auditLogEntry   │
│ (Auth/DataSource │     │  response.locals 暂存    │     │  提取并补充字段      │     │  事件发射        │
│  /DataQuery/...) │     │  (tj_audit_logs_meta_data)│     │                     │     │  (fire-and-forget)│
└──────────────────┘     └─────────────────────────┘     └─────────────────────┘     └──────────────────┘
```

## 二、核心组件与文件

| 组件 | 文件路径 | 作用 |
|------|---------|------|
| 请求上下文中间件 | [request-context/middleware.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/request-context/middleware.ts) | 创建请求上下文，生成 transactionId、route、start_time |
| 请求上下文服务 | [request-context/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/request-context/service.ts) | 提供 `setLocals` 等静态方法操作 response.locals |
| 响应拦截器 | [response.interceptor.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/interceptors/response.interceptor.ts) | 响应返回前读取 locals 并发射 auditLogEntry 事件 |
| 常量定义 | [app/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/index.ts) | 定义 `AUDIT_LOGS_REQUEST_CONTEXT_KEY = 'tj_audit_logs_meta_data'` |
| 模块信息配置 | [module-info.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/module-info.ts) | 各模块的 feature 配置（含 auditLogsKey、skipAuditLogs 等） |
| 审计日志类型 | [audit-logs/types/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/audit-logs/types/index.ts) | `AuditLogFields` 接口定义 |
| 审计日志实体 | [audit_log.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/audit_log.entity.ts) | 数据库表结构定义 |

## 三、业务服务写入审计日志

业务服务通过 `RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, data)` 将审计日志数据写入 `response.locals.tj_audit_logs_meta_data`。

### 3.1 登录（Auth Module）

**文件**：[auth/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/service.ts)

**涉及方法**：
- `login()` - 普通密码登录（行 162-170）
- `resetPassword()` - 重置密码（行 233-239）
- `forgotPassword()` - 忘记密码（行 251-257）
- `superAdminLogin()` - 超级管理员登录
- OAuth 系列登录方法

**写入数据示例（密码登录）**：
```typescript
RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, {
  userId: user.id,
  organizationId: organization.id,
  resourceId: user.id,
  resourceName: user.email,
  resourceData: {
    auth_method: 'password',
  },
});
```

**Feature 配置**（[auth/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/constants/feature.ts)）：
- `LOGIN`: `auditLogsKey: 'USER_LOGIN'`
- `FORGOT_PASSWORD`: `auditLogsKey: 'USER_PASSWORD_FORGOT'`
- `RESET_PASSWORD`: `auditLogsKey: 'USER_PASSWORD_RESET'`
- `OAUTH_SIGN_IN`: `auditLogsKey: 'USER_LOGIN'`

### 3.2 DataQuery 运行

**文件**：[data-queries/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts)（行 372-383）

**写入数据示例**：
```typescript
const auditData = {
  userId: user.id,
  organizationId: user.organizationId,
  resourceId: dataQuery?.id,
  resourceName: dataQuery?.name,
  metadata: enrichedMetadata,  // 包含 appId、appName、dataSourceType 等
  resourceData: {
    dataSourceId: dataSource?.id,
    dataSourceName: dataSource?.name,
  },
};
RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, auditData);
```

**Feature 配置**（[data-queries/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/constants/feature.ts)）：
- `RUN_EDITOR`: `auditLogsKey: 'DATA_QUERY_RUN'`
- `RUN_VIEWER`: `auditLogsKey: 'DATA_QUERY_RUN'`
- `PREVIEW`: `auditLogsKey: 'DATA_QUERY_RUN'`

### 3.3 DataSource 创建/更新/删除

**文件**：[data-sources/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts)

#### 创建（行 144-160）
```typescript
RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, {
  userId: user.id,
  organizationId: user.organizationId,
  resourceId: dataSource?.id,
  resourceName: dataSource?.name,
  resourceData: {
    dataSourceKind: dataSource?.kind,
    dataSourceScope: dataSource?.scope,
    appId: dataSource?.app?.id || null,
    appVersionId: dataSource?.appVersionId,
    environmentId: environment_id,
    pluginId: pluginId,
  },
  metadata: {
    createdAt: dataSource?.createdAt,
  },
});
```

#### 更新（行 183-197）
```typescript
RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, {
  userId: user.id,
  organizationId: user.organizationId,
  resourceId: dataSourceId,
  resourceName: name,
  resourceData: {
    dataSourceKind: dataSource?.kind,
    dataSourceScope: dataSource?.scope,
    appId: dataSource?.app?.id || null,
    appVersionId: dataSource?.appVersionId,
    environmentId: environmentId,
    updatedFields: Object.keys(updateDataSourceDto),
  },
  metadata: updateDataSourceDto,
});
```

#### 删除（行 235-249）
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

**Feature 配置**（[data-sources/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/constants/feature.ts)）：
- `CREATE`: `auditLogsKey: 'DATA_SOURCE_CREATE'`
- `UPDATE`: `auditLogsKey: 'DATA_SOURCE_UPDATE'`
- `DELETE`: `auditLogsKey: 'DATA_SOURCE_DELETE'`

### 3.4 Session 终止（登出）

**文件**：[session/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts)（行 58-64）

**写入数据**：
```typescript
const auditLogData = {
  userId: user.id,
  organizationId: user.organizationId,
  resourceId: user.id,
  resourceName: user.email,
};
RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, auditLogData);
```

**Feature 配置**（[session/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/constants/feature.ts)）：
- `LOG_OUT`: `auditLogsKey: 'USER_LOGOUT'`

## 四、RequestContext 与 response.locals

### 4.1 RequestContextMiddleware

在 [middleware.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/request-context/middleware.ts) 中，每个请求进入时：

1. 使用 `AsyncLocalStorage` 创建请求上下文
2. 生成 15 位随机 `transactionId`
3. 构造 `[METHOD] originalUrl` 格式的 route
4. 记录请求开始时间 `tj_start_time`
5. 将这些信息存入 `response.locals`

### 4.2 RequestContext.setLocals

[service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/request-context/service.ts) 中的静态方法：

```typescript
static setLocals(key: string, data: any) {
  const context = this.currentContext;
  if (!context) {
    console.error('RequestContext is not set');
    return;
  }
  if (!context.res.locals) {
    context.res.locals = {};
  }
  context.res.locals[key] = data;
}
```

业务服务通过该方法将审计数据写入 `response.locals.tj_audit_logs_meta_data`。

## 五、ResponseInterceptor 转换逻辑

[response.interceptor.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/interceptors/response.interceptor.ts) 是审计日志产生的核心转换点。

### 5.1 处理流程

```
1. 从 response.locals 获取审计数据 (logsData)
2. 通过 Reflector 获取 module 和 features (来自 @InitFeature 装饰器)
3. 前置检查（跳过条件判定）
4. 从 MODULE_INFO 获取 featureInfo
5. 补充 transaction metadata
6. 检查请求是否成功
7. 获取 IP 和 User-Agent
8. 发射 auditLogEntry 事件
```

### 5.2 字段补充逻辑

#### resourceType
从类级别的 `tjModuleId` 装饰器元数据获取，对应 `MODULES` 枚举值。

#### actionType
优先级从高到低：
1. `logsData.actionType` - 业务服务显式设置
2. `featureInfo.auditLogsKey` - Feature 配置中定义
3. `features[0]` - Feature ID 本身

#### ipAddress
```typescript
const clientIp = (request as any)?.clientIp;
ipAddress: clientIp || (request && requestIp.getClientIp(request))
```

#### userAgent
```typescript
userAgent: request?.headers['user-agent']
```

#### transaction metadata
```typescript
logsData.metadata = {
  ...logsData.metadata,
  transactionId: response.locals.tj_transactionId,  // 15位随机数
  totalDuration: timeTaken,                          // 请求耗时(ms)
  route: response.locals.tj_route,                   // [METHOD] URL 格式
};
```

## 六、Feature Metadata 说明

`FeatureConfig` 接口定义在 [app/types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/types.ts)（行 17-25）：

```typescript
export interface FeatureConfig {
  license?: LICENSE_FIELD;        // 许可证要求
  auditLogsKey?: string;          // 审计日志 actionType 键名
  skipAuditLogs?: boolean;        // 是否跳过审计日志
  isPublic?: boolean;             // 是否为公开接口
  isSuperAdminFeature?: boolean;  // 是否为超级管理员功能
  shouldNotSkipPublicApp?: boolean;
  allowFailedAuditLogs?: boolean; // 是否允许失败请求也记录审计日志
}
```

### 6.1 各模块 Feature Metadata 示例

| 模块 | Feature | auditLogsKey | skipAuditLogs | allowFailedAuditLogs |
|------|---------|--------------|---------------|---------------------|
| AUTH | LOGIN | USER_LOGIN | - | - |
| AUTH | FORGOT_PASSWORD | USER_PASSWORD_FORGOT | - | - |
| SESSION | LOG_OUT | USER_LOGOUT | - | - |
| DATA_SOURCE | CREATE | DATA_SOURCE_CREATE | - | - |
| DATA_SOURCE | UPDATE | DATA_SOURCE_UPDATE | - | - |
| DATA_SOURCE | DELETE | DATA_SOURCE_DELETE | - | - |
| DATA_QUERY | RUN_EDITOR | DATA_QUERY_RUN | - | - |
| DATA_QUERY | RUN_VIEWER | DATA_QUERY_RUN | - | - |

## 七、跳过审计日志的条件

在 `ResponseInterceptor.intercept()` 中，以下任一条件满足则跳过日志记录：

1. **response 对象不存在**（行 30-32）
   ```typescript
   if (!response) { return; }
   ```

2. **features 不存在或为空**（行 42-44）
   ```typescript
   if (!features || features.length === 0) { return; }
   ```

3. **featureInfo 不存在**（行 47）
   ```typescript
   if (!featureInfo || ...) { return; }
   ```

4. **featureInfo.skipAuditLogs 为 true**（行 47）
   ```typescript
   if (... || featureInfo?.skipAuditLogs || ...) { return; }
   ```

5. **logsData 不存在**（行 47）
   ```typescript
   if (... || !logsData || ...) { return; }
   ```

6. **logsData.userId 不存在**（行 47）
   ```typescript
   if (... || !logsData?.userId) { return; }
   ```

7. **请求失败且不允许失败审计**（行 68、70）
   ```typescript
   const isSuccess = response.statusCode >= 200 && response.statusCode < 300;
   if (isSuccess || featureInfo.allowFailedAuditLogs) {
     // 发射事件
   }
   ```

## 八、Fire-and-Forget 模式与可观测性边界

### 8.1 Fire-and-Forget 模式

`ResponseInterceptor` 通过 `EventEmitter2.emit()` 发射 `auditLogEntry` 事件（行 74-80）：

```typescript
try {
  const clientIp = (request as any)?.clientIp;
  this.eventEmitter.emit('auditLogEntry', {
    ...logsData,
    ipAddress: clientIp || (request && requestIp.getClientIp(request)),
    userAgent: request?.headers['user-agent'],
    resourceType: module,
    actionType: logsData?.actionType || featureInfo?.auditLogsKey || features[0],
  });
} catch (error) {
  this.logger.error('Failed to create audit log:', error);
}
```

这是一个 **fire-and-forget** 模式，特点是：
- 事件发射是同步调用，但监听器的执行可能是异步的
- 主请求响应不会等待审计日志处理完成
- 事件处理失败不会影响主请求的成功响应

### 8.2 可观测性边界

| 边界类型 | 说明 |
|---------|------|
| **请求-响应边界** | 审计日志的处理结果不体现在 API 响应中，调用方无法通过响应确认日志是否成功记录 |
| **异常静默边界** | 事件发射被 try-catch 包裹，失败仅记录 logger.error，不会抛出异常或中断主流程 |
| **时序边界** | 审计日志写入（事件消费）发生在响应返回之后，存在时间差 |
| **数据一致性边界** | 业务操作成功但审计日志丢失时，没有回滚或补偿机制 |
| **监控盲区** | 若事件监听器异常或未注册，审计日志会静默丢失，需要独立监控告警 |

### 8.3 潜在风险

1. **审计日志丢失**：事件监听器故障或性能问题可能导致审计日志丢失，且不易被发现
2. **顺序不一致**：高并发下审计日志的写入顺序可能与请求顺序不一致
3. **数据不完整**：如果事件消费端处理逻辑有 bug，可能导致部分字段丢失
4. **难以排查**：审计日志的生成链路跨越多个层级，问题排查需要追踪事件传递路径

## 九、AuditLog 数据库实体结构

[audit_log.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/audit_log.entity.ts)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | uuid | 主键 |
| user_id | string | 用户 ID |
| organization_id | string | 组织/工作区 ID |
| resource_id | string | 资源 ID |
| resource_name | string | 资源名称 |
| resource_type | enum(MODULES) | 资源类型 |
| resource_data | simple-json | 资源详情数据 |
| action_type | string | 操作类型 |
| ip_address | string | IP 地址 |
| metadata | simple-json | 元数据（含 transactionId、totalDuration 等） |
| created_at | timestamp | 创建时间 |

## 十、总结

ToolJet 的审计日志系统采用 **"请求上下文暂存 + 拦截器统一转换 + 事件驱动解耦"** 的架构设计：

- **优点**：业务代码与审计日志代码解耦，不影响主请求性能，扩展性好
- **关键点**：`response.locals.tj_audit_logs_meta_data` 是数据传递的桥梁
- **跳过条件**：7 种条件会跳过审计日志，其中 `skipAuditLogs` 和 `userId` 检查是关键门控
- **可观测性挑战**：fire-and-forget 模式带来了审计日志可能静默丢失的风险，需要配套监控机制
