# BackgroundProcessor 后台任务系统架构详解

## 概述

BackgroundProcessor 是一个通用的后台任务处理系统，支持多任务异步执行、进度追踪（SSE）、多 Pod 部署（Redis Pub/Sub）以及事务化执行。本文档详细解析从任务启动到进度推送的完整工作流程。

---

## 一、核心文件结构

| 文件 | 职责 |
|------|------|
| [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts) | 核心服务：`startJob` 入口、任务调度、执行引擎、权重计算 |
| [store.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/store.ts) | Redis 存储层：元数据管理、Pub/Sub 分发、本地 Subject 管理 |
| [service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/service.ts) | 业务服务层：SSE 事件流获取、用户归属校验 |
| [controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/controller.ts) | REST 控制器：`/jobs/:jobId/events` SSE 端点 |
| [types/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/types/index.ts) | 类型定义：`ProcessInfo`、`ProcessOptions`、`SseProgressEvent` 等 |
| [module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/module.ts) | NestJS 模块定义 |

---

## 二、startJob 立即返回 jobId 机制

### 2.1 调用流程

[BackgroundProcessorUtilService.startJob](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L91-L138) 采用 **Fire-and-Forget** 模式实现立即返回：

```
调用方 → startJob(tasks, options)
    │
    ├─ 1. 生成 UUID 作为 jobId (uuidv4)
    ├─ 2. resolveWeightages() 计算各任务权重
    ├─ 3. jobStore.set() 将元数据写入 Redis + 创建本地 Subject
    ├─ 4. this.execute() —— 不 await，后台异步执行
    │   └─ catch() 捕获异常并打日志
    └─ 5. 立即 return { jobId, processInfo } ← 同步返回
```

### 2.2 关键代码逻辑

```typescript
// 1. 生成唯一 jobId
const jobId = uuidv4();

// 2. 解析权重（同步计算）
const resolvedInfo = this.resolveWeightages(processInfos);

// 3. 注册到 Redis 存储（异步，但 await 了）
const subject = await this.jobStore.set(jobId, { resolvedInfo, userId });

// 4. Fire-and-Forget：后台执行，不阻塞返回
this.execute(jobId, processInfos, resolvedInfo, subject, {
  isRunParallel,
  isBreakOnError,
  manager,
  onSuccess,
  onFailure,
}).catch((err) => {
  this.transactionLogger.error(`...`, err);
});

// 5. 立即返回
return { jobId, processInfo: resolvedInfo };
```

> **设计要点**：`execute()` 没有 `await`，任务在后台异步运行，调用方立刻拿到 `jobId` 用于后续 SSE 订阅。

---

## 三、三种任务执行模式

### 3.1 顺序执行（默认）

**触发条件**：`isRunParallel = false`（默认值）

[execute 方法 - 顺序分支](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L373-L401)

- 按数组顺序逐个执行任务
- 每个任务执行前发出 `running` 事件，完成后发出 `completed`/`failed` 事件
- `isBreakOnError = true`（默认）时，第一个失败任务触发后停止后续任务
- `isBreakOnError = false` 时，继续执行剩余任务

### 3.2 并行执行

**触发条件**：`isRunParallel = true`

[execute 方法 - 并行分支](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L298-L325)

- 使用 `Promise.allSettled()` 并发执行所有任务
- 每个任务独立发射 `running` / `completed` / `failed` 事件
- `completedWeightage` 为原子累加（由于并行，进度百分比非严格递增）
- `isBreakOnError` 对并行执行意义有限（已全部启动），仅影响最终生命周期回调

### 3.3 事务化顺序执行

**触发条件**：`manager` 选项不为 `undefined`

[execute 方法 - 事务分支](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L326-L372)

- 所有任务共享同一个 `EntityManager`，在同一个数据库事务中执行
- 任一任务失败 → 整个事务回滚
- `onSuccess` / `onFailure` 回调也在事务内执行

---

## 四、weightage 自动分配机制

[resolveWeightages 方法](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L142-L191)

### 4.1 分配规则

1. **显式权重优先**：已设置 `weightage` 的任务使用指定值
2. **剩余权重均分**：未设置权重的任务平分 `100 - explicitSum`
3. **全部未设置**：所有任务平均分配，各占 `100 / n`

### 4.2 校验规则

- 权重不能为负数
- 显式权重之和不能超过 100（允许 1e-9 的浮点误差）
- 权重类型必须为有效数字

### 4.3 示例

```typescript
const tasks = [
  { task: 'A', process: ..., weightage: 30 },  // 显式 30%
  { task: 'B', process: ... },                  // 隐式，自动计算
  { task: 'C', process: ... },                  // 隐式，自动计算
];
// 结果：A=30, B=35, C=35 （剩余70由B、C均分）
```

---

## 五、manager 选项限制

[startJob - manager 校验](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L98-L111)

### 5.1 使用约束

当传入 `manager` 选项时，必须同时满足：

| 选项 | 必须值 | 原因 |
|------|--------|------|
| `isRunParallel` | `false` | 事务不能跨并行任务（每个连接同一时间只能有一个事务） |
| `isBreakOnError` | `true` | 事务语义要求遇到错误必须中止并回滚 |

违反任一条件会抛出 `Error`。

### 5.2 manager 的两种取值

- `manager: true`：内部通过 [dbTransactionWrap](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/database.helper.ts#L19-L29) 开启新事务
- `manager: EntityManager`：复用调用方已有的事务上下文

---

## 六、onSuccess / onFailure 事务语义

[dispatchLifecycle 函数](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L282-L295)

### 6.1 调用判定逻辑

```
anyFailed = 是否有任务失败
isBreakOnError = 是否遇错即停

if (anyFailed && isBreakOnError) → onFailure
else                             → onSuccess
```

> 注意：`isBreakOnError = false` 时，即使有任务失败也调用 `onSuccess`（因为整体流程未中断）。

### 6.2 事务模式下的语义

| 回调 | 执行时机 | 事务上下文 | 抛出异常的影响 |
|------|----------|------------|----------------|
| `onSuccess` | 所有任务成功后，事务提交前 | ✅ 在事务内 | 整个事务回滚 |
| `onFailure` | 任务失败后，事务回滚前 | ✅ 在事务内 | 其 DB 写入随事务一起回滚 |

### 6.3 非事务模式下的语义

- `onSuccess` / `onFailure` 都在所有任务执行完毕后调用
- 没有事务保护，回调中的 DB 操作各自独立

---

## 七、Redis 元数据与 Pub/Sub 架构

[BackgroundProcessorJobStore](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/store.ts)

### 7.1 Redis Key 设计

| 类型 | Key 格式 | 内容 | TTL |
|------|----------|------|-----|
| 元数据 | `bg_job:{jobId}` | `{ userId, resolvedInfo(JSON), createdAt }` | 3600 秒（1 小时） |
| Pub/Sub 通道 | `bg_job_events:{jobId}` | 事件流（SseProgressEvent） | 通道随 job 生命周期 |

### 7.2 TTL 机制

- 常量 `JOB_TTL_SECONDS = 3600`
- 使用 `SETEX` 命令写入元数据时自动设置过期
- 防止任务异常终止后元数据永久残留
- 1 小时的窗口足够绝大多数后台任务完成

### 7.3 Pub/Sub 工作原理

```
┌─────────────┐          ┌──────────┐          ┌─────────────┐
│  Executor Pod│ publish  │  Redis   │ subscribe│ Subscriber Pod│
│              │─────────▶│ Channel  │─────────▶│               │
│  执行任务     │  事件    │          │  事件    │  SSE 推送给客户端│
└─────────────┘          └──────────┘          └─────────────┘
```

- 执行 Pod（启动任务的 Pod）通过 `publishEvent()` 发布事件
- 订阅 Pod（接收 SSE 请求的 Pod）通过 `subscribeToEvents()` 订阅通道
- 每个 job 有独立的 Redis 通道，互不干扰
- 专用的 subscriber Redis 连接（Pub/Sub 需要专用连接）

### 7.4 本地 Subject 与 Redis 的关系

每个 Pod 维护 `localSubjects: Map<jobId, Subject>`：

- **执行 Pod**：任务启动时创建 Subject，同时订阅 Redis 通道（虽然自己用不上跨 pod 事件，但保持统一）
- **订阅 Pod**：SSE 连接建立时创建 Subject，通过 Redis 消息驱动 `localSubject.next(event)`
- 本地 Subject 是 SSE 数据流的直接来源，Redis 是跨 Pod 分发的桥梁

---

## 八、/jobs/:jobId/events SSE 端点

[BackgroundProcessorController.streamEvents](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/controller.ts#L77-L101)

### 8.1 端点信息

- **路径**：`GET /jobs/:jobId/events`
- **认证**：JWT Bearer Token（`JwtAuthGuard`）
- **响应类型**：`text/event-stream`（SSE）

### 8.2 请求处理流程

```
客户端 EventSource 连接
    │
    ▼
JwtAuthGuard 校验 JWT → 提取用户信息
    │
    ▼
controller.streamEvents(jobId, user)
    │
    ├─ 1. processor.getEventStream(jobId, userId)
    │   ├─ 1.1 从 Redis 读取 job 元数据
    │   ├─ 1.2 若不存在 → 返回 null → 404 NotFound
    │   ├─ 1.3 校验 userId 归属 → 不匹配 → 'FORBIDDEN' → 403
    │   └─ 1.4 调用 jobStore.subscribeToEvents() → 返回 Observable
    │
    ├─ 2. map 转换为 MessageEvent 格式 (type: 'progress')
    │
    └─ 3. finalize → 客户端断开时调用 cleanupSubscription
```

### 8.3 JWT 用户归属校验

[getEventStream - 归属校验](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/service.ts#L33-L38)

- Job 元数据中存储了创建时传入的 `userId`
- SSE 请求时，从 JWT 解析出当前用户 `id`
- 两者必须匹配才能访问事件流
- 若 `userId` 未设置（`undefined`），则不做校验（兼容无主任务）

---

## 九、done 事件与生命周期

[execute - finally 块](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L402-L422)

### 9.1 done 事件的意义

`done` 是任务的**最终收尾事件**，在 `finally` 块中发出，保证无论成功失败都会发送：

```typescript
{
  jobId: string,
  task: '',           // 空字符串，表示不是某个具体任务
  weightage: 0,
  completedWeightage: number,  // 最终完成的总权重
  status: 'done'
}
```

### 9.2 done 事件触发的清理动作

[store.ts - message 处理](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/store.ts#L66-L71)

订阅 Pod 收到 `done` 事件后：
1. `localSubject.complete()` —— 关闭 SSE 流
2. `localSubjects.delete(jobId)` —— 从本地映射移除
3. `subscriber.unsubscribe(channel)` —— 取消 Redis 通道订阅

---

## 十、cleanupSubscription 机制

### 10.1 触发时机

[controller.ts - finalize](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/controller.ts#L97-L99)

- 客户端主动断开 SSE 连接（浏览器关闭、网络中断）
- Observable 正常完成（收到 `done` 事件后）

### 10.2 清理动作

[cleanupSubscription 方法](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/store.ts#L204-L225)

```typescript
cleanupSubscription(jobId):
  1. 检查 localSubject 是否存在且未关闭
  2. localSubject.complete()        // 通知所有观察者结束
  3. localSubjects.delete(jobId)    // 从本地 Map 移除
  4. subscriber.unsubscribe(channel) // 取消 Redis 通道订阅
```

### 10.3 与 delete() 的区别

| 方法 | 调用方 | 清理范围 |
|------|--------|----------|
| `delete(jobId)` | 执行 Pod（任务完成时） | Redis 元数据 + 本地 Subject + 取消订阅 |
| `cleanupSubscription(jobId)` | 订阅 Pod（客户端断开时） | 仅本地 Subject + 取消订阅（不删元数据） |

> 执行 Pod 在任务结束后调用 `delete()`，会删除 Redis 中的 job 元数据；订阅 Pod 只清理自己的订阅资源。

---

## 十一、main.ts 全局 prefix 排除 /jobs 的关系

[main.ts - configureUrlPrefix](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/main.ts#L186-L203)

### 11.1 全局前缀配置

```typescript
app.setGlobalPrefix(urlPrefix + 'api', { exclude: pathsToExclude });
```

默认情况下，所有 API 路由都有 `/api` 前缀（如果配置了 `SUB_PATH` 还会加子路径前缀）。

### 11.2 为什么排除 /jobs

```typescript
pathsToExclude.push({ path: '/jobs', method: RequestMethod.ALL });
pathsToExclude.push({ path: '/jobs/{*path}', method: RequestMethod.ALL });
```

**历史原因**：`/jobs` 路径同时被 **Bull Board**（任务队列管理面板）使用。Bull Board 是一个独立的 UI 面板，它期望直接挂载在 `/jobs` 路径下，不希望被 `/api` 前缀修饰。

### 11.3 影响范围

- BackgroundProcessor 的 SSE 端点 `/jobs/:jobId/events` **没有** `/api` 前缀
- 客户端直接访问 `/jobs/{jobId}/events` 即可
- 需要在网关/反向代理层面确保 `/jobs` 路径正确路由到后端服务

---

## 十二、完整调用链时序图

```
客户端                    调用方Controller    BackgroundProcessor       Redis
  │                           │                      │                    │
  │ POST /my-feature/start    │                      │                    │
  │──────────────────────────▶│                      │                    │
  │                           │ startJob(tasks, opts)│                    │
  │                           │─────────────────────▶│                    │
  │                           │                      │ 1. uuidv4()        │
  │                           │                      │ 2. resolveWeightages
  │                           │                      │ 3. SETEX bg_job:xxx │
  │                           │                      │───────────────────▶│
  │                           │                      │ 4. SUBSCRIBE channel│
  │                           │                      │───────────────────▶│
  │                           │                      │                    │
  │                           │  return {jobId}      │                    │
  │                           │◀─────────────────────│                    │
  │◀──────────────────────────│                    │                    │
  │    { jobId: "xxx" }       │                    │                    │
  │                           │                    │                    │
  │  (后台继续执行)            │                    │ execute() 后台运行  │
  │                           │                    │  ├─ running 事件    │
  │                           │                    │  │  PUBLISH ────────▶│
  │                           │                    │  ├─ completed 事件  │
  │                           │                    │  │  PUBLISH ────────▶│
  │                           │                    │  └─ ...             │
  │                           │                    │                    │
  │ SSE /jobs/:jobId/events   │                    │                    │
  │───────────────────────────────────────────────▶│                    │
  │                           │                    │ getMetadata()      │
  │                           │                    │ GET bg_job:xxx     │
  │                           │                    │───────────────────▶│
  │                           │                    │◀───────────────────│
  │                           │                    │ 校验 userId        │
  │                           │                    │ subscribeToEvents()│
  │                           │                    │ SUBSCRIBE channel  │
  │                           │                    │───────────────────▶│
  │                           │                    │                    │
  │   progress events         │                    │◀───────────────────│
  │◀───────────────────────────────────────────────│                    │
  │                           │                    │                    │
  │   ... (任务继续执行) ...  │                    │                    │
  │                           │                    │ done 事件          │
  │                           │                    │ PUBLISH done ─────▶│
  │                           │                    │ delete(jobId)      │
  │                           │                    │ DEL bg_job:xxx     │
  │                           │                    │───────────────────▶│
  │   done event              │                    │                    │
  │◀───────────────────────────────────────────────│                    │
  │   (SSE 连接关闭)          │                    │                    │
```

---

## 十三、关键设计总结

| 特性 | 实现方式 | 所在模块 |
|------|----------|----------|
| 立即返回 jobId | Fire-and-Forget（不 await execute） | [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L126-L134) |
| 顺序执行 | for 循环 + await | [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L373-L401) |
| 并行执行 | Promise.allSettled | [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L298-L325) |
| 事务执行 | dbTransactionWrap + EntityManager 传递 | [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L326-L372) |
| 权重自动分配 | 剩余权重均分算法 | [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L142-L191) |
| 跨 Pod 事件推送 | Redis Pub/Sub + 本地 Subject | [store.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/store.ts) |
| 用户归属校验 | Redis 元数据 userId 比对 | [service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/service.ts#L33-L38) |
| TTL 自动过期 | Redis SETEX 3600s | [store.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/store.ts#L13-L13) |
| done 收尾事件 | finally 块统一发出 | [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/util.service.ts#L409-L416) |
| 客户端断开清理 | RxJS finalize + cleanupSubscription | [controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/background-processor/controller.ts#L97-L99) |
| 全局前缀排除 | setGlobalPrefix exclude 配置 | [main.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/main.ts#L198-L200) |
