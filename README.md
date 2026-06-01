# nu_plugin_browse

Nushell headless 浏览器插件，基于 [chaser-oxide](https://github.com/ccheshirecat/chaser-oxide)（CDP 协议），需要系统安装 Chrome、Chromium 或 Edge。

## 浏览器检测

插件启动时自动查找系统浏览器，无需手动配置路径。查找顺序（由 chaser-oxide 提供）：

1. **`CHROME` 环境变量** — 如果设置了 `CHROME` 且路径存在，直接使用
2. **PATH 查找** — 在系统 PATH 中依次查找：`chrome`、`chrome-browser`、`chromium`、`chromium-browser`、`msedge`、`microsoft-edge`
3. **Windows 注册表** — 读取 `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\chrome.exe`
4. **默认安装路径** — 检查各平台常见安装位置（Windows: `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`；macOS: `/Applications/Google Chrome.app/...` 等）

如果以上均未找到，启动时会返回错误 `"Could not auto detect a chrome executable"`。可通过设置环境变量指定：

```nu
$env.CHROME = "C:/path/to/chrome.exe"
browse https://example.com
```

```nu
cargo build --release
plugin rm browse; plugin add target/release/nu_plugin_browse.exe; plugin use browse
```

> **注意：** Nu 会缓存插件签名。重新编译后必须先 `plugin rm` 再 `plugin add`，否则新 flag 不会被识别。

## 命令总览

| 命令 | 作用 | 浏览器生命周期 |
|------|------|---------------|
| `browse <url>` | 一次性：启动→导航→返回结果→关闭 | 自动 |
| `browse open` | 启动持久会话（预热或直接打开 URL） | 手动 |
| `browse goto <url>` | 导航 + 注入 init 脚本 | 手动 |
| `browse ready` | 在当前页面执行 JS（不导航） | 手动 |
| `browse cookie [name?]` | 获取/设置/删除 cookie | 手动 |
| `browse status [session]` | 检查会话活跃状态 | — |
| `browse close [session]` | 优雅关闭（`--all` 关闭所有） | 手动 |
| `browse state` | 列出所有会话及状态 | — |

### 三阶段模型（持久会话）

```
browse open [--session x] [--headed] [--no-stealth] [--profile p] [--url <target>] [--init-script <file>] [--init-content <js>] [--isolated-init-script <file>] [--isolated-init-content <js>] [--debug]
    → 启动浏览器。--url 通过 browser.new_page 直接打开目标站点（绕过 CDP navigate，适用于 x.com 等阻止 CDP 导航的站点），否则预热导航到 example.com → status: "ready"
    → --init-script/--init-content 注册 init 脚本（CDP 稳定注入，跨导航持久）
browse goto <url> [--trace ...] [--timeout ...] [--no-stealth] [--debug]
    → 导航（init 脚本从 session 自动加载） → status: "ready"
browse ready --eval <js> [--isolated-eval <js>] [--session x] [--debug] [--eval-timeout <duration>] [--linger <duration>]
    → 在当前页面执行 JS → status: "ready" | "timeout"
```

浏览器状态：**ready**（空闲，页面已加载） | **open**（`browse status`：CDP 连接正常） | **frozen**（`browse status`：进程无响应） | **closed**（`browse status`：无活跃会话）

### 参数分布

| 参数 | browse | open | goto | ready | 用途 |
|------|--------|------|------|-------|------|
| `--init-content` | ✅ | — | — | — | Init 脚本（内联 JS，主世界，一次性） |
| `--init-script` | ✅ | — | — | — | Init 脚本（文件路径，主世界，一次性） |
| `--isolated-init-content` | ✅ | — | — | — | Init 脚本（内联 JS，隔离世界，一次性） |
| `--isolated-init-script` | ✅ | — | — | — | Init 脚本（文件路径，隔离世界，一次性） |
| `--init-content` | — | ✅ | — | — | Init 脚本（内联 JS，主世界，跨导航持久） |
| `--init-script` | — | ✅ | — | — | Init 脚本（文件路径，主世界，跨导航持久） |
| `--isolated-init-content` | — | ✅ | — | — | Init 脚本（内联 JS，隔离世界，跨导航持久） |
| `--isolated-init-script` | — | ✅ | — | — | Init 脚本（文件路径，隔离世界，跨导航持久） |
| `--eval` | ✅ | — | — | ✅ | 页面加载后执行 JS（主世界） |
| `--isolated-eval` | ✅ | — | — | ✅ | 页面加载后执行 JS（隔离世界） |
| `--profile` | ✅ | ✅ | 继承 | — | 浏览器指纹，启动时设定 |
| `--no-stealth` | ✅ | ✅ | ✅ | — | 禁用 Stealth 模式（默认启用） |
| `--headed` | — | ✅ | — | — | 显示浏览器窗口 |
| `--url` | — | ✅ | — | — | 直接打开 URL（`browser.new_page()`，绕过 CDP navigate） |
| `--trace` | ✅ | — | ✅ | ✅ | 导航时网络追踪 |
| `--trace-first` | ✅ | — | ✅ | ✅ | 输出首个匹配的 ID 配对请求+响应+body（与 --trace 互斥） |
| `--timeout` | ✅ | — | ✅ | — | 导航等待控制 |
| `--debug` | ✅ | ✅ | ✅ | ✅ | Console 捕获 + init 错误监控（Runtime.enable） |
| `--eval-timeout` | — | — | — | ✅ | Eval 超时（默认 15s，超时 → status "timeout"） |
| `--linger` | — | — | — | ✅ | --trace 收尾监听时间（默认 --eval-timeout） |
| `--session` | — | ✅ | ✅ | ✅ | 会话名称 |

---

## `browse <url>` — 一次性模式

启动临时浏览器，请求完成后自动关闭。

**参数：** `<url>`（必填）、`--init-content`、`--init-script`、`--isolated-init-content`、`--isolated-init-script`、`--eval`/`-e`、`--isolated-eval`、`--trace`、`--trace-first`（与 `--trace` 互斥）、`--timeout`/`-t`、`--profile`、`--no-stealth`、`--debug`

```nu
browse https://example.com
browse https://example.com --eval "document.title"
browse https://example.com --init-content "window.__x = 1;" --eval "window.__x"
browse https://example.com --trace '.*'
browse https://example.com --profile windows-gamer
# 管道输入 → 自动作为 --isolated-eval 执行
"document.title" | browse https://example.com
```

**返回 record：**

```nu
{
    status:   string,              # "success"（灾难性故障才非 success）
    url:      string,              # 请求的 URL
    session:  string,              # ephemeral 时为 ""
    duration: int,                 # 执行耗时（ms），始终存在
    idle:     string,              # "skipped"|"network-idle"|"timeout"|"binding"|"deadline"|""
    errors:   list<record>,        # init + eval 错误合并，无则 []
    console:  list<string>,        # console.log/warn/error 捕获输出（无 --debug 时为 []）
    binding:  any,                 # done 协议 data 字段（结构化 Nu Value），无则 null
    result:   any,                 # --eval/--isolated-eval 返回值（结构化 Nu Value），无则 null
    network:  list<record>,        # 始终存在，无 --trace/--trace-first 时为 []
}
```

`binding` 和 `result` 的类型是「JSON 可表示的 Nu Value」（七变体之一），由 `json_value_to_nu` 从 CDP 返回的 JSON 直接构造。**不需要 `from json`**。不会出现 `duration` / `range` / `closure` / `binary` / `cell-path` 等 Nu 特有类型 —— JS 侧产不出。

### 扁平契约（browse / open / goto / ready 统一）

所有 key 始终存在并使用 sentinel 默认值（详见 `src/output.rs::build_output`）：

| Sentinel | 含义 |
|----------|------|
| `null`（Nothing） | "无值" — `binding` / `result` |
| `[]`（空列表） | "什么都没发生" — `errors` / `console` / `network` |
| `""`（空串） | "空内容" — `url` / `session` / `idle` |

- `binding`（done 协议数据）和 `result`（eval 返回值）都是结构化 Nu Value，**不需要 `from json`**
- `errors` 合并 init 错误（在前）与 eval 错误（在后），每条为 `{message, kind, line?, column?}`

### `console` 捕获（`--debug`）

设置 `--debug` 时，插件启用 `Runtime.enable` 并捕获 JS 的 `console.log/warn/error` 等输出（通过 CDP `Runtime.consoleAPICalled` 事件）和 init 脚本异常（通过 `EventExceptionThrown`）。`Runtime.enable` 是可检测的反爬信号，仅在需要 console 输出或错误捕获时使用。

格式为 `[Log] message`。

- **有 `--debug` 时：** `console` 包含捕获的条目；`errors` 包含 init 脚本异常；无输出时为 `[]`
- **无 `--debug` 时：** `console` 始终为 `[]`，init 脚本异常不被捕获（最大隐蔽性）
- 固定 key 契约：`console` **始终存在**，不会缺失

### 错误记录结构

```nu
{ message: string, line?: int, column?: int, kind: string }
# kind: "js_exception" — JS 异常（含行列号）
# kind: "error"       — 通用错误
```

### `network` 条目

```nu
# 请求（body 仅 POST 数据可解码为 UTF-8 时存在）：
{id: string, type: "request", method: string, url: string, headers: record, body?: string}

# 响应（body 仅在获取成功时存在）：
{id: string, type: "response", status: int, url: string, mime: string, headers: record, body?: string}
```

- `id` — CDP `requestId`，请求与响应共享，可精确配对
- `headers` — Nu record（非 JSON 字符串），直接访问字段如 `$entry.headers.host`
- `body` — 可选字段：
  - **request**：取自 CDP `postDataEntries` 第一项 `bytes`（base64 解码 → UTF-8 字符串）。仅可解码为 UTF-8 时记录；二进制上传体或解码失败时 `body` 字段不出现。
  - **response**：取自 CDP `GetResponseBody`（字符串）

### `idle` 字段

扁平字符串（非嵌套 record）。

| 值 | 含义 |
|------|------|
| `"skipped"` | 无等待 / 仅固定 `--timeout` |
| `"network-idle"` | 网络空闲确认（`--trace`） |
| `"timeout"` | 网络空闲超时（10s，`--trace`） |
| `"binding"` | Init 脚本通过 `__browse_done()` 提前终止等待 |
| `"deadline"` | `--timeout` 全局截止时间到期 |
| `""` | 不适用（如 `browse ready`） |

---

## `browse open` — 启动会话

启动持久浏览器。默认预热导航到 `https://example.com`（建立 TLS/DNS/网络栈就绪状态）。可设置 `--profile` 指纹和 stealth 设置。

**`--url <target>`** 通过 `browser.new_page(url)` 直接打开目标站点，绕过 CDP `Page.navigate`。适用于 x.com 等检测并阻止 CDP 程序化导航的站点。

如果同名 session 已存在，会先关闭旧浏览器进程再重新启动。

**参数：** `--session`/`-s`、`--headed`、`--no-stealth`、`--profile`、`--url`、`--init-script`、`--init-content`、`--isolated-init-script`、`--isolated-init-content`、`--debug`

```nu
browse open                                    # 默认会话，预热到 example.com
browse open --session grok                     # 命名会话
browse open --session grok --headed          # 显示浏览器窗口
browse open --profile windows-gamer             # 指纹预设
browse open --url https://x.com/home            # 直接打开目标 URL
browse open --session twitter --headed --url https://x.com/home
browse open --session grok --headed --init-script ./browse-sdk.min.js --url https://grok.com
```

**返回扁平 record：** 与 `browse <url>` 同构（`status: "ready"`，`binding` 携带 SDK done 信号 + 状态数据）。当使用 `--url` + init scripts 时，SDK 服务调用 `__browse_done({data})` 后 `binding` 包含其数据。

---

## `browse goto <url>` — 导航 + 注入

连接已有会话，导航到新 URL。Init 脚本从 session 文件自动加载（通过 `browse open --init-script/--init-content` 注册）。

**参数：** `<url>`（必填）、`--session`/`-s`、`--trace`、`--trace-first`（与 `--trace` 互斥）、`--timeout`/`-t`、`--no-stealth`、`--debug`

```nu
browse goto https://example.com
browse goto https://example.com --trace 'response:api\.'
browse goto https://example.com --debug --timeout 10sec
```

**返回扁平 record：** 字段与 `browse <url>` 相同（`session` 为会话名，错误查看 `errors`，eval 返回值在 `result`，done 数据在 `binding`，`idle` 为字符串 idle 原因）。

---

## `browse ready` — 执行 JS

在已打开会话的当前页面上执行 JavaScript，不导航。

**参数：** `--eval`、`--isolated-eval`、`--session`/`-s`、`--trace`、`--trace-first`、`--debug`/`-d`、`--eval-timeout`、`--linger`

`--eval` 和 `--isolated-eval` 可同时使用，主世界先执行。

`--eval-timeout` 是 eval 执行截止时间（默认 15s）。`--linger`（默认等于 `--eval-timeout`）控制 `--trace` 模式下 eval 返回后继续监听网络的时长，用于捕获 `setTimeout` 触发的异步请求。两者独立：有 `--trace` 时总超时 = `eval-timeout + linger + 2s`，无 `--trace` 时 = `eval-timeout`。

```nu
browse ready --eval "document.title"
browse ready --isolated-eval "document.querySelectorAll('a').length"
browse ready --eval "document.title" --isolated-eval "1+1" --session grok
browse ready --eval "(console.log('debug'), 1+1)" --debug
browse ready --eval "await fetch(url)" --eval-timeout 5sec
```

**返回扁平 record：** 与 `browse <url>` 同构：

```nu
{
    session:  string,              # 会话名称
    status:   string,              # "ready"（错误查看 errors）| "timeout"（--eval-timeout 超时）
    url:      string,              # 当前页面 URL
    duration: int,                 # 执行耗时（ms）
    idle:     string,              # browse ready 固定为 ""
    errors:   list<record>,        # eval 错误，无则 []
    console:  list<string>,        # --debug 时捕获 console.log/warn/error，否则 []
    binding:  nothing,            # browse ready 固定为 null
    result:   nothing|bool|int|float|string|list|record,  # eval 返回值（JSON→Nu），无则 null
    network:  list<record>,        # 始终存在，无 --trace/--trace-first 时为 []
}
```

必须指定 `--eval` 或 `--isolated-eval` 其中之一，否则抛出 LabeledError。

---

## `browse cookie [name?]` — Cookie 操作

三种模式，由 flag 决定：

| Flag | 操作 | 返回值 |
|------|------|--------|
| （无） | 列出所有 cookie | `table<{name, value, domain, path, expires, http_only, secure, session, same_site}>` |
| `<name>` | 按名筛选 | 同上，过滤后 |
| `--set name=value --domain d` | 设置 cookie | `{status: "success", name}` |
| `<name> --delete` | 删除 cookie | `{status: "success", deleted}` |

**参数：** `[name]`（可选）、`--session`/`-s`、`--set`、`--domain`、`--path`、`--http-only`、`--secure`、`--delete`

```nu
browse cookie                                    # 列出全部
browse cookie session_id                         # 按名筛选
browse cookie --set "token=abc123" --domain .example.com
browse cookie token --delete
browse cookie --set "key=val" --domain .example.com --path /api --http-only --secure
```

---

## `browse status [session]` — 会话状态

读取 `.session` 文件，尝试 CDP 连接。返回活跃状态 record。不启动浏览器。

**参数：** `[session]`（可选）

```nu
browse status
browse status grok
```

**返回 record：**

```nu
{session: string, status: "open"|"frozen"|"closed", url: string, port?: int}
```

| status | 含义 | url | port |
|--------|------|-----|------|
| `"open"` | 浏览器运行中，CDP 连接正常 | 页面 URL | 端口号 |
| `"frozen"` | session 文件存在但浏览器无响应 | `""` | 端口号 |
| `"closed"` | 无活跃会话 | `""` | — |

---

## `browse close [session]` — 关闭

通过 CDP `Browser.close` 优雅关闭。frozen 状态则通过端口强制终止 Chrome 进程。

**参数：** `[session]`（可选）、`--all`

```nu
browse close              # 默认会话
browse close grok         # 命名会话
browse close --all        # 所有会话
```

**返回 table：**

```nu
[{session: string, status: "closed"}]
```

幂等 — 关闭已关闭的会话仍返回 `"closed"`。

`--all` 不能与 session 名同时指定（抛出 LabeledError）。

---

## `browse state` — 列出所有会话

扫描 `$HOME/.nu_browse/profiles/` 下所有会话目录，逐个通过 CDP 连接检查状态。

**参数：** 无

```nu
browse state
```

**返回 table：**

```nu
[{session: string, status: "open"|"frozen"|"closed", url: string, port: int|null}]
```

---

## 会话管理

- **目录结构：** `$HOME/.nu_browse/profiles/{session}/`
- **默认会话：** `$HOME/.nu_browse/profiles/default/`
- **命名会话：** `$HOME/.nu_browse/profiles/<name>/`
- **名称规则：** `[a-zA-Z0-9_-]`，最长 64 字符
- **Session 文件：** JSON `SessionData`（含 `ws_url` + init script 路径），原子写入（`.session.tmp` → rename）
- **文件锁：** `fs4` 排他锁防止并发访问同一会话

---

## Init 脚本注入

Init 脚本在页面脚本执行前注入 JavaScript（CDP `Page.addScriptToEvaluateOnNewDocument`）。可同时使用文件 + 内联：文件内容在前，内联代码在后。

### 持久模式（`browse open`）

跨导航持久，路径存储在 session 文件中，`goto`/`ready` 自动加载。

| Flag | 世界 | 输入 | 示例 |
|------|------|------|------|
| `--init-content <js>` | 主世界 | 内联 JS | `--init-content "window.__x = 1;"` |
| `--init-script <path>` | 主世界 | 文件路径 | `--init-script ./hook.js` |
| `--isolated-init-content <js>` | 隔离世界 | 内联 JS | `--isolated-init-content "var x = 1;"` |
| `--isolated-init-script <path>` | 隔离世界 | 文件路径 | `--isolated-init-script ./hook.js` |

### 一次性模式（`browse <url>`）

仅在当前导航生效，不存储在 session 中。

| Flag | 世界 | 输入 | 示例 |
|------|------|------|------|
| `--init-content <js>` | 主世界 | 内联 JS | `--init-content "window.__x = 1;"` |
| `--init-script <path>` | 主世界 | 文件路径 | `--init-script ./hook.js` |
| `--isolated-init-content <js>` | 隔离世界 | 内联 JS | `--isolated-init-content "var x = 1;"` |
| `--isolated-init-script <path>` | 隔离世界 | 文件路径 | `--isolated-init-script ./hook.js` |

- CDP 稳定注入（页面导航/iframe 自动重新注入）
- 自动注入 `window.__browse_done()` 函数，用于提前终止等待并传递数据（DOM meta tag 通道，零 `Runtime.enable`）
- 错误捕获需要 `--debug`（启用 `Runtime.enable`）
- 路径相对于 nushell CWD 解析

---

## `--eval` / `--isolated-eval` — JS 执行

页面加载后执行 JavaScript。可同时使用 — 主世界优先，然后隔离世界。

**参数值支持两种输入方式：**
- **内联 JS：** 直接传入 JavaScript 代码字符串
- **文件路径：** 传入 `.js` 文件路径（相对 nushell PWD 解析），插件自动读取文件内容执行

| Flag | 世界 | 访问范围 | Stealth | 机制 |
|------|------|---------|---------|------|
| `--eval` | 主世界 | 页面全局变量、框架状态 | 可被页面脚本检测 | `raw_page.evaluate` |
| `--isolated-eval` | 隔离世界 | 仅 DOM（通过 `grantUniversalAccess`） | 安全 | `CreateIsolatedWorld` + `Evaluate` |

返回值自动解析：JSON → Nushell 类型。错误进入顶层 `errors` 字段，不会抛出异常。

```nu
# 主世界 — 访问页面框架状态
browse ready --eval "window.__NEXT_DATA__"

# 隔离世界 — 访问 DOM，stealth 安全
browse ready --isolated-eval "document.querySelectorAll('a').length"

# 从文件执行
browse ready --isolated-eval ./extract-links.js
```

---

## `--trace <pattern>` — 网络追踪

追踪网络请求和响应，启用网络空闲等待（最多 10s）。请求和响应的 URL 都用同一个 pattern 匹配。双方向追踪（如 `".*"`）时自动按 CDP `requestId` 配对，仅输出配对成功的条目；单方向追踪（如 `"response"`）直接输出所有匹配条目。

| Pattern | 收集请求 | 收集响应 | URL 过滤 |
|---------|---------|---------|----------|
| `".*"` 或纯正则 | ✅ | ✅ | regex 匹配 |
| `"request"` | ✅ | — | 不过滤 |
| `"response"` | — | ✅ | 不过滤 |
| `"request:<regex>"` | ✅ | — | regex 匹配 |
| `"response:<regex>"` | — | ✅ | regex 匹配 |

```nu
browse https://example.com --trace '.*'
browse https://example.com --trace 'response:json'
browse https://example.com --trace 'request:api\.example\.com'
```

**`--trace-first`：** 与 `--trace` 互斥。自动启用网络追踪（默认捕获请求+响应），`network` 字段仅输出首个 ID 配对（request + response + body）。通过 CDP `requestId` 配对请求和响应。`body` 尝试 JSON 解析为结构化数据。

---

## `--profile <preset>` — 浏览器指纹

浏览器指纹预设（在 `browse open` 或 `browse <url>` 时设置）：

| 预设 | 系统 | GPU | 内存 | 核心 |
|------|------|-----|------|------|
| `native`（默认） | 系统原生 | 系统 GPU | — | — |
| `windows-gamer` | Windows | RTX 4080 | 32 GB | 16 |
| `windows-office` | Windows | UHD 630 | 16 GB | 8 |
| `macos-arm` | macOS ARM | Apple Silicon | — | — |
| `macos-intel` | macOS Intel | — | — | — |
| `linux` | Linux | — | — | — |

---

## 等待策略

`--timeout N`（默认 15s）是全局硬截止时间，从导航开始到返回的整个流程中生效。所有内部机制（binding、ntrace、goto）正常运作，但 `--timeout` 作为最外层硬截止优先。`--timeout` **不是 sleep** — 不传时，无 binding/ntrace 的 wait phase 直接返回 `skipped`。

| 参数 | 内部机制 | `idle` |
|------|----------|-------------------|
| 仅 `--trace` | 网络空闲循环（最多 10s） | `"network-idle"` / `"timeout"` |
| `--trace` + init scripts | 同上 + binding 捷径 | + `"binding"` |
| 仅 init scripts | Binding 等待（上限为 `--timeout` 或 15s） | `"binding"` / `"skipped"` |
| 仅 `--timeout` | 固定等待（goto 时间从 N 中扣除） | `"skipped"` |
| 无任何参数 | goto 完成后立即返回 | `"skipped"` |
| `--timeout` 到期 | 全局截止覆盖所有机制 | `"deadline"` |

### Binding 协议（done 信号）

Init 脚本可通过 `window.__browse_done()` 提前终止等待并传递数据。插件自动注入 `__browse_done` 函数，它将数据写入 DOM `<meta name="__browse_done">` 标签，Rust 端通过隔离世界轮询读取（零 `Runtime.enable`，完全隐蔽）。

```js
// 在 --init-content 或 --init-script 文件中：
setTimeout(() => {
    window.__browse_done({reason: 'scraped', data: {title: document.title}});
}, 100);
```

`data` 从 JSON 解析为 Nushell Value，放入顶层 `binding` 字段。

---

## 错误处理

`status` 字段反映的是**浏览器/连接健康状态**，不作 per-phase 错误判断。
JS 层面的错误全部放在顶层 `errors` 字段中（init 错误在前，eval 错误在后），`status` 仅在灾难性故障时为非 `"success"`/`"ready"`。

| 错误 | 处理方式 | 示例 |
|------|----------|------|
| Eval JS 错误 | `errors`（`status` 仍为 `"ready"`） | `ReferenceError` |
| Init 脚本错误 | `errors`（需 `--debug`，否则不捕获） | `SyntaxError` |
| 无效 URL | 抛出 `LabeledError` | `browse baidu.com` |
| 会话未打开 | 抛出 `LabeledError` | 已关闭 session 上 `browse goto` |
| 非法会话名 | 抛出 `LabeledError` | `--session "bad.name"` |
| ready 无 --eval/--isolated-eval | 抛出 `LabeledError` | 单独的 `browse ready` |
| ready --eval-timeout 超时 | `status: "timeout"` + `errors` | `browse ready --eval ... --eval-timeout 1sec` |
| close 同时指定 --all + session 名 | 抛出 `LabeledError` | `browse close grok --all` |
| --trace + --trace-first 同时指定 | 抛出 `LabeledError`（互斥） | `browse URL --trace '.*' --trace-first` |

### 错误记录结构

所有 `errors` 都是 `list<record>`，每条含 `message` 和 `kind` 字段：

```nu
# JS 异常（CDP init 脚本错误，含行列号）
{ message: "ReferenceError: x is not defined", line: 1, column: 23, kind: "js_exception" }

# 通用错误（eval 异常）
{ message: "eval error: TypeError: ...", kind: "error" }
```

使用示例：
```nu
# 检查是否有 JS 异常
$result.errors | where kind == "js_exception"

# 取第一条错误消息
$result.errors | first | get message
```

---

## 日志

插件使用 `log` crate 进行结构化日志输出。由于 Nu 会吞掉插件子进程的 stderr，需要通过环境变量指定日志文件：

```nu
# 设置日志文件路径
$env:BROWSE_LOG_FILE = "/tmp/browse.log"
browse https://example.com
cat /tmp/browse.log
```

不设置 `BROWSE_LOG_FILE` 时，日志输出到 stderr（适用于直接二进制调试）。

日志级别：
- `INFO` — 生命周期事件（导航开始/完成、会话打开/关闭、binding 触发）
- `WARN` — 可恢复问题（warmup 超时、goto 超时、网络空闲超时）
- `DEBUG` — 详细阶段进度（binding 注册、事件监听、eval 结果）

---

## 测试

```bash
cargo build --release

# 快速测试（无浏览器，<1s）
# MCP 方式（推荐）：单条 evaluate 执行
cd D:\opensource\nu_plugin_browse; plugin rm browse; plugin add target/release/nu_plugin_browse.exe; plugin use browse; source tests/test_fast.nu

# nu -c fallback：
nu -c 'plugin rm browse; plugin add target/release/nu_plugin_browse.exe; plugin use browse; source tests/test_fast.nu'

# 全部测试（113 项 = test_fast 10 + test_error 14 + test_basic 18 + test_js_worlds 22 + test_persistent 24 + test_network 13 + test_stealth 12）
# MCP 方式逐个执行，或 fallback：
nu -c 'plugin rm browse; plugin add target/release/nu_plugin_browse.exe; plugin use browse; source tests/test_fast.nu; source tests/test_error.nu; source tests/test_basic.nu; source tests/test_js_worlds.nu; source tests/test_persistent.nu; source tests/test_network.nu; source tests/test_stealth.nu'
```

| 测试文件 | 数量 | 速度 | 覆盖 |
|----------|------|------|------|
| `test_fast.nu` | 10 | fast | 参数验证 |
| `test_error.nu` | 14 | fast | 错误路径边界、resolve_js 启发式 |
| `test_basic.nu` | 18 | slow | 导航、eval、init script、profile |
| `test_js_worlds.nu` | 22 | slow | 跨世界隔离、binding 协议、console |
| `test_persistent.nu` | 24 | slow | session 生命周期、cookie、多 session、trace linger |
| `test_network.nu` | 13 | slow | trace、trace-first、JSON body、request body |
| `test_stealth.nu` | 12 | slow | 反检测（bot.sannysoft.com） |

支持 `$env.TEST_FROM = N` 从指定编号开始跳过前面的测试。

---

## License

MIT
