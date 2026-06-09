# FeatureAbilityGuard 权限系统分析

## 概述

FeatureAbilityGuard 是 ToolJet 权限系统的核心守卫，基于 NestJS 的 `CanActivate` 接口实现，结合 CASL (Ability) 权限库，对受保护的 API 请求进行多层权限校验。

---

## 一、FeatureAbilityGuard 请求处理流程

### 1.1 装饰器元数据读取

请求进入守卫后，首先从装饰器元数据中读取模块和功能信息：

**模块元数据 (class 级别)**

通过 `@InitModule(MODULES.APP)` 装饰器在控制器类上设置，使用 key `'tjModuleId'` 存储。

- 装饰器定义: [init-module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/decorators/init-module.ts)
- 模块枚举: [modules.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/modules.ts)

```typescript
export const InitModule = (moduleId: MODULES) => SetMetadata('tjModuleId', moduleId);
```

**功能元数据 (method 级别)**

通过 `@InitFeature(FEATURE_KEY.CREATE)` 装饰器在路由方法上设置，使用 key `'tjFeatureId'` 存储。

- 装饰器定义: [init-feature.decorator.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/decorators/init-feature.decorator.ts)

```typescript
export const InitFeature = (featureId: any) => SetMetadata('tjFeatureId', featureId);
```

**读取逻辑**

在 [ability.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/guards/ability.guard.ts#L45-L46) 的 `canActivate` 方法中：

```typescript
const module = cloneDeep(this.reflector.get<MODULES>('tjModuleId', context.getClass()));
let features = cloneDeep(this.reflector.get<string[]>('tjFeatureId', context.getHandler()));
```

如果 features 不是数组，会被包装成数组。如果没有 features，直接返回 `false` 拒绝访问。

### 1.2 License 校验

License 校验是第一道关卡，遍历所有 features 逐一检查：

**Feature 配置结构**

每个 feature 在 `MODULE_INFO` 中都有对应的配置，类型为 `FeatureConfig`：

- 配置定义: [types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/types.ts#L17-L25)
- 模块信息总表: [module-info.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/module-info.ts)

```typescript
export interface FeatureConfig {
  license?: LICENSE_FIELD;        // 关联的 License 字段
  isPublic?: boolean;             // 是否公开功能
  isSuperAdminFeature?: boolean;  // 是否超级管理员功能
  shouldNotSkipPublicApp?: boolean; // 公开应用时是否仍需校验
  auditLogsKey?: string;          // 审计日志 key
  skipAuditLogs?: boolean;
  allowFailedAuditLogs?: boolean;
}
```

**License 校验逻辑**

在 [ability.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/guards/ability.guard.ts#L63-L95) 中：

```typescript
for (const feature of features) {
  const featureInfo: FeatureConfig = MODULE_INFO?.[module]?.[feature];
  
  const licenseRequired: LICENSE_FIELD = featureInfo?.license;
  
  // 场景1: 公开功能 + 无组织ID → 检查实例级 License
  if (licenseRequired && featureInfo.isPublic && !(app?.organizationId || user?.organizationId)) {
    if (!(await this.licenseTermsService.getLicenseTermsInstance(licenseRequired))) {
      throw new HttpException('...', 451);
    }
  }
  
  // 场景2: 无 License 要求 + 无组织ID → 直接通过
  if (licenseRequired && !(app?.organizationId || user?.organizationId)) {
    return true;
  }
  
  // 场景3: 组织级 License 检查
  if (licenseRequired && !(await this.licenseTermsService.getLicenseTerms(
    licenseRequired,
    app?.organizationId || user?.organizationId
  ))) {
    throw new HttpException('...', 451);
  }
}
```

License 服务实现: [terms.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/services/terms.service.ts)

### 1.3 公开功能 (isPublic) 检查

如果 feature 配置了 `isPublic: true`，跳过所有后续权限校验，直接返回 `true`。

```typescript
if (featureInfo.isPublic) {
  return true;
}
```

示例: 健康检查接口 [features.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/features.ts)

### 1.4 超级管理员功能 (isSuperAdminFeature) 检查

如果 feature 配置了 `isSuperAdminFeature: true`，但当前用户不是超级管理员，则抛出 `ForbiddenException`。

```typescript
if (featureInfo.isSuperAdminFeature && !isSuperAdmin(user)) {
  throw new ForbiddenException({
    message: 'You do not have permission to access this resource',
    organizationId: app?.organizationId,
  });
}
```

### 1.5 公开应用 (publicApp) 检查

应用级别的公开检查，需要结合 `shouldNotSkipPublicApp` 配置：

```typescript
// 应用公开且不需要跳过公开应用检查 → 直接通过
if (app?.isPublic && !featureInfo.shouldNotSkipPublicApp) {
  return true;
}

// 应用公开但需要跳过公开应用检查且用户不存在 → 拒绝
if (app?.isPublic && featureInfo.shouldNotSkipPublicApp && !user) {
  return false;
}
```

示例: `VALIDATE_PRIVATE_APP_ACCESS` 功能配置了 `shouldNotSkipPublicApp: true`，即使应用公开也需要校验。

参考: [apps/constants/features.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/constants/features.ts#L17)

### 1.6 用户缺失处理

遍历完所有 features 后，如果没有用户且应用不公开，直接拒绝：

```typescript
if (!user && !app?.isPublic) {
  return false;
}
```

### 1.7 AbilityFactory 构建与权限计算

如果用户存在，通过 AbilityFactory 构建 CASL Ability 对象：

```typescript
const abilityFactory = await this.moduleRef.resolve(this.getAbilityFactory());
const ability = await abilityFactory.createAbility(
  user,
  { moduleName: module, features },
  resourceArray,
  request
);
```

**AbilityFactory 基类**

定义在 [ability-factory.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/ability-factory.ts)

核心流程：

1. 调用 `abilityService.resourceActionsPermission()` 从数据库加载用户权限（会缓存到 `request.tj_user_permissions`）
2. 解析出 `superAdmin`、`isAdmin`、`isBuilder`、`isEndUser` 等角色标识
3. 调用各模块实现的 `defineAbilityFor()` 方法，根据权限数据定义 CASL 规则

**各模块 AbilityFactory 实现**

每个模块都有自己的 FeatureAbilityFactory，例如：

- Apps 模块: [apps/ability/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/ability/index.ts)
- Licensing 模块: [licensing/ability/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/ability/index.ts)
- Feature 模块: [feature/ability/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/feature/ability/index.ts)

**Apps 模块的权限定义示例**

[apps/ability/app.ability.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/ability/app.ability.ts) 中 `defineAppAbility` 函数：

- **Admin/SuperAdmin**: 拥有所有权限
- **普通用户**: 根据 `isAllAppsEditable`、`editableAppsId`、`isAllAppsViewable`、`viewableAppsId` 等确定权限
- **应用所有者 (isAppOwner)**: 额外拥有删除权限
- **环境访问**: 合并 `environmentAccess`（默认）和 `appSpecificEnvironmentAccess`（应用特定）的权限（UNION 逻辑）

### 1.8 资源级权限校验 (resourceId)

构建完 Ability 后，使用 `ability.can()` 对每个 feature 进行校验：

```typescript
const resourceId = request.tj_resource_id;

const forbiddenFeature = features.find(
  (feature: string) => !ability.can(feature, this.getSubjectType(), resourceId || undefined)
);

if (forbiddenFeature) {
  throw new ForbiddenException({
    message: errorMessage || 'You do not have permission to access this resource',
    organizationId: app?.organizationId,
  });
}
```

`resourceId` 通过 `request.tj_resource_id` 传递，通常由前置的守卫/中间件（如 `ValidAppGuard`）设置。

---

## 二、Session 权限组合逻辑

Session 返回给前端的权限数据由多层权限组合而成，主要通过 `AbilityService.resourceActionsPermission()` 方法计算。

核心服务: [ability/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/service.ts)

### 2.1 权限数据结构

[ability/types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/types.ts)

```typescript
export interface UserPermissions {
  // 角色标识
  isAdmin: boolean;
  isSuperAdmin: boolean;
  isBuilder: boolean;
  isEndUser: boolean;
  
  // 全局操作权限
  appCreate: boolean;
  appDelete: boolean;
  appPromote: boolean;
  appRelease: boolean;
  dataSourceCreate: boolean;
  dataSourceDelete: boolean;
  folderCreate: boolean;
  folderDelete: boolean;
  // ...
  
  // 资源级权限
  [MODULES.APP]?: UserAppsPermissions;
  [MODULES.GLOBAL_DATA_SOURCE]?: UserDataSourcePermissions;
  [MODULES.FOLDER]?: UserFolderPermissions;
}
```

### 2.2 Group Permissions 基础

用户的权限基于其所属的用户组（GroupPermissions），通过 `getUserPermissionsQuery` 查询：

[ability/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/util.service.ts#L91-L132)

查询逻辑：
- 从 `group_permissions` 表出发
- 通过 `group_users` 关联用户
- 左连接 `granular_permissions` 获取细粒度权限
- 根据资源类型进一步连接 apps / data sources / folders 的权限表

### 2.3 默认角色判定

[ability/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/service.ts#L53-L88)

```typescript
// 1. 检查是否有 Admin 组
const adminGroup = permissions.some((group) => group.name === USER_ROLE.ADMIN);
userPermissions.isAdmin = adminGroup;

// 2. 非 Admin 则判定是 Builder 还是 EndUser
if (!adminGroup) {
  const isBuilder = await this.abilityUtilService.isBuilder(user);
  if (isBuilder) {
    userPermissions.isBuilder = true;
  } else {
    userPermissions.isEndUser = true;
  }
}
```

**角色层级关系**：
- **Admin**: 拥有所有权限
- **Builder**: 可以创建和编辑应用
- **EndUser**: 只能查看和使用应用

### 2.4 全局权限聚合

使用 reduce 对所有 group 的权限进行 OR 聚合：

[ability/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/service.ts#L56-L76)

```typescript
const userPermissions: UserPermissions = permissions.reduce((acc, group) => {
  return {
    appCreate: acc.appCreate || group.appCreate,
    appDelete: acc.appDelete || group.appDelete,
    appRelease: acc.appRelease || group.appRelease,
    dataSourceCreate: acc.dataSourceCreate || group.dataSourceCreate,
    // ... 更多字段
  };
}, DEFAULT_USER_PERMISSIONS);
```

只要用户在任何一个组中拥有某项权限，就拥有该项权限（UNION 逻辑）。

### 2.5 App 权限组合

App 权限是最复杂的，由多层来源组合而成，核心方法是 `createUserAppsPermissions`：

[ability/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/util.service.ts#L183-L409)

#### 2.5.1 组成部分

App 权限由以下几部分组合而成：

1. **默认组权限 (isAll)**  
   用户组配置了"所有应用"权限时生效

2. **自定义组权限 (granular permissions)**  
   用户组配置了特定应用列表的权限

3. **用户拥有的应用 (Owner)**  
   用户自己创建的应用自动获得编辑权限

4. **文件夹权限推导**  
   用户拥有文件夹权限 → 文件夹内的应用也获得相应权限

#### 2.5.2 默认组权限 (isAll = true)

```typescript
defaultGroupPermissions.forEach((permission) => {
  const appsPermission = permission?.appsGroupPermissions;
  
  userAppsPermissions.isAllEditable ||= appsPermission.canEdit;
  userAppsPermissions.isAllViewable ||= appsPermission.canView;
  userAppsPermissions.hideAll ||= appsPermission.hideFromDashboard;
  
  // 环境访问权限也使用 UNION 逻辑
  userAppsPermissions.environmentAccess.development ||= appsPermission.canAccessDevelopment;
  userAppsPermissions.environmentAccess.staging ||= appsPermission.canAccessStaging;
  userAppsPermissions.environmentAccess.production ||= appsPermission.canAccessProduction;
  userAppsPermissions.environmentAccess.released ||= appsPermission.canAccessReleased;
});
```

#### 2.5.3 自定义组权限 (isAll = false)

```typescript
customGroupPermissions.forEach((permission) => {
  const appsPermission = permission?.appsGroupPermissions;
  const groupApps = appsPermission?.groupApps?.map(item => item.appId) || [];
  
  if (appsPermission.canEdit) {
    userAppsPermissions.editableAppsId.push(...groupApps);
  }
  if (appsPermission.canView) {
    userAppsPermissions.viewableAppsId.push(...groupApps);
  }
  
  // 应用特定的环境访问权限
  for (const appId of groupApps) {
    const existing = userAppsPermissions.appSpecificEnvironmentAccess[appId];
    existing.development ||= appsPermission.canAccessDevelopment;
    // ... staging, production, released
  }
});
```

#### 2.5.4 用户拥有的应用 (Owner)

```typescript
const appsOwnedByUser = await manager.find(AppBase, {
  where: {
    userId: user.id,
    organizationId: user.organizationId,
    type: APP_TYPES.FRONT_END,
  },
});
const appsIdOwnedByUser = appsOwnedByUser.map(app => app.id);
userAppsPermissions.editableAppsId = Array.from(
  new Set([...userAppsPermissions.editableAppsId, ...appsIdOwnedByUser])
);
```

用户自己创建的应用自动加入可编辑列表。

#### 2.5.5 文件夹权限推导

文件夹权限会进一步推导为应用权限：

**层级关系**：
- `canEditFolder` > `canEditApps` > `canViewApps`
- 高级权限隐含所有低级权限

```typescript
// 1. 用户拥有的文件夹 → 文件夹内的应用可编辑
const ownedFolderApps = await manager
  .createQueryBuilder(FolderApp, 'folderApp')
  .innerJoin('folderApp.folder', 'folder')
  .where('folder.createdBy = :userId', { userId: user.id })
  .select('folderApp.appId')
  .getMany();

// 2. 用户有编辑权限的文件夹 → 文件夹内的应用可编辑
// 3. 用户有查看权限的文件夹 → 文件夹内的应用可查看

// 推导的应用获得完整环境访问权限
for (const appId of folderDerivedAppIds) {
  userAppsPermissions.appSpecificEnvironmentAccess[appId] = {
    development: true,
    staging: true,
    production: true,
    released: true,
  };
}
```

#### 2.5.6 环境访问权限的合并

环境访问权限有两个层级，使用 UNION (OR) 逻辑合并：

- **默认环境权限** (`environmentAccess`): 来自 isAll=true 的组
- **应用特定环境权限** (`appSpecificEnvironmentAccess`): 来自细粒度权限和文件夹推导

```typescript
static canAccessAppInEnvironment(permissions, appId, environment): boolean {
  const appSpecificAccess = permissions.appSpecificEnvironmentAccess?.[appId]?.[environment] ?? false;
  const defaultAccess = permissions.environmentAccess?.[environment] ?? false;
  return appSpecificAccess || defaultAccess;
}
```

### 2.6 DataSource 权限组合

[ability/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/service.ts#L167-L172)

```typescript
async createUserDataSourcesPermissions(
  dataSourcesGranularPermissions: GranularPermissions[]
): Promise<UserDataSourcePermissions> {
  const userDataSourcesPermissions = { ...DEFAULT_USER_DATA_SOURCE_PERMISSIONS };
  return userDataSourcesPermissions;
}
```

注意：Community Edition 中 builder 可以使用所有 datasource：

```typescript
if (userPermissions.isBuilder) {
  userPermissions[MODULES.GLOBAL_DATA_SOURCE].isAllUsable = true;
}
```

### 2.7 Folder 权限组合

[ability/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/service.ts#L123-L165)

```typescript
createUserFolderPermissions(folderGranularPermissions): UserFolderPermissions {
  const userFolderPermissions = {
    ...DEFAULT_USER_FOLDER_PERMISSIONS,
    editableFoldersId: [],
    viewableFoldersId: [],
    editAppsInFoldersId: [],
  };

  for (const gp of folderGranularPermissions) {
    const folderPerms = gp.foldersGroupPermissions;
    
    // 权限层级: canEditFolder > canEditApps > canViewApps
    const canEditFolder = folderPerms.canEditFolder;
    const canEditApps = folderPerms.canEditFolder || folderPerms.canEditApps;
    const canViewApps = folderPerms.canEditFolder || folderPerms.canEditApps || folderPerms.canViewApps;

    if (gp.isAll) {
      // 全部文件夹权限
      if (canEditFolder) userFolderPermissions.isAllEditable = true;
      if (canViewApps) userFolderPermissions.isAllViewable = true;
      if (canEditApps) userFolderPermissions.isAllEditApps = true;
    } else {
      // 特定文件夹权限
      const folderIds = folderPerms.groupFolders?.map(gf => gf.folderId) || [];
      if (canEditFolder) userFolderPermissions.editableFoldersId.push(...folderIds);
      if (canViewApps) userFolderPermissions.viewableFoldersId.push(...folderIds);
      if (canEditApps) userFolderPermissions.editAppsInFoldersId.push(...folderIds);
    }
  }
  
  // 去重
  userFolderPermissions.editableFoldersId = [...new Set(userFolderPermissions.editableFoldersId)];
  // ...
  return userFolderPermissions;
}
```

### 2.8 Session 返回的权限数据

在登录和切换组织时，通过 `getPermissionDataToAuthorize` 方法获取权限数据：

[session/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts#L184-L249)

```typescript
async getPermissionDataToAuthorize(user, manager) {
  const groupPermissions = await this.getAllGroupsOfUser(user, manager);
  const userPermissions = await this.abilityService.resourceActionsPermission(
    user,
    {
      organizationId: user.organizationId,
      resources: [
        { resource: MODULES.APP },
        { resource: MODULES.GLOBAL_DATA_SOURCE },
        { resource: MODULES.FOLDER },
      ],
    },
    manager
  );

  let role = await this.rolesRepository.getUserRole(user.id, user.organizationId, manager);
  // 超级管理员没有组织角色时，获取 admin 角色
  if (superAdmin && !role) {
    role = await this.rolesRepository.getAdminRoleOfOrganization(user.organizationId, manager);
  }

  return {
    id: user.id,
    email: user.email,
    admin: isAdmin,
    superAdmin,
    appGroupPermissions: userPermissions[MODULES.APP],
    dataSourceGroupPermissions: userPermissions[MODULES.GLOBAL_DATA_SOURCE],
    folderGroupPermissions: userPermissions[MODULES.FOLDER],
    role,
    groupPermissions,
    userPermissions,
  };
}
```

---

## 三、完整校验流程图

```
请求到达
  │
  ▼
读取装饰器元数据
  ├─ class 级: tjModuleId (InitModule)
  └─ method 级: tjFeatureId (InitFeature)
  │
  ▼
License 校验
  ├─ 公开功能且无组织ID → 实例级License
  ├─ 有组织ID → 组织级License
  └─ 无License且无组织ID → 直接通过
  │
  ▼
isPublic 功能? ──是──→ 通过 ✓
  │
  ▼ 否
isSuperAdminFeature?
  ├─ 是 + 非超管 → 拒绝 ✗
  └─ 否 (或超管) → 继续
  │
  ▼
app.isPublic?
  ├─ 是 + !shouldNotSkipPublicApp → 通过 ✓
  ├─ 是 + shouldNotSkipPublicApp + 无user → 拒绝 ✗
  └─ 否 → 继续
  │
  ▼
无user且app非公开? ──是──→ 拒绝 ✗
  │
  ▼ 否 (有user)
构建 Ability (AbilityFactory)
  ├─ 从 DB 加载 group permissions
  ├─ 判定角色 (Admin/Builder/EndUser)
  ├─ 聚合全局权限 (OR 逻辑)
  ├─ 计算 App 权限 (组权限 + owner + 文件夹推导)
  ├─ 计算 DataSource 权限
  └─ 计算 Folder 权限
  │
  ▼
资源级权限校验 (ability.can)
  ├─ 有 resourceId → 校验特定资源
  └─ 无 resourceId → 校验全局权限
  │
  ▼
全部通过? ──是──→ 通过 ✓
  └─ 否 → 拒绝 ✗ (ForbiddenException)
```

---

## 四、关键文件索引

| 模块 | 文件路径 | 说明 |
|------|---------|------|
| 守卫基类 | [ability.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/guards/ability.guard.ts) | AbilityGuard 核心逻辑 |
| Ability 工厂基类 | [ability-factory.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/ability-factory.ts) | AbilityFactory 抽象基类 |
| 模块装饰器 | [init-module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/decorators/init-module.ts) | @InitModule 装饰器 |
| 功能装饰器 | [init-feature.decorator.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/decorators/init-feature.decorator.ts) | @InitFeature 装饰器 |
| 模块枚举 | [modules.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/modules.ts) | MODULES 枚举 |
| 模块信息总表 | [module-info.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/module-info.ts) | MODULE_INFO 配置 |
| Feature 配置类型 | [types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/types.ts) | FeatureConfig 接口 |
| License 服务 | [terms.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/services/terms.service.ts) | LicenseTermsService |
| Ability 服务 | [ability/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/service.ts) | AbilityService 核心实现 |
| Ability 工具服务 | [ability/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/util.service.ts) | 权限计算工具方法 |
| 权限类型定义 | [ability/types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/ability/types.ts) | UserPermissions 等类型 |
| Session 工具服务 | [session/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts) | Session 权限生成 |
| Apps 守卫 | [apps/ability/guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/ability/guard.ts) | FeatureAbilityGuard (Apps版) |
| Apps Ability | [apps/ability/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/ability/index.ts) | FeatureAbilityFactory (Apps版) |
| Apps 权限定义 | [apps/ability/app.ability.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/apps/ability/app.ability.ts) | defineAppAbility 函数 |
