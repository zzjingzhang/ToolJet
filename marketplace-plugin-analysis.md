# Marketplace 插件进入数据库与消费流程分析

## 概述

ToolJet 的 Marketplace 插件体系通过四个主要入口将插件元数据与代码注入系统：
1. **HTTP 安装接口** (`POST /plugins/install`)
2. **HTTP 重载接口** (`POST /plugins/:id/reload`)
3. **Spec 获取接口** (`GET /plugins/specs/:pluginKind/:specName`)
4. **脚本安装** (`scripts/plugins-install.ts`)

插件文件（`manifest.json`、`operations.json`、`icon.svg`、`index.js`、`openapi-specs/*`）经过 **发现 → base64 编码 → 写入 `files` 表 → 建立 `Plugin` 实体关联 → 前端/查询运行时消费** 的完整链路。

---

## 一、核心数据模型

### 1. Plugin 实体

[plugin.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/plugin.entity.ts)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | PrimaryKey | 插件实例 ID |
| `pluginId` | string | 插件种类标识（如 `openai`、`hubspot`），全局唯一 |
| `name` / `description` | string | 展示元信息 |
| `repo` | string | GitHub 仓库地址（可选） |
| `version` | string | 插件版本 |
| `indexFileId` | string | 关联 `files` 表的 index.js |
| `operationsFileId` | string | 关联 `files` 表的 operations.json |
| `iconFileId` | string | 关联 `files` 表的 icon.svg |
| `manifestFileId` | string | 关联 `files` 表的 manifest.json |
| `specFilesMap` | jsonb | `{ specName: fileId }` 映射，存储 OpenAPI spec 文件 |

### 2. File 实体

[file.entity.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/entities/file.entity.ts)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | PrimaryKey | 文件 ID |
| `filename` | string | 文件名（如 `index`、`manifest`、`accounting`） |
| `data` | bytea | base64 编码后的文件内容，以二进制存储 |

---

## 二、入口一：`/plugins/install` — HTTP 安装

### 2.1 调用链

```
前端 MarketplacePlugins.jsx
  → plugins.service.js: installPlugin()
    → PluginsController.install() [POST /plugins/install]
      → PluginsService.install()
        → PluginsUtilService.fetchPluginFiles()   // 发现文件
        → PluginsUtilService.create()             // 写入 DB
          → FilesRepository.createOne() × N       // 每个文件一条记录
          → manager.save(Plugin)                  // 建立关联
```

### 2.2 文件发现：`fetchPluginFiles()`

[util.service.ts#L120-L262](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/util.service.ts#L120-L262)

根据 `repo` 参数是否存在，走两条路径：

#### A. GitHub Release 路径 (`fetchPluginFilesFromRepo`)

1. 请求 `https://api.github.com/repos/{repo}/releases/latest` 获取最新发布
2. 并行下载 `zipball_url`（源码包）和 `assets[0].browser_download_url`（编译后的 `index.js`）
3. 用 `jszip` 解压 zipball，**通过文件名模式匹配**发现关键文件：
   - `manifest.json` → `manifestFileKey`
   - `icon.svg` → `iconFileKey`
   - `operations.json` → `operationsFileKey`
   - `openapi-specs/*.yaml` 或 `openapi-specs/*.json` → `specFileKeys[]`
4. 提取所有 spec 文件内容到 `specFiles: Record<string, string>`，key 为文件名去扩展名

#### B. S3 / 本地开发路径 (`fetchPluginFilesFromS3`)

**生产环境**：从 `TOOLJET_MARKETPLACE_URL`（默认 S3）下载：
- `{host}/marketplace-assets/{id}/dist/index.js`
- `{host}/marketplace-assets/{id}/lib/operations.json`
- `{host}/marketplace-assets/{id}/lib/icon.svg`
- `{host}/marketplace-assets/{id}/lib/manifest.json`

**Spec 文件发现**：先解析 `operations.json`，通过 `extractSpecNamesFromOperations()` 递归查找所有 `specUrl` / `spec_url` 字段，提取 `@spec/{kind}/{name}` 中的 name 列表，再逐个尝试 `.yaml` → `.json` 下载。

**开发环境**：从本地 `../marketplace/plugins/{id}/` 目录读取，直接扫描 `openapi-specs/` 目录获取 spec 文件。

### 2.3 base64 编码与 File 实体写入

[util.service.ts#L58-L118](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/util.service.ts#L58-L118)

在 `create()` 方法中：

```typescript
// 四大核心文件并行创建
Object.keys(files).map(async (key) => {
  const fileDto = new CreateFileDto();
  fileDto.data = encode(file);    // js-base64 encode
  fileDto.filename = key;         // "index" / "operations" / "icon" / "manifest"
  uploadedFiles[key] = await this.filesRepository.createOne(fileDto, manager);
});
```

`encode()` 来自 `js-base64` 库，将 ArrayBuffer/string 编码为 base64 字符串，存入 `files.data`（bytea 列）。

### 2.4 specFilesMap 的构建

[util.service.ts#L326-L343](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/util.service.ts#L326-L343)

`storeSpecFiles()` 方法：
- 遍历 `specFiles: Record<string, string>`
- 每个 spec 创建一个 `File` 实体（同样 base64 编码）
- 返回 `specFilesMap: { [specName]: fileId }`

命名约定：`openapi-specs/accounting.yaml` → `specFilesMap["accounting"] = "uuid-file-id"`

### 2.5 Plugin 实体关联

最终 `Plugin` 实体保存时，通过以下字段建立与 `files` 表的关系：
- `indexFileId` = `uploadedFiles.index.id`
- `operationsFileId` = `uploadedFiles.operations.id`
- `iconFileId` = `uploadedFiles.icon.id`
- `manifestFileId` = `uploadedFiles.manifest.id`
- `specFilesMap` = `{ specName: fileId, ... }`（jsonb 列）

---

## 三、入口二：`/plugins/:id/reload` — HTTP 重载

### 3.1 调用链

```
前端 InstalledPlugins.jsx
  → plugins.service.js: reloadPlugin()
    → PluginsController.reload() [POST /plugins/:id/reload]
      → PluginsService.reload()
        → PluginsUtilService.fetchPluginFiles()      // 重新拉取
        → FilesRepository.updateOne() × N            // 原地更新已有 File
        → PluginsUtilService.updateSpecFilesForReload() // 更新 spec 文件
        → manager.save(updatedPlugin)                // 更新 Plugin
```

### 3.2 四大文件的原地更新

[service.ts#L93-L144](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/service.ts#L93-L144)

与 `install` 不同，`reload` 不创建新的 `File` 实体，而是**复用现有 fileId 做原地更新**：

```typescript
Object.keys(files).map(async (key) => {
  const fileDto = new UpdateFileDto();
  fileDto.data = encode(file);
  fileDto.filename = key;
  uploadedFiles[key] = await this.fileRepository.updateOne(
    plugin[`${key}FileId`],  // 使用已有的 fileId
    fileDto,
    manager
  );
});
```

### 3.3 Spec 文件更新策略与陈旧 Spec 问题

[util.service.ts#L351-L388](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/util.service.ts#L351-L388)

`updateSpecFilesForReload()` 采用三种策略：

| 场景 | 处理方式 |
|------|----------|
| spec 名在新旧版本中都存在 | **原地更新** `File` 实体（同 fileId） |
| spec 名是新增的 | 创建新的 `File` 实体，加入 `specFilesMap` |
| spec 名只在旧版本中存在（已删除） | **保留为孤儿 File 实体**，`specFilesMap` 中移除引用 |

**为什么会留下陈旧 spec？**

代码注释明确标注为技术债务（Tech Debt）：

```typescript
// Stale specs (in currentMap but not in newSpecFiles) are left as orphan File entities.
// TODO: Delete stale spec files as part of the broader plugin file cleanup effort.
```

原因：
1. `specFilesMap` 只记录"当前有效"的 spec 映射，旧 spec 对应的 `File` 行仍存在于 `files` 表中但失去引用
2. 这是全插件范围的问题——卸载插件时也不会删除 index/operations/icon/manifest 对应的 File 实体
3. 属于预存的技术债务，并非 spec 特有

---

## 四、入口三：`/plugins/specs/:pluginKind/:specName` — Spec 获取

### 4.1 调用链

```
前端 openapi.service.js: fetchSpecFromUrl()
  → 检测到 URL 以 "@spec/" 开头
  → 替换为 {apiUrl}/plugins/specs/{kind}/{name}
    → PluginsController.getSpec()
      → PluginsService.findByKind(pluginKind)
      → 从 plugin.specFilesMap[specName] 取出 fileId
      → FilesRepository.getOne(fileId)
      → base64 decode 后返回
```

### 4.2 前端 `@spec/` 协议解析

[openapi.service.js#L11-L17](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/frontend/src/_services/openapi.service.js#L11-L17)

```javascript
function resolveSpecUrl(url) {
  if (typeof url === 'string' && url.startsWith('@spec/')) {
    const specPath = url.slice('@spec/'.length);
    return `${config.apiUrl}/plugins/specs/${specPath}`;
  }
  return url;
}
```

`operations.json` 中的 `specUrl: "@spec/hubspot/contacts"` 会被前端解析为：
`https://api.example.com/plugins/specs/hubspot/contacts`

### 4.3 后端 Spec 分发

[controller.ts#L57-L73](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/controller.ts#L57-L73)

```typescript
async getSpec(@Param('pluginKind') pluginKind, @Param('specName') specName, @Res() res) {
  const plugin = await this.pluginsService.findByKind(pluginKind);
  const fileId = plugin.specFilesMap[specName];
  const file = await this.filesRepository.getOne(fileId);

  const content = decode(file.data.toString('utf8'));  // base64 解码
  const isJson = specName.endsWith('.json') || content.trimStart().startsWith('{');
  res.setHeader('Content-Type', isJson ? 'application/json' : 'text/yaml');
  res.setHeader('Cache-Control', 'public, max-age=3600');
  res.send(content);
}
```

注意：
- 路由 `@Get('specs/:pluginKind/:specName')` 必须声明在 `@Get(':id')` **之前**，否则 `:id` 通配符会捕获 `specs` 路径
- 返回时设置了 1 小时的 HTTP 缓存

---

## 五、入口四：`plugins-install.ts` 脚本安装

### 5.1 脚本入口

[plugins-install.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/scripts/plugins-install.ts)

脚本通过 `PLUGINS_TO_INSTALL` 环境变量指定要安装的插件 ID 列表，启动时批量安装。

### 5.2 AppModule.register 上下文复用

脚本的核心是**复用应用程序的完整 NestJS DI 容器**：

```typescript
async function bootstrap() {
  const nestApp = await NestFactory.createApplicationContext(
    await AppModule.register({ IS_GET_CONTEXT: true }),  // 关键
    { logger: ['error', 'warn'] }
  );

  await validateAndInstallPlugins(nestApp);
  await nestApp.close();
  process.exit(0);
}
```

#### `AppModule.register({ IS_GET_CONTEXT: true })` 的作用

[app/module.ts#L93-L208](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/app/module.ts#L93-L208)

`register` 是一个静态工厂方法，返回 `DynamicModule`：

1. **动态加载模块**：通过 `AppModuleLoader.loadModules(configs)` 加载所有静态和动态模块
2. **条件注册**：各子模块的 `register(configs, isMainImport)` 方法根据 `IS_GET_CONTEXT` 决定是否注册控制器等
3. **PluginsModule** 的行为：
   [plugins/module.ts#L7-L20](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/module.ts#L7-L20)

```typescript
static async register(configs, isMainImport = false): Promise<DynamicModule> {
  return {
    module: PluginsModule,
    controllers: isMainImport ? [PluginsController] : [],  // 脚本模式下无 HTTP 控制器
    providers: [PluginsService, FilesRepository, PluginsUtilService, ...],
    exports: [PluginsUtilService, PluginsService],
  };
}
```

#### 为什么脚本可以复用业务逻辑

通过 `nestApp.get(PluginsService)` 获取服务实例后，脚本直接调用 `pluginsService.install(dto)`，**与 HTTP 接口走的是完全相同的业务逻辑**（`PluginsService.install` → `PluginsUtilService.create` → 写入数据库）。

### 5.3 脚本安装流程

```
1. 读取 PLUGINS_TO_INSTALL 环境变量
2. 从 assets/marketplace/plugins.json 查找插件详情
3. 用 class-validator 校验 CreatePluginDto
4. 检查 pluginId 是否已存在（跳过已安装）
5. 逐个调用 pluginsService.install()
   → 内部逻辑与 HTTP 安装完全一致
6. 输出安装结果，关闭应用上下文
```

### 5.4 `plugins-reload.ts` 脚本

[plugins-reload.ts](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/scripts/plugins-reload.ts)

与安装脚本模式相同，也是通过 `AppModule.register({ IS_GET_CONTEXT: true })` 创建上下文，然后调用 `pluginsService.reload()`。

特殊逻辑：
- **Cloud 版本**：从 `PLUGINS_TO_RELOAD` 环境变量获取列表
- **非 Cloud 版本**：重载数据库中**所有**已安装的插件
- 由于 `pluginId` 没有唯一性约束（UI 层面限制），脚本会找到所有同 `pluginId` 的插件实例一并 reload

---

## 六、前端 / 查询运行时如何消费插件

### 6.1 前端消费

#### 插件列表获取
[controller.ts#L45-L54](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/plugins/controller.ts#L45-L54)

`GET /plugins` 返回所有已安装插件，返回时做两层解码：
- `iconFile.data` → 转 UTF-8 字符串（SVG 文本）
- `manifestFile.data` → base64 解码 → JSON.parse

#### 数据源关联插件
[data-sources/service.ts#L73-L94](file:///Users/zhangjing/Desktop/so-coders/0609-under/ToolJet/server/src/modules/data-sources/service.ts#L73-L94)

当数据源关联了 Marketplace 插件（`pluginId` 非空）时，返回数据源列表会一并加载插件的 `iconFile`、`manifestFile`、`operationsFile`，并对后两者做 base64 解码 + JSON.parse。

前端拿到这些数据后，用于渲染：
- 数据源选择器中的插件图标和名称
- 查询编辑器中的操作列表（来自 operations.json）
- 数据源配置表单（来自 manifest.json 的 properties 定义）

### 6.2 查询运行时消费

查询运行时（`@tooljet/plugins` 包）通过加载 `index.js` 中的插件类来执行实际的查询操作。`index.js` 文件内容存储在 `files` 表中，由插件运行时加载器读取并执行（通常在沙箱/worker 环境中）。

---

## 七、总结：全链路对比

| 维度 | install 安装 | reload 重载 | plugins-install 脚本 |
|------|-------------|------------|---------------------|
| 入口 | HTTP POST | HTTP POST | Node 脚本 |
| 鉴权 | JWT + 能力守卫 | JWT + 能力守卫 | 无（脚本直连 DB） |
| 文件发现 | fetchPluginFiles | fetchPluginFiles | 同 install |
| File 实体 | 创建新行 | 原地更新 | 同 install |
| specFilesMap | 全新构建 | 更新/新增，保留陈旧项 | 同 install |
| 业务逻辑 | PluginsService.install | PluginsService.reload | 复用 PluginsService |
| Nest 上下文 | HTTP 请求上下文 | HTTP 请求上下文 | AppModule.register ApplicationContext |

### 关键设计要点

1. **统一存储模型**：所有插件文件（含 spec）都通过 `File` 实体存储，base64 编码后存入 bytea 列
2. **specFilesMap 作为索引**：jsonb 列实现动态键值映射，无需为 spec 建关联表
3. **`@spec/` 协议层**：前端通过虚拟协议间接访问，解耦 operations.json 与实际存储位置
4. **脚本复用完整 DI**：通过 `AppModule.register({ IS_GET_CONTEXT: true })` 创建非 HTTP 应用上下文，脚本与 HTTP 接口共享 100% 业务逻辑
5. **已知技术债务**：reload/卸载会留下孤儿 File 实体，需统一的插件文件清理机制
