# AppVersion 保存/自动保存与编辑权限分析

## 概述

本文档分析 ToolJet 后端在处理前端保存/自动保存 AppVersion 时，如何根据版本状态、当前环境 priority、MULTI_ENVIRONMENT License、Git Sync allowEditing、APP_JS_LIBRARIES License 和模块草稿依赖等因素，决定是否允许编辑、发布或提升环境。同时详细说明 getVersion 响应中各关键字段的改写逻辑。

---

## 一、核心 API 入口

### 1.1 保存版本
- **接口**: `PUT /v2/apps/:id/versions/:versionId`
- **控制器**: [VersionControllerV2.updateVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/controller.v2.ts#L49-L54)
- **服务**: [VersionService.update()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L299-L359)

### 1.2 自动保存
自动保存通过多个细分端点处理不同类型的更改：
- 组件增删改
- 页面增删改
- 全局设置更新
- 页面设置更新

前端服务: [appVersion.service.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/appVersion.service.js#L123-L173)

### 1.3 获取版本
- **接口**: `GET /v2/apps/:id/versions/:versionId`
- **服务**: [VersionService.getVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L166-L266)

### 1.4 提升环境
- **接口**: `PUT /v2/apps/:id/versions/:versionId/promote`
- **服务**: [VersionService.promoteVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L383-L451)

---

## 二、版本状态与类型

### 2.1 版本状态 (AppVersionStatus)
定义在 [app_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts#L14-L18)

| 状态 | 说明 |
|------|------|
| `DRAFT` | 草稿状态，可编辑 |
| `PUBLISHED` | 已保存/已发布，通常不可编辑 |
| `RELEASED` | 已发布（对外发布） |

### 2.2 版本类型 (AppVersionType)
定义在 [app_version.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_version.entity.ts#L28-L31)

| 类型 | 说明 |
|------|------|
| `VERSION` | 普通版本 |
| `BRANCH` | 分支版本（Git 功能分支） |

---

## 三、保存时的权限检查逻辑

### 3.1 保存核心流程

`VersionService.update()` 方法的检查流程：

```
1. 开启数据库事务
2. 查找当前版本
3. 如果是发布操作 (status=PUBLISHED) 且不是 module：
   → 检查模块草稿依赖 (checkDraftModulesInApp)
4. 如果版本已发布 (status=PUBLISHED)：
   → 只允许修改名称和描述
   → 如果启用了组织级 Git Sync，连名称和描述都不能改
5. 执行版本更新
6. 提交事务
```

### 3.2 发布时的模块草稿依赖检查

**方法**: [VersionUtilService.checkDraftModulesInApp()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/util.service.ts#L266-L325)

**触发条件**: 当保存时 `status === PUBLISHED` 且应用类型不是 `module`

**检查逻辑**: 遍历版本中所有 ModuleViewer 组件，解析每个模块引用，遇到以下情况阻止保存：

| 匹配类型 | 阻止原因 | 错误信息 |
|----------|----------|----------|
| `no-row` | 模块不存在或没有可用版本 | Module "xxx" has no saved version yet. Save the module first. |
| `orphan-fallback` | 模块版本 ID 无效（找不到对应行） | Module "xxx" pin is invalid. Pin a saved version. |
| `unpinned-fallback` | 模块未指定版本（跟随最新草稿） | Module "xxx" has active draft pinned. Pin a saved version. |
| `pin-hit + DRAFT` | 指定的版本处于草稿状态 | Module "xxx" version "yyy" is still in draft. Save the module first. |

**模块解析机制** 详见 [module-ref.util.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts)

### 3.3 已发布版本的编辑限制

**代码位置**: [VersionService.update()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L309-L322)

当版本状态为 `PUBLISHED` 时：
- **仅允许修改**: 名称 (name) 和描述 (description)
- 如果启用了组织级 Git Sync (`organizationGit?.isEnabled`)：
  - **连名称和描述都不能改**
  - 抛出异常: `Cannot edit name or description of a saved version.`

---

## 四、编辑器冻结 (freezeEditor) 判定逻辑

### 4.1 getVersion 中的 freezeEditor 计算

**代码位置**: [VersionService.getVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L191-L235)

`shouldFreezeEditor` 由以下因素共同决定，**任一条件满足即为 true**：

#### 因素 1：环境 priority + MULTI_ENVIRONMENT License

```
if (hasMultiEnvLicense) {
  currentEnvironment = 获取版本当前环境
  shouldFreezeEditor = currentEnvironment.priority > 1
}
```

- **有 MULTI_ENVIRONMENT License**: 根据当前环境的 priority 判断
  - `priority === 1` (development): 不冻结 → 可编辑
  - `priority > 1` (staging/production): 冻结 → 不可编辑

- **无 MULTI_ENVIRONMENT License**: 强制使用 development 环境，不冻结

相关代码: [service.ts#L193-L206](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L193-L206)

#### 因素 2：Git Sync allowEditing

```
if (appGit) {
  shouldFreezeEditor = !appGit.allowEditing || shouldFreezeEditor
}
```

- 如果应用配置了 Git Sync (`app_git_sync` 记录存在)
- 且 `allowEditing === false`
- 则冻结编辑器

**注意**: 对于分支版本 (BRANCH type)，不受 canonical appGit 的 allowEditing 限制（分支副本默认可编辑）。

相关代码: [service.ts#L224-L232](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L224-L232)

#### 因素 3：版本状态

```
if (appVersion?.status === AppVersionStatus.PUBLISHED) {
  shouldFreezeEditor = true
}
```

- 版本状态为 `PUBLISHED` 时，冻结编辑器

相关代码: [service.ts#L233-L235](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L233-L235)

### 4.2 shouldFreezeEditor 综合判断表

| 版本状态 | 环境 priority | Multi-Env License | Git Sync allowEditing | 结果 (冻结?) |
|----------|--------------|-------------------|----------------------|-------------|
| DRAFT | 1 | 有 | true | 否 |
| DRAFT | 1 | 有 | false | 是 |
| DRAFT | 2 | 有 | true | 是 |
| DRAFT | 1 | 无 | true | 否 |
| PUBLISHED | 任意 | 任意 | 任意 | 是 |

### 4.3 另一种 freezeEditor 判定 (AppsUtilService)

[AppsUtilService.shouldFreezeEditor()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/util.service.ts#L924-L954) 提供了更细粒度的判定，考虑了版本类型和工作区分支：

- `PUBLISHED` 状态 → 冻结
- `VERSION` 类型 + `DRAFT` 状态 + 无分支 → 不冻结
- `VERSION` 类型 + 非 DRAFT 状态 → 冻结
- 启用了工作区分支时，`VERSION` 类型的草稿也冻结（编辑必须在功能分支上进行）
- `BRANCH` 类型 → 可编辑（不受 allowEditing 限制）
- 其他情况受 `appGit.allowEditing` 影响

---

## 五、getVersion 响应字段改写详解

### 5.1 响应结构概览

`getVersion()` 返回的响应结构：

```javascript
{
  ...appData,               // 应用基本信息
  editing_version: { ... }, // 编辑中的版本（camelCase 格式）
  pages: [...],             // 页面列表
  events: [...],            // 事件列表
  should_freeze_editor: boolean,  // 是否冻结编辑器
  modules: [...]            // 引用的模块列表
}
```

### 5.2 currentEnvironment 字段改写

**代码位置**: [service.ts#L183-L189](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L183-L189) 和 [service.ts#L203-L206](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L203-L206)

**改写规则**:

| 场景 | 处理方式 |
|------|----------|
| 无 MULTI_ENVIRONMENT License | 强制设置为 development 环境 (priority=1) 的 ID |
| 有 MULTI_ENVIRONMENT License | 保留版本原有的 currentEnvironmentId |

**实现方式**:
```javascript
if (!hasMultiEnvLicense) {
  const developmentEnv = await getByPriority(user.organizationId)
  appCurrentEditingVersion['currentEnvironmentId'] = developmentEnv.id
}
```

### 5.3 theme 字段改写

**代码位置**: [service.ts#L219-L236](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L219-L236)

**改写规则**:
1. 从 `editingVersion.globalSettings.theme.id` 读取主题 ID
2. 调用 `organizationThemesUtilService.getTheme()` 获取完整主题信息
3. 用完整主题对象替换 `globalSettings.theme`

```javascript
const appTheme = await organizationThemesUtilService.getTheme(
  user.organizationId,
  editingVersion?.globalSettings?.theme?.id
)
editingVersion['globalSettings']['theme'] = appTheme
```

### 5.4 JS 库字段改写 (libraries / preloadedScript)

**代码位置**: [service.ts#L239-L247](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L239-L247)

**License 字段**: `LICENSE_FIELD.APP_JS_LIBRARIES` (`appJsLibrariesEnabled`)

**改写规则**:

| APP_JS_LIBRARIES License | globalSettings.libraries | globalSettings.preloadedScript |
|-------------------------|--------------------------|--------------------------------|
| 有 (true) | 保留原值 | 保留原值 |
| 无 (false) | **删除该字段** | **删除该字段** |

```javascript
const hasJsLibrariesAccess = await licenseTermsService.getLicenseTerms(
  LICENSE_FIELD.APP_JS_LIBRARIES,
  app.organizationId
)
if (!hasJsLibrariesAccess) {
  delete editingVersion['globalSettings']['libraries']
  delete editingVersion['globalSettings']['preloadedScript']
}
```

**设计意图**: 前端直接加载后端返回的字段，因此后端通过删除字段来控制功能可用性，作为许可控制的安全闸门。

### 5.5 modules 字段改写

**代码位置**: [service.ts#L258-L263](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L258-L263)

**改写规则**:
1. 调用 `appUtilService.fetchModules(app, false, undefined)` 获取应用引用的所有模块
2. 对每个模块调用 `prepareResponse()`（与主应用相同的处理逻辑）
3. 每个模块都应用相同的规则：
   - currentEnvironment 改写
   - freezeEditor 计算
   - theme 注入
   - JS 库字段控制

**模块获取逻辑**: [AppsUtilService.fetchModules()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/util.service.ts#L781-L860)

- 从版本的所有 ModuleViewer 组件中提取模块 ID
- 按 `co_relation_id` 查询模块应用
- 支持分支感知：如果父应用在功能分支上，模块也使用对应分支的版本

---

## 六、提升环境 (Promote) 的权限检查

### 6.1 提升环境核心流程

**方法**: [VersionService.promoteVersion()](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/service.ts#L383-L451)

### 6.2 检查项

| 检查项 | 条件 | 异常 |
|--------|------|------|
| MULTI_ENVIRONMENT License | 必须有此 License | `You do not have permissions to perform this action` |
| 版本状态 | 不能是 DRAFT | `You cannot promote a draft version. Please save the version before promoting.` |
| 环境一致性 | 请求的 currentEnvironmentId 必须与版本当前环境一致 | `NotAcceptableException` (406) |

### 6.3 提升逻辑

通过所有检查后：
1. 查找下一个 priority 的环境 (`priority > currentEnvironment.priority`，按 priority 升序取第一个)
2. 更新版本的 `currentEnvironmentId` 为下一个环境
3. 如果 `promotedFrom` 有值，设为 null
4. 更新 `updatedAt`

---

## 七、相关 License 字段汇总

| License 字段 | 枚举值 | 影响范围 |
|-------------|--------|----------|
| `MULTI_ENVIRONMENT` | `multiEnvironmentEnabled` | 环境切换、编辑器冻结、环境提升 |
| `APP_JS_LIBRARIES` | `appJsLibrariesEnabled` | JS 库功能（libraries、preloadedScript） |
| `GIT_SYNC` | `gitSyncEnabled` | Git Sync 功能 |
| `MODULES` | `modulesEnabled` | 模块功能 |

定义在 [licensing/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/index.ts#L94-L148)

---

## 八、Git Sync 相关字段

### 8.1 AppGitSync 实体

定义在 [app_git_sync.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/app_git_sync.entity.ts)

关键字段：
- `allowEditing`: boolean - 是否允许在 Git Sync 启用时编辑应用
- `appId`: string - 关联的应用 ID
- `organizationGitId`: string - 关联的组织级 Git Sync 配置

### 8.2 对编辑器的影响

- `allowEditing === false` → 冻结编辑器
- 分支版本 (BRANCH type) 不受此限制
- 模块同样受此规则约束（在 prepareResponse 中统一处理）

---

## 九、模块草稿依赖深度解析

### 9.1 模块版本解析机制

定义在 [module-ref.util.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/versions/module-ref.util.ts)

ModuleViewer 组件存储两个关键属性：
- `moduleAppId`: 模块的 `co_relation_id`（稳定标识）
- `moduleVersionId`: 版本的 `module_reference_id`（UUID）

#### 解析优先级：

1. **UUID pin 匹配** → 优先匹配消费者分支上的行，再匹配默认分支
2. **版本名称匹配** → 匹配 version_type='version' 的行
3. **分支名称简写** → 匹配默认分支上的版本
4. **orphan fallback** → pin 存在但无匹配，回退到最新版本
5. **unpinned fallback** → pin 为空，使用最新版本
6. **no-row** → 无可用版本

### 9.2 保存/发布/提升时的模块检查

| 操作 | 检查方法 | 检查内容 |
|------|----------|----------|
| 保存 (PUBLISHED) | `checkDraftModulesInApp` | 所有引用的模块不能是 DRAFT 状态 |
| 提升环境 | `checkModulesPromotableToEnvironment` | 所有引用的模块必须已提升到目标环境或更高 |
| 发布 (RELEASE) | `checkModulesReleasedInApp` | 所有引用的模块必须是已发布版本 |

---

## 十、总结

### 10.1 编辑权限决策树

```
是否允许编辑？
├── 版本状态 = PUBLISHED? → 否
├── 有 MULTI_ENVIRONMENT License?
│   ├── 是 → 环境 priority > 1? → 是 → 否
│   └── 否 → 继续
├── 有 Git Sync 配置?
│   ├── 是 → allowEditing = false? → 是 → 否
│   └── 否 → 继续
├── 版本类型 = BRANCH? → 是 → 是
└── 默认 → 是
```

### 10.2 getVersion 字段改写总结

| 字段 | 改写因素 | 改写方式 |
|------|----------|----------|
| `should_freeze_editor` | 版本状态 + 环境 priority + License + Git Sync | 布尔值计算 |
| `editing_version.currentEnvironmentId` | MULTI_ENVIRONMENT License | 无 License 时强制为 dev |
| `editing_version.globalSettings.theme` | 组织主题配置 | ID 替换为完整主题对象 |
| `editing_version.globalSettings.libraries` | APP_JS_LIBRARIES License | 无 License 时删除字段 |
| `editing_version.globalSettings.preloadedScript` | APP_JS_LIBRARIES License | 无 License 时删除字段 |
| `modules` | 应用中引用的所有模块 | 每个模块应用相同规则 |

### 10.3 发布与提升的前置条件

| 操作 | 版本状态要求 | License 要求 | 模块要求 |
|------|-------------|-------------|----------|
| 保存 (DRAFT → PUBLISHED) | DRAFT | - | 所有模块不能是 DRAFT |
| 提升环境 | PUBLISHED | MULTI_ENVIRONMENT | 所有模块已提升到目标环境 |
| 发布 (RELEASE) | - | - | 所有模块必须是已发布版本 |
