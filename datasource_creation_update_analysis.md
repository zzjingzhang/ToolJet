# ToolJet 全局 DataSource 创建/更新流程深度分析

## 1. 核心数据模型

ToolJet 使用三层数据模型来管理全局数据源（Global DataSource）的版本化和环境隔离：

### 1.1 实体关系图

```
DataSource (数据源主表)
    │
    └── DataSourceVersion (数据源版本 - 按分支隔离)
            │  ├── isDefault: 是否为默认版本（主分支快照）
            │  ├── branchId: 关联的工作区分支 ID
            │  └── versionFromId: 来源版本 ID（克隆溯源）
            │
            └── DataSourceVersionOptions (版本选项 - 按环境隔离)
                    ├── environmentId: 环境 ID
                    └── options: JSON 格式的配置选项
```

### 1.2 各实体关键字段

#### DataSource ([data_source.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source.entity.ts))
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `co_relation_id` | string | 关联 ID，用于 Git Sync 序列化时标识数据源 |
| `name` | string | 数据源名称 |
| `kind` | string | 数据源类型（如 restapi, postgresql 等） |
| `type` | enum | `static` / `default` / `sample` |
| `scope` | enum | `local` / `global` - 本文关注 `global` 范围 |
| `organizationId` | string | 组织 ID |
| `pluginId` | string | 关联的插件 ID |

#### DataSourceVersion ([data_source_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source_version.entity.ts))
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `dataSourceId` | string | 关联的数据源 ID |
| `branchId` | string | 关联的工作区分支 ID（可为 null） |
| `isDefault` | boolean | 是否为默认版本（主分支快照） |
| `isActive` | boolean | 是否激活（软删除标记） |
| `name` | string | 版本名称 |
| `versionFromId` | string | 来源版本 ID（克隆溯源） |

**唯一约束**：`(dataSourceId, branchId)` 组合唯一

#### DataSourceVersionOptions ([data_source_version_options.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source_version_options.entity.ts))
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `dataSourceVersionId` | string | 关联的数据源版本 ID |
| `environmentId` | string | 环境 ID |
| `options` | json | 配置选项（包含 encrypted、workspace_constant 等标记） |

**唯一约束**：`(dataSourceVersionId, environmentId)` 组合唯一

---

## 2. 创建 DataSource 的流程

创建逻辑主要位于 [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L70-L144) 的 `create()` 方法中。

### 2.1 前置验证

在创建之前，会调用 `validateNoConstantsOnDefaultBranch()` 进行验证：

- **验证逻辑**：如果在 options 中检测到 `workspace_constant` 标记，或加密字段的值包含 `{{constants...}}` 或 `{{secrets...}}` 引用
- **验证规则**：在默认分支（master/main）上不允许直接添加工作区常量，必须通过 PR 流程
- **异常抛出**：如果违反规则，抛出 `BadRequestException`

参见：[validateNoConstantsOnDefaultBranch](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L44-L68)

### 2.2 创建主数据源记录

无论分支情况如何，都会先创建 `DataSource` 主记录：

1. 生成唯一名称
2. 设置 `scope = global`
3. 设置 `co_relation_id = id`（用于 Git Sync 序列化）

### 2.3 分支感知的版本创建

根据是否传入 `branchId`，创建逻辑分为两条路径：

#### 路径 A：有 branchId（Feature Branch 模式）

**代码位置**：[util.service.ts#L103-L115](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L103-L115)

调用 `createDataSourceVersionForBranchWithOptions()` 方法：

- **只创建分支特定的 DSV**，不创建默认 DSV
- 默认 DSV 会在分支合并到 main 时通过 Git pull/反序列化创建
- 这样设计的目的是防止数据源在分支合并前出现在 main 分支上

具体步骤：
1. 创建 `DataSourceVersion`，设置 `branchId = 当前分支ID`，`isActive = true`
2. 为当前选中环境创建 `DataSourceVersionOptions`，写入完整的解析后选项
3. 为其他环境创建 `DataSourceVersionOptions`，但敏感数据会被重置（`resetSecureData = true`）

#### 路径 B：无 branchId（默认分支 / 无 Git Sync 模式）

**代码位置**：[util.service.ts#L116-L140](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L116-L140)

1. 调用 `createDataSourceInAllEnvironments()` 创建默认 DSV + 所有环境的 DSVO
2. 为选中环境写入完整选项
3. 为其他环境写入敏感数据被重置的选项

---

## 3. 更新 DataSource 的流程

更新逻辑主要位于 [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L278-L512) 的 `update()` 方法中。

### 3.1 关键变量：shouldUpdateDefault

**代码位置**：[util.service.ts#L313-L325](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L313-L325)

`shouldUpdateDefault` 是决定数据写入路径的核心变量：

```typescript
let shouldUpdateDefault = true;
if (branchId) {
  const branch = await manager.findOne(WorkspaceBranch, {
    where: { id: branchId },
    select: ['id', 'isDefault'],
  });
  shouldUpdateDefault = !!branch?.isDefault;
}
```

**判定规则**：
- `branchId` 为空 → `shouldUpdateDefault = true`（无 Git Sync 或许可证过期）
- `branchId` 是默认分支 → `shouldUpdateDefault = true`
- `branchId` 是 feature branch → `shouldUpdateDefault = false`

**作用**：
- `isDefault = true` 的 DSV 是 "主分支快照"
- 只有在默认分支上编辑时才更新默认 DSV
- Feature branch 的编辑不应改变主分支的回退版本

### 3.2 多环境感知的更新路径

根据 `MULTI_ENVIRONMENT` 许可证状态，更新分为两条主路径：

#### 路径 A：多环境已启用（isMultiEnvEnabled = true）

**代码位置**：[util.service.ts#L327-L402](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L327-L402)

1. 只处理当前选中的环境（envToUpdate）
2. 如果 `shouldUpdateDefault = true`，更新默认 DSV 的选项
3. 如果有 `branchId`（全局数据源），同时更新分支 DSV 的选项
   - 如果分支 DSV 不存在，会自动创建（从默认 DSV 克隆）
   - 克隆时会**复制 credential 记录**，确保分支之间不共享凭证引用

#### 路径 B：多环境未启用（isMultiEnvEnabled = false）

**代码位置**：[util.service.ts#L403-L484](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L403-L484)

1. 遍历所有环境
2. 每个环境都执行相同的更新逻辑
3. 同样遵循 `shouldUpdateDefault` 规则
4. 这样设计是为了用户后续购买企业版时能无缝迁移

### 3.3 分支 DSV 的自动创建与克隆

当在 feature branch 上更新一个已存在的数据源（该数据源是在 Git Sync 启用前创建的），会自动创建分支 DSV：

**代码位置**：[util.service.ts#L351-L395](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L351-L395)

克隆流程：
1. 查找默认 DSV（`isDefault = true`）
2. 遍历所有环境，从默认 DSV 的 DSVO 复制 options
3. **关键**：对加密字段（有 `credential_id` 且 `encrypted = true`），会创建新的 credential 记录
   - 读取原始 credential 的值
   - 创建新的 credential 记录
   - 用新的 `credential_id` 替换旧的
4. 保存新的 DSVO

**为什么要克隆 credential？**
- 避免分支之间共享凭证引用
- 确保分支上的修改（如更新密码）不会影响主分支
- 实现真正的分支隔离

---

## 4. 关键字段与配置的影响分析

### 4.1 encrypted 字段

**位置**：options 对象中的每个配置项都有 `encrypted` 布尔标记

**作用**：
- 标记该配置项是否需要加密存储
- 为 `true` 时，值会存储在独立的 `credentials` 表中，options 中只保存 `credential_id`
- 为 `false` 时，值直接以明文存储在 options 的 `value` 字段中

**数据结构**：
```typescript
// 加密字段
{
  credential_id: "uuid",
  encrypted: true,
  workspace_constant?: "{{constants.db_password}}"  // 可选
}

// 非加密字段
{
  value: "some-value",
  encrypted: false
}
```

**相关方法**：
- 创建时：[parseOptionsForCreate()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L167-L204)
- 更新时：[parseOptionsForUpdate()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L529-L628)

### 4.2 workspace_constant 字段

**位置**：options 对象中，与 `encrypted` 配合使用

**作用**：
- 标记该加密字段的值是一个工作区常量引用（如 `{{constants.db_host}}` 或 `{{secrets.api_key}}`）
- 值不会被直接加密存储，而是保存常量引用字符串
- 运行时通过 `resolveConstants()` 方法解析为实际值

**验证约束**：
- `validateNoConstantsOnDefaultBranch()` 确保默认分支上不能直接添加常量
- 必须通过 PR 流程（feature branch → merge）来添加常量

**解析时机**：
- 测试连接时解析：[testConnection()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L754-L816)
- 调用查询方法时解析：[parseSourceOptions()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L1122-L1189)

### 4.3 OAuth 相关字段

OAuth 数据源有特殊的处理逻辑，涉及 `tokenData`、`access_token`、`refresh_token` 等字段。

**关键特性**：

1. **Token 获取**：
   - 通过 `authorizeOauth2()` 方法获取访问令牌
   - 令牌可以是单用户模式或多用户模式（`multiple_auth_enabled`）

2. **Token 刷新与传播**：
   - 当 OAuth 令牌刷新时，会调用 `propagateTokenToAllBranches()` 方法
   - **重要**：令牌会传播到所有分支版本，而不仅限于当前分支
   - 这是因为 OAuth 令牌被认为是"分支不变的"（branch-invariant）

   参见：[propagateTokenToAllBranches()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L1248-L1270)

3. **多用户认证**：
   - `multiple_auth_enabled = true` 时，tokenData 是一个数组，每个元素包含 `user_id`
   - 每个用户有自己的 OAuth 令牌

### 4.4 MULTI_ENVIRONMENT License

**许可证字段**：`LICENSE_FIELD.MULTI_ENVIRONMENT` = `'multiEnvironmentEnabled'`

参见：[licensing/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/index.ts#L110)

**对数据写入的影响**：

| 场景 | 多环境启用 | 多环境禁用 |
|------|-----------|-----------|
| 更新范围 | 只更新指定环境 | 更新所有环境 |
| 环境切换 | 不同环境有独立配置 | 所有环境配置相同 |
| 凭证管理 | 每个环境有独立的 credential | 所有环境共享相同的 credential 值 |

**检查位置**：
- 更新时：[util.service.ts#L300-L303](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L300-L303)
- 读取时：[app-environments/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-environments/util.service.ts)

### 4.5 validateNoConstantsOnDefaultBranch

**代码位置**：[util.service.ts#L44-L68](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L44-L68)

**触发时机**：
- 创建数据源时（如果有 branchId）
- 更新数据源时（如果有 branchId）

**检测条件**（满足任一即触发）：
1. option 有 `workspace_constant` 属性
2. option 是 `encrypted` 且值包含 `{{constants` 或 `{{secrets`

**约束目的**：
- 遵循 PRD 要求：加密字段中的工作区常量必须经过 PR 工作流
- 防止直接在 master 分支上添加敏感常量引用
- 强制通过代码审查流程来管理敏感配置

### 4.6 branchId 和 environmentId 的组合影响

`branchId` 和 `environmentId` 是决定数据写入路径的两个核心参数，它们的组合产生了 2×2 = 4 种基本场景：

| 场景 | branchId | environmentId | 写入目标 |
|------|----------|---------------|----------|
| 无 Git Sync + 单环境 | 无 | 有（默认） | 默认 DSV 的默认环境 DSVO |
| 无 Git Sync + 多环境 | 无 | 有 | 默认 DSV 的指定环境 DSVO |
| Feature Branch + 单环境 | 有 | 有（默认） | 分支 DSV 的默认环境 DSVO |
| Feature Branch + 多环境 | 有 | 有 | 分支 DSV 的指定环境 DSVO |

**读取路径**（以 `getOptions()` 为例）：
参见：[app-environments/util.service.ts#L236-L313](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-environments/util.service.ts#L236-L313)

1. 如果有 `branchId`，先尝试从分支 DSV 读取
2. 如果分支 DSV 不存在或没有对应环境的选项，回退到默认 DSV
3. 默认 DSV 是 "最终回退" 版本

---

## 5. 数据写入路径决策树

### 5.1 创建时的决策流程

```
创建 DataSource
     │
     ├── 有 branchId？
     │    ├── 是 → Feature Branch 模式
     │    │    └── 创建分支 DSV + 所有环境 DSVO
     │    │         （不创建默认 DSV，合并时再创建）
     │    └── 否 → 默认模式
     │         └── 创建默认 DSV + 所有环境 DSVO
     │
     └── 前置检查：validateNoConstantsOnDefaultBranch
          （仅当有 branchId 且是默认分支时检查）
```

### 5.2 更新时的决策流程

```
更新 DataSource
     │
     ├── 计算 shouldUpdateDefault
     │    ├── 无 branchId → true
     │    ├── 是默认分支 → true
     │    └── 是 feature branch → false
     │
     ├── 多环境是否启用？
     │    ├── 是 → 只更新指定环境
     │    │    ├── shouldUpdateDefault = true
     │    │    │   └── 更新默认 DSV 的该环境 DSVO
     │    │    └── 有 branchId
     │    │         ├── 分支 DSV 不存在 → 从默认 DSV 克隆（含 credential 克隆）
     │    │         └── 更新分支 DSV 的该环境 DSVO
     │    └── 否 → 更新所有环境
     │         ├── 遍历所有环境
     │         ├── shouldUpdateDefault = true → 更新默认 DSV
     │         └── 有 branchId → 更新分支 DSV
     │
     └── 更新 DataSource 主表的 name 等字段
          └── 如果 shouldUpdateDefault，同步更新默认 DSV 的 name
```

---

## 6. 特殊场景分析

### 6.1 从无 Git Sync 到有 Git Sync 的迁移

当一个组织启用 Git Sync 功能时，已存在的全局数据源只有默认 DSV。当用户第一次在 feature branch 上编辑该数据源时：

1. 系统检测到分支 DSV 不存在
2. 自动从默认 DSV 克隆出分支 DSV
3. 克隆时会复制所有环境的 DSVO
4. **关键**：加密字段的 credential 会被重新创建（不共享引用）

参见：[util.service.ts#L355-L395](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L355-L395)

### 6.2 OAuth Token 的跨分支传播

与普通配置项不同，OAuth 令牌在刷新后会传播到所有分支：

**原因**：
- OAuth 令牌代表用户的授权状态
- 令牌过期后刷新是全局事件，不应该只影响一个分支
- 避免每个分支都需要重新进行 OAuth 授权

**实现**：
- `propagateTokenToAllBranches()` 方法遍历所有激活的 DSV
- 更新每个 DSV 对应环境的 DSVO 中的 `tokenData`
- 这是少数几个会跨分支写入的操作之一

### 6.3 删除操作的分支感知

删除全局数据源时，行为也受分支影响：

参见：[service.ts#L205-L251](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts#L205-L251)

- **有 branchId（feature branch）**：软删除，设置 `DataSourceVersion.isActive = false`
- **无 branchId（默认分支）**：硬删除，直接删除 DataSource 记录

---

## 7. 关键代码文件索引

| 文件 | 作用 |
|------|------|
| [data_source.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source.entity.ts) | DataSource 实体定义 |
| [data_source_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source_version.entity.ts) | DataSourceVersion 实体定义 |
| [data_source_version_options.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/data_source_version_options.entity.ts) | DataSourceVersionOptions 实体定义 |
| [util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts) | 核心业务逻辑（创建/更新/解析） |
| [service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts) | 服务层入口 |
| [controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/controller.ts) | API 控制器 |
| [repository.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/repository.ts) | 数据访问层 |
| [app-environments/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-environments/util.service.ts) | 环境工具服务（选项读写） |
| [workspace_branch.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/workspace_branch.entity.ts) | 工作区分支实体 |
| [licensing/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/index.ts) | 许可证常量定义 |
