# 会话与邀请链接状态决策分析文档

## 概述

本文档分析 ToolJet 系统在**会话已存在**或**邀请链接打开**时，如何根据不同的 organization/user 状态决定返回以下响应类型：

- 正常 session payload
- invite signup payload（邀请注册）
- workspace signup 状态
- archived 组织错误
- 需要重新登录

涉及的核心服务：
- [SessionService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts) — 会话主逻辑
- [SessionUtilService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts) — 权限数据与 payload 生成
- [AuthService](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/service.ts) — 认证与邀请登录分支
- [InvitedUserSessionAuthGuard](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/guards/invited-user-session.guard.ts) — 邀请会话守卫
- [JwtStrategy](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/jwt/jwt.strategy.ts) — JWT 验证策略

---

## 一、核心状态常量

### 用户状态 (USER_STATUS)
定义于 [lifecycle.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/users/constants/lifecycle.ts#L41-L46)

| 状态 | 说明 |
|------|------|
| `INVITED` | 已邀请，未激活 |
| `VERIFIED` | 已验证邮箱，未激活 |
| `ACTIVE` | 活跃用户 |
| `ARCHIVED` | 已归档 |

### 工作空间用户状态 (WORKSPACE_USER_STATUS)
定义于 [lifecycle.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/users/constants/lifecycle.ts#L124-L128)

| 状态 | 说明 |
|------|------|
| `INVITED` | 已邀请加入工作空间 |
| `ACTIVE` | 工作空间活跃成员 |
| `ARCHIVED` | 已从工作空间归档 |

### 工作空间状态 (WORKSPACE_STATUS)
定义于 [lifecycle.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/users/constants/lifecycle.ts#L36-L39)

| 状态 | 说明 |
|------|------|
| `ACTIVE` | 活跃 |
| `ARCHIVE` | 已归档 |

### 来源 (SOURCE / WORKSPACE_USER_SOURCE)
定义于 [lifecycle.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/users/constants/lifecycle.ts#L15-L29)

| 来源 | 说明 |
|------|------|
| `SIGNUP` | 自主注册 |
| `INVITE` | 被邀请 |
| `WORKSPACE_SIGNUP` | 通过工作空间注册 |

---

## 二、正常会话获取流程 (GET /session)

### 入口
- Controller: [SessionController.getSessionDetails](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/controller.ts#L38-L48)
- Guard: `SessionAuthGuard` (JWT 验证)
- Service: [SessionService.getSessionDetails](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts#L68-L104)

### 决策流程

```
用户携带 JWT Cookie 请求 GET /session
        │
        ▼
  SessionAuthGuard (JWT 验证)
        │
        ├─→ 会话失效/过期 ──→ 401 Unauthorized（需重新登录）
        │
        ▼
  getSessionDetails(user, workspaceSlug, appId)
        │
        ├─→ 有 workspaceSlug 或 appData.organizationId?
        │     │
        │     ├─→ 是：查找组织
        │     │     │
        │     │     ├─→ 组织不存在 ──→ 404 NotFound
        │     │     │
        │     │     └─→ 组织存在：检查是否为活跃成员
        │     │           │
        │     │           ├─→ 是活跃成员/应用公开/超管 ──→ currentOrganization = 组织
        │     │           │
        │     │           └─→ 非活跃成员且非公开应用 ──→ currentOrganization = null
        │     │
        │     └─→ 否：currentOrganization = null
        │
        ▼
  sessionUtilService.generateSessionPayload()
        │
        ▼
  返回正常 session 数据
```

### generateSessionPayload 输出字段
定义于 [SessionUtilService.generateSessionPayload](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts#L386-L423)

| 字段 | 含义 | 决定因素 |
|------|------|----------|
| `id / email / firstName / lastName` | 用户基本信息 | 直接来自 user 对象 |
| `noWorkspaceAttachedInTheSession` | 无可用工作空间 | 用户无任何活跃工作空间 **且** 不是超级管理员 |
| `isAllWorkspacesArchived` | 所有工作空间都已归档 | 超级管理员视角：所有非归档状态的工作空间数为 0 |
| `currentOrganizationId` / `currentOrganizationSlug` / `currentOrganizationName` | 当前工作空间 | 优先级：传入组织 > 默认组织 > 第一个组织 |
| `isFirstUserOnboardingCompleted` | 首用户引导是否完成 | 首用户且实例未 onboarded 时检查 |
| `isOnboardingCompleted` | 引导是否完成 | 目前恒为 `true` |

---

## 三、邀请会话验证流程 (POST /session/invited-user-session)

### 入口
- Controller: [SessionController.getInvitedUserSessionDetails](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/controller.ts#L31-L36)
- Guard: `InvitedUserSessionAuthGuard`
- Service: [SessionService.validateInvitedUserSession](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts#L106-L198)

### Guard 层前置检查 (InvitedUserSessionAuthGuard)

[InvitedUserSessionAuthGuard.canActivate](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/guards/invited-user-session.guard.ts#L37-L140)

```
请求 POST /session/invited-user-session
    (body: { organizationToken, accountToken })
        │
        ▼
  有 organizationToken?
        │
        ├─→ 否 ──→ 400 BadRequest("缺少 workspace invite token")
        │
        ▼
  根据 token 查找邀请用户
        │
        ├─→ 用户不存在 ──→ 400 BadRequest { isInvalidInvitationUrl: true }
        │
        ▼
  邀请的组织是否已归档?
        │
        ├─→ 是 ──→ 400 BadRequest { isWorkspaceArchived: true }
        │
        ▼
  是纯组织邀请(仅 organizationToken)且用户未激活?
        │
        ├─→ 是 ──→ 400 BadRequest { accountIsNotActivatedYet: true }
        │
        ▼
  尝试 JWT 验证 (super.canActivate)
        │
        ├─→ 成功 ──→ 检查当前会话用户与邀请用户 email 是否匹配
        │     │
        │     ├─→ 不匹配 ──→ 403 Forbidden { isInvitationTokenAndUserMismatch: true }
        │     │
        │     └─→ 匹配 ──→ 继续，检查登录方法兼容性
        │           (password login 但组织未启用 form login → 403)
        │
        └─→ 失败 (Unauthorized) ──→ 调用 onInvalidSession()
              │
              ├─→ 有 accountToken 且用户状态为 INVITED/VERIFIED
              │     └─→ 返回 invitedUser（通过验证，进入邀请注册流程）
              │
              ├─→ organizationUserSource === SIGNUP (工作空间注册来源)
              │     └─→ 返回 invitedUser（通过验证）
              │
              └─→ 其他情况
                    └─→ 检查组织状态 → 归档则 400
                    └─→ 403 Forbidden（需重新登录接受邀请）
```

### onInvalidSession 决策
[InvitedUserSessionAuthGuard.onInvalidSession](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/guards/invited-user-session.guard.ts#L142-L166)

| 条件 | 结果 | 说明 |
|------|------|------|
| 有 `accountToken` + 用户状态为 `INVITED`/`VERIFIED` | ✅ 通过，返回 invitedUser | 用户未激活，走邀请注册流程 |
| `organizationUserSource === SIGNUP` | ✅ 通过，返回 invitedUser | 工作空间注册来源，允许绕过 |
| 其他情况（用户已激活但无有效会话） | ❌ 403 Forbidden | 需要重新登录后再接受邀请 |
| 组织已归档 | ❌ 400 "Organization is Archived" | 组织不可用 |

---

## 四、validateInvitedUserSession 核心决策逻辑

[SessionService.validateInvitedUserSession](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts#L106-L198)

Guard 验证通过后，进入 Service 层的状态机决策：

```
validateInvitedUserSession(user, invitedUser, tokens)
        │
        ├─→ 输入参数解构
        │     ├─ tokens: { accountToken, organizationToken }
        │     └─ invitedUser: { email, firstName, lastName, status,
        │                        organizationStatus, organizationUserSource,
        │                        invitedOrganizationId, source }
        │
        ▼
  【检查点 1】邀请的组织是否存在且活跃?
        │
        ├─→ 组织不存在 或 status !== ACTIVE
        │     └─→ 400 BadRequest("Organization is Archived")
        │
        ▼
  【检查点 2】账户是否待激活? (accountYetToActive)
        │
        │  条件: organizationAndAccountInvite &&
        │         [INVITED, VERIFIED].includes(invitedUserStatus)
        │
        ├─→ 是（账户待激活）
        │     │
        │     ├─→ 是实例注册邀请?
        │     │    (有 accountToken && 无 organizationToken && source === SIGNUP)
        │     │
        │     ├─→ 是工作空间注册邀请?
        │     │    (双token && source === WORKSPACE_SIGNUP)
        │     │
        │     ├─→ 以上任一为真
        │     │     └─→ 返回 isWorkspaceSignUpInvite payload
        │     │           { email, name, invitedOrganizationName,
        │     │             isWorkspaceSignUpInvite: true, source }
        │     │
        │     └─→ 都不是（普通邀请，账户未激活）
        │           └─→ 406 NotAcceptable
        │                 { error: "Account is not activated yet",
        │                   isAccountNotActivated: true,
        │                   inviteeEmail, redirectPath }
        │
        ▼
  【检查点 3】是否为 workspace signup 状态?
        │
        │  条件: organizationStatus === INVITED &&
        │         有 organizationToken &&
        │         invitedUserStatus === ACTIVE &&
        │         organizationUserSource === SIGNUP
        │
        ├─→ 是（活跃用户 + 组织邀请 + 注册来源）
        │     └─→ 返回 { organizationUserSource }
        │           （前端据此展示 workspace signup 页面）
        │
        ▼
  【检查点 4】是否为旧版双 Token 邀请?
        │
        │  条件: 双token && 用户 ACTIVE && 组织状态 INVITED
        │
        ├─→ 是
        │     └─→ 生成 organizationInviteUrl（仅组织邀请链接）
        │
        ▼
  【检查点 5】生成正常会话 payload
        │
        ├─→ 获取用户默认/当前组织
        ├─→ 检查是否无活跃工作空间 (noActiveWorkspaces)
        ├─→ 调用 generateSessionPayload 生成权限数据
        │
        ▼
  返回最终响应
    { ...sessionPayload,
      invitedOrganizationName,
      noActiveWorkspaces,
      name,
      organizationInviteUrl? }
```

### 五种决策结果详解

#### 结果 1: Organization is Archived（组织归档错误）
- **触发条件**: 邀请的组织不存在，或状态不是 `ACTIVE`
- **代码位置**: [service.ts#L126-L128](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts#L126-L128)
- **响应**: `400 BadRequestException("Organization is Archived")`

#### 结果 2: invite signup payload（邀请注册）
- **触发条件**: 
  - 同时有 `accountToken` 和 `organizationToken`
  - 用户状态为 `INVITED` 或 `VERIFIED`（账户未激活）
  - 来源为 `SIGNUP`（实例注册）或 `WORKSPACE_SIGNUP`（工作空间注册）
- **代码位置**: [service.ts#L131-L144](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts#L131-L144)
- **响应字段**:
  ```json
  {
    "email": "...",
    "name": "...",
    "invited_organization_name": "...",
    "is_workspace_sign_up_invite": true,
    "source": "signup" | "workspace_signup"
  }
  ```
- **前端行为**: 展示邀请注册页面，设置密码后激活账户

#### 结果 3: workspace signup 状态
- **触发条件**:
  - 用户状态为 `ACTIVE`（账户已激活）
  - 组织用户状态为 `INVITED`
  - 有 `organizationToken`
  - 组织用户来源为 `SIGNUP`
- **代码位置**: [service.ts#L157-L168](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts#L157-L168)
- **响应字段**: `{ organization_user_source: "signup" }`
- **前端行为**: 展示 workspace signup / 加入工作空间确认页面

#### 结果 4: 正常 session + 邀请信息
- **触发条件**: 
  - 用户已激活
  - 不是 workspace signup 来源
  - 有正常会话或通过邀请验证
- **代码位置**: [service.ts#L169-L197](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts#L169-L197)
- **响应字段**:
  - 完整 session payload（用户信息、组织信息、权限等）
  - `invited_organization_name`: 被邀请的组织名称
  - `no_active_workspaces`: 用户是否无活跃工作空间
  - `name`: 用户全名
  - `organization_invite_url` (可选): 旧版双 token 邀请时的纯组织邀请链接

#### 结果 5: 需要重新登录
- **触发位置**: Guard 层 `onInvalidSession`
- **触发条件**: 
  - 无有效 JWT 会话
  - 用户已激活（非 INVITED/VERIFIED）
  - 不是 `SIGNUP` 来源
- **响应**: `403 Forbidden`，携带 `invited_organization_slug`
- **前端行为**: 跳转到登录页，登录后再接受邀请

---

## 五、SessionUtilService 权限数据生成

### generateLoginResultPayload
[SessionUtilService.generateLoginResultPayload](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts#L78-L182)

登录成功后生成完整的登录结果 payload：

| 字段 | 来源 | 说明 |
|------|------|------|
| `organization_id` / `organization` | 传入的 organization | 当前登录组织 |
| `is_current_organization_archived` | `organization.status === ARCHIVE` | 当前组织是否归档 |
| `current_organization_id` / `current_organization_slug` | organization | 当前组织标识 |
| `no_workspace_attached_in_the_session` | 无 organization 时 | 会话中无工作空间 |
| `no_active_workspaces` | `checkUserWorkspaceStatus()` | 用户是否无任何活跃工作空间 |
| 权限相关 | `getPermissionDataToAuthorize()` | 见下文 |

### getPermissionDataToAuthorize
[SessionUtilService.getPermissionDataToAuthorize](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts#L184-L249)

生成用户授权所需的完整权限数据：

| 字段 | 来源 | 说明 |
|------|------|------|
| `id` / `email` / `first_name` / `last_name` / `avatar_id` | user 对象 | 用户基本信息 |
| `admin` | `userPermissions.isAdmin` | 是否组织管理员 |
| `super_admin` | `userPermissions.isSuperAdmin` | 是否超级管理员 |
| `app_group_permissions` | `userPermissions[MODULES.APP]` | 应用权限 |
| `data_source_group_permissions` | `userPermissions[MODULES.GLOBAL_DATA_SOURCE]` | 数据源权限 |
| `folder_group_permissions` | `userPermissions[MODULES.FOLDER]` | 文件夹权限 |
| `role` | `rolesRepository.getUserRole()` | 用户角色 |
| `group_permissions` | `getAllGroupsOfUser()` | 用户所在所有组 |
| `user_permissions` | `abilityService.resourceActionsPermission()` | 资源级权限 |
| `metadata` | 解密后的 userMetadata | 用户元数据 |
| `sso_user_info` | userDetails | SSO 用户信息 |

### checkUserWorkspaceStatus
[SessionUtilService.checkUserWorkspaceStatus](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts#L273-L294)

判断用户是否**没有**任何活跃工作空间：

```
条件: 用户的 organizationUsers 中
      └─→ status === ACTIVE
          └─→ organization.status === ACTIVE

结果为空 → 返回 true（无活跃工作空间）
```

---

## 六、AuthService 邀请登录分支

### AuthService.login 中的邀请重定向处理
[AuthService.login](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/service.ts#L44-L183)

```
login(response, appAuthDto, organizationId?, loggedInUser?)
        │
        ▼
  判断是否为邀请重定向?
  (redirectTo 以 /organization-invitations/ 或 /invitations/ 开头)
        │
        ├─→ 是（邀请重定向模式）
        │     │
        │     ├─→ 按 INVITED 状态查找用户
        │     │    (findByEmail(email, organizationId, [INVITED]))
        │     │
        │     ├─→ 用户不存在或无密码 ──→ 401 "Invalid credentials"
        │     │
        │     ├─→ 密码不匹配 ──→ 401 "Invalid credentials"
        │     │
        │     └─→ 验证通过：organizationId = undefined
        │           （使用默认组织登录，不直接使用邀请组织）
        │
        └─→ 否（正常登录模式）
              └─→ authUtilService.validateLoginUser()
                    （仅允许 ACTIVE / ARCHIVED 状态的用户）
        │
        ▼
  确定登录后进入的组织
        │
        ├─→ 有默认组织且未归档 ──→ 使用默认组织
        ├─→ 有其他活跃组织 ──→ 选第一个
        ├─→ 允许个人工作空间 ──→ 创建新工作空间
        └─→ 都没有且非邀请 ──→ 401 "User is not assigned to any workspaces"
        │
        └─→ 邀请重定向且无组织：organizationId = invitingOrganizationId
        │
        ▼
  生成登录结果 payload
  (sessionUtilService.generateLoginResultPayload)
```

### 邀请登录的特殊点
1. **用户状态放宽**: 正常登录只允许 `ACTIVE`/`ARCHIVED` 用户，邀请模式允许 `INVITED` 状态
2. **组织选择差异**: 邀请登录时即使没有活跃组织，也会用 `invitingOrganizationId` 作为占位
3. **密码校验**: 邀请登录要求用户必须已有密码（不能是 SSO 等无密码账户）

---

## 七、JWT 策略中的会话验证

### JwtStrategy.validate
[JwtStrategy.validate](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/jwt/jwt.strategy.ts#L38-L138)

JWT 验证时对用户状态的处理：

```
JWT Token 解析出 payload
        │
        ▼
  validateUserSession(userId, sessionId, organizationId?)
        │
        ├─→ 会话不存在/过期 ──→ 401 Unauthorized
        │
        ▼
  根据场景不同查找用户
        │
        ├─→ isGetUserSession: 按 email 全局查找用户
        │     （不校验组织，返回所有组织信息）
        │
        ├─→ isInviteSession: 查找 ACTIVE 用户，使用默认组织
        │     （邀请流程不需要组织校验）
        │
        └─→ 普通场景: 按 email + organizationId 查找 ACTIVE 用户
              │
              ├─→ 用户不存在 ──→ handleUnauthorizedUser()
              │     │
              │     ├─→ 用户是 ARCHIVED ──→ USER_ARCHIVED_IN_ORGANIZATION
              │     ├─→ 用户是 INVITED ──→ USER_INVITED_IN_ORGANIZATION
              │     └─→ 用户不存在 ──→ USER_NOT_EXISTED
              │
              └─→ 用户存在：设置 organizationId
```

### validateUserSession 详细逻辑
[SessionUtilService.validateUserSession](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts#L513-L570)

| 检查项 | 失败结果 |
|--------|----------|
| 会话存在且未过期 | 401 Unauthorized |
| 用户状态为 ACTIVE | 401 Unauthorized |
| PAT 会话需验证 PAT 有效性和过期时间 | 401 Unauthorized |
| 常规会话过期 | 401 "User session expired" |

会话验证通过后会**自动续期**：
- 常规会话：延长至 `USER_SESSION_EXPIRY` 配置（默认 10 天）
- PAT 会话：根据 PAT 的 `sessionExpiryMinutes` 续期

---

## 八、完整状态决策矩阵

### 邀请链接打开时的状态决策

| # | 用户状态 | 组织用户状态 | 组织状态 | 有有效会话? | Token 类型 | 来源 | 结果 |
|---|----------|-------------|----------|------------|-----------|------|------|
| 1 | - | - | 不存在/ARCHIVE | - | 任意 | - | ❌ 400 组织归档 |
| 2 | INVITED/VERIFIED | - | ACTIVE | 否 | 仅组织邀请 | - | ❌ 400 账户未激活 |
| 3 | INVITED/VERIFIED | - | ACTIVE | 否 | 双 Token | SIGNUP/WORKSPACE_SIGNUP | ✅ invite signup payload |
| 4 | INVITED/VERIFIED | - | ACTIVE | 否 | 双 Token | INVITE | ❌ 406 账户未激活（需重发激活） |
| 5 | ACTIVE | INVITED | ACTIVE | 否 | 仅组织邀请 | SIGNUP | ✅ workspace signup 状态 |
| 6 | ACTIVE | INVITED | ACTIVE | 否 | 仅组织邀请 | INVITE | ❌ 403 需重新登录 |
| 7 | ACTIVE | INVITED | ACTIVE | 是（用户匹配） | 任意 | 任意 | ✅ 正常 session + 邀请信息 |
| 8 | ACTIVE | INVITED | ACTIVE | 是（用户不匹配） | 任意 | 任意 | ❌ 403 用户不匹配 |
| 9 | ACTIVE | ACTIVE | ACTIVE | 是 | 任意 | 任意 | ✅ 正常 session |
| 10 | ARCHIVED | - | - | - | 任意 | - | ❌ 401 无权限 |

### 正常会话获取时的状态决策

| # | 用户状态 | 组织状态 | 有活跃工作空间? | 是超管? | 结果 |
|---|----------|----------|----------------|--------|------|
| 1 | ACTIVE | ACTIVE | 是 | - | ✅ 正常 session（含组织信息） |
| 2 | ACTIVE | - | 否 | 否 | ✅ session + `no_workspace_attached_in_the_session: true` |
| 3 | ACTIVE | - | 否 | 是 | ✅ session + `is_all_workspaces_archived: true` |
| 4 | 非 ACTIVE | - | - | - | ❌ 401 会话失效 |
| 5 | ACTIVE | ARCHIVE | 其他活跃组织 | - | ✅ 切换到其他活跃组织 |
| 6 | ACTIVE | ARCHIVE | 无其他活跃组织 | 否 | ✅ session + 无工作空间标记 |

---

## 九、关键代码索引

| 功能模块 | 文件位置 | 关键函数/行号 |
|----------|----------|--------------|
| 正常会话获取 | [session/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts) | `getSessionDetails` L68-L104 |
| 邀请会话验证 | [session/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/service.ts) | `validateInvitedUserSession` L106-L198 |
| 会话 payload 生成 | [session/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts) | `generateSessionPayload` L386-L423 |
| 登录结果生成 | [session/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts) | `generateLoginResultPayload` L78-L182 |
| 权限数据生成 | [session/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/util.service.ts) | `getPermissionDataToAuthorize` L184-L249 |
| 邀请会话守卫 | [session/guards/invited-user-session.guard.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/guards/invited-user-session.guard.ts) | `canActivate` L37-L140 |
| 登录服务 | [auth/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/auth/service.ts) | `login` L44-L183 |
| JWT 策略 | [session/jwt/jwt.strategy.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/session/jwt/jwt.strategy.ts) | `validate` L38-L138 |
| 状态常量 | [users/constants/lifecycle.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/users/constants/lifecycle.ts) | 全文 |
