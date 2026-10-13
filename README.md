# dsh-openrouter-providers

中文 | [English](README.en.md)

DeepSeek Harness 插件：填写 OpenRouter 请求使用的**提供商列表**与**量化位数限制**，并把它们作为 `provider.only` / `provider.order` / `provider.quantizations` 路由参数注入到所有 OpenRouter 模型请求中。可选地把这些请求改经 [billion-context](https://github.com/ranxianglei/billion-context) 的压缩代理发出。配置由插件自持，写入 `$DSH_HOME/openrouter-providers.json`（未设置 `DSH_HOME` 时为 `~/.dsh/openrouter-providers.json`），重启后自动恢复。

**适配版本**：DeepSeek Harness `0.1.5-rc.1` ~ `0.2.1-alpha.2`（`engines.dsh: >=0.1.5-rc.1 <0.3.0`）。本插件不 import 任何 `@deepseek-ai/dsh-*` 包，唯一 peer 是 `@deepseek-ai/cordis`，因此不受 0.1.7 / 0.2.x 破坏性改动影响；跨 0.2 主版本已在真实 `0.2.0-rc.2`、`0.2.1-alpha.1` 与 `0.2.1-alpha.2` 上逐项复验（配置读写往返、`llm/stream` 重路由、真实 OpenRouter 路由、billion-context 代理接入、插件页显示名与图标、设置页表单渲染）。

> **为什么不再用 settings 服务**：DSH `0.1.7` 删除了 settings 命名空间注册（`settings.register` / `SettingsScope` / `watch`），改为只枚举 profile 条目 Config 上的 `.volatile()` 字段。依赖旧 API 的插件在 0.1.7 上会**静默失效**——`apply` 提前返回，HTTP 路由与 `llm/stream` 重路由都不再注册，设置页显示「无法读取当前状态（Host 端不可用）」，且请求注入完全不生效。本插件改为自持配置文档，并只把旧命名空间当作**能力探测的可选增强**（见下），因此在新旧宿主上行为一致。

**配置入口（双栈）**：DSH `0.1.6-alpha.2` 把插件配置从设置搬到新的侧栏**「插件」页**，并退役了 `settings.plugin.item`。本插件同时注册两个 slot，由宿主声明决定哪个生效（未声明的那个永不触发，不会报错）：

| 宿主 | slot | 位置 |
| --- | --- | --- |
| DSH ≥ `0.1.6-alpha.2` | `plugins.bundle.config`（键=包名） | 侧栏**插件** → 本组合包页面内的配置表单 |
| DSH < `0.1.6-alpha.2` | `settings.plugin.item` | **设置** → **插件** → **插件配置** 内的折叠卡片 |

旧宿主上，那半卡片按 Host 已服务的 settings 命名空间派发，所以本插件会**探测** `settings.register` 是否存在，存在才注册一个空的 pass-through 命名空间（只为让卡片能被派发，不承载任何取值）；0.1.7 起该 API 不存在，跳过即可，不影响其余功能。

## 功能

- **配置表单**：填写提供商 slug 列表（每行一个）、选择路由模式、选择量化位数限制：
  - **仅允许这些提供商** → 请求体注入 `provider: { only: [...], allow_fallbacks: false }`
  - **按顺序优先尝试** → 注入 `provider: { order: [...], allow_fallbacks: true }`
  - **量化位数限制** → 注入 `provider: { quantizations: ['int4' | 'int8' | ...] }`（可选，默认不限制；合法值见 [OpenRouter Quantization](https://openrouter.ai/docs/guides/routing/provider-selection#quantization)）
  - 可整体开关；保存后写入插件自持的配置文件（`$DSH_HOME/openrouter-providers.json`）。新「插件」页只有保存会写入，离开页面丢弃暂存修改；旧折叠卡片另提供「撤销」按钮与「未保存」徽标。
- **界面跟随 DSH 语言**：文案来自插件注册的 `openrouter-providers` locale 命名空间（`zh` / `en` 双语词典），切换语言后界面即时重渲染，无需刷新页面；宿主没有 locale 服务时回退中文文案。
- **「插件」页显示名称与图标**：包内 `locale/zh.json` 与 `locale/en.json` 的 `meta.title` / `meta.description` 提供本地化显示名（中文「OpenRouter 提供商列表」/ 英文 `OpenRouter Providers`），因此插件列表里显示的是可读名称而非裸包名 `dsh-openrouter-providers`；`package.json` 的 `icon: "./icon.svg"` 提供卡片与行内图标（36×36 三层渐隐圆角条，示意「按列表逐级收窄提供商」）。两者都跟随 DSH 界面语言。注意 `exports` 必须包含 `"./locale/*.json": "./locale/*.json"`，否则 Node 的 exports 解析会抛 `ERR_PACKAGE_PATH_NOT_EXPORTED`，显示名静默回退为包名；`icon` 必须是**相对路径**、扩展名限 `.svg/.png/.jpg/.jpeg/.webp`、须位于包目录内且 ≤256 KiB，写错会报错（但只丢图标，名称仍生效）。
- **请求注入**：监听 `llm/stream` waterfall——当请求的 provider 路由为 `openrouter`（已启用且列表非空或设置了量化限制）时，把请求重路由到插件自研的 chat-completions adapter，由它构造请求体注入 `provider` 字段；`reasoning.effort`（off/low/medium/high/max，均为 OpenRouter 合法值）按契约透传。会话日志与 UI 仍显示 `openrouter`。
- **billion-context 压缩代理（可选）**：把代理开关设为**自动**后，插件会探测本机运行中的 [billion-context](https://github.com/ranxianglei/billion-context) 代理，命中则把请求改经它的 `/bili/` 入口发出（`<origin>/bili/https://openrouter.ai/api/v1/chat/completions`），从而获得其上下文压缩；探测不到代理时**直连**，不会因为装了本插件而让生成失败。代理开关与「提供商限制」总开关**互相独立**：只想压缩、不想限制路由时，关掉提供商限制、单独开启代理即可。
- **凭据**：复用现有 `OPENROUTER_API_KEY`（通过 `credentials` 服务解析，与 `llm-pi-ai` 的 `apiKeyEnv` 一致）。
- **应用归属（App Attribution）**：请求携带 `HTTP-Referer: https://github.com/deepseek-ai/deepseek-harness`、`X-OpenRouter-Title: DeepSeek Harness OpenRouter` 与 `X-OpenRouter-Categories: cli-agent` 头，使 OpenRouter 界面/排行榜中显示为 `DeepSeek Harness OpenRouter` 而非 Unknown（[OpenRouter App Attribution 文档](https://openrouter.ai/docs/app-attribution)）。

## 安装（bundle 挂载）

插件以 **bundle 包**形式挂载，与其他插件（dshmarket、dsh-recall-plugin 等）一致，通过 `cordis.patch.yml` 的 insert 行自动合并进 profile composition。

### 方式 A：从 npm 安装（推荐）

在 profile 目录（如 `~/.dsh/profiles/web/`）执行：

```bash
pnpm add dsh-openrouter-providers@latest
```

然后把包加入 profile 的 `package.json` 的 `dsh.profile.bundles` 列表：

```json
"dsh": { "profile": { "bundles": [ ..., "dsh-openrouter-providers" ] } }
```

再重启 `dsh web`。bundle 协调器自动合并包内的 `cordis.patch.yml`（insert 行：`id: openrouter-providers`）。

### 方式 B：从 GitHub 安装

```bash
pnpm add dsh-openrouter-providers@github:MoRanYue/dsh-openrouter-providers
```

其余步骤与方式 A 相同（加入 `dsh.profile.bundles` 并重启）。

### 方式 C：本地路径

```bash
pnpm add dsh-openrouter-providers@file:D:\\path\\to\\dsh-openrouter-providers
# 然后同样把包名加入 dsh.profile.bundles 并重启
```

### 手动方式（不依赖 bundle）

在 `~/.dsh/profiles/web/cordis.patch.yml` 中追加：

```yaml
- insert:
    - id: openrouter-providers
      name: 'dsh-openrouter-providers'
```

并在 profile 的 `package.json` 中把包加入 `dependencies` 与 `dsh.profile.bundles`，然后 `pnpm install` 并重启。

## 使用

1. 确保模型的 provider 路由为 `openrouter`（模型选择器中选中 OpenRouter 下的模型）。
2. 打开对应宿主的配置入口（见上表：DSH ≥ `0.1.6-alpha.2` 用侧栏**插件** → 本组合包页面；更早的宿主用**设置** → **插件** → **插件配置**），填写提供商 slug（如 `DeepInfra`、`Together`），选择路由模式与量化位数限制，保存。
3. 之后的 OpenRouter 请求都会携带注入的 provider 路由参数。
4. 如需上下文压缩：先启动 billion-context 代理（`bili`），再把本插件的**代理**一项设为**自动**并保存。表单下方会显示是否已接入（`代理已接入: http://127.0.0.1:18790`）或未检测到；检测到即生效，无需重启 DSH。

> 提供商 slug 格式参见 [OpenRouter Provider Routing 文档](https://openrouter.ai/docs/guides/routing/provider-selection)（`order`/`only` 字段）。

## 工作原理（简要）

- **传输层**：动态插件环境没有 `fetch` 内置，adapter 通过 `subprocess` 服务派生 `node -e` 子进程执行 HTTP + SSE 流式解析（文本/推理/工具调用增量、usage、`[DONE]`、错误分类 AUTH/RATE_LIMIT/INVALID_REQUEST/SERVER 等）。
- **状态持久化**：配置由插件自持，写入 `$DSH_HOME/openrouter-providers.json`（`DSH_HOME` 未设置时为 `~/.dsh/…`），与 settings 服务无关。首次启动且配置文件仍为出厂默认值时，会按顺序尝试从旧位置恢复一次：`<workspaceRoot>/.dsh-plugins/openrouter-providers.json`（v1.0.4 及以前）、`$DSH_HOME/settings.yaml.imported`（0.1.7 移除设置文档时留下的、含本插件 section 的改名文件）。任一处有值即迁移，之后不再覆盖已有配置。
- **Client 通信**：插件配置卡通过 HTTP API `GET/POST /api/openrouter-providers/state` 读写状态（Host 端经 `webServer` 注册）。客户端对非 2xx 响应会显示状态码与响应片段，不再把所有失败笼统报成「Host 端不可用」。
- **adapter 契约**：插件注册的是普通对象 adapter（不是 `LlmAdapter` 子类），没有基类默认值兜底，所以必须**逐项**实现宿主会调用的成员。除 `stream` 外，`LlmRuntime` 会调用 `providerInfo` / `providerRetryPolicy` / `imageRequestPricing` / `listModels` / `resolveModel` / `prepareCall`——少任何一个，对应调用点就会抛 `TypeError: adapter.<name> is not a function`。本插件的 `listModels()` 返回空数组，因此该合成路由不会出现在模型目录里（只能由 `llm/stream` 在传输层内部使用）；`imageRequestPricing()` 返回 `undefined`，让 token meter 与 spill policy 沿用各自的中性估算。
- **billion-context 代理接入**：代理地址**不硬编码端口**，按 billion-context 自身的发现顺序解析：`BILI_MCP_PROXY` 环境变量 → `$XDG_STATE_HOME/billion-context/proxy-origin`（裸 URL 或实例 JSON 两种形态都支持）→ 同目录 `port-zone.json` 的 `lanes.dsh` → 兜底 `http://127.0.0.1:8787`；每个候选都先请求 `/__bili/health` 确认 `ok: true` 才采用。探测**刻意用 `node:http`/`node:https` 而非 `globalThis.fetch`**——billion-context 挂载后会把宿主的 `fetch` 补丁成「重写 URL 为 `<origin>/bili/<原URL>`」，若用 `fetch` 探测，探测请求本身会被改写成指向代理自己的请求，导致永远「探测成功」。结果缓存 30 秒并在后台刷新，请求路径只读缓存（不会为探测阻塞生成）；设置页每次加载都会重新探测，并显示当前是否真的接入。请求会带 `x-acp-session: <sessionId>` 头，让代理按会话稳定分桶累积压缩状态（该值来自宿主在 `GenerateOptions.sessionId` 上盖的会话标识，专为这类传输层元数据提供）。

## 限制

- 图像输入不支持（模型请求含图片时返回 `UNSUPPORTED_CONTENT`）。
- `only` 模式下 `allow_fallbacks=false`：列表中的提供商全部不可用时请求失败（OpenRouter 行为）。
- 代理接入是**按请求**判断的：代理中途停止时，最多 30 秒（探测缓存 TTL）后自动回退直连；代理中途启动时同样最多 30 秒后自动接入，或打开一次设置页立即生效。
- 代理与「提供商限制」共用同一条传输通道（本插件的 adapter），因此开启代理时请求同样绕过宿主原生的 `openrouter` 路由——如果其它插件（如 billion-context 的 dsh-native）依赖宿主 `globalThis.fetch` 做拦截，那些拦截不会作用于本插件的流量，这是自持传输层的必然结果。

## License

MIT
