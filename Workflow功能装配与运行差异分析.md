# Workflow功能装配与运行差异分析

## 1. AppModule何时导入WorkflowsModule

### 1.1 条件导入逻辑

WorkflowsModule 的导入由 **Cloud/非Cloud** 版本决定，在 [AppModule.register()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/module.ts#L93-L208) 方法中通过条件判断：

```typescript
if (getTooljetEdition() !== TOOLJET_EDITIONS.Cloud) {
  conditionalImports.push(await WorkflowsModule.register(configs, true));
  conditionalImports.push(
    BullBoardModule.forRoot({
      route: '/jobs',
      adapter: ExpressAdapter,
      middleware: basicAuth({
        challenge: true,
        users: { admin: process.env.TOOLJET_QUEUE_DASH_PASSWORD },
      }),
    })
  );
}
```

### 1.2 版本判断机制

版本判断通过 [getTooljetEdition()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/utils.helper.ts#L666-L671) 函数实现：

- 优先读取环境变量 `TOOLJET_EDITION`
- 其次读取环境变量配置文件中的 `TOOLJET_EDITION`
- 默认值为 `'ce'`

### 1.3 导入矩阵

| 版本 | WorkflowsModule | BullBoard (根) |
|------|-----------------|----------------|
| CE   | ✅ 导入         | ✅ 导入        |
| EE   | ✅ 导入         | ✅ 导入        |
| Cloud | ❌ 不导入      | ❌ 不导入      |

**注意**：Cloud 版本完全禁用了 WorkflowsModule，同时也不注册 BullBoard 根模块。

---

## 2. WorkflowsModule如何通过SubModule.getProviders加载实现

### 2.1 SubModule.getProviders机制

[SubModule](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/sub-module.ts#L3-L19) 是一个抽象基类，提供了动态加载 provider 的能力：

```typescript
export abstract class SubModule {
  protected static async getProviders(
    configs: { IS_GET_CONTEXT: boolean },
    module: string,
    paths: string[]
  ): Promise<any> {
    const importPath = await getImportPath(configs.IS_GET_CONTEXT);
    const providers = {};
    for (const path of paths) {
      const fullPath = `${importPath}/${module}/${path}`;
      const imported = await import(fullPath);
      Object.assign(providers, imported);
    }
    return providers;
  }
}
```

### 2.2 getImportPath路径切换

[getImportPath()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/index.ts#L11-L35) 根据版本动态决定模块加载路径：

| 版本 | 基础路径 | 说明 |
|------|----------|------|
| CE | `dist/src/modules` | 从开源代码目录加载 |
| EE | `dist/ee` | 从企业版目录加载 |
| Cloud | `dist/ee` | 与 EE 使用相同目录 |

**关键点**：CE 版本加载的是 `src/modules/workflows/` 下的 stub 实现，EE/Cloud 版本加载的是 `ee/workflows/` 下的完整实现。

### 2.3 WorkflowsModule加载的Providers

[WorkflowsModule.register()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/module.ts#L48-L233) 通过 `getProviders` 加载以下三类模块的 provider：

**Workflows 模块自身**（28个文件）：
- Controllers: `workflows.controller`, `workflow-executions.controller`, `workflow-schedules.controller`, `workflow-webhooks.controller`, `workflow-bundles.controller`
- Services: `workflow-executions.service`, `workflow-webhooks.service`, `workflow-schedules.service`, `workflow-scheduler.service`, `workflow-execution-queue.service`, `workflow-termination-registry`, `workflow-stream.service`, `schedule-bootstrap.service`, `npm-registry.service`, `bundle-generation.service`, `bundle-service.factory`, `agent-node.service`, `workflow-version.util.service`, `python-executor.service`, `security-mode-detector.service`, `python-bundle-generation.service`, `pypi-registry.service`
- Processors: `workflow-schedule.processor`, `workflow-execution.processor`
- Listeners: `app-actions.listener`
- Ability: `app`

**Apps 模块**（5个文件）：
- `service`, `services/page.service`, `services/event.service`, `services/component.service`, `services/page.util.service`

**Organization-constants 模块**（1个文件）：
- `service`

---

## 3. Bull队列和BullBoard feature注册

### 3.1 Bull队列注册

在 [WorkflowsModule.register()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/module.ts#L145-L151) 中注册了两个 BullMQ 队列：

```typescript
const WORKFLOW_SCHEDULE_QUEUE = 'workflow-schedule-queue';
const WORKFLOW_EXECUTION_QUEUE = 'workflow-execution-queue';

BullModule.registerQueue({ name: WORKFLOW_SCHEDULE_QUEUE }),
BullModule.registerQueue({ name: WORKFLOW_EXECUTION_QUEUE }),
```

### 3.2 BullBoard feature注册

同时将两个队列注册到 BullBoard 用于可视化监控（[module.ts#L153-L160](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/module.ts#L153-L160)）：

```typescript
BullBoardModule.forFeature({
  name: WORKFLOW_SCHEDULE_QUEUE,
  adapter: BullMQAdapter,
}),
BullBoardModule.forFeature({
  name: WORKFLOW_EXECUTION_QUEUE,
  adapter: BullMQAdapter,
}),
```

### 3.3 队列配置

详细配置在 [queue-config.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/constants/queue-config.ts) 中定义：

| 配置项 | 值 | 说明 |
|--------|----|------|
| 优先级 - MANUAL/WEBHOOK | 0 | 最高优先级 |
| 优先级 - SCHEDULED | 1 | 次高优先级 |
| 重试次数 | 0 | 默认不重试 |
| 重试策略 | exponential, 2s | 指数退避 |
| 完成保留数 | 100 | 保留最近100条成功记录 |
| 失败保留数 | 50 | 保留最近50条失败记录 |
| 默认超时 | 60秒 | 可通过 WORKFLOW_TIMEOUT_SECONDS 配置 |
| 并发数 | 5 | 可通过 TOOLJET_WORKFLOW_CONCURRENCY 配置 |

### 3.4 注册范围

| 维度 | 队列注册 | BullBoard feature | BullBoard 根模块 |
|------|----------|-------------------|------------------|
| CE | ✅ WorkflowsModule内 | ✅ WorkflowsModule内 | ✅ AppModule条件导入 |
| EE | ✅ WorkflowsModule内 | ✅ WorkflowsModule内 | ✅ AppModule条件导入 |
| Cloud | ❌ 无 | ❌ 无 | ❌ 无 |
| 主进程 | ✅ 注册 | ✅ 注册 | ✅ 注册 |
| Worker | ✅ 注册 | ✅ 注册 | ✅ 注册 |

---

## 4. WORKER=true时才注册的processor/scheduler

### 4.1 条件注册逻辑

在 [WorkflowsModule.register()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/module.ts#L213-L223) 中，通过两层条件控制：

```typescript
...(isMainImport
  ? [
      WorkflowStreamService,
      AppsActionsListener,
      ...(process.env.WORKER === 'true'
        ? [WorkflowScheduleProcessor, WorkflowExecutionProcessor, ScheduleBootstrapService]
        : []),
    ]
  : []),
```

### 4.2 两层条件说明

| 条件 | 控制的providers | 说明 |
|------|-----------------|------|
| `isMainImport === true` | WorkflowStreamService, AppsActionsListener | 作为主模块导入时才注册 |
| `process.env.WORKER === 'true'` | WorkflowScheduleProcessor, WorkflowExecutionProcessor, ScheduleBootstrapService | Worker进程才注册处理器和启动服务 |

### 4.3 各组件职责

| 组件 | 职责 | CE版行为 |
|------|------|----------|
| [WorkflowScheduleProcessor](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/processors/workflow-schedule.processor.ts) | 处理调度队列中的定时任务 | Stub: 抛出"Method not implemented." |
| [WorkflowExecutionProcessor](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/processors/workflow-execution.processor.ts) | 处理执行队列中的工作流执行任务 | Stub: 抛出"Method not implemented." |
| [ScheduleBootstrapService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/services/schedule-bootstrap.service.ts) | 模块启动时恢复已有的调度任务 | Stub: 仅打印日志，无实际操作 |

### 4.4 进程部署矩阵

| 进程类型 | isMainImport | WORKER=true | 注册的组件 |
|----------|--------------|-------------|------------|
| 主进程 (Web) | true | false | WorkflowStreamService, AppsActionsListener |
| Worker进程 | true | true | WorkflowStreamService, AppsActionsListener, + Processors + ScheduleBootstrap |
| 非主导入 | false | - | 无 |

**设计目的**：支持分离部署——可以运行纯HTTP的主进程实例和专门的Worker实例，提高可扩展性。

---

## 5. CE stub在controller/service层的行为

### 5.1 CE stub总体行为

CE 版本（Community Edition）的 Workflow 模块所有核心功能都是 **stub 实现**，遵循以下模式：

- 实现对应的 interface（保持类型兼容）
- 所有方法抛出 `Error` 或返回空值
- 部分方法有明确的"企业版功能"提示

### 5.2 Controllers层stub行为

| Controller | CE stub行为 | 文件 |
|------------|-------------|------|
| [WorkflowsController](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/controllers/workflows.controller.ts) | create() 抛出 "Method not implemented." | workflows.controller.ts |
| [WorkflowExecutionsController](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/controllers/workflow-executions.controller.ts) | 所有端点（create, status, show, index, getExecutions, getExecutionNodes, previewQueryNode, trigger, streamWorkflowExecution）均抛出 "Method not implemented." | workflow-executions.controller.ts |
| [WorkflowSchedulesController](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/controllers/workflow-schedules.controller.ts) | 所有端点（create, findAll, findOne, update, activate, remove）均抛出 "Method not implemented." | workflow-schedules.controller.ts |
| [WorkflowWebhooksController](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/controllers/workflow-webhooks.controller.ts) | 所有端点（triggerWorkflow, triggerWorkflowAsync, getExecutionStatus, triggerWorkflowStream, updateWorkflow）均抛出 "Method not implemented." | workflow-webhooks.controller.ts |
| [WorkflowBundlesController](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/controllers/workflow-bundles.controller.ts) | 所有端点抛出 500 错误，消息为 "Enterprise feature: Package management requires ToolJet Enterprise Edition" | workflow-bundles.controller.ts |

### 5.3 Services层stub行为

| Service | CE stub行为 | 文件 |
|---------|-------------|------|
| [WorkflowExecutionsService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/services/workflow-executions.service.ts) | 所有方法抛出 "Method not implemented." | workflow-executions.service.ts |
| [WorkflowSchedulesService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/services/workflow-schedules.service.ts) | 所有方法抛出 "Method not implemented." | workflow-schedules.service.ts |
| [WorkflowSchedulerService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/services/workflow-scheduler.service.ts) | 所有方法抛出 "Workflow scheduling is an Enterprise feature. Please upgrade to use workflow scheduling." | workflow-scheduler.service.ts |
| [WorkflowWebhooksService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/services/workflow-webhooks.service.ts) | triggerWorkflow() 和 updateWorkflow() 空返回；resolveVersionId() 和 getEnvironmentId() 抛出 "Method not implemented." | workflow-webhooks.service.ts |
| [WorkflowExecutionQueueService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/services/workflow-execution-queue.service.ts) | enqueue() 和 terminate() 抛出 "Method not implemented." | workflow-execution-queue.service.ts |
| [ScheduleBootstrapService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/services/schedule-bootstrap.service.ts) | onModuleInit() 无操作，仅打印日志 "ℹ️ Workflow scheduling is available in Enterprise Edition" | schedule-bootstrap.service.ts |

### 5.4 Processors层stub行为

| Processor | CE stub行为 | 文件 |
|-----------|-------------|------|
| [WorkflowScheduleProcessor](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/processors/workflow-schedule.processor.ts) | process() 抛出 "Method not implemented." | workflow-schedule.processor.ts |
| [WorkflowExecutionProcessor](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/processors/workflow-execution.processor.ts) | process() 抛出 "Method not implemented." | workflow-execution.processor.ts |

### 5.5 stub设计模式

CE stub 采用 **接口驱动** 的设计模式：

1. 在 `interfaces/` 目录定义接口（如 `IWorkflowExecutionsService`）
2. CE 版本实现这些接口，但所有方法抛出错误
3. EE 版本提供真正的实现
4. 通过 `SubModule.getProviders` 根据版本动态加载

这种设计的好处：
- **类型安全**：CE 和 EE 版本遵循相同的接口契约
- **平滑升级**：从 CE 升级到 EE 只需切换版本，无需修改代码
- **错误明确**：调用 CE 版本的企业功能时会得到清晰的错误提示

---

## 6. WorkflowGuard如何统计日/月执行限制

### 6.1 Guard层级结构

Workflow 相关的 Guard 有两个：

| Guard | 职责 | 文件 |
|-------|------|------|
| [WorkflowGuard](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/workflow.guard.ts) | 限制工作流执行次数（日/月） | licensing/guards/workflow.guard.ts |
| [WorkflowCountGuard](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/workflowcount.guard.ts) | 限制工作流总数 | licensing/guards/workflowcount.guard.ts |
| [WorkflowAccessGuard](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/guards/workflow-access.guard.ts) | 工作流访问权限校验 | workflows/guards/workflow-access.guard.ts |

### 6.2 WorkflowGuard执行限制逻辑

[WorkflowGuard](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/workflow.guard.ts#L9-L103) 的核心流程：

#### 6.2.1 获取workflowId

从请求体中获取 `appVersionId` 或 `appId`，然后查询数据库得到 workflow 的 appId。

#### 6.2.2 获取License限制

```typescript
const workflowsLimit = await this.licenseTermsService.getLicenseTerms(
  LICENSE_FIELD.WORKFLOWS, 
  organizationId
);
```

如果 `workflowsLimit.workspace` 和 `workflowsLimit.instance` 都不存在，抛出 451 错误：
> "Workflow is not enabled in the license, contact admin"

#### 6.2.3 Workspace级别限制统计

使用原生 SQL 查询统计 workspace 级别的执行次数：

```sql
SELECT
  COUNT(*) FILTER (WHERE we.created_at >= date_trunc('day', current_date)) as daily_count,
  COUNT(*) as monthly_count
FROM apps a
INNER JOIN app_versions av ON av.app_id = a.id
INNER JOIN workflow_executions we ON we.app_version_id = av.id
WHERE a.organization_id = $1
AND we.created_at >= date_trunc('month', current_date)
AND we.created_at < date_trunc('month', current_date) + interval '1 month'
```

**统计逻辑**：
- `daily_count`: 当天（从午夜开始）的执行次数
- `monthly_count`: 当月（从月初开始）的执行次数

**限制检查**：
- 日限制：`daily_count >= daily_executions` → 抛出 451
- 月限制：`monthly_count >= monthly_executions` → 抛出 451

#### 6.2.4 Instance级别限制统计

类似地，查询整个实例的执行次数：

```sql
SELECT
  COUNT(*) FILTER (WHERE we.created_at >= date_trunc('day', current_date)) as daily_count,
  COUNT(*) as monthly_count
FROM workflow_executions we
WHERE we.created_at >= date_trunc('month', current_date)
AND we.created_at < date_trunc('month', current_date) + interval '1 month'
```

### 6.3 UNLIMITED特殊处理

如果限制值为 `LICENSE_LIMIT.UNLIMITED`（值为 `'UNLIMITED'`），则跳过对应检查：

```typescript
const needsDailyCheck = workflowsLimit.workspace.daily_executions !== LICENSE_LIMIT.UNLIMITED;
const needsMonthlyCheck = workflowsLimit.workspace.monthly_executions !== LICENSE_LIMIT.UNLIMITED;
```

### 6.4 WorkflowCountGuard总数限制

[WorkflowCountGuard](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/workflowcount.guard.ts) 统计工作流总数：

- **Workspace级别**：统计 `organizationId` 下 `type = 'workflow'` 的 app 数量
- **Instance级别**：统计整个实例中 `type = 'workflow'` 的 app 数量
- 超过 `workflows.workspace.total` 或 `workflows.instance.total` 时抛出 451

### 6.5 默认限制值（BASIC_PLAN）

在 [PlanTerms.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/PlanTerms.ts#L34-L46) 中定义了 BASIC 计划的默认限制：

| 维度 | 工作流总数 | 日执行次数 | 月执行次数 |
|------|-----------|-----------|-----------|
| Workspace | 200 | 500 | 10,000 |
| Instance | 1,000 | 25,000 | 50,000 |

另外还有 TEAM_PLAN 版本，其 instance 级别的限制均为 UNLIMITED。

### 6.6 错误响应码

所有 License 相关的限制都使用 **HTTP 451** 状态码（Unavailable For Legal Reasons），表示因许可证限制而不可用。

---

## 7. Cloud/非Cloud和License限制下的差异总结

### 7.1 版本维度差异总览

| 维度 | CE (社区版) | EE (企业版) | Cloud (云版) |
|------|-------------|-------------|--------------|
| **WorkflowsModule** | ✅ 导入（stub） | ✅ 导入（完整实现） | ❌ 不导入 |
| **BullBoard根模块** | ✅ 导入 | ✅ 导入 | ❌ 不导入 |
| **Provider来源** | src/modules/workflows (stub) | ee/workflows (完整) | - |
| **Bull队列** | ✅ 注册（空转） | ✅ 注册（工作） | ❌ 无 |
| **Workflow功能** | ❌ 不可用（stub抛错） | ✅ 可用（受license限制） | ❌ 不可用 |
| **License限制** | BASIC_PLAN（默认） | 按license配置 | - |

### 7.2 进程维度差异总览

| 组件 | 主进程 (WORKER=false) | Worker进程 (WORKER=true) |
|------|----------------------|--------------------------|
| Controllers | ✅ 全部注册 | ✅ 全部注册 |
| Services | ✅ 全部注册 | ✅ 全部注册 |
| Bull队列 | ✅ 注册（可投递） | ✅ 注册（可消费） |
| BullBoard | ✅ 注册 | ✅ 注册 |
| WorkflowStreamService | ✅ 注册 | ✅ 注册 |
| AppsActionsListener | ✅ 注册 | ✅ 注册 |
| WorkflowScheduleProcessor | ❌ 不注册 | ✅ 注册 |
| WorkflowExecutionProcessor | ❌ 不注册 | ✅ 注册 |
| ScheduleBootstrapService | ❌ 不注册 | ✅ 注册 |

### 7.3 License限制维度

| 限制类型 | Workspace级别 | Instance级别 |
|----------|--------------|--------------|
| 工作流总数 | ✅ `workflows.workspace.total` | ✅ `workflows.instance.total` |
| 日执行次数 | ✅ `workflows.workspace.daily_executions` | ✅ `workflows.instance.daily_executions` |
| 月执行次数 | ✅ `workflows.workspace.monthly_executions` | ✅ `workflows.instance.monthly_executions` |
| 执行超时 | ✅ `workflows.execution_timeout` | - |

**限制优先级**：
1. 先检查 workspace 级别限制
2. 再检查 instance 级别限制
3. 两者是 **AND** 关系，任一超限都会拒绝

### 7.4 部署架构建议

根据上述差异，推荐的部署架构：

```
┌─────────────────────────────────────────────────────┐
│                    Load Balancer                    │
└──────────────────────┬──────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
┌───────────────┐             ┌───────────────┐
│  主进程实例    │             │  Worker实例   │
│  (WORKER=false)│             │ (WORKER=true) │
│               │             │               │
│ - HTTP API    │             │ - 调度处理    │
│ - 控制台      │             │ - 执行处理    │
│ - Bull投递    │             │ - 启动恢复    │
└───────┬───────┘             └───────┬───────┘
        │                              │
        └──────────────┬───────────────┘
                       ▼
              ┌────────────────┐
              │   Redis (Bull)  │
              │  队列 + BullBoard│
              └────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │   PostgreSQL    │
              │  workflow_exec  │
              └────────────────┘
```

### 7.5 关键代码参考

| 功能 | 文件路径 |
|------|----------|
| AppModule条件导入 | [module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/module.ts#L165-L177) |
| WorkflowsModule定义 | [module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/module.ts) |
| SubModule.getProviders | [sub-module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/sub-module.ts) |
| getImportPath版本切换 | [constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/index.ts#L11-L35) |
| 队列配置 | [queue-config.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workflows/constants/queue-config.ts) |
| 执行次数限制Guard | [workflow.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/workflow.guard.ts) |
| 总数限制Guard | [workflowcount.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/guards/workflowcount.guard.ts) |
| 默认License条款 | [PlanTerms.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/PlanTerms.ts) |
| getTooljetEdition函数 | [utils.helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/utils.helper.ts#L666-L671) |
