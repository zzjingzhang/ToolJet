# OpenAIMarketplace 插件执行路径追踪

## 概述

本文档以 OpenAI Marketplace 插件为例，完整追踪一个 chat / image_generation / generate_embedding 操作从 DataQuery 执行到插件 VM 实例化再到具体 operation 函数的调用路径。同时详细说明 plugin-selector 运行机制、manifest 配置、operations.json UI 约束、以及 query_operations 中的核心业务逻辑。

---

## 一、整体调用链路总览

```
前端触发查询
  ↓
DataQueriesService.runQueryOnBuilder() / runQueryForApp()
  ↓
DataQueriesUtilService.runQuery()
  ↓
DataQueriesUtilService.fetchServiceAndParsedParams()
  ├─ 解析 sourceOptions (数据源配置)
  ├─ 解析 parsedQueryOptions (查询参数，支持 {{变量}} 替换)
  └─ 获取 service 实例
      ↓
      PluginsServiceSelector.getService()
        ├─ isMarketplacePlugin = !!pluginId
        └─ findMarketplacePluginService()
            ↓
            从 DB 读取 plugins.indexFile (base64 编码的 JS 代码)
            ↓
            使用 Node.js vm 模块 createContext + runInNewContext
            ↓
            实例化 sandbox.module.exports.default
            ↓
      返回插件 service 实例
  ↓
service.run(sourceOptions, parsedQueryOptions, ...)
  ↓
Openai.run()  [marketplace/plugins/openai/lib/index.ts]
  ├─ getConnection() → 初始化 OpenAI SDK 实例
  └─ switch(operation):
      ├─ chat → getChatCompletion()
      ├─ image_generation → generateImage()
      └─ generate_embedding → generateEmbedding()
```

---

## 二、从 DataQuery 到插件 VM 实例化的完整路径

### 2.1 入口：DataQueriesService

查询执行的入口在 [service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts)，提供两个主要方法：

- **`runQueryOnBuilder()`** (L253-L277)：在编辑器中运行查询，支持更新 options
- **`runQueryForApp()`** (L279-L294)：在已发布应用中运行查询

两者最终都调用 `runAndGetResult()`，后者委托给 `DataQueriesUtilService.runQuery()`。

### 2.2 核心执行：DataQueriesUtilService.runQuery()

[util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L66-L386) 是查询执行的核心，主要步骤：

1. **获取数据源配置** (L110-L118)：通过 `appEnvironmentUtilService.getOptions()` 从环境变量中解析数据源选项
2. **获取服务与解析参数** (L120-L128)：调用 `fetchServiceAndParsedParams()`
3. **超时控制** (L131-L137)：使用 `AbortControllerHandler` 管理查询超时
4. **执行查询** (L182-L195)：调用 `service.run()` 执行实际查询
5. **OAuth 刷新重试** (L206-L337)：如果遇到 `OAuthUnauthorizedClientError`，自动刷新 token 并重试

### 2.3 获取服务实例：fetchServiceAndParsedParams()

[util.service.ts#L432-L459](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts#L432-L459) 方法完成三件事：

```typescript
const sourceOptions = await this.dataSourceUtilService.parseSourceOptions(...);
const parsedQueryOptions = await this.parseQueryOptions(...);
const service = await this.pluginsSelectorService.getService(dataSource.pluginId, dataSource.kind);
```

- `sourceOptions`：解密数据源配置（如 apiKey）
- `parsedQueryOptions`：解析查询选项中的模板变量（`{{variable}}`、`{{constants.xxx}}`、`{{secrets.xxx}}`）
- `service`：通过 plugin-selector 获取插件服务实例

### 2.4 插件选择器：PluginsServiceSelector.getService()

[plugin-selector.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/services/plugin-selector.service.ts#L25-L37) 的 `getService()` 方法根据数据源类型返回不同的服务：

```typescript
if (isToolJetDatabaseKind) return this.tooljetDbDataOperationsService;
if (isMarketplacePlugin) return await this.findMarketplacePluginService(pluginId);
return new allPlugins[kind]();
```

- **ToolJet Database**：直接返回内置服务
- **Marketplace 插件**：通过 `findMarketplacePluginService()` 动态加载
- **内置插件**：从 `@tooljet/plugins` 直接实例化

---

## 三、plugin-selector 如何运行数据库中的 indexFile

### 3.1 核心方法：findMarketplacePluginService()

[plugin-selector.service.ts#L39-L200](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/services/plugin-selector.service.ts#L39-L200) 是 marketplace 插件动态加载的核心。

### 3.2 从数据库读取 indexFile

```typescript
const plugin = await this.pluginsRepository.findById(pluginId, ['indexFile']);
decoded = decode(plugin.indexFile.data.toString());
this.plugins[pluginId] = decoded;
```

- **存储方式**：插件代码以 base64 编码的二进制数据存储在 `files` 表中，通过 `plugins.index_file_id` 关联
- **实体定义**：[plugin.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/plugin.entity.ts) 中 `indexFile` 是 `@OneToOne` 关联到 `File` 实体
- **缓存策略**：非开发模式下，解码后的代码缓存在 `this.plugins[pluginId]` 中，避免重复读取数据库
- **开发模式**：`ENABLE_MARKETPLACE_DEV_MODE=true` 时跳过缓存，每次都重新读取

### 3.3 使用 VM 沙箱执行代码

使用 Node.js 内置的 `vm` 模块在隔离环境中执行插件代码：

```typescript
import { runInNewContext, createContext } from 'vm';

const moduleWrapper = { exports: {} as PluginModule };

const sandbox = createContext({
  ...global,
  module: moduleWrapper,
  exports: moduleWrapper.exports,
  require: require,
  process: process,
  console: console,
  Buffer: Buffer,
  setTimeout, setInterval, ... // 定时器
  TextEncoder, TextDecoder, ... // 文本编码
  URL, URLSearchParams, ...     // URL API
  fetch, Headers, ...           // Fetch API
  crypto, ...                   // 加密
  JSON, Math, Promise, ...      // 内置对象
  __dirname: process.cwd(),
  __filename,
  global,
  // ... 上百个全局 API
});

runInNewContext(decoded, sandbox);
const service = new sandbox.module.exports.default();
```

**关键设计要点：**

1. **CommonJS 兼容**：通过 `module` 和 `exports` 对象模拟 CommonJS 模块系统
2. **沙箱安全**：使用 `createContext()` 创建独立上下文，插件代码无法直接访问外部作用域
3. **API 白名单**：显式注入允许插件使用的全局 API（如 `fetch`、`crypto`、`Buffer` 等）
4. **require 透传**：插件可以通过 `require()` 加载依赖包（依赖需已安装在主进程中）

### 3.4 插件代码格式

indexFile 是打包后的 JavaScript 文件，导出一个默认类，实现 `QueryService` 接口（包含 `run()` 和 `testConnection()` 方法）。

---

## 四、OpenAI 插件 manifest 如何声明 apiKey

### 4.1 manifest.json 结构

[manifest.json](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/manifest.json) 是插件的数据源配置清单，定义了连接数据源所需的参数。

### 4.2 apiKey 的声明方式

```json
{
  "properties": {
    "apiKey": {
      "type": "string",
      "title": "API Key",
      "description": "Enter your OpenAI API Key"
    }
  },
  "tj:encrypted": ["apiKey"],
  "required": ["apiKey"],
  "tj:ui:properties": {
    "apiKey": {
      "order": 10,
      "$ref": "#/properties/apiKey",
      "key": "apiKey",
      "label": "API Key",
      "widget": "password-v3",
      "placeholder": "sk-...",
      "help_text": "For generating API Key, visit: <a href='...'>OpenAI Console</a>"
    }
  }
}
```

### 4.3 各字段含义

| 字段 | 位置 | 作用 |
|------|------|------|
| `properties.apiKey` | 顶层 | JSON Schema 定义：类型、标题、描述 |
| `tj:encrypted` | 顶层 | 声明 `apiKey` 需要加密存储。保存时通过 `CredentialsService` 加密，读取时解密 |
| `required` | 顶层 | 声明 `apiKey` 是必填项 |
| `tj:ui:properties.apiKey.widget` | UI 配置 | 使用 `password-v3` 组件渲染（密码输入框，带掩码） |
| `tj:ui:properties.apiKey.order` | UI 配置 | 字段在表单中的显示顺序 |
| `tj:ui:properties.apiKey.placeholder` | UI 配置 | 输入框占位符 |
| `tj:ui:properties.apiKey.help_text` | UI 配置 | 帮助文本，支持 HTML 链接 |

### 4.4 apiKey 在运行时的使用

在插件的 `getConnection()` 方法中 ([index.ts#L67-L83](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/index.ts#L67-L83))：

```typescript
async getConnection(sourceOptions: SourceOptions): Promise<OpenAI> {
  const { apiKey, organizationId = null } = sourceOptions;
  const openai = new OpenAI({ apiKey: apiKey });
  return openai;
}
```

`sourceOptions` 中的 `apiKey` 已经由 `parseSourceOptions()` 解密，可直接使用。

---

## 五、operations.json 如何约束 UI 输入

### 5.1 作用与定位

[operations.json](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/operations.json) 定义了查询编辑器的 UI 结构。前端 `DynamicForm` 组件根据此 schema 动态渲染表单。

前端渲染入口在 [QueryEditors/index.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/QueryManager/QueryEditors/index.js#L46-L50)：

```jsx
export const source = (props) => (
  <div className="query-editor-dynamic-form-container">
    <DynamicForm 
      schema={props.pluginSchema} 
      {...props} 
      computeSelectStyles={computeSelectStyles} 
      layout="horizontal" 
    />
  </div>
);
```

`pluginSchema` 来自 `selectedDataSource.plugin.operations_file.data`，即数据库中存储的 operations.json 内容。

### 5.2 分层级联结构

operations.json 使用**两级 dropdown 级联**的方式组织字段：

```
operation (下拉选择：chat / image_generation / generate_embedding)
  ├─ chat
  │   └─ model (下拉选择：gpt-5, gpt-4o, o1, ...)
  │       ├─ gpt-5.2 → { system_prompt, message_history, prompt, max_tokens, temperature, ... }
  │       ├─ gpt-5.1 → { ... }
  │       ├─ o3 → { ... }
  │       └─ ...
  ├─ image_generation
  │   └─ model (下拉选择：gpt-image-1, dall-e-3, dall-e-2)
  │       ├─ gpt-image-1 → { prompt, size }
  │       ├─ dall-e-3 → { prompt, size }
  │       └─ dall-e-2 → { prompt, size }
  └─ generate_embedding
      └─ model_embedding (下拉选择：text-embedding-3-small, ...)
          ├─ text-embedding-3-small → { input_M1, encoding_format_M1, dimensions_M1 }
          ├─ text-embedding-3-large → { input_M2, encoding_format_M2, dimensions_M2 }
          └─ text-embedding-ada-002 → { input_M3, encoding_format_M3 }
```

### 5.3 核心字段类型

#### 5.3.1 dropdown-component-flip

用于操作类型和模型选择的级联下拉。当选择不同值时，显示对应分组的字段。

```json
{
  "operation": {
    "label": "Operation",
    "key": "operation",
    "type": "dropdown-component-flip",
    "list": [
      { "value": "chat", "name": "Chat" },
      { "value": "image_generation", "name": "Generate AI Image(s)" },
      { "value": "generate_embedding", "name": "Generate embedding" }
    ]
  }
}
```

`DynamicForm` 组件检测到 `dropdown-component-flip` 类型时，会根据当前选中值（`options[flipComponentDropdown.key].value`）动态显示 `properties[selector]` 下的字段。

#### 5.3.2 codehinter

代码编辑器输入框，用于输入提示词、消息历史等长文本。

```json
{
  "prompt": {
    "label": "User message",
    "key": "prompt",
    "type": "codehinter",
    "placeholder": "Draft an email or other piece of writing",
    "height": "150px"
  }
}
```

支持 `width`、`height`、`placeholder`、`tooltip`、`mandatory` 等 UI 属性。

#### 5.3.3 dropdown

普通下拉选择框。

```json
{
  "encoding_format_M1": {
    "label": "Encoding format",
    "key": "encoding_format_M1",
    "type": "dropdown",
    "list": [
      { "value": "float", "name": "Float" },
      { "value": "base64", "name": "Base64" }
    ]
  }
}
```

### 5.4 默认值

`defaults` 字段定义表单初始值，前端 `getDefaultOptions()` 函数会读取这些默认值。

```json
{
  "defaults": {
    "encoding_format_M1": "float",
    "encoding_format_M2": "float",
    "encoding_format_M3": "float"
  }
}
```

### 5.5 字段约束机制

| 属性 | 作用 |
|------|------|
| `mandatory` | 是否必填（布尔值） |
| `placeholder` | 输入框占位提示 |
| `tooltip` | 悬停提示信息 |
| `help_text` | 帮助文本（支持 HTML） |
| `width` / `height` | 控件尺寸 |
| `disabled` | 是否禁用 |
| `default` | 默认值（配合 disabled 使用） |
| `order` | 字段排序（在 manifest 的 tj:ui:properties 中） |

---

## 六、query_operations 核心逻辑详解

[query_operations.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/query_operations.ts) 包含三个核心函数：`getChatCompletion()`、`generateImage()`、`generateEmbedding()`。

### 6.1 getChatCompletion() — 聊天补全

#### 6.1.1 max_tokens vs max_completion_tokens 的模型选择

**核心逻辑** ([query_operations.ts#L56-L100](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/query_operations.ts#L56-L100))：

```typescript
const modelName = model?.toLowerCase() || '';

// 识别"现有"模型 —— 必须使用 max_tokens
const isExistingModel = 
  modelName.includes('gpt-4o') || 
  modelName.includes('gpt-4.0') || 
  modelName.includes('gpt-4-turbo') || 
  modelName.includes('gpt-3.5-turbo');

if (tokenLimit) {
  if (isExistingModel) {
    requestPayload.max_tokens = tokenLimit;      // 旧模型：max_tokens
  } else {
    requestPayload.max_completion_tokens = tokenLimit; // 新模型：max_completion_tokens
  }
}
```

**分类规则：**

| 使用 `max_tokens` 的旧模型 | 使用 `max_completion_tokens` 的新模型 |
|---------------------------|-------------------------------------|
| gpt-4o 系列 | gpt-5 系列（gpt-5, gpt-5.1, gpt-5.2, gpt-5-mini, gpt-5-nano） |
| gpt-4-turbo | o 系列（o1, o3, o3-mini, o4-mini） |
| gpt-3.5-turbo | gpt-4.1 系列 |

**背景**：OpenAI 新模型（GPT-5、o 系列、GPT-4.1）使用 `max_completion_tokens` 参数替代旧的 `max_tokens`，语义更清晰（仅限制输出 token 数，不包含输入）。

#### 6.1.2 推理模型跳过 temperature

**核心逻辑** ([query_operations.ts#L102-L107](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/query_operations.ts#L102-L107))：

```typescript
// 推理模型（o 系列、gpt-5）不支持 temperature
const isReasoning = modelName.startsWith('o') || modelName.startsWith('gpt-5');
if (!isReasoning) {
  requestPayload.temperature = 
    typeof temperature === 'string' ? parseFloat(temperature) : temperature || 0;
}
```

**推理模型列表：**
- o 系列：o1, o3, o3-mini, o4-mini（所有以 `o` 开头的模型）
- gpt-5 系列：gpt-5, gpt-5.1, gpt-5.2, gpt-5-mini, gpt-5-nano

**原因**：推理模型（Reasoning Models）通过内部思维链进行推理，其输出确定性不由 `temperature` 参数控制，因此不支持该参数。传入会导致 API 报错。

#### 6.1.3 消息组装

```typescript
let parsedMessages: any[] = [];

// 1. 系统提示词
if (system_prompt && typeof system_prompt === 'string' && system_prompt.trim() !== '') {
  parsedMessages.push({ role: 'system', content: system_prompt });
}

// 2. 消息历史（JSON 字符串 → 数组）
if (message_history && message_history !== '') {
  const historyArray = typeof message_history === 'string' 
    ? JSON.parse(message_history) : message_history;
  if (Array.isArray(historyArray)) {
    parsedMessages.push(...historyArray);
  }
}

// 3. 当前用户消息
if (prompt && prompt !== '') {
  parsedMessages.push({ role: 'user', content: String(prompt) });
}
```

### 6.2 generateImage() — 图像生成

#### 6.2.1 DALL-E 图片大小限制

**核心函数 `getSizeEnum()`** ([query_operations.ts#L5-L39](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/query_operations.ts#L5-L39))：

```typescript
const getSizeEnum = (model, size) => {
  // DALL-E 3：只允许 1024x1024, 1792x1024, 1024x1792
  if (model === 'dall-e-3') {
    switch (size) {
      case '1024x1024': return '1024x1024';
      case '1792x1024': return '1792x1024';
      case '1024x1792': return '1024x1792';
      default: return '1024x1024'; // 默认
    }
  }

  // DALL-E 2：只允许 1024x1024, 512x512, 256x256
  if (model === 'dall-e-2') {
    switch (size) {
      case '1024x1024': return '1024x1024';
      case '512x512': return '512x512';
      case '256x256': return '256x256';
      default: return '1024x1024'; // 默认
    }
  }

  return '1024x1024'; // 未知模型的默认值
};
```

**各模型支持的尺寸：**

| 模型 | 支持尺寸 | 默认值 |
|------|----------|--------|
| dall-e-3 | 1024×1024, 1792×1024, 1024×1792 | 1024×1024 |
| dall-e-2 | 256×256, 512×512, 1024×1024 | 1024×1024 |
| gpt-image-1 | 同 dall-e-3 验证逻辑（无独立 switch） | 1024×1024 |

**设计意图**：
- 白名单校验：只允许 OpenAI API 支持的尺寸值
- 优雅降级：非法或空值自动回退到默认尺寸，避免 API 报错
- 模型差异：DALL-E 3 支持宽屏和竖屏尺寸，DALL-E 2 仅支持正方形

#### 6.2.2 返回格式差异

```typescript
if (finalModel === 'gpt-image-1') {
  return { status: 'success', message: 'Image generated successfully',
    data: { b64_json: response.data[0].b64_json } };
}
// DALL·E → 返回 URL
return { status: 'success', message: 'Image generated successfully',
  data: { url: response.data[0].url } };
```

- **gpt-image-1**：返回 base64 编码的图片数据（`b64_json`）
- **dall-e-2 / dall-e-3**：返回图片 URL

### 6.3 generateEmbedding() — 向量生成

**核心逻辑** ([query_operations.ts#L149-L177](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/query_operations.ts#L149-L177))：

由于不同 embedding 模型的参数字段名不同（后缀 `_M1`、`_M2`、`_M3`），需要根据模型选择对应的字段：

```typescript
switch (model) {
  case 'text-embedding-3-small':
    input = rawOptions.input__m1 || options.input_M1;
    encoding_format = rawOptions.encoding_format__m1 || options.encoding_format_M1;
    dimensions = rawOptions.dimensions__m1 || options.dimensions_M1;
    break;
  case 'text-embedding-3-large':
    input = rawOptions.input__m2 || options.input_M2;
    // ...
    break;
  case 'text-embedding-ada-002':
    input = rawOptions.input__m3 || options.input_M3;
    // （无 dimensions 参数）
    break;
}

const embedding = await openai.embeddings.create({
  model: model,
  input: input,
  encoding_format: encoding_format as any,
  ...(dimensions && { dimensions: Number(dimensions) }),
});
```

**字段命名规则**（与 operations.json 对应）：

| 模型 | input 字段 | encoding_format 字段 | dimensions 字段 |
|------|-----------|---------------------|----------------|
| text-embedding-3-small | input_M1 | encoding_format_M1 | dimensions_M1 |
| text-embedding-3-large | input_M2 | encoding_format_M2 | dimensions_M2 |
| text-embedding-ada-002 | input_M3 | encoding_format_M3 | （不支持） |

---

## 七、关键文件索引

| 文件 | 作用 |
|------|------|
| [plugin.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/plugin.entity.ts) | Plugin 数据库实体定义 |
| [plugin-selector.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/services/plugin-selector.service.ts) | 插件选择器，VM 沙箱执行 |
| [data-queries/service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/service.ts) | DataQuery 服务入口 |
| [data-queries/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-queries/util.service.ts) | 查询执行核心逻辑 |
| [data-sources/util.service.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/util.service.ts) | 数据源工具服务（解密等） |
| [openai/lib/index.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/index.ts) | OpenAI 插件主入口 |
| [openai/lib/manifest.json](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/manifest.json) | 数据源配置清单 |
| [openai/lib/operations.json](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/operations.json) | 查询操作 UI 定义 |
| [openai/lib/query_operations.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/query_operations.ts) | 核心操作实现 |
| [openai/lib/types.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/marketplace/plugins/openai/lib/types.ts) | TypeScript 类型定义 |
| [DynamicForm.jsx](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_components/DynamicForm.jsx) | 前端动态表单渲染组件 |
| [QueryEditors/index.js](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/AppBuilder/QueryManager/QueryEditors/index.js) | 查询编辑器入口 |
