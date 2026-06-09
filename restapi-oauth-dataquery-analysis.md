# RESTAPI/OAuth DataQuery 执行流程深度分析

## 一、sourceOptions 和 queryOptions 解析流程

### 1.1 总体架构

DataQuery 执行的入口在 [data-queries/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts) 的 `runAndGetResult` 方法，核心解析逻辑由 [data-queries/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts) 的 `runQuery` 方法驱动。

解析流程分为两条独立的路径：
- **sourceOptions**：数据源级别的配置（连接信息、认证参数等）
- **queryOptions**：查询级别的配置（请求方法、URL路径、请求体等）

### 1.2 sourceOptions 解析流程

#### 第一步：环境与 Branch 感知的选项获取

在 `runQuery` 方法中（[util.service.ts#L110-L118](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L110-L118)）：

```typescript
const branchId =
  dataQuery?.appVersion?.versionType === AppVersionType.BRANCH 
    ? dataQuery.appVersion.branchId 
    : undefined;

const dataSourceOptions = await this.appEnvironmentUtilService.getOptions(
  dataSource.id,
  organizationId,
  envId,
  branchId
);
```

**Branch 解析逻辑**（[app-environments/util.service.ts#L236-L313](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-environments/util.service.ts#L236-L313)）：

1. 优先查找 branch 对应的 `DataSourceVersion`（DSV）
2. 通过 DSV 查找对应环境的 `DataSourceVersionOptions`（DSVO）
3. 如果 branch 路径找不到，回退到 `isDefault: true` 的默认 DSV
4. 每个环境有独立的 DSVO 记录，实现环境隔离

#### 第二步：sourceOptions 的常量与 secrets 解析

通过 `dataSourceUtilService.parseSourceOptions` 方法解析（[data-sources/util.service.ts#L1122-L1189](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L1122-L1189)）：

**解析顺序：**

1. **非加密字段的常量替换**：遍历所有 options，如果 value 包含 `{{constants.}}`、`{{secrets.}}` 或 `{{globals.server.}}`，调用 `resolveConstants` 解析
2. **数组类型递归解析**：对于 headers、url_params 等数组类型选项，递归解析每个元素
3. **加密字段解密与常量替换**：
   - 加密字段通过 `credentialService.getValue(credential_id)` 解密
   - 解密后如果包含常量引用，再次调用 `resolveConstants` 解析
   - 最终将解密并解析后的值放入 `parsedOptions`

**常量/Secrets 解析核心**（[data-sources/util.service.ts#L692-L733](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L692-L733)）：

```typescript
async resolveConstants(str: string, organizationId: string, environmentId: string, user?: User): Promise<string> {
  const regex = /\{\{(constants|secrets)\.(.*?)\}\}/g;
  // 对每个匹配项：
  // 1. 根据 prefix 确定类型（GLOBAL 或 SECRET）
  // 2. 调用 organizationConstantsUtilService.getOrgEnvironmentConstant 查询
  // 3. 通过 encryptionService.decryptColumnValue 解密值
  // 4. 替换回原字符串
}
```

> **关键点**：常量和 secrets 都是环境感知的（environment-specific），不同环境可以有不同的常量值。

### 1.3 queryOptions 解析流程

通过 `dataQueryUtilService.parseQueryOptions` 方法解析（[data-queries/util.service.ts#L520-L669](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L520-L669)）。

#### 解析算法（栈式遍历）

使用深度优先的栈遍历算法，递归处理所有嵌套的对象、数组和字符串：

```typescript
while (stack.length > 0) {
  const { obj, key, parent } = stack.pop();
  // Case 1: Object - 压入所有子属性
  // Case 2: Array - 压入所有数组元素
  // Case 3: String - 执行各种模板替换
}
```

#### 五种替换规则（按代码顺序）

| 规则 | 触发条件 | 处理方式 |
|------|----------|----------|
| a. 双大括号整体替换 | 字符串同时包含 `{{` 和 `}}` | 用 `options[resolvedValue]` 直接替换整个属性值 |
| b. 常量/Secrets/Globals | 包含 `{{constants.`、`{{secrets.` 或 `{{globals.server.` | 调用 `resolveConstants` 解析 |
| d. 单变量替换 | 整个字符串是单个 `{{variable}}` 或 `{{{object}}}` | 用 `options[resolvedValue]` 替换整个属性值 |
| c. 多变量字符串插值 | 字符串中有多个 `{{var}}` 但不是单个变量 | 逐个替换，对象/数组会被 JSON.stringify |
| f. （已移除）| `%%` 语法 | 已废弃 |

#### 工作流 Bundle 增强

如果是工作流上下文（`opts.workflow` 存在），会使用增强的解析路径（[util.service.ts#L529-L548](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L529-L548)）：

1. 通过 `getQueryVariables` 从 bundle 中提取所有模板变量
2. 将 bundle 变量合并到 options 中
3. 再执行标准的 parseQueryOptionsInternal 逻辑

`getQueryVariables` 函数（[server/lib/utils.ts#L180-L219](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/lib/utils.ts#L180-L219)）会：
- 递归遍历 options 中的所有 `{{var}}` 模式
- 通过 `isolated-vm` 沙箱执行 JavaScript 表达式
- 支持 NPM 包 bundle 注入

---

## 二、OAuthUnauthorizedClientError 刷新 Token 与重试条件

### 2.1 错误抛出的两个源头

#### 源头一：REST API 插件的 401 响应

在 [restapi/lib/index.ts#L337-L339](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/restapi/lib/index.ts#L337-L339)：

```typescript
if (sourceOptions['auth_type'] === 'oauth2' && error?.response?.statusCode === 401) {
  throw new OAuthUnauthorizedClientError('Unauthorized status from API server', error.message, result);
}
```

**触发条件**：
- auth_type 为 `oauth2`
- HTTP 响应状态码为 401

#### 源头二：Refresh Token 调用失败

在 [common/lib/oauth.ts#L354-L360](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/oauth.ts#L354-L360)：

```typescript
if (error.response?.statusCode >= 400 && error.response?.statusCode < 500) {
  throw new OAuthUnauthorizedClientError(
    'Unauthorized status from Oauth server',
    JSON.stringify({ statusCode: error.response?.statusCode, message: error.response?.body }),
    result
  );
}
```

**触发条件**：
- 刷新 token 请求返回 4xx 状态码

### 2.2 刷新 Token 与重试决策树

在 [data-queries/util.service.ts#L206-L337](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L206-L337) 中处理：

```
捕获 OAuthUnauthorizedClientError
        │
        ▼
  获取当前用户的 token
  (getCurrentUserToken)
        │
        ├─► 有 refresh_token 吗？
        │       │
        │       ├─ 是 ──► 调用 service.refreshToken()
        │       │          │
        │       │          ├─ 成功 ──► 更新 OAuth access token
        │       │          │           重新解析 sourceOptions
        │       │          │           重试查询（仅重试1次）
        │       │          │
        │       │          └─ 失败 ──► 是 OAuthUnauthorizedClientError 吗？
        │       │                         │
        │       │                         ├─ 是 ──► 返回 needs_oauth（需重新授权）
        │       │                         └─ 否 ──► 抛出组合错误信息
        │       │
        │       └─ 否 ──► 数据源类型在白名单中吗？
        │                       │
        │                       ├─ 是 ──► 返回 needs_oauth
        │                       └─ 否 ──► 直接抛出原错误
        │
        └─（无 token 分支已在上面的"否"分支覆盖）
```

### 2.3 关键重试条件总结

**刷新 Token 的前提条件：**
1. 错误类型必须是 `OAuthUnauthorizedClientError`（通过 `constructor.name` 判断，而非 `instanceof`）
2. 当前用户必须有有效的 `refresh_token`
3. 多认证模式下需要能根据 userId 找到对应的 token 记录

**重试的限制：**
- 仅重试 **一次**（刷新 token 后重新执行查询，没有循环重试机制）
- 重试前会更新数据库中的 token（`updateOAuthAccessToken`）
- token 更新会传播到所有 branch 版本（`propagateTokenToAllBranches`）

**返回 needs_oauth 的条件：**
1. 刷新 token 时又遇到 `OAuthUnauthorizedClientError`（refresh_token 也失效了）
2. 没有 refresh_token，但数据源类型在白名单中（restapi、openapi、graphql、googlesheets、googlesheetsv2、slack、zendesk）

---

## 三、各功能模块的处理位置

### 3.1 Public App 多认证处理

**位置**：[data-queries/util.service.ts#L143-L150](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L143-L150)

```typescript
if (appToUse?.isPublic && sourceOptions['multiple_auth_enabled']) {
  throw new QueryError(
    'Authentication required for all users should be turned off since the app is public',
    '',
    {}
  );
}
```

**规则**：公共应用 + 多认证模式 = 直接报错，不允许执行。

**多认证 Token 选择逻辑**（[utils.helper.ts#L89-L100](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/utils.helper.ts#L89-L100)）：

```typescript
export const getCurrentToken = (isMultiAuthEnabled, tokenData, userId, isAppPublic) => {
  if (isMultiAuthEnabled) {
    if (!tokenData || !Array.isArray(tokenData)) return null;
    return !isAppPublic
      ? tokenData.find(token => token.user_id === userId)
      : userId
        ? tokenData.find(token => token.user_id === userId)
        : tokenData[0];  // public app + 无 userId 时取第一个 token
  } else {
    return tokenData;
  }
};
```

### 3.2 tj-x-forwarded-for Header

**位置**：[data-queries/util.service.ts#L152-L161](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L152-L161)

```typescript
if (dataSource.kind === 'restapi') {
  const customXFFHeader = ['tj-x-forwarded-for'];
  if (RequestContext?.currentContext?.req) {
    customXFFHeader.push(requestIp.getClientIp(RequestContext?.currentContext?.req));
  }
  if (!sourceOptions['headers']) {
    sourceOptions['headers'] = [customXFFHeader];
  } else {
    sourceOptions['headers'].push(customXFFHeader);
  }
}
```

**特点**：
- 仅对 `restapi` 类型的数据源生效
- 通过 `request-ip` 库获取客户端真实 IP
- 作为 header 追加到 sourceOptions 的 headers 数组中
- 在 service.run() 调用之前注入

### 3.3 FORWARD_RESTAPI_COOKIES（Cookie 转发）

**位置**：[data-queries/util.service.ts#L78](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L78) 和 [L162-L175](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L162-L175)

```typescript
const forwardRestCookies = this.configService.get<string>('FORWARD_RESTAPI_COOKIES') === 'true';

if (forwardRestCookies) {
  const cookies = RequestContext?.currentContext?.req?.headers?.cookie || '';
  if (cookies) {
    const cookieArray = cookies.split('; ');
    const filteredCookies = cookieArray.filter(cookie => !cookie.startsWith('tj_auth_token='));
    const filteredCookiesString = filteredCookies.join('; ');
    if (filteredCookiesString) {
      const cookieHeader = ['Cookie', filteredCookiesString];
      sourceOptions['headers'].push(cookieHeader);
    }
  }
}
```

**安全过滤**：
- 过滤掉 `tj_auth_token=`（ToolJet 自身的认证 token）
- 其他 cookie 原样转发给目标 API

### 3.4 Set-Cookie 回写

**位置**：[data-queries/util.service.ts#L344-L346](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L344-L346) 和 [L479-L518](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L479-L518)

```typescript
if (forwardRestCookies && dataQuery.kind === 'restapi' && result.responseHeaders) {
  this.setCookiesBackToClient(response, result.responseHeaders);
}
```

**回写逻辑**（`setCookiesBackToClient`）：
1. 从响应头中提取 `set-cookie`
2. 解析每个 cookie 的名称、值和属性
3. 支持的属性：max-age、expires、httponly、secure、domain、path、samesite 等
4. 通过 Express 的 `response.cookie()` 方法回写给客户端
5. expires 属性会被转换为 maxAge（相对时间）

### 3.5 审计上下文

**位置**：[data-queries/util.service.ts#L358-L384](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L358-L384)（在 `finally` 块中）

```typescript
finally {
  if (user) {
    const auditData = {
      userId: user.id,
      organizationId: user.organizationId,
      resourceId: dataQuery?.id,
      resourceName: dataQuery?.name,
      metadata: enrichedMetadata,  // 包含 appId、appName、dataSourceType、查询状态等
      resourceData: {
        dataSourceId: dataSource?.id,
        dataSourceName: dataSource?.name,
      },
    };
    RequestContext.setLocals(AUDIT_LOGS_REQUEST_CONTEXT_KEY, auditData);
  }
}
```

**关键点**：
- 在 `finally` 块中执行，确保无论成功失败都会记录
- 仅当有用户（user 存在）时才记录
- 通过 `RequestContext` 存储，由后续的拦截器/中间件消费
- 常量定义：`AUDIT_LOGS_REQUEST_CONTEXT_KEY = 'tj_audit_logs_meta_data'`（[app/constants/index.ts#L55](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/constants/index.ts#L55)）

---

## 四、容易误判的边界条件

### 边界条件一：`globals.server` 变量实际上不会被解析

**容易误判的点**：

多个地方的 matcher 正则都包含 `globals.server`，让人以为它是和 `constants`、`secrets` 并列的第三种变量类型：

- `parseSourceOptions` 中的 matcher：`/\{\{(constants|secrets|globals.server)\..*?\}\}/g`（[data-sources/util.service.ts#L1125](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L1125)）
- `resolveValue` 中的 matcher：`/{{constants|secrets|globals.server\..+?}}/g`（[data-sources/util.service.ts#L744](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts#L744)）
- `parseQueryOptions` 中的判断：`resolvedValue.includes('{{globals.server.')`（[data-queries/util.service.ts#L598](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L598)）

**实际情况**：

真正执行解析的 `resolveConstants` 函数的正则是：

```typescript
const regex = /\{\{(constants|secrets)\.(.*?)\}\}/g;
```

**只匹配 `constants` 和 `secrets`，不匹配 `globals.server`！**

后果：
- 包含 `{{globals.server.XXX}}` 的字符串会被检测到，并传入 `resolveConstants`
- 但 `resolveConstants` 的正则不匹配，直接原样返回
- 最终 `{{globals.server.XXX}}` 会作为字面量保留，不会被解析

**为什么会有这种不一致？**
- matcher 用于"判断是否需要调用 resolveConstants"
- 但 resolveConstants 内部的正则更严格
- 可能是历史遗留或未完成的功能

### 边界条件二：OAuth 错误判断使用 constructor.name 而非 instanceof

**容易误判的点**：

代码中多处使用 `api_error.constructor.name === 'OAuthUnauthorizedClientError'` 来判断错误类型（[util.service.ts#L206](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L206)、[L226](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L226)）。

**为什么容易误判：**

1. **跨包/跨 realm 的问题**：如果错误对象来自不同的包实例（如 marketplace plugins  vs packages/plugins），即使类名相同，`instanceof` 也会返回 false。使用 `constructor.name` 似乎是为了兼容这种情况。

2. **但这带来了新问题**：任何名为 `OAuthUnauthorizedClientError` 的类都会被匹配，即使完全不相关。

3. **刷新 token 调用的参数不一致**：

   在 `runQuery` 中调用 refreshToken 时：
   ```typescript
   accessTokenDetails = await service.refreshToken(
     sourceOptions,
     dataSource.id,      // 第二个参数是 dataSource.id
     user?.id,
     appToUse?.isPublic
   );
   ```

   但在 restapi 插件中，`refreshToken` 方法的签名是：
   ```typescript
   async refreshToken(sourceOptions: any, error: any, userId: string, isAppPublic: boolean)
   ```

   第二个参数应该是 `error`，但实际传入的是 `dataSource.id`！

   再看 `getRefreshedToken` 的实现（[oauth.ts#L287](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/oauth.ts#L287)），它的第二个参数确实叫 `error`，但函数体内并没有使用这个 `error` 参数来做除了错误消息以外的关键逻辑判断。

   这是一个**潜在的 bug**：`dataSource.id` 被当作 `error` 传入了，但因为 `error` 参数在 `getRefreshedToken` 中仅用于错误消息拼装，所以功能上可能暂时没问题，但语义上完全错误。

### 边界条件三：公共应用与多认证的交互逻辑不一致

**容易误判的点**：

1. **查询执行前的检查**（[util.service.ts#L143-L150](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L143-L150)）：
   - 公共应用 + 多认证 = **直接报错**

2. **getCurrentToken 中的逻辑**（[utils.helper.ts#L89-L100](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/utils.helper.ts#L89-L100)）：
   - 公共应用 + 有 userId = 按 userId 查找
   - 公共应用 + 无 userId = 取第一个 token（`tokenData[0]`）

**矛盾点**：
- 按道理，如果公共应用不允许多认证，那么 `getCurrentToken` 中处理公共应用的逻辑就是死代码
- 但实际上，可能存在某些路径（如 OAuth 插件内部直接调用 `getCurrentToken`）绕过了前面的检查
- 这意味着"公共应用不能用多认证"的规则可能只在特定路径上生效

### 边界条件四：parseQueryOptions 中替换规则的执行顺序

**容易误判的点**：

`parseQueryOptionsInternal` 中的替换规则不是互斥的，而是**顺序执行**的，后面的规则可能会覆盖前面的结果。

规则执行顺序：
1. 规则 a：整体替换（检测到 `{{` 和 `}}` 就用 `options[值]` 替换整个属性）
2. 规则 b：常量/Secrets 解析（如果包含 `{{constants.` 等）
3. 规则 d：单变量替换（整个字符串是单个变量）
4. 规则 c：多变量插值（字符串中有多个变量）

**潜在问题**：
- 规则 a 和规则 d 看起来功能重叠
- 规则 b 在规则 a 之后执行，如果规则 a 把整个字符串替换成了非字符串，规则 b 就不会执行了
- 这种顺序依赖的逻辑很容易因为调整顺序而引入 bug

例如，如果一个值是 `{{constants.API_KEY}}`：
- 规则 a 会尝试用 `options['{{constants.API_KEY}}']` 替换（通常是 undefined）
- 然后规则 b 检测到 `{{constants.`，调用 resolveConstants 正确解析

如果规则 b 在规则 a 之前，行为就不同了。

---

## 五、核心文件索引

| 功能模块 | 核心文件 | 关键行号 |
|----------|----------|----------|
| DataQuery 执行入口 | [data-queries/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts) | L300-L343 |
| 查询执行核心逻辑 | [data-queries/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts) | L66-L386 |
| 环境+Branch 选项解析 | [app-environments/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app-environments/util.service.ts) | L236-L313 |
| sourceOptions 解析 | [data-sources/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts) | L1122-L1189 |
| 常量/Secrets 解析 | [data-sources/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts) | L692-L733 |
| queryOptions 解析 | [data-queries/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts) | L520-L669 |
| REST API 插件主逻辑 | [restapi/lib/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/restapi/lib/index.ts) | L47-L476 |
| OAuth 通用逻辑 | [common/lib/oauth.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/oauth.ts) | 全文 |
| Token 选择工具 | [common/lib/utils.helper.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/utils.helper.ts) | L89-L100 |
| 错误类型定义 | [common/lib/query.error.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/plugins/packages/common/lib/query.error.ts) | 全文 |
| 模板变量提取 | [server/lib/utils.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/lib/utils.ts) | L180-L219 |
