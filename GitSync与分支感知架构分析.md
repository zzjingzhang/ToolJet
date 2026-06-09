# GitSync 与分支感知架构分析

## 一、核心问题：前端完整 API vs 后端 Stub 实现

### 1.1 现象描述

在 ToolJet CE（社区版）源码中，存在一个显著的架构不对称：

- **前端**：`git_sync.service.js` 暴露了 **30+ 个完整的 API 方法**，涵盖了从 Git 配置管理、分支操作、Pull/Push 到标签管理的全流程功能
- **后端**：GitSync、AppGit、WorkspaceBranches 三个模块的 controller 和 service **全部为 stub 实现**，所有方法均抛出 `NotFoundException` 或 `Error('Method not implemented.')`

### 1.2 后端 Stub 实现清单

#### GitSync 模块

- [git-sync/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/git-sync/controller.ts)
  - 8 个路由端点，全部 `throw new NotFoundException()`
  - 包括：`getOrgGitStatusByOrgId`、`create`、`update`、`setFinalizeConfig`、`changeStatus`、`deleteConfig`、`getOrgGitByOrgId`、`toggleEnvConfig`

- [git-sync/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/git-sync/service.ts)
  - 9 个方法，全部 `throw new Error('Method not implemented.')`

#### AppGit 模块

- [app-git/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-git/controller.ts)
  - 9 个路由端点，全部 `throw new NotFoundException()`
  - 包括：`getAppsMetaFile`、`gitSyncApp`、`getAppMetaFile`、`getAppConfig`、`createGitApp`、`pullGitAppChanges`、`renameAppOrVersion`、`updateAppGitConfigs`、`getAppGitConfigs`

- [app-git/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-git/service.ts)
  - 6 个方法，全部 `throw new Error('Method not implemented.')`

#### WorkspaceBranches 模块

- [workspace-branches/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workspace-branches/service.ts)
  - 11 个方法，全部 `throw new NotFoundException()`
  - 包括：`list`、`createBranch`、`switchBranch`、`deleteBranch`、`deleteWorkspaceBranch`、`pushWorkspace`、`pullWorkspace`、`ensureAppDraft`、`checkForUpdates`、`listRemoteBranches`、`getPullRequests`

### 1.3 前端完整 API 清单

[git_sync.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/git_sync.service.js) 中定义的 API 方法：

| 分类 | 方法名 | 说明 |
|------|--------|------|
| **配置管理** | `create` | 创建 Git 同步配置 |
| | `getGitConfig` | 获取 Git 配置 |
| | `updateConfig` | 更新配置 |
| | `setFinalizeConfig` | 最终确认配置 |
| | `deleteConfig` | 删除配置 |
| | `updateStatus` | 更新启用状态 |
| | `getGitStatus` | 获取状态 |
| | `saveProviderConfigs` | 保存提供商配置 |
| | `updateEnvConfigs` | 更新环境配置 |
| | `testProviderConnection` | 测试连接 |
| **应用同步** | `syncAppVersion` | 同步应用版本 |
| | `gitPush` | 推送应用到 Git |
| | `getAppConfig` | 获取应用配置 |
| | `checkForUpdates` | 检查更新 |
| | `checkForUpdatesByAppName` | 按名称检查更新 |
| | `gitPull` | 拉取 Git 变更 |
| | `confirmPullChanges` | 确认拉取变更 |
| | `importGitApp` | 导入 Git 应用 |
| **应用编辑控制** | `updateAppEditState` | 更新应用编辑状态 |
| | `getAppGitConfigs` | 获取应用 Git 配置 |
| | `updateGitConfigs` | 更新 Git 配置 |
| | `getGitConfigs` | 获取 Git 配置 |
| **分支管理** | `getAllBranches` | 获取所有分支 |
| | `createBranch` | 创建分支 |
| | `getPullRequests` | 获取 PR 列表 |
| | `switchBranch` | 切换分支 |
| **标签管理** | `createGitTag` | 创建 Git 标签 |
| | `checkTagExists` | 检查标签是否存在 |

### 1.4 原因分析：CE/EE 双轨架构

这种"前端完整 + 后端 stub"的模式是典型的 **开源核心 + 商业扩展** 架构设计：

1. **CE 定义接口契约**：CE 版本定义了完整的 Controller 接口（如 `IGitSyncController`、`IGitSyncService`）和路由结构，保证了前后端接口契约的一致性

2. **EE 提供实际实现**：企业版通过模块覆盖/继承的方式提供真实实现。证据包括：
   - 代码中存在 `@ee/git-sync/providers/dto/provider-config.dto` 的引用路径
   - [platform-git-pull-registry.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/platform-git-pull-registry.ts) 定义了 `IPlatformGitPullService` 接口，并通过 `registerPlatformGitPullService()` 函数供 EE 版本注册实现
   - `GitSyncController` 实现了 `IGitSyncController` 接口，EE 版本可通过依赖注入替换实现
   - [edition.helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/edition.helper.ts) 文件暗示了版本区分机制

3. **前端统一代码库**：前端不分 CE/EE，使用同一份代码，通过功能开关（feature flag）和后端响应来控制 UI 显示

4. **渐进式启用**：即使在 CE 中，相关的数据库表、实体定义、核心业务逻辑也已就绪，EE 许可证启用后即可激活完整功能

---

## 二、DataSource 中的 BranchId 感知逻辑

### 2.1 核心实体模型

[DataSource](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source.entity.ts) 与 [DataSourceVersion](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source_version.entity.ts) 构成了数据源的分支感知模型：

```
DataSource (主表，定义数据源元数据)
  └── DataSourceVersion (版本表，按分支隔离配置)
         ├── branch_id: 关联 WorkspaceBranch
         ├── is_default: 是否为默认版本（主分支快照）
         ├── is_active: 是否激活（软删除标记）
         ├── version_from_id: 父版本
         └── DataSourceVersionOptions (各环境的配置选项)
```

**DataSourceVersion 关键字段**：
- `branch_id`：分支 ID，可为 null（非 Git 同步模式）
- `is_default`：是否为默认版本（对应主分支）
- `is_active`：软删除标记，分支上删除数据源时设置为 false
- `version_from_id`：从哪个版本派生而来
- `name`：版本名称
- `meta_timestamp`：元数据时间戳
- `pulled_at`：从 Git 拉取的时间

**唯一约束**：`@Unique(['dataSourceId', 'branchId'])` —— 每个数据源在每个分支上只有一个活跃版本

### 2.2 分支感知的 CRUD 逻辑

#### 创建（create）

[data-sources/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts) 中的 `create` 方法：

- **有 branchId（分支模式）**：
  - 只创建分支专属的 DSV（DataSourceVersion），**不创建默认 DSV**
  - 通过 `createDataSourceVersionForBranchWithOptions()` 方法实现
  - 默认 DSV 会在分支合并到主分支时通过 Git pull/deserialize 创建
  - 目的：防止数据源在合并前就出现在主分支上

- **无 branchId（普通模式）**：
  - 创建默认 DSV（`is_default = true`, `branch_id = null`）
  - 在所有环境中初始化选项

- **安全验证**：
  - `validateNoConstantsOnDefaultBranch()`：如果在默认分支上添加了 workspace constant 引用，则抛出错误
  - PRD 要求：加密字段中的 workspace constant 必须走 PR 流程

#### 更新（update）

- **shouldUpdateDefault 判定**：
  - 无 branchId → 更新默认 DSV
  - branchId 是默认分支 → 更新默认 DSV
  - branchId 是 feature 分支 → **不更新**默认 DSV，只更新分支 DSV
  - 原则：feature 分支的编辑不应改变主分支的 fallback

- **分支 DSV 自动创建**：
  - 如果数据源在该分支上还没有 DSV，会自动创建
  - 从默认 DSV 克隆选项和凭证
  - 凭证会被重新创建（克隆 credential 的值，但生成新的 credential_id）
  - 原因：分支之间不能共享凭证引用

- **凭证克隆机制**：
  ```typescript
  // 遍历所有加密选项，克隆 credential
  for (const key of Object.keys(sourceOptions)) {
    const opt = sourceOptions[key];
    if (opt?.credential_id && opt?.encrypted) {
      const originalValue = await this.credentialService.getValue(opt.credential_id);
      const newCredential = await this.credentialService.create(originalValue || '', manager);
      sourceOptions[key] = { ...opt, credential_id: newCredential.id };
    }
  }
  ```

#### 删除（delete）

[data-sources/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts) 中的 `delete` 方法：

- **有 branchId（全局数据源 + 分支模式）**：
  - **软删除**：设置 `DataSourceVersion.isActive = false`
  - 不物理删除，保留数据以便分支切换
  - 合并到主分支时才会真正删除

- **无 branchId**：
  - 物理删除数据源

#### 调用时的选项解析（invokeMethod）

- 全局数据源在调用时，根据 `branchId` 解析对应分支的选项
- `effectiveBranchId = dataSource.scope === GLOBAL ? branchId || null : null`
- 本地数据源（scope=local）不分支隔离，随 app_version 走

### 2.3 特殊的 Token 传播机制

OAuth 刷新令牌时，会调用 `propagateTokenToAllBranches()` 将 token 传播到**所有分支**的 DSV：

- 原因：OAuth token 是用户级的，不应该因分支不同而不同
- 实现：遍历所有 `isActive = true` 的 DSV，更新 tokenData
- 这是分支隔离的一个例外（安全便利 > 严格隔离）

---

## 三、AppVersion 中的 BranchId 感知逻辑

### 3.1 核心实体模型

[app_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts) 的关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `branch_id` | uuid | 关联 WorkspaceBranch，可为 null |
| `version_type` | enum | `VERSION`（正式版本）/ `BRANCH`（分支草稿） |
| `module_reference_id` | uuid | 模块版本的稳定引用 ID，跨分支/Git 保持一致 |
| `parent_version_id` | uuid | 父版本 ID |
| `is_stub` | boolean | 是否为桩版本（占位符，待从 Git 拉取） |
| `status` | enum | DRAFT / PUBLISHED / RELEASED |
| `co_relation_id` | varchar | 跨实例稳定标识，Git 同步时使用 |
| `pulled_at` | timestamp | 上次从 Git 拉取的时间 |
| `source_tag` | varchar | 来源标签 |

**关键枚举**：
```typescript
export enum AppVersionType {
  VERSION = 'version',   // 正式保存的版本
  BRANCH = 'branch',     // 分支上的草稿版本
}
```

### 3.2 版本列表的分支感知

[versions/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts) 中的 `getAllVersions` 方法：

- 普通应用：`versionRepository.getVersionsInApp(app.id, effectiveBranchId)`
- 模块应用：调用 `listModuleVersions()`（详见 ModuleViewer 章节）
- Workflow 类型的 app 不使用分支隔离

### 3.3 编辑器冻结机制（shouldFreezeEditor）

`getVersion()` 方法中的冻结逻辑：

```typescript
let shouldFreezeEditor = false;

// 1. Git 同步的应用：allowEditing = false 时冻结
if (appGit) {
  shouldFreezeEditor = !appGit.allowEditing || shouldFreezeEditor;
}

// 2. 已发布版本也冻结
if (appVersion?.status === AppVersionStatus.PUBLISHED) {
  shouldFreezeEditor = true;
}
```

其中 `appGit` 的查找逻辑：
- 先按 `app.id` 查找 `AppGitSync` 记录
- 如果没找到但 app 有 `co_relation_id`（且不等于自身 id），则按 `co_relation_id` 查找
- 这是为了处理 branch-copy 应用：平台级 Git 同步创建的分支应用没有自己的 app_git_sync 记录

### 3.4 版本创建/删除的分支约束

#### 创建约束

[versions/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts) 中的 `validateDraftVersionConstraints`：

- 当 `organizationGit.isBranchingEnabled` 为 true 时
- 且要创建的版本类型不是 `BRANCH` 类型
- 且已存在非 BRANCH 类型的 DRAFT 版本
- → 抛出错误："Only one draft version is allowed when branching is enabled."

含义：启用分支模式后，只能有一个正式草稿版本，其他编辑都在分支草稿中进行。

#### 名称/描述修改限制

```typescript
// 在 versionService.update() 中
if (appVersion.status === AppVersionStatus.PUBLISHED) {
  const organizationGit = await this.organizationGitRepository.findOrgGitByOrganizationId(...);
  if (organizationGit?.isEnabled) {
    throw new BadRequestException('Cannot edit name or description of a saved version.');
  }
}
```

启用 Git 同步后，已发布版本的名称和描述不能直接修改（需要通过 Git 流程）。

### 3.5 co_relation_id 的作用

`co_relation_id` 是跨分支、跨 Git 实例的稳定标识：

- **App 实体**、**DataSource**、**DataQuery**、**Page**、**Component**、**EventHandler** 等都有此字段
- 创建时，如果没有 `co_relation_id`，会将其设为自身 id
- Git push 时序列化到 JSON 中，使用 `co_relation_id` 作为标识
- Git pull 时根据 `co_relation_id` 匹配现有记录或创建新记录
- 作用：保证同一个逻辑资源在不同分支、不同工作区中能被识别为"同一个"

---

## 四、ModuleViewer 与分支感知

### 4.1 ModuleViewer 的引用机制

ModuleViewer 组件在其 properties 中存储两个关键值：

- `moduleAppId`：模块的 `co_relation_id`（稳定标识，跨分支/Git 一致）
- `moduleVersionId`：模块版本的 `module_reference_id`（UUID，稳定标识）

> 空字符串的 `moduleVersionId` 表示"未固定"（unpinned），跟随当前分支的最新版本

### 4.2 模块版本解析规则

[versions/module-ref.util.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts) 中的 `resolveModuleRef` 函数实现了完整的解析逻辑：

```
                    ┌─────────────────────┐
                    │  moduleReferenceId  │
                    └─────────┬───────────┘
                              │
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
   有值且是 UUID        有值但非 UUID         空值/缺失
           │                  │                  │
    匹配 module_      匹配 version_name    视为 unpinned
    reference_id       或 branch_name          │
           │                  │                  │
           └──────────┬───────┘                  │
                      ▼                          ▼
               找到匹配行？ ────否────→  orphan 降级
                      │
                      是
                      ▼
             ┌───────────────────┐
             │ 优先消费者分支行   │
             │ 否则默认分支行     │
             └───────────────────┘
```

**详细规则**：

1. **pin-hit（精确匹配）**：
   - `moduleReferenceId` 是有效 UUID
   - 优先在**消费者分支**（consumer branch）上匹配 `module_reference_id`
   - 如果消费者分支没有，回退到**默认分支**（default branch）
   - 也支持按版本名匹配（向后兼容）

2. **unpinned（未固定）**：
   - `moduleReferenceId` 为空或缺失
   - 取消费者分支上的最新非 stub 版本
   - 没有则取默认分支的最新版本

3. **orphan-fallback（孤立降级）**：
   - `moduleReferenceId` 有值但找不到匹配
   - 降级到 unpinned 的逻辑（最新版本）

### 4.3 模块版本列表

`listModuleVersions()` 函数返回的版本列表包括：
- 消费者分支上的**分支草稿**（如果有）
- 默认分支上的所有**已保存版本**（version_type = 'version'）

> 为什么不只返回当前分支的？因为 feature 分支上通常只有草稿，用户需要能选择默认分支的已发布版本。

### 4.4 Feature Branch 的 Pin 继承机制

`reconcileModuleViewerPinsFromDefault()` 函数处理一个重要的边缘情况：

**问题**：从默认分支创建 feature 分支后，分支上的 ModuleViewer 组件的 `moduleVersionId` 可能是旧的（创建分支时的快照），而默认分支上的 pin 可能已经更新。

**解决方案**：从默认分支继承 pin 值：
- 通过 `component.co_relation_id` 匹配两个分支上的同一个 ModuleViewer 组件
- 如果默认分支的 pin 值已设置且与 feature 分支不同，则复制过来
- 只在默认分支有值时才继承（空值不继承）
- 目的：确保 feature 分支创建后，模块 pin 是最新的

### 4.5 完整的解析器：resolveAllModuleViewersForVersion

这个函数一次性解析指定 app_version 下所有 ModuleViewer 的实际渲染版本，返回详细的匹配结果，包括：

- `matchKind`：pin-hit / orphan-fallback / unpinned-fallback / no-row
- `resolved`：解析到的版本行详细信息
- `pinnedValue`：原始 pin 值

用途：
- Save/Promote/Release 等操作前的验证
- 检查模块版本是否符合环境要求
- 避免在每个守卫处重复 JOIN 逻辑

---

## 五、WorkspaceBranch 实体与元数据哈希

### 5.1 WorkspaceBranch 实体结构

[workspace_branch.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/workspace_branch.entity.ts)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `organization_id` | uuid | 工作区 ID |
| `branch_name` | varchar | 分支名称 |
| `is_default` | boolean | 是否为默认分支（主分支） |
| `source_branch_id` | uuid | 源分支 ID（从哪个分支创建） |
| `created_by` | varchar | 创建者 |
| `app_meta_hash` | varchar(64) | 应用元数据哈希 |
| `data_source_meta_hash` | varchar(64) | 数据源元数据哈希 |
| `module_meta_hash` | varchar(64) | 模块元数据哈希 |

**唯一约束**：`@Unique(['organizationId', 'name'])`

### 5.2 三个元数据哈希的作用

`appMetaHash`、`dataSourceMetaHash`、`moduleMetaHash` 分别对应三类资源的 Git 元数据文件哈希：

- **app_meta_hash**：对应 `.meta/appMeta.json` 的哈希
- **data_source_meta_hash**：对应 `.meta/dataSourceMeta.json` 的哈希
- **module_meta_hash**：对应 `.meta/moduleMeta.json` 的哈希

这些哈希用于：
1. 快速判断分支上的资源是否与 Git 远程一致
2. 增量同步时只处理有变化的资源
3. `checkForUpdates` 接口通过比较哈希值判断是否有更新

### 5.3 与其他实体的关联

- **AppVersion.branch_id** → WorkspaceBranch.id
- **DataSourceVersion.branch_id** → WorkspaceBranch.id
- 一对多关系：一个 WorkspaceBranch 对应多个 AppVersion 和 DataSourceVersion

---

## 六、AppGitSync 与 allowEditing 机制

### 6.1 AppGitSync 实体结构

[app_git_sync.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_git_sync.entity.ts)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `git_app_name` | varchar | Git 中的应用名称 |
| `last_commit_id` | varchar | 上次提交 ID |
| `last_commit_message` | varchar | 上次提交信息 |
| `last_commit_user` | varchar | 上次提交用户 |
| `git_app_id` | varchar | Git 应用 ID |
| `git_version_name` | varchar | Git 版本名称 |
| `git_version_id` | varchar | Git 版本 ID |
| `organization_git_id` | uuid | 组织 Git 配置 ID |
| `app_id` | uuid | 关联的应用 ID |
| `version_id` | uuid | 关联的版本 ID |
| `last_push_date` | timestamp | 上次推送时间 |
| `last_pull_date` | timestamp | 上次拉取时间 |
| **`allow_editing`** | **boolean** | **是否允许编辑（默认 false）** |

### 6.2 allowEditing 的作用

`allowEditing` 是 Git 同步应用的编辑锁：

- **默认值**：`false`（不允许直接编辑）
- **作用**：从 Git 拉取的应用默认处于只读状态，防止用户直接编辑导致与 Git 不同步
- **开启后**：用户可以在本地编辑，然后通过 Git Push 提交回去

**编辑冻结判定**（在 `versionService.getVersion()` 中）：
```typescript
if (appGit) {
  shouldFreezeEditor = !appGit.allowEditing || shouldFreezeEditor;
}
```

### 6.3 前端编辑状态切换

前端通过 `gitSyncService.updateAppEditState(appId, allowEditing)` 调用切换编辑状态，对应后端路由 `PUT /app-git/:appId/configs`。

---

## 七、Feature Branch 拉取后的影响实体梳理

当一个 feature branch 被拉取（git pull / switch branch）后，以下实体和字段会受到影响：

### 7.1 数据源版本（DataSourceVersion）

**受影响的字段**：

| 字段 | 影响方式 | 说明 |
|------|----------|------|
| `branch_id` | 设置为目标分支 ID | 标识此 DSV 属于哪个分支 |
| `is_active` | true | 拉取后激活 |
| `is_default` | false | feature 分支上的 DSV 不是默认版本 |
| `name` | 从 Git 元数据读取 | 数据源名称 |
| `version_from_id` | 可能指向源分支版本 | 继承关系 |
| `pulled_at` | 设置为当前时间 | 记录拉取时间 |
| `data_source_id` | 关联到 DataSource 主记录 | 通过 co_relation_id 匹配 |

**关键逻辑**：
- 全局数据源（scope=global）在每个分支上有独立的 DSV
- 本地数据源（scope=local）随 AppVersion 走，不单独分支
- 凭证（credentials）会被克隆，各分支不共享 credential_id
- OAuth token 例外：刷新时会传播到所有分支

### 7.2 模块 Pin（ModuleViewer）

**受影响的组件属性**：

| 属性 | 影响方式 | 说明 |
|------|----------|------|
| `moduleAppId.value` | 从 Git 读取的 co_relation_id | 模块的稳定标识 |
| `moduleVersionId.value` | 从 Git 读取的 module_reference_id | 固定的版本引用 |

**关键逻辑**：
- 拉取后，`reconcileModuleViewerPinsFromDefault()` 会检查并从默认分支继承 pin 值
- 如果 feature 分支上的 pin 比默认分支旧，会被更新
- 解析时优先匹配消费者分支的版本，找不到才回退到默认分支
- pin 状态：pinned / unpinned / orphan-fallback

**受影响的 AppVersion 字段**：
- `module_reference_id`：模块版本的稳定 ID
- `co_relation_id`：模块 app 的稳定 ID

### 7.3 版本创建/删除

**创建版本时**：

| 字段 | 分支模式下的行为 |
|------|------------------|
| `branch_id` | 设置为当前分支 ID |
| `version_type` | `BRANCH`（分支草稿）或 `VERSION`（正式版本） |
| `is_stub` | 可能为 true（桩版本，待拉取完整数据） |
| `parent_version_id` | 指向源版本 |
| `co_relation_id` | 继承或生成新的稳定 ID |

**约束**：
- 启用分支模式后，只能有一个非 BRANCH 类型的 DRAFT 版本
- 已发布版本的名称/描述不能直接修改（Git 模式下）

**删除版本时**：
- 分支草稿可以删除
- 删除时需要检查依赖（如被 ModuleViewer 引用的模块版本）

### 7.4 Git allowEditing

**受影响的实体**：`AppGitSync`

| 字段 | 说明 |
|------|------|
| `allow_editing` | 是否允许编辑（默认 false） |
| `last_pull_date` | 拉取时间 |
| `last_commit_id` | 上次提交 ID |

**行为**：
- 从 Git 新拉取的应用 `allowEditing = false`（只读）
- 用户可以手动开启 `allowEditing`
- `shouldFreezeEditor` 由 `!appGit.allowEditing` 和版本状态共同决定
- 分支拷贝的应用（co_relation_id ≠ id）继承源应用的 app_git_sync 配置

### 7.5 其他受影响实体

#### App 实体
- `co_relation_id`：跨分支稳定标识
- `current_version_id`：当前编辑版本
- 分支模式下，每个分支可能有独立的 app 实例或共享 app 实例（取决于实现层级）

#### WorkspaceBranch 自身
- `app_meta_hash`：更新为新拉取的应用元数据哈希
- `data_source_meta_hash`：更新为新拉取的数据源元数据哈希
- `module_meta_hash`：更新为新拉取的模块元数据哈希
- 这些哈希用于 check-for-updates 比较

---

## 八、总结：CE/EE 架构分工

### CE 版本提供的能力

1. **完整的数据模型**：所有实体、表结构、字段定义都已就绪
2. **核心业务逻辑**：DataSource、AppVersion、ModuleViewer 的分支感知逻辑完整实现
3. **接口契约**：Controller 和 Service 的接口定义完整
4. **Stub 实现**：抛出 NotFoundException，保证接口可达但功能不可用
5. **注册机制**：通过 registry 模式（如 PlatformGitPullService）供 EE 注入实现
6. **前端 UI**：完整的前端代码，通过 feature flag 和后端响应控制显示

### EE 版本提供的能力

1. **Git 操作实现**：真实的 Git clone、pull、push、branch 管理
2. **序列化/反序列化**：应用定义与 Git 文件格式的互相转换
3. **PR 工作流**：Pull Request 的创建、查看、合并
4. **冲突解决**：Git 合并冲突的处理逻辑
5. **平台级同步**：workspace 级别的 Git 同步（区别于单应用同步）

### 为什么 DataSource/AppVersion/ModuleViewer 在 CE 中有完整分支逻辑？

这些模块的分支感知不是"Git 功能"，而是**架构基础能力**：

1. **数据模型需要提前就位**：数据库表结构、实体关系必须在 CE 中就设计好，否则 EE 启用后数据不兼容
2. **核心业务依赖分支上下文**：即使不用 Git，多分支的数据隔离逻辑也是 EE 功能的基础
3. **渐进式启用**：CE 用户可以看到分支相关字段但不使用，升级到 EE 后无缝激活
4. **模块解析的通用性**：ModuleViewer 的 pin/unpin 机制本身就是有用的功能，不绑定 Git
5. **接口一致性**：前后端的分支参数传递是统一架构，不因 CE/EE 而改变

---

## 附录：关键文件索引

| 模块 | 文件路径 | 说明 |
|------|----------|------|
| **GitSync** | [git-sync/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/git-sync/controller.ts) | Stub 控制器 |
| | [git-sync/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/git-sync/service.ts) | Stub 服务 |
| **AppGit** | [app-git/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-git/controller.ts) | Stub 控制器 |
| | [app-git/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-git/service.ts) | Stub 服务 |
| **WorkspaceBranches** | [workspace-branches/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workspace-branches/controller.ts) | 控制器（路由完整） |
| | [workspace-branches/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/workspace-branches/service.ts) | Stub 服务 |
| **DataSource** | [data-sources/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts) | 完整分支感知逻辑 |
| | [data-sources/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts) | DSV 创建/更新核心逻辑 |
| **AppVersion** | [versions/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts) | 版本服务 |
| | [versions/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts) | 版本工具服务 |
| | [versions/services/create.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/services/create.service.ts) | 版本创建服务 |
| **ModuleViewer** | [versions/module-ref.util.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts) | 模块引用解析核心 |
| **实体** | [app_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts) | 应用版本实体 |
| | [data_source_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source_version.entity.ts) | 数据源版本实体 |
| | [workspace_branch.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/workspace_branch.entity.ts) | 工作区分支实体 |
| | [app_git_sync.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_git_sync.entity.ts) | 应用 Git 同步实体 |
| **注册机制** | [platform-git-pull-registry.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/platform-git-pull-registry.ts) | EE 服务注册 |
| **前端** | [git_sync.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/git_sync.service.js) | 前端 Git 服务 |
| | [workspaceBranchesStore.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_stores/workspaceBranchesStore.js) | 前端分支状态 |
