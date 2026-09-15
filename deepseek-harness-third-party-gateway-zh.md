# DeepSeek Harness 接入第三方 OpenAI 兼容网关：配置与排障记录

本文记录把 DeepSeek Harness（`dsh`）接到一个**第三方 OpenAI 兼容网关**（自建/中转，用私有 CA 签发证书）时踩到的四类问题、根因与修复方案。

内容已做脱敏处理：网关域名、CA 名称、环境变量名、provider id 均以占位符表示，不含任何真实密钥或内部地址。替换占位符后即可复用。

适用读者：在自有机器上跑 `dsh web`，并使用非官方模型网关的开发者。

**占位符约定**

| 占位符 | 含义 | 本文示例值 |
| --- | --- | --- |
| `<GATEWAY>` | 网关主机名 | `api.example.net` |
| `<GATEWAY_ENV>` | 存放网关密钥的环境变量/凭据引用名 | `GATEWAY_API_KEY` |
| `<CA_NAME>` | 网关所用私有 CA 的名称 | `Example Root CA` |
| `$DSH_HOME` | Harness 主目录 | 默认 `~/.dsh` |
| `<MODEL>` | 网关提供的模型 id | `deepseek-v4.1-flash` |

---

## 一、问题总览

| # | 现象 | 根因层级 | 修复位置 |
| --- | --- | --- | --- |
| 1 | 对话报 `{"message":"Connection error.","code":"TRANSPORT"}` 并重试 5 次 | Node TLS 信任 | 启动环境变量 |
| 2 | 模型选择器里**没有思考级别**可选 | 手工声明的路由缺 reasoning 元数据 | `settings.yaml` |
| 3 | 上下文 / 输出上限配错 | 配置值 | `settings.yaml` |
| 4 | 用 `web_search` 报 `has no API key for "DEEPSEEK_API_KEY"` | web 搜索是独立子系统 | 自定义 provider 插件 |

---

## 二、问题 1：私有 CA 网关在 Node 下 TLS 握手失败

### 现象

对话请求全部失败，会话日志里能看到固定的重试序列：

```
provider: <provider>, retry: 1..5,
failure: {"message":"Connection error.","code":"TRANSPORT"}
```

注意 **`"Connection error."` 是被上游库吞掉后的泛化消息**，不是真实错误。

### 根因

网关证书由**私有 CA** 签发（链：leaf → 中间 CA → 自签根 CA），而这个根 CA **只在操作系统的钥匙串里被信任**。

- `curl` / 浏览器：读系统钥匙串 → 握手成功（`SSL certificate verify ok`）。
- **Node.js：不读系统钥匙串**，只加载编译进二进制的公共根证书（本机实测 146 个）。私有 CA 不在其中 → 握手失败，底层错误是：

```
SELF_SIGNED_CERT_IN_CHAIN | self-signed certificate in certificate chain
```

该错误被上层（`pi-ai` 的 `fetch` 封装）归一化为 `"Connection error."`，再被分类成 `TRANSPORT`，于是表现为「网络错误 + 无意义重试」。

### 关键认知：不是「把 pem 放到某个目录」就行

Node 的根证书是**编译进二进制**的，磁盘上没有可追加的 CA 包。Node 只有三套**可切换**的信任来源，**每套都需要一个开关**：

| pem 放在哪 | 还必须加的开关 |
| --- | --- |
| 系统钥匙串 | `NODE_USE_SYSTEM_CA=1` |
| 任意文件 | `NODE_EXTRA_CA_CERTS=/path/to/ca.pem` |
| OpenSSL 的 `SSL_CERT_FILE` / `SSL_CERT_DIR` | `NODE_OPTIONS=--use-openssl-ca` |

实测对比（同一个 pem、同一个位置，只差开关）：

```
默认                                        FAIL  SELF_SIGNED_CERT_IN_CHAIN
SSL_CERT_FILE=/tmp/ca.pem                   FAIL  SELF_SIGNED_CERT_IN_CHAIN   ← 只放文件，无效
SSL_CERT_FILE=/tmp/ca.pem + --use-openssl-ca  OK
NODE_USE_SYSTEM_CA=1                        OK    ← 读系统钥匙串
NODE_EXTRA_CA_CERTS=/tmp/ca.pem             OK
```

### 验证

```bash
# 不带开关
node -e "fetch('https://<GATEWAY>/v1/models').then(r=>console.log('OK',r.status)).catch(e=>console.log('FAIL',e.cause?.code||e.code))"
# → FAIL SELF_SIGNED_CERT_IN_CHAIN

# 带开关
NODE_USE_SYSTEM_CA=1 node -e "fetch('https://<GATEWAY>/v1/models').then(r=>console.log('OK',r.status)).catch(e=>console.log('FAIL',e.cause?.code||e.code))"
# → OK 401   （401 是没带 key 的正常响应，说明 TLS 已通）
```

也可以对比信任集大小：

```bash
node -e "const t=require('tls');console.log('default:',t.getCACertificates('default').length)"
NODE_USE_SYSTEM_CA=1 node -e "const t=require('tls');console.log('with system:',t.getCACertificates('default').length)"
# default: 146   →   with system: 168   （叠加，不是替换）
```

### 修复：启动前导出环境变量

把开关写进 shell 启动文件（`~/.zshrc`）：

```bash
# 让 Node 除自带的公共根证书外，也信任系统钥匙串里的 CA。
# 用于私有 CA 签发证书的内部端点，否则 Node 会报 SELF_SIGNED_CERT_IN_CHAIN
# 而 curl/浏览器正常。Node 只在启动时读取，无法由 .env 设置。
export NODE_USE_SYSTEM_CA=1
```

然后新开终端（或 `exec zsh`）再启动：

```bash
dsh web
```

**GUI 启动的桌面版**不继承终端环境，需要：

```bash
launchctl setenv NODE_USE_SYSTEM_CA 1   # macOS；之后退出 App 重新打开
```

### 两个坑

1. **不要写进 `.env`**。`dsh` 把 `NODE_EXTRA_CA_CERTS` / `SSL_CERT_FILE` / `SSL_CERT_DIR` / `NODE_TLS_REJECT_UNAUTHORIZED` 等列为 bootstrap-only，任何 `.env`（含项目目录和 `$DSH_HOME/.env`）声明它们都会**直接报错**而非静默忽略。
2. **`NODE_USE_SYSTEM_CA` 只在进程启动瞬间读取**。实测运行时再注入无效：

```bash
node -e "process.env.NODE_USE_SYSTEM_CA='1'; fetch('https://<GATEWAY>/v1/models').then(r=>console.log('OK')).catch(e=>console.log('FAIL',e.cause?.code))"
# → FAIL SELF_SIGNED_CERT_IN_CHAIN   （太晚了）
```

### 安全性权衡

`NODE_USE_SYSTEM_CA=1` 只是把「系统钥匙串的 CA」**追加**到 Node 的公共根证书之上（146 → 168），不是替换，因此公共站点不受影响（实测 `example.com`、公共 API 均正常）。

代价是：Node 从此也信任钥匙串里的一切，包括公司 MDM 或抓包代理安装的根证书。但浏览器和 `curl` 本来用的就是这个信任集，对个人开发机通常可以接受；去掉该变量即完全恢复，无残留。

---

## 三、问题 2：思考级别不可选

### 现象

在 settings 里给模型设 `reasoningEffort`，选择器里却没有档位可选，请求也可能直接失败。

### 根因

第三方网关属于**手工声明的路由**（hand-declared route）——上游库的内置模型目录里没有它。上游解析逻辑大致是：

```js
function resolveModelReasoning(provider, entry, base) {
  const efforts = entry.reasoningEfforts;
  if (efforts === void 0) return { reasoning: base?.reasoning ?? false };  // base 不存在 → false
  // ...
}
function reasoningInfo(model, defaultLevel) {
  if (!model.reasoning) return {};   // 不返回 reasoning 字段 = 选择器不提供任何档位
  // ...
}
```

`base` 是内置目录中的同名条目；手工路由没有，于是 `reasoning: false` → 返回空 → **一个档位都不显示**。

与此同时，默认档位的校验会失败：

```js
function resolveReasoningLevel(model, effort) {
  if (effort === void 0) return void 0;
  if (getSupportedThinkingLevels(model).some(l => l === effort)) return effort;
  throw new LlmError(`... does not support reasoning effort "${effort}"`, "UNSUPPORTED_REASONING_EFFORT");
}
```

此时该模型支持的档位**只有 `off`**，所以默认设成 `high` 会直接抛 `UNSUPPORTED_REASONING_EFFORT`。

### 修复：显式声明 `reasoningEfforts`

在 `settings.yaml` 的模型条目里补上即可。**可用档位完全由你声明什么决定**，固定 7 个：

```
off、minimal、low、medium、high、xhigh、max
```

```yaml
llm-pi-ai:
  providers:
    <provider>:
      apiKeyEnv: <GATEWAY_ENV>
      api: openai-completions
      baseURL: https://<GATEWAY>/v1
      models:
        - id: <MODEL>
          name: <MODEL>
          contextWindow: 1000000
          maxTokens: 393216
          # 键 = 选择器提供的档位；值 = 线上发送的 reasoning_effort 字面量。
          # 只有 off 允许留空，表示「该档有效、选中时不发送任何参数」。
          reasoningEfforts:
            off:
            minimal: minimal
            low: low
            medium: medium
            high: high
            xhigh: xhigh
            max: max

agent-default-model:
  provider: <provider>
  model: <MODEL>
  reasoningEffort: high
```

### 必须知道的真相：这些档位多半是「名义上的」

用**非法类型探测**（传坏值看网关是否 400）可以判断网关真正解析了哪些参数：

| 参数 | 网关是否解析 |
| --- | --- |
| `enable_thinking`（布尔） | ✅ 会校验，传 `"bogus"` → 400 |
| `reasoning_effort` | ❌ 传 `123` / `"ultra"` 都 200 |
| `thinking` / `thinking.type` | ❌ 200 |
| `thinking_budget` / `thinking_token_budget` / `thinking_budget_tokens` |  200 |

但 `reasoning_effort` **并非完全无效**——交错多轮实测输出是系统性不同的：

```
baseline               rt=87 / 87 / 115
reasoning_effort=high  rt=101 / 101 / 101
enable_thinking=false  rt=115 / 115 / 115
```

结论：它确实影响生成，但语义不透明、**非单调**（不是「越高想得越多」）。另外该网关**无法真正关闭思考**——所有组合下都仍会输出 `reasoning_content`，因此 `off` 只表示「不发送参数」，等于网关默认。若不想提供这个误导性档位，把 `off:` 那一行删掉即可（`reasoningEfforts` 只要求至少有一个非 `off` 档）。

> 该网关的错误格式形如 `InternalError.Algo.*`（`InternalError.Algo.InvalidParameter` 等），是某云厂商平台的错误码 —— 说明它是一个套在该平台外面的中转，而非原生 DeepSeek 端点。这也是为什么它在 Anthropic 协议上的行为与官方不同。

---

## 四、问题 3：上下文与输出上限

不要照抄别处的数字，**实测**网关的硬上限。

用 `max_completion_tokens`（OpenAI 兼容字段）逐值试探边界：

```bash
for n in 384000 393216 393217 400000; do
  printf "%-8s " $n
  curl -s -o /dev/null -w "HTTP %{http_code}\n" https://<GATEWAY>/v1/chat/completions \
    -H "Authorization: Bearer $<GATEWAY_ENV>" -H "Content-Type: application/json" \
    -d "{\"model\":\"<MODEL>\",\"messages\":[{\"role\":\"user\",\"content\":\"hi\"}],\"max_completion_tokens\":$n}"
done
```

实测结果：

```
384000  → HTTP 200
393216  → HTTP 200
393217  → HTTP 400  Range of max_completion_tokens ...
400000  → HTTP 400
```

即上限正好是 **393216**（= 384 × 1024）。这与该模型的公开规格「1M 上下文 / 393K 最大输出 / 393K 最大思维链」一致。

最终配置：

```yaml
contextWindow: 1000000
maxTokens: 393216
```

顺带确认的能力（该网关均支持，无需额外配置）：

| 能力 | 结果 |
| --- | --- |
| 工具调用 `tool_calls` | ✅ |
| 流式 `stream` | ✅（增量里带 `reasoning_content`） |
| `max_completion_tokens` 字段 | ✅（上限 393216） |

---

## 五、问题 4：`web_search` 要求官方 API Key

### 现象

```
Error: DeepSeek search has no API key for "DEEPSEEK_API_KEY";
store it through the credentials service (the web Models page writes it),
export it in the launching environment, or set a literal "apiKey" in the
web-search-deepseek config
```

用户会疑惑：**我明明用的是自定义 provider，为什么还查官方 key？**

### 根因：web 搜索是独立子系统，与 chat provider 无关

组合树（`dsh --profile web --dump-config`）中的相关行：

```yaml
- id: web
  config:
    searchProvider: deepseek-official     # 搜索后端被钉死为官方
    fetchProvider: http
- id: web-search-deepseek
  config:
    apiKeyEnv: DEEPSEEK_API_KEY           # 与 chat 的 provider 无关
- id: tool-web
  config: { fetch: true, searchTimeoutMs: 60000 }
```

`dsh-web-search-deepseek` 走的是 **DeepSeek 官方 Anthropic 兼容 Messages API**（默认 `https://api.deepseek.com/anthropic/v1`），带原生 `web_search_20250305` 服务端工具。而 `llm-pi-ai` 的 provider 只管 **chat completions**，两者互不影响。

顺带一提：即使把官方 chat 路线清空（如 `llm-deepseek: models: []`），**也管不到这个搜索 provider** —— 它是另一个包、另一个 `apiKeyEnv`。

### 为什么不能直接把 baseURL 换掉

网关支持哪些搜索调用方式，实测如下：

| 方式 | 结果 | 返回来源？ |
| --- | --- | --- |
| OpenAI Chat + `enable_search: true` | ✅ 有效（返回实时数据） | ❌ 不返回 |
| Responses API + `web_search` 工具 | ✅ **有效** | ✅ **返回来源 URL** |
| Anthropic Messages + `web_search_20250305` | ❌ 不生效（按文档在 `system` 里加客户端标识也无效） | — |

而官方那个 provider 偏偏走的是**第三条**：即使网关支持 Responses API 搜索，官方 provider 也用不上它。

### 关键认知：搜索后端是可插拔的

`ctx.web` 是 **provider 抽象层**，不是写死的：

```js
// dsh-web：公开接口
registerSearchProvider(provider)   // 任何插件都能注册自己的搜索后端
registerFetchProvider(provider)
// 选择规则：searchProvider 只是从「已注册的 provider」里挑一个
this.searchProviderId = config.searchProvider ?? process.env.DSH_WEB_SEARCH_PROVIDER;
```

provider 接口只有三个东西：

```ts
interface WebSearchProvider {
  readonly id: string;
  available(): boolean;                                          // 廉价本地检查，禁止网络调用
  search(request: WebSearchRequest, signal?: AbortSignal): Promise<WebSearchResult>;
}
interface WebSearchRequest { readonly query: string; readonly maxResults?: number }
interface WebSearchResult  { readonly sources: WebSearchSource[]; readonly truncated: boolean; readonly content?: string }
interface WebSearchSource  { readonly url: string; readonly title?: string; readonly snippet?: string; readonly publishedAt?: string }
```

所以正解是：**自己写一个 provider，走网关支持的 Responses API**。

### 修复：自定义 search provider

#### 1) 插件文件

放到 `$DSH_HOME/profiles/web/plugins/gateway-search.mjs`。

注意用 **`.mjs`**：profile 目录的 `package.json` 没有声明 `"type": "module"`，`.js` 会被当作 CommonJS。

```js
/**
 * web-search-gateway — 走网关 Responses API 的 ctx.web 搜索 provider。
 *
 * 背景：内置的 @deepseek-ai/dsh-web-search-deepseek 使用 Anthropic Messages API 的
 * web_search_20250305 服务端工具，因而需要 DeepSeek 官方密钥；本网关不支持该服务端
 * 工具，但支持 Responses API 的 web_search 工具并返回真实来源 URL：
 *
 *   POST {baseURL}/responses
 *   { "model": …, "input": …, "tools": [{ "type": "web_search" }] }
 *   → output[].web_search_call.action.sources[].url
 *
 * 由 profile 的 cordis.patch.yml 挂载；把 web.config.searchProvider 钉到它的 id 即可。
 */

import z from "@deepseek-ai/schemastery";
import { credentialRef } from "@deepseek-ai/dsh-credentials";
import { launchEnvironmentOf } from "@deepseek-ai/dsh-launch-environment";
import { WebError } from "@deepseek-ai/dsh-web";

export const name = "web-search-gateway";
export const inject = ["web"];

const USER_AGENT = "deepseek-harness/0.0.1 gateway-search/0.1.0";

const DEFAULTS = {
  id: "gateway",
  apiKeyEnv: "GATEWAY_API_KEY",
  baseURL: "https://api.example.net/v1",
  model: "deepseek-v4.1-flash",
  maxOutputTokens: 4096,
  timeoutMs: 60000
};

export const Config = z.object({
  id: z.string().default(DEFAULTS.id),
  apiKey: z.string().role("secret"),
  apiKeyEnv: z.string().role("credential-ref").default(DEFAULTS.apiKeyEnv),
  baseURL: z.string().default(DEFAULTS.baseURL),
  model: z.string().default(DEFAULTS.model),
  maxOutputTokens: z.number().step(1).min(1).default(DEFAULTS.maxOutputTokens),
  timeoutMs: z.number().step(1).min(1).default(DEFAULTS.timeoutMs)
});

/** 把行配置投影成一次搜索读取的选项。 */
function resolveOptions(ctx, config) {
  const apiKeyEnv = credentialRef(config.apiKeyEnv ?? DEFAULTS.apiKeyEnv);
  const literalApiKey =
    typeof config.apiKey === "string" && config.apiKey.length > 0 ? config.apiKey : void 0;
  return {
    id: config.id ?? DEFAULTS.id,
    ...(literalApiKey === void 0 ? {} : { apiKey: literalApiKey }),
    resolveApiKey: async () => {
      const credentials = ctx.get("credentials");
      if (credentials !== void 0) return (await credentials.resolve(apiKeyEnv))?.value;
      const ambient = launchEnvironmentOf(ctx).get(apiKeyEnv);
      return ambient !== void 0 && ambient.value.length > 0 ? ambient.value : void 0;
    },
    apiKeyEnv,
    baseURL: config.baseURL ?? DEFAULTS.baseURL,
    model: config.model ?? DEFAULTS.model,
    maxOutputTokens: config.maxOutputTokens ?? DEFAULTS.maxOutputTokens,
    timeoutMs: config.timeoutMs ?? DEFAULTS.timeoutMs
  };
}

/**
 * 把 Responses 响应体映射成归一化结果：遍历 output[] 里的 web_search_call
 * （原生搜索工具的 action.sources[]）与 url_citation 注解，按 URL 去重。
 * maxResults 的截断由 web 服务层负责，故这里 truncated 恒为 false。
 */
function mapResponse(body) {
  const seen = new Set();
  const sources = [];
  const push = (url, title, snippet) => {
    if (typeof url !== "string" || url.length === 0 || seen.has(url)) return;
    seen.add(url);
    sources.push({
      url,
      ...(typeof title === "string" && title.length > 0 ? { title } : {}),
      ...(typeof snippet === "string" && snippet.length > 0 ? { snippet } : {})
    });
  };
  for (const item of Array.isArray(body?.output) ? body.output : []) {
    if (item?.type === "web_search_call")
      for (const source of item.action?.sources ?? []) push(source?.url);
    if (item?.type !== "message") continue;
    for (const content of item.content ?? [])
      for (const annotation of content?.annotations ?? [])
        if (annotation?.type === "url_citation")
          push(annotation.url, annotation.title, annotation.text ?? annotation.cited_text);
  }
  if (sources.length === 0)
    throw new WebError(
      "gateway returned no web-search sources; the request may not have triggered native web search",
      "WEB_PROVIDER_ERROR"
    );
  return { sources, truncated: false };
}

function throwIfAborted(signal) {
  if (signal?.aborted === true)
    throw new WebError("gateway search aborted", "WEB_ABORTED", { cause: signal.reason });
}

class GatewaySearchProvider {
  constructor(resolveOptions) {
    this.resolveOptions = resolveOptions;
  }

  /** 稳定的注册键；由 web.config.searchProvider 钉定。 */
  get id() {
    return this.resolveOptions().id;
  }

  /** 只做廉价本地检查，绝不发起网络调用。 */
  available() {
    const options = this.resolveOptions();
    return URL.canParse(options.baseURL) && options.model.length > 0;
  }

  async search(request, signal) {
    const options = this.resolveOptions();
    const apiKey = await this.credential(options, signal);
    throwIfAborted(signal);
    const endpoint = `${options.baseURL.replace(/\/+$/u, "")}/responses`;
    const timeout = AbortSignal.timeout(options.timeoutMs);
    let response;
    try {
      response = await fetch(endpoint, {
        method: "POST",
        redirect: "error",
        headers: {
          authorization: `Bearer ${apiKey}`,
          "content-type": "application/json",
          accept: "application/json",
          "user-agent": USER_AGENT
        },
        body: JSON.stringify({
          model: options.model,
          input: `Perform a web search for the query: ${request.query}`,
          tools: [{ type: "web_search" }],
          max_output_tokens: options.maxOutputTokens
        }),
        signal: signal === void 0 ? timeout : AbortSignal.any([signal, timeout])
      });
    } catch (error) {
      throwIfAborted(signal);
      throw new WebError(
        `gateway search request to ${JSON.stringify(endpoint)} failed: ${String(error)}`,
        "WEB_PROVIDER_ERROR",
        { cause: error }
      );
    }
    if (!response.ok) {
      let detail = `HTTP ${response.status}`;
      try {
        const parsed = await response.json();
        const message =
          typeof parsed?.error === "string"
            ? parsed.error
            : parsed?.error?.message ?? parsed?.message;
        if (typeof message === "string" && message.length > 0) detail += `: ${message}`;
      } catch {
        /* 保留仅含状态码的详情 */
      }
      throw new WebError(
        `gateway search endpoint ${JSON.stringify(endpoint)} returned ${detail}`,
        "WEB_PROVIDER_ERROR"
      );
    }
    try {
      return mapResponse(await response.json());
    } catch (error) {
      throwIfAborted(signal);
      if (error instanceof WebError) throw error;
      throw new WebError(
        `gateway search returned an unprocessable response body: ${String(error)}`,
        "WEB_PROVIDER_ERROR",
        { cause: error }
      );
    }
  }

  /** 每次操作解析一次凭据，不把密钥留在 provider 上。 */
  async credential(options, signal) {
    throwIfAborted(signal);
    if (typeof options.apiKey === "string" && options.apiKey.length > 0) return options.apiKey;
    const resolved = await options.resolveApiKey?.();
    if (typeof resolved === "string" && resolved.length > 0) return resolved;
    throw new WebError(
      `gateway search has no API key for "${options.apiKeyEnv ?? DEFAULTS.apiKeyEnv}"; store it through the credentials service (the web Models page writes it), export it in the launching environment, or set a literal "apiKey" in the web-search-gateway config`,
      "WEB_PROVIDER_CREDENTIAL_MISSING"
    );
  }
}

/** 把网关搜索 provider 注册进 ctx.web。 */
export function apply(ctx, config) {
  ctx.web.registerSearchProvider(new GatewaySearchProvider(() => resolveOptions(ctx, config)));
}
```

> 依赖 `@deepseek-ai/schemastery`、`@deepseek-ai/dsh-credentials`、`@deepseek-ai/dsh-launch-environment`、`@deepseek-ai/dsh-web` 均可从 `$DSH_HOME/profiles/node_modules` 解析到，无需额外安装。

#### 2) profile patch

修改 `$DSH_HOME/profiles/web/cordis.patch.yml`（先备份）：

```yaml
# 把 web 搜索钉到本地 provider，替代内置的 deepseek-official。
# patch 会整段替换目标行的 config，所以未改动的 fetchProvider 也必须重述。
- id: web
  config:
    searchProvider: gateway
    fetchProvider: http

# 从 ./plugins/gateway-search.mjs 注册（路径相对 patch 文件解析）
- insert:
    - id: web-search-gateway
      name: ./plugins/gateway-search.mjs
      config:
        id: gateway
        apiKeyEnv: GATEWAY_API_KEY
        baseURL: https://api.example.net/v1
        model: deepseek-v4.1-flash
        maxOutputTokens: 4096
        timeoutMs: 60000
```

#### 3) 把密钥存进凭据服务

存到 `$DSH_HOME/.credentials.yaml` 的 `refs`（或从 Web UI 的 Models 页写入）：

```yaml
version: 1
refs:
  GATEWAY_API_KEY: <your-key>
```

也可以直接在启动环境里 `export GATEWAY_API_KEY=...`（provider 会先查凭据服务，再回落到启动环境）。

---

## 六、验证

### 1) 组合是否正确

```bash
dsh --profile web --dump-config | grep -A3 -E "id: web$|id: web-search-gateway"
```

期望看到：

```yaml
- id: web
  config:
    searchProvider: gateway
    fetchProvider: http
...
- id: web-search-gateway
  name: file:///.../plugins/gateway-search.mjs
```

### 2) provider 是否能跑通（不启动服务）

写一个临时脚本，用假 ctx 直接加载插件并真实打一次搜索：

```js
// $DSH_HOME/profiles/web/plugins/.selftest.mjs
import { apply, name, inject } from "./gateway-search.mjs";

const KEY = process.env.GATEWAY_API_KEY;
let provider;
const ctx = {
  web: { registerSearchProvider(p) { provider = p; } },
  get(n) { return n === "credentials" ? { resolve: async () => ({ value: KEY }) } : undefined; }
};

apply(ctx, { id: "gateway", apiKeyEnv: "GATEWAY_API_KEY", baseURL: "https://api.example.net/v1", model: "deepseek-v4.1-flash", maxOutputTokens: 4096, timeoutMs: 60000 });
console.log("id =", provider.id, "| available =", provider.available());
const r = await provider.search({ query: "杭州明天天气", maxResults: 8 });
console.log("sources =", r.sources.length);
for (const s of r.sources.slice(0, 5)) console.log("  -", s.url);
```

```bash
cd $DSH_HOME/profiles/web/plugins
NODE_USE_SYSTEM_CA=1 GATEWAY_API_KEY=<your-key> node ./.selftest.mjs
```

期望输出：

```
id = gateway | available = true
sources = 8
  - https://...
```

### 3) 真实启动

```bash
dsh web
```

启动只打印监听地址、无异常即为通过。之后模型调用 `web_search` 就会走自己的网关，**不再需要官方密钥**（`web_fetch` 本来也不需要）。

### 4) TLS 修复验证

```bash
node -e "fetch('https://<GATEWAY>/v1/models').then(r=>console.log('OK',r.status)).catch(e=>console.log('FAIL',e.cause?.code))"
```

在配好 `NODE_USE_SYSTEM_CA=1` 的新终端里应输出 `OK 401`。

---

## 七、最终配置汇总

`$DSH_HOME/settings.yaml`：

```yaml
llm-pi-ai:
  providers:
    <provider>:
      apiKeyEnv: <GATEWAY_ENV>
      api: openai-completions
      baseURL: https://<GATEWAY>/v1
      models:
        - id: <MODEL>
          name: <MODEL>
          contextWindow: 1000000
          maxTokens: 393216
          reasoningEfforts:
            off:
            minimal: minimal
            low: low
            medium: medium
            high: high
            xhigh: xhigh
            max: max

agent-default-model:
  provider: <provider>
  model: <MODEL>
  reasoningEffort: high
```

`~/.zshrc`：

```bash
export NODE_USE_SYSTEM_CA=1
```

`$DSH_HOME/profiles/web/cordis.patch.yml`：见第五章第 2 节。

`$DSH_HOME/profiles/web/plugins/gateway-search.mjs`：见第五章第 1 节。

---

## 八、回滚

| 改动 | 回滚方式 |
| --- | --- |
| `~/.zshrc` 的 `NODE_USE_SYSTEM_CA` | 删掉该行并 `exec zsh` |
| `settings.yaml` | 恢复备份 `settings.yaml.bak.*` |
| `cordis.patch.yml` | 恢复备份；或直接写回 `[]`（空数组会禁用该 patch 层） |
| 自定义 provider | 删掉插件文件，并从 patch 中移除 `insert` 段 |
| 桌面版 `launchctl setenv` | `launchctl unsetenv NODE_USE_SYSTEM_CA` 后重启 App |

---

## 九、排查经验小结

1. **`"Connection error."` / `TRANSPORT` 不要当成网络问题** —— 很可能是被上游吞掉的 TLS 错误。先在 Node 里单独复现那一句 `fetch`。
2. **Node 的 CA 与系统钥匙串是两套东西** —— 「curl 能通」不代表 Node 能通；反之亦然。
3. **放文件不等于生效** —— Node 的信任来源必须靠开关切换。
4. **对手工路由，能力要显式声明** —— 思考级别、上下文、输出上限都不会自己出现。
5. **网关参数要用「非法类型探测」验证** —— 传坏值看是否 400，比看文档更可靠。
6. **chat 与 web 搜索是两条独立的线** —— 换 chat provider 不会影响 `web_search` 的凭据要求。
7. **搜索后端可插拔** —— 官方只发了一个 provider，不代表只能用那一个。