# ExternalAPI 与 SCIM 对外集成入口安全机制对比分析

## 概述

本文档对 ToolJet 源码中两个对外集成入口——**ExternalAPI** 和 **SCIM** ——在安全开关、License 校验、认证方式、动态模块导入和 CE stub 行为等方面进行全面对比分析。

---

## 一、安全开关对比

### 1.1 ExternalAPI 安全开关

**开关变量**: `ENABLE_EXTERNAL_API`

**位置**: [external-api-security.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/guards/external-api-security.guard.ts#L18-L21)

```typescript
const isConfigEnabled = this.configService.get<string>('ENABLE_EXTERNAL_API') === 'true';
if (!isConfigEnabled) {
  throw new ForbiddenException('External API is disabled');
}
```

**特点**:
- 环境变量级别的总开关
- 值为 `'true'` 时启用
- 关闭时抛出 `ForbiddenException`，消息为 "External API is disabled"

### 1.2 SCIM 安全开关

**开关变量**: `SCIM_ENABLED`

**位置**: [scim-auth.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/guards/scim-auth.guard.ts#L12-L15)

```typescript
const scimEnabled = this.configService.get<string>('SCIM_ENABLED') === 'true';
if (!scimEnabled) {
  throw new UnauthorizedException('SCIM not enabled');
}
```

**特点**:
- 环境变量级别的总开关
- 值为 `'true'` 时启用
- 关闭时抛出 `UnauthorizedException`，消息为 "SCIM not enabled"

### 1.3 对比总结

| 特性 | ExternalAPI | SCIM |
|------|-------------|------|
| 环境变量 | `ENABLE_EXTERNAL_API` | `SCIM_ENABLED` |
| 启用值 | `'true'` | `'true'` |
| 异常类型 | `ForbiddenException` (403) | `UnauthorizedException` (401) |
| 异常消息 | "External API is disabled" | "SCIM not enabled" |

---

## 二、License 校验对比

### 2.1 ExternalAPI License 校验

**License 字段**: `LICENSE_FIELD.EXTERNAL_API` → 值为 `'externalApiEnabled'`

**定义位置**: [licensing/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/index.ts#L126)

**校验位置**: [external-api-security.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/guards/external-api-security.guard.ts#L17-L24)

```typescript
const hasLicense = await this.licenseTermsService.getLicenseTermsInstance(LICENSE_FIELD.EXTERNAL_API);
// ...
if (!hasLicense) {
  throw new ForbiddenException('You do not have access to this resource');
}
```

**Feature 配置**: [external-apis/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/constants/feature.ts)

所有 External API 功能点均配置了 `license: LICENSE_FIELD.EXTERNAL_API`，包括：
- 用户管理（GET_ALL_USERS, GET_USER, CREATE_USER, UPDATE_USER 等）
- 工作空间管理
- 应用管理（GET_ALL_WORKSPACE_APPS, IMPORT_APP, EXPORT_APP 等）
- 组管理（CREATE_GROUP, UPDATE_GROUP, LIST_GROUPS, GET_GROUP, DELETE_GROUP）
- 模块管理（LIST_MODULES, EXPORT_MODULE, IMPORT_MODULE）
- Git 集成相关功能

### 2.2 SCIM License 校验

**License 字段**: `LICENSE_FIELD.SCIM` → 值为 `'scimEnabled'`

**定义位置**: [licensing/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/index.ts#L129)

**Feature 配置**: [scim/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/constants/feature.ts)

所有 SCIM 功能点均配置了 `license: LICENSE_FIELD.SCIM`，包括：
- 用户管理（GET_ALL_USERS, GET_USER, CREATE_USER, UPDATE_USER, PATCH_USER, DELETE_USER）
- 组管理（GET_ALL_GROUPS, GET_GROUP, CREATE_GROUP, UPDATE_GROUP, PATCH_GROUP, DELETE_GROUP）
- SCIM 配置（GET_SP_CONFIG, GET_RESOURCE_TYPES, GET_SCHEMAS, GET_SCHEMA）

**注意**: SCIM 的 `ScimAuthGuard` 本身不直接做 License 校验，License 校验通过 `FeatureAbilityGuard` 实现。

### 2.3 对比总结

| 特性 | ExternalAPI | SCIM |
|------|-------------|------|
| License 字段 | `LICENSE_FIELD.EXTERNAL_API` (`externalApiEnabled`) | `LICENSE_FIELD.SCIM` (`scimEnabled`) |
| 校验位置 | `ExternalApiSecurityGuard` 直接校验 | 通过 `FeatureAbilityGuard` 校验 |
| 校验层级 | 实例级 (`getLicenseTermsInstance`) | 实例级（isPublic 功能） |
| 异常消息 | "You do not have access to this resource" | "Oops! Your current plan doesn't have access to this feature..." |

---

## 三、认证方式对比

### 3.1 ExternalAPI 认证方式

**认证方式**: HTTP Basic Auth

**Token 变量**: `EXTERNAL_API_ACCESS_TOKEN`

**位置**: [external-api-security.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/guards/external-api-security.guard.ts#L26-L31)

```typescript
const authHeader = request.headers['authorization'];
const externalApiAccessToken = this.configService.get<string>('EXTERNAL_API_ACCESS_TOKEN');

if (!authHeader || authHeader !== `Basic ${externalApiAccessToken}`) {
  throw new ForbiddenException('Unauthorized');
}
```

**特点**:
- 仅支持 Basic Auth 一种方式
- Token 直接作为 Basic Auth 的凭证（不是用户名:密码格式，而是直接将 token 放在 Basic 后面）
- 认证失败抛出 `ForbiddenException`，消息为 "Unauthorized"

### 3.2 SCIM 认证方式

**认证方式**: 支持两种认证方式

1. **HTTP Basic Auth**: `SCIM_BASIC_AUTH_USER` / `SCIM_BASIC_AUTH_PASS`
2. **Bearer Token / Header Token**: `SCIM_HEADER_AUTH_TOKEN`

**位置**: [scim-auth.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/guards/scim-auth.guard.ts#L10-L39)

```typescript
// Basic Auth 检查
if (authHeader.startsWith('Basic ')) {
  const base64Credentials = authHeader.split(' ')[1];
  const decoded = Buffer.from(base64Credentials, 'base64').toString('utf8');
  const [username, password] = decoded.split(':');

  const validUser = this.configService.get<string>('SCIM_BASIC_AUTH_USER');
  const validPass = this.configService.get<string>('SCIM_BASIC_AUTH_PASS');

  if (username === validUser && password === validPass) return true;
  throw new UnauthorizedException('Invalid Basic credentials');
}

// Bearer / Header Token 检查
const token = authHeader.split(' ')[1];
const validToken = this.configService.get<string>('SCIM_HEADER_AUTH_TOKEN');

if (token === validToken || authHeader === validToken) return true;
throw new UnauthorizedException('Invalid token');
```

**特点**:
- 支持标准 HTTP Basic Auth（用户名 + 密码格式）
- 支持 Bearer Token 认证
- 支持直接将 token 放在 Authorization header 中（不带 Bearer 前缀）
- 认证失败抛出 `UnauthorizedException`

### 3.3 对比总结

| 特性 | ExternalAPI | SCIM |
|------|-------------|------|
| 认证方式 | Basic Auth（单一 token） | Basic Auth（用户名+密码）+ Bearer Token + 直接 Token |
| 环境变量 | `EXTERNAL_API_ACCESS_TOKEN` | `SCIM_BASIC_AUTH_USER` / `SCIM_BASIC_AUTH_PASS` / `SCIM_HEADER_AUTH_TOKEN` |
| Basic Auth 格式 | `Basic <token>`（直接 token） | `Basic <base64(username:password)>`（标准格式） |
| Bearer Token | 不支持 | 支持（`Bearer <token>` 或直接 `<token>`） |
| 异常类型 | `ForbiddenException` (403) | `UnauthorizedException` (401) |

---

## 四、动态模块导入机制

### 4.1 动态导入核心机制

**核心类**: `SubModule`

**位置**: [app/sub-module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/sub-module.ts)

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

### 4.2 导入路径决定逻辑

**函数**: `getImportPath`

**位置**: [app/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/index.ts#L11-L35)

```typescript
export const getImportPath = async (isGetContext?: boolean, edition?: TOOLJET_EDITIONS) => {
  const repoType = edition || getTooljetEdition() || TOOLJET_EDITIONS.CE;
  let baseDir = 'dist';

  if (isGetContext) {
    const isSrcPresent = await checkIfSrcPresent();
    baseDir = isSrcPresent ? '' : baseDir;
  }

  switch (repoType) {
    case TOOLJET_EDITIONS.CE:
      return `${join(process.cwd(), baseDir, 'src/modules')}`;
    case TOOLJET_EDITIONS.EE:
      return `${join(process.cwd(), baseDir, 'ee')}`;
    case TOOLJET_EDITIONS.Cloud:
      return `${join(process.cwd(), baseDir, 'ee')}`;
    default:
      return `${join(process.cwd(), baseDir, 'src/modules')}`;
  }
};
```

### 4.3 ExternalAPI 模块动态导入

**位置**: [external-apis/module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/module.ts)

```typescript
export class ExternalApiModule extends SubModule {
  static async register(configs?: { IS_GET_CONTEXT: boolean }, isMainImport: boolean = false): Promise<DynamicModule> {
    const {
      ExternalApisController,
      ExternalApisService,
      ExternalApiUtilService,
      ExternalApisAppsController,
      ExternalApisGroupsController,
      ExternalApisModulesController,
    } = await this.getProviders(configs, 'external-apis', [
      'controller',
      'service',
      'util.service',
      'controllers/apps.controller',
      'controllers/groups.controller',
      'controllers/modules.controller',
    ]);
    // ...
  }
}
```

**导入的组件**:
- `controller` - 主控制器（用户相关接口）
- `service` - 主服务
- `util.service` - 工具服务
- `controllers/apps.controller` - 应用控制器
- `controllers/groups.controller` - 组控制器
- `controllers/modules.controller` - 模块控制器

### 4.4 SCIM 模块动态导入

**位置**: [scim/module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/module.ts)

```typescript
export class ScimModule extends SubModule {
  static async register(configs?: { IS_GET_CONTEXT: boolean }, isMainImport: boolean = false): Promise<DynamicModule> {
    const {
      ScimService,
      ScimController,
      ScimUsersController,
      ScimUsersService,
      ScimGroupsController,
      ScimGroupsService,
    } = await this.getProviders(configs, 'scim', [
      'controller',
      'service',
      'controllers/scim-users.controller',
      'services/scim-users.service',
      'controllers/scim-groups.controller',
      'services/scim-groups.service',
    ]);
    // ...
  }
}
```

**导入的组件**:
- `controller` - 主控制器
- `service` - 主服务
- `controllers/scim-users.controller` - SCIM 用户控制器
- `services/scim-users.service` - SCIM 用户服务
- `controllers/scim-groups.controller` - SCIM 组控制器
- `services/scim-groups.service` - SCIM 组服务

**依赖关系**: SCIM 模块依赖 ExternalApiModule

```typescript
imports: [await ExternalApiModule.register(configs), await GroupPermissionsModule.register(configs)],
```

---

## 五、CE Stub 行为分析

### 5.1 ExternalAPI CE Stub

#### 5.1.1 主控制器 Stub

**位置**: [external-apis/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controller.ts)

```typescript
@Controller('ext')
export class ExternalApisController implements IExternalApisController {
  constructor() {}

  @UseGuards(ExternalApiSecurityGuard)
  @Get('users')
  async getAllUsers() {
    throw new NotFoundException();
  }
  // ... 所有方法均抛出 NotFoundException
}
```

**特点**:
- 路由前缀: `/ext`
- 使用 `ExternalApiSecurityGuard` 守卫
- 所有方法均抛出 `NotFoundException`（404）

#### 5.1.2 组控制器 Stub

**位置**: [external-apis/controllers/groups.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/groups.controller.ts)

**关键点**: 使用 `@ee/auth/guards/external-api-security.guard` 导入

```typescript
import { ExternalApiSecurityGuard } from '@ee/auth/guards/external-api-security.guard';
```

**CE 中的含义**:
- tsconfig.json 中配置了路径映射：`"@ee/*": ["ee/*"]`
- 在 CE 环境中，`ee` 目录不存在或为 stub
- TypeScript 编译时会解析到 `ee/auth/guards/external-api-security.guard`
- 但由于 CE 版本没有 ee 目录，实际运行时通过动态导入加载 `src/modules` 下的实现

**Stub 行为**:
- 使用 `@InitFeature(FEATURE_KEY.CREATE_GROUP)` 等装饰器标记功能点
- 使用 `@UseGuards(ExternalApiSecurityGuard)` 应用安全守卫
- 所有方法均抛出 `NotFoundException`

#### 5.1.3 应用控制器 Stub

**位置**: [external-apis/controllers/apps.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/apps.controller.ts)

```typescript
@Controller('ext')
@InitModule(MODULES.EXTERNAL_APIS)
@UseGuards(FeatureAbilityGuard)
export class ExternalApisAppsController implements IExternalApisAppsController {
  pullNewAppFromGit(createMode: string, payload: AppGitPullDto): Promise<any> {
    throw new Error('Method not implemented.');
  }
  // ...
}
```

**特点**:
- 使用 `@InitModule(MODULES.EXTERNAL_APIS)` 标记模块
- 使用 `@UseGuards(FeatureAbilityGuard)` 应用功能权限守卫
- 方法抛出 `Error('Method not implemented.')`

#### 5.1.4 模块控制器 Stub

**位置**: [external-apis/controllers/modules.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/modules.controller.ts)

与应用控制器类似，使用 `FeatureAbilityGuard`，方法抛出 `Error('Method not implemented.')`。

#### 5.1.5 服务 Stub

**位置**: [external-apis/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/service.ts)

```typescript
@Injectable()
export class ExternalApisService implements IExternalApisService {
  constructor() { }
  async getAllUsers(lookupKey?: string, groupNamesString?: string, manager?: EntityManager) {
    throw new Error('Method not implemented.');
  }
  // ...
}
```

### 5.2 SCIM CE Stub

#### 5.2.1 主控制器 Stub

**位置**: [scim/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/controller.ts)

```typescript
@Controller('scim/v2')
@InitModule(MODULES.SCIM)
export class ScimController { }
```

**特点**:
- 路由前缀: `/scim/v2`
- 使用 `@InitModule(MODULES.SCIM)` 标记模块
- 空类，无任何方法

#### 5.2.2 用户控制器 Stub

**位置**: [scim/controllers/scim-users.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/controllers/scim-users.controller.ts)

```typescript
@Controller('scim/v2')
@InitModule(MODULES.SCIM)
export class ScimUsersController { }
```

#### 5.2.3 组控制器 Stub

**位置**: [scim/controllers/scim-groups.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/controllers/scim-groups.controller.ts)

```typescript
@Controller('scim/v2')
@InitModule(MODULES.SCIM)
export class ScimGroupsController { }
```

#### 5.2.4 服务 Stub

**位置**: 
- [scim/services/scim-users.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/services/scim-users.service.ts)
- [scim/services/scim-groups.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/services/scim-groups.service.ts)

```typescript
@Injectable()
export class ScimUsersService { }

@Injectable()
export class ScimGroupsService { }
```

**特点**: 完全空的服务类

### 5.3 对比总结

| 特性 | ExternalAPI CE Stub | SCIM CE Stub |
|------|---------------------|--------------|
| 控制器实现 | 有方法定义，抛出 `NotFoundException` | 空类，无方法 |
| 服务实现 | 有方法定义，抛出 `Error('Method not implemented.')` | 空类 |
| 守卫应用 | 主控制器使用 `ExternalApiSecurityGuard` | 无守卫（在 EE 版本中通过 ScimAuthGuard） |
| 功能标记 | 使用 `@InitFeature` 装饰器 | 使用 `@InitModule` 装饰器 |
| 路由前缀 | `/ext` | `/scim/v2` |

---

## 六、rawBody 解析机制

### 6.1 rawBody 解析器

**函数**: `rawBodyBuffer`

**位置**: [helpers/bootstrap.helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/bootstrap.helper.ts#L38-L42)

```typescript
export function rawBodyBuffer(req: any, res: any, buf: Buffer, encoding: BufferEncoding) {
  if (buf && buf.length) {
    req.rawBody = buf.toString(encoding || 'utf8');
  }
}
```

### 6.2 全局启用

**位置**: [main.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/main.ts)

```typescript
app.use(json({ verify: rawBodyBuffer, limit: maxSize }));
app.use(urlencoded({
  verify: rawBodyBuffer,
  // ...
}));
```

### 6.3 与 ExternalAPI / SCIM 的关系

**CSRF 豁免**:

**位置**: [helpers/bootstrap.helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/bootstrap.helper.ts#L305-L315)

```typescript
const exemptPrefixes = [
  '/health',
  '/api/health',
  '/jobs',
  '/api/v2/webhooks/',
  '/api/organization/payment/webhooks',
  '/api/scim/',
  '/api/ext/',
  '/api/sso/saml/',
  '/api/oauth/saml/',
];
```

**说明**:
- ExternalAPI (`/api/ext/`) 和 SCIM (`/api/scim/`) 都被豁免了 CSRF Origin 检查
- 因为这两个接口使用 API Token 认证而非 Cookie 认证
- rawBody 本身不是专门为 SCIM 或 ExternalAPI 设计的，而是全局启用的
- rawBody 主要用于 webhook 签名验证场景（如 Stripe webhook）

---

## 七、FeatureAbilityGuard 保护机制

### 7.1 AbilityGuard 基类

**位置**: [app/guards/ability.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/guards/ability.guard.ts)

**核心流程**:

1. **获取模块和功能信息**
   ```typescript
   const module = cloneDeep(this.reflector.get<MODULES>('tjModuleId', context.getClass()));
   let features = cloneDeep(this.reflector.get<string[]>('tjFeatureId', context.getHandler()));
   ```

2. **License 校验**
   ```typescript
   const licenseRequired: LICENSE_FIELD = featureInfo?.license;
   if (licenseRequired && featureInfo.isPublic && !(app?.organizationId || user?.organizationId)) {
     // 公共功能 + 无组织ID -> 检查实例级 License
     if (!(await this.licenseTermsService.getLicenseTermsInstance(licenseRequired))) {
       throw new HttpException('...', 451);
     }
   }
   ```

3. **公共功能快速通过**
   ```typescript
   if (featureInfo.isPublic) {
     return true;
   }
   ```

4. **用户权限校验（基于 Ability）**
   ```typescript
   const ability = await abilityFactory.createAbility(user, { moduleName: module, features }, resourceArray, request);
   const forbiddenFeature = features.find(
     (feature: string) => !ability.can(feature, this.getSubjectType(), resourceId || undefined)
   );
   ```

### 7.2 ExternalAPI FeatureAbilityGuard

**位置**: [external-apis/ability/guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/ability/guard.ts)

```typescript
@Injectable()
export class FeatureAbilityGuard extends AbilityGuard {
  protected getAbilityFactory() {
    return FeatureAbilityFactory;
  }

  protected getSubjectType() {
    return User;
  }
}
```

**应用范围**:
- `ExternalApisAppsController` - 应用相关接口
- `ExternalApisModulesController` - 模块相关接口

**保护的功能点**（来源: [external-apis/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/constants/feature.ts)）:

| 类别 | 功能点 |
|------|--------|
| 用户 | GET_ALL_USERS, GET_USER, CREATE_USER, UPDATE_USER, REPLACE_USER_WORKSPACES, UPDATE_USER_WORKSPACE, GET_ALL_WORKSPACES, UPDATE_USER_ROLE, UPDATE_USER_METADATA, GET_USER_METADATA |
| 应用 | GET_ALL_WORKSPACE_APPS, IMPORT_APP, EXPORT_APP, PULL_NEW_APP, PULL_EXISTING_APP, PUSH_APP_VERSION, CREATE_ORG_GIT, AUTO_RELEASE_APP, SAVE_APP_VERSION |
| 组 | CREATE_GROUP, UPDATE_GROUP, LIST_GROUPS, GET_GROUP, DELETE_GROUP |
| 模块 | LIST_MODULES, EXPORT_MODULE, IMPORT_MODULE |
| 其他 | GENERATE_PAT, VALIDATE_PAT_SESSION |

**所有功能点均为 `isPublic: true`**，意味着：
- License 校验通过后直接放行
- 不进行用户级别的权限校验
- 因为 External API 使用 Token 认证而非用户会话

### 7.3 SCIM FeatureAbilityGuard

**位置**: [scim/ability/guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/ability/guard.ts)

```typescript
@Injectable()
export class FeatureAbilityGuard extends AbilityGuard {
  protected getAbilityFactory() {
    return FeatureAbilityFactory;
  }

  protected getSubjectType() {
    return User;
  }
}
```

**保护的功能点**（来源: [scim/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/constants/feature.ts)）:

| 类别 | 功能点 |
|------|--------|
| 用户 | GET_ALL_USERS, GET_USER, CREATE_USER, UPDATE_USER, PATCH_USER, DELETE_USER |
| 组 | GET_ALL_GROUPS, GET_GROUP, CREATE_GROUP, UPDATE_GROUP, PATCH_GROUP, DELETE_GROUP |
| 配置 | GET_SP_CONFIG, GET_RESOURCE_TYPES, GET_SCHEMAS, GET_SCHEMA |

**所有功能点均为 `isPublic: true`**，与 ExternalAPI 相同。

### 7.4 两层守卫机制

#### ExternalAPI 的两层守卫

1. **第一层**: `ExternalApiSecurityGuard`（方法级）
   - 检查 `ENABLE_EXTERNAL_API` 开关
   - 检查 `LICENSE_FIELD.EXTERNAL_API` License
   - 检查 `EXTERNAL_API_ACCESS_TOKEN` 认证

2. **第二层**: `FeatureAbilityGuard`（控制器类级，仅 Apps 和 Modules 控制器）
   - 检查功能级别的 License
   - 由于 `isPublic: true`，License 通过后直接放行

> **注意**: 主控制器（用户接口）和组控制器只使用 `ExternalApiSecurityGuard`，不使用 `FeatureAbilityGuard`。而应用和模块控制器使用 `FeatureAbilityGuard`，但由于 isPublic，实际只做 License 校验。

#### SCIM 的两层守卫（EE 版本推测）

1. **第一层**: `ScimAuthGuard`
   - 检查 `SCIM_ENABLED` 开关
   - 检查认证（Basic Auth 或 Token）

2. **第二层**: `FeatureAbilityGuard`
   - 检查 `LICENSE_FIELD.SCIM` License
   - 由于 `isPublic: true`，License 通过后直接放行

---

## 八、`@ee/auth/guards/external-api-security.guard` 导入在 CE 源码中的含义

### 8.1 路径映射配置

**位置**: [server/tsconfig.json](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/tsconfig.json#L23-L26)

```json
"paths": {
  "@ee/*": ["ee/*"],
  // ...
}
```

### 8.2 导入位置

**文件**: [external-apis/controllers/groups.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/groups.controller.ts#L18)

```typescript
import { ExternalApiSecurityGuard } from '@ee/auth/guards/external-api-security.guard';
```

### 8.3 CE 版本中的含义

这是 ToolJet CE/EE 双版本架构的典型设计模式：

1. **TypeScript 编译层面**:
   - `@ee/*` 路径映射到 `ee/*` 目录
   - 在 CE 仓库中，`ee` 目录可能不存在或为占位符
   - 但由于动态导入机制，实际运行时不依赖这个静态导入

2. **运行时动态导入**:
   - `SubModule.getProviders()` 根据版本动态加载模块
   - CE 版本从 `src/modules/external-apis/controllers/groups.controller` 加载
   - EE 版本从 `ee/external-apis/controllers/groups.controller` 加载
   - 每个版本的控制器导入对应版本的 guard

3. **CE 版本的实际行为**:
   - CE 版本的 groups.controller.ts 静态导入 `@ee/auth/guards/external-api-security.guard`
   - 但 CE 版本还有一个位于 `src/modules/auth/guards/external-api-security.guard.ts` 的实现
   - 这是因为主 controller.ts 从 `@modules/auth/guards/external-api-security.guard` 导入

4. **设计意图**:
   - 保持代码结构一致性
   - EE 版本可以有更复杂的安全校验逻辑
   - CE 版本的 stub 导入 `@ee/` 路径是为了与 EE 版本的文件结构保持一致
   - 实际运行时通过动态模块加载决定使用哪个版本的实现

### 8.4 矛盾点解释

观察发现：
- [external-apis/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controller.ts#L5) 从 `@modules/auth/guards/external-api-security.guard` 导入
- [external-apis/controllers/groups.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/groups.controller.ts#L18) 从 `@ee/auth/guards/external-api-security.guard` 导入

这可能是因为：
1. 主控制器是较早实现的，使用 CE 版本路径
2. groups 控制器是较新实现的，遵循了 EE 优先的导入模式
3. 在 EE 版本中，两个文件都会从 `@ee/` 路径导入
4. CE 版本中，`@ee/auth/guards/external-api-security.guard` 实际上可能解析不到，但由于动态模块导入机制，运行时会替换整个 controller 文件

---

## 九、综合对比表

| 维度 | ExternalAPI | SCIM |
|------|-------------|------|
| **路由前缀** | `/api/ext/` | `/api/scim/v2/` |
| **安全开关** | `ENABLE_EXTERNAL_API` | `SCIM_ENABLED` |
| **License 字段** | `LICENSE_FIELD.EXTERNAL_API` (`externalApiEnabled`) | `LICENSE_FIELD.SCIM` (`scimEnabled`) |
| **认证方式** | Basic Auth（单一 token） | Basic Auth（用户名+密码）+ Bearer Token |
| **环境变量** | `EXTERNAL_API_ACCESS_TOKEN` | `SCIM_BASIC_AUTH_USER` / `SCIM_BASIC_AUTH_PASS` / `SCIM_HEADER_AUTH_TOKEN` |
| **认证守卫** | `ExternalApiSecurityGuard` | `ScimAuthGuard` |
| **权限守卫** | `FeatureAbilityGuard`（Apps/Modules 控制器） | `FeatureAbilityGuard` |
| **功能公开性** | 所有功能 `isPublic: true` | 所有功能 `isPublic: true` |
| **CSRF 豁免** | 是 (`/api/ext/`) | 是 (`/api/scim/`) |
| **CE Stub 控制器** | 有方法，抛 404 | 空类 |
| **CE Stub 服务** | 有方法，抛 Error | 空类 |
| **模块依赖** | 无 | 依赖 ExternalApiModule |
| **接口范围** | 用户、组、应用、模块、Git、PAT | 用户、组、SCIM 配置 |

---

## 十、接口保护链路总结

### 10.1 ExternalAPI 用户/组接口保护链路

```
请求 → ExternalApiSecurityGuard
  ├─ 检查 ENABLE_EXTERNAL_API 开关
  ├─ 检查 LICENSE_FIELD.EXTERNAL_API License
  └─ 检查 EXTERNAL_API_ACCESS_TOKEN (Basic Auth)
        → 全部通过 → 执行业务逻辑（EE 版本）
        → 失败 → 抛出 403 Forbidden
```

**CE 版本**: 守卫通过后，控制器方法抛出 `NotFoundException`。

### 10.2 ExternalAPI 应用/模块接口保护链路

```
请求 → FeatureAbilityGuard (类级)
  ├─ 检查模块/功能信息
  ├─ 检查功能对应的 License
  └─ isPublic: true → 直接放行（不校验用户权限）
        → 执行业务逻辑（EE 版本）
```

> **注意**: Apps 和 Modules 控制器的 CE 版本代码中未看到 `ExternalApiSecurityGuard`，只有 `FeatureAbilityGuard`。这可能意味着 EE 版本会在动态加载时添加额外的安全守卫。

### 10.3 SCIM 接口保护链路（推测 EE 版本）

```
请求 → ScimAuthGuard
  ├─ 检查 SCIM_ENABLED 开关
  └─ 检查认证（Basic Auth 或 Bearer Token）
        → 通过 → FeatureAbilityGuard
              ├─ 检查功能对应的 License
              └─ isPublic: true → 直接放行
                    → 执行业务逻辑（EE 版本）
        → 失败 → 抛出 401 Unauthorized
```

**CE 版本**: 控制器和服务均为 stub，无实际功能。

---

## 附录：关键文件索引

| 文件 | 说明 |
|------|------|
| [external-api-security.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/guards/external-api-security.guard.ts) | ExternalAPI 安全守卫（CE 版本） |
| [scim-auth.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/guards/scim-auth.guard.ts) | SCIM 认证守卫 |
| [external-apis/controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controller.ts) | ExternalAPI 主控制器（CE Stub） |
| [external-apis/controllers/groups.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/groups.controller.ts) | ExternalAPI 组控制器（CE Stub） |
| [external-apis/controllers/apps.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/apps.controller.ts) | ExternalAPI 应用控制器（CE Stub） |
| [external-apis/controllers/modules.controller.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/controllers/modules.controller.ts) | ExternalAPI 模块控制器（CE Stub） |
| [external-apis/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/constants/feature.ts) | ExternalAPI 功能配置 |
| [scim/constants/feature.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/constants/feature.ts) | SCIM 功能配置 |
| [external-apis/module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/external-apis/module.ts) | ExternalAPI 模块定义 |
| [scim/module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/scim/module.ts) | SCIM 模块定义 |
| [app/sub-module.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/sub-module.ts) | 动态模块导入基类 |
| [app/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/index.ts) | 版本判断与导入路径 |
| [app/guards/ability.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/guards/ability.guard.ts) | 能力守卫基类 |
| [licensing/constants/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/licensing/constants/index.ts) | License 字段定义 |
| [helpers/bootstrap.helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/helpers/bootstrap.helper.ts) | rawBody 解析与安全配置 |
| [tsconfig.json](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/tsconfig.json) | TypeScript 路径映射配置 |
