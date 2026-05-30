# Spec 文件管理与加载机制

wps365-cli 基于 OpenAPI 3.0 规范文件驱动命令生成。本文档描述 spec 文件的存储位置、自动下载、增量更新与自定义覆盖的完整机制。

## 概述

CLI 在运行时读取两类 spec 文件：

| 文件 | 路径 | 内容 | 行数 |
|------|------|------|------|
| API 规范 | `spec/api/365.yaml` | OpenAPI 3.0 全量接口定义（801 paths） | ~74k |
| 精装目录 | `spec/curated/365.yaml` | 精装命令的声明式目录（152 commands） | ~5k |

此外还有自定义覆盖目录（详见[自定义覆盖](#自定义覆盖)）。

## 存储位置

spec 文件存放在配置目录下，路径因操作系统而异：

| OS | 路径 |
|----|------|
| macOS | `~/Library/Application Support/wps365-cli/spec/` |
| Linux | `~/.local/share/wps365-cli/spec/` 或 `$XDG_DATA_HOME/wps365-cli/spec/` |
| Windows | `%APPDATA%\wps365-cli\spec\` |

目录结构：

```
spec/
├── api/
│   ├── 365.yaml              # 官方 API 规范
│   ├── 365.yaml.md5          # 缓存校验
│   └── customs/              # 用户自定义 API 覆盖
│       └── my-extension.yaml
├── curated/
│   ├── 365.yaml              # 官方精装目录
│   ├── 365.yaml.md5          # 缓存校验
│   └── customs/              # 用户自定义精装覆盖
│       └── my-commands.yaml
└── osh/                      # OSH 网关 spec（按需下载）
    └── ...
```

## 自动下载

### 触发时机

当 CLI 执行任何精装命令时，内部调用 `EnsureLocalSpecs` → `ensureSpecs`，检查 spec 文件是否存在：

- 两个 spec 文件都存在 → 跳过下载，直接加载
- 任一文件缺失 → 自动从远程下载

可通过环境变量控制：

```bash
# 禁止自动下载（缺省：true）
WPS365_SPEC_AUTO_DOWNLOAD=false
```

禁用后，若 spec 文件缺失，精装命令将不可用，但仍可使用 `api` 命令直接调用接口。

### 下载源

默认远程地址：

```
https://open.wps.cn/cli/specs/v1/{api,curated}/365.yaml
```

可通过环境变量覆盖：

```bash
WPS365_SPEC_BASE_URL=https://your-mirror.example.com/specs
```

内部通过 `effectiveSpecBaseURL` 解析最终地址。

### 缓存与增量更新

每次下载后，CLI 将文件的 MD5 哈希存为 `.md5` 后缀文件。下次启动时 `checkSpecUpdates` 比对远程哈希：

- 哈希未变 → 跳过下载
- 哈希变化 → 重新下载并更新 `.md5` 文件

`specURLWithHash` 函数在下载 URL 后追加哈希后缀用于缓存控制。

### 手动更新

```bash
# 检查并下载最新 spec
wps365-cli spec update
```

## 加载流程

```
用户执行命令
  │
  ├─ EnsureLocalSpecs()           # 确保本地 spec 存在
  │   └─ ensureSpecs()            # 缺失时触发自动下载
  │
  ├─ Load() / LoadWithOverrides() # 加载 spec 到内存
  │   ├─ loadAPISpec()            # 解析 api/365.yaml
  │   ├─ loadOfficialCatalog()    # 解析 curated/365.yaml
  │   ├─ loadOshCatalog()        # 解析 osh/ 目录（如存在）
  │   └─ loadCatalogDir()        # 解析 customs/ 目录
  │
  └─ 命令执行                      # 根据加载的 spec 路由到具体处理逻辑
```

加载后，CLI 将 OpenAPI paths 注册为 `api` 子命令，将 curated commands 注册为语义化子命令。

## spec 子命令

| 命令 | 说明 |
|------|------|
| `spec update` | 检查远程更新并下载 |
| `spec set --api <file>` | 替换官方 API 规范文件 |
| `spec set --curated <file>` | 替换官方精装目录文件 |
| `spec add --custom-api <file>` | 添加自定义 API 覆盖到 customs 目录 |
| `spec add --custom-curated <file>` | 添加自定义精装覆盖到 customs 目录 |

`spec set` 是替换操作——将官方 spec 替换为指定文件，CLI 后续使用替换后的版本。

`spec add` 是叠加操作——在官方 spec 基础上添加自定义覆盖，两者合并生效。

## 自定义覆盖

### API 覆盖

将 OpenAPI 3.0 格式的 YAML 文件放入 `spec/api/customs/` 目录（或使用 `spec add --custom-api`）。CLI 加载时会合并官方 spec 与所有 customs 文件。

适用场景：
- 补充官方 spec 尚未收录的接口
- 覆盖官方 spec 中描述不准确的字段
- 内部测试环境使用不同的接口定义

### 精装命令覆盖

将精装目录格式的 YAML 文件放入 `spec/curated/customs/` 目录（或使用 `spec add --custom-curated`）。CLI 加载时会合并官方目录与所有 customs 文件。

适用场景：
- 为高频接口添加更友好的命令别名
- 为团队内部接口创建专用命令
- 调整官方命令的默认参数或 body 绑定

### 自定义 API spec 格式要求

`spec/api/customs/` 下的文件必须符合 OpenAPI 3.0 规范。只需定义需要覆盖或补充的 paths，不需要重复官方 spec 中已有的内容。

示例——补充一个内部接口：

```yaml
openapi: "3.0.0"
info:
  title: Custom API extensions
  version: "1.0"
paths:
  /v7/internal/reports:
    get:
      summary: 获取内部报表
      operationId: getInternalReport
      responses:
        "200":
          description: 成功
```

### 自定义精装目录格式要求

`spec/curated/customs/` 下的文件必须遵循精装目录格式：

```yaml
version: 1
commands:
  - id: my.resource.action
    command: my resource action
    summary: 我的自定义命令
    method: GET
    path: /v7/my/resource
    args: []
    flags:
      - name: verbose
        type: bool
        required: false
        to: query.verbose
    body:
      bindings: []
    examples:
      - command: 'wps365-cli my resource action --verbose'
```

关键字段：
- `version` 必须为 `1`
- `id` 在所有目录中必须唯一，冲突时 customs 优先
- `command` 定义 CLI 命令路径（空格分隔资源层级）
- `body.bindings` 的 `transform` 支持：`split_csv`、`to_int`、`to_bool`、`parse_json`、`trim`、`wrap`、`negate`

### 合并优先级

当官方 spec 与 customs 存在重叠定义时：

- **精装目录**：以 `id` 为键，customs 中的命令**整体替换**同 id 的官方命令（非字段级合并）
- **API 规范**：以 path 为键，customs 中的 path 定义**覆盖**同 path 的官方定义（path 级替换，非字段级合并）

简言之：重叠时 customs 整体替换，不会做字段级部分合并。

## OSH 网关 Spec

OSH（企业网关）模式的 spec 处理与标准模式不同：

- 通过 `pullOSHSpecs` / `syncOSHZip` 单独下载
- 以 ZIP 格式传输，本地解压
- 存放在 `spec/osh/` 目录下
- 由 `loadOshCatalog` 加载

OSH spec 的下载受 OSH 认证状态控制，仅在 `auth login --osh` 后才可获取。

## 环境变量速查

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `WPS365_SPEC_AUTO_DOWNLOAD` | `true` | 是否自动下载缺失的 spec |
| `WPS365_SPEC_BASE_URL` | `https://open.wps.cn/cli/specs/v1` | spec 远程下载地址 |
| `WPS365_CONFIG_DIR` | 系统默认 | 配置目录（含 spec 子目录） |

> **注意区分**：`WPS365_SPEC_BASE_URL` 控制 spec 文件的下载源（YAML 规范文件从哪里拉取），而 `WPS365_API_BASE` 控制 API 请求的目标端点（运行时 HTTP 请求发往哪里）。两者相互独立——可以使用官方 spec 描述文件，同时将 API 请求指向内部测试环境。

## 实现包

核心逻辑位于 `wps365-cli/internal/specfile` 包：

| 函数 | 职责 |
|------|------|
| `EnsureLocalSpecs` | 入口：确保本地 spec 就绪 |
| `ensureSpecs` | 检查并触发下载 |
| `checkSpecUpdates` | 增量更新检查 |
| `downloadSpec` | 下载单个 spec 文件 |
| `specURLWithHash` | 构造带哈希的下载 URL |
| `effectiveSpecBaseURL` | 解析远程地址 |
| `Load` / `LoadWithOverrides` | 加载并合并所有 spec |
| `loadAPISpec` | 解析 OpenAPI 3.0 YAML |
| `loadOfficialCatalog` | 解析精装目录 |
| `loadOshCatalog` | 解析 OSH 目录 |
| `loadCatalogDir` | 解析 customs 目录 |
| `pullOSHSpecs` / `syncOSHZip` | OSH spec 下载与解压 |
