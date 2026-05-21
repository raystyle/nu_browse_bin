# nu_plugin_browse

Nushell headless 浏览器插件，基于 [chaser-oxide](https://github.com/0xchasercat/chaser-oxide)（CDP 协议），需要系统安装 Chrome、Chromium 或 Edge。

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
browse open [--session x] [--with-head] [--no-stealth] [--profile p] [--url <target>] [--init-js <file>] [--init-code <js>] [--iso-init-js <file>] [--iso-init-code <js>] [--debug]
    → 启动浏览器。--url 通过 browser.new_page 直接打开目标站点（绕过 CDP navigate，适用于 x.com 等阻止 CDP 导航的站点），否则预热导航到 example.com → status: "ready"
    → --init-js/--init-code 注册 init 脚本（CDP 稳定注入，跨导航持久）
browse goto <url> [--ntrace ...] [--wait ...] [--no-stealth] [--debug]
    → 导航（init 脚本从 session 自动加载） → status: "ready"
browse ready --eval <js> [--iso-eval <js>] [--session x]
    → 在当前页面执行 JS → status: "ready"
```

浏览器状态：**ready**（空闲，页面已加载） | **loading**（导航中） | **unknown**（冻结/崩溃）

### 参数分布

| 参数 | browse | open | goto | ready | 用途 |
|------|--------|------|------|-------|------|
| `--code` | ✅ | — | — | — | Init 脚本（内联 JS，主世界，一次性） |
| `--js` | ✅ | — | — | — | Init 脚本（文件路径，主世界，一次性） |
| `--iso-code` | ✅ | — | — | — | Init 脚本（内联 JS，隔离世界，一次性） |
| `--iso-js` | ✅ | — | — | — | Init 脚本（文件路径，隔离世界，一次性） |
| `--init-code` | — | ✅ | — | — | Init 脚本（内联 JS，主世界，跨导航持久） |
| `--init-js` | — | ✅ | — | — | Init 脚本（文件路径，主世界，跨导航持久） |
| `--iso-init-code` | — | ✅ | — | — | Init 脚本（内联 JS，隔离世界，跨导航持久） |
| `--iso-init-js` | — | ✅ | — | — | Init 脚本（文件路径，隔离世界，跨导航持久） |
| `--eval` | ✅ | — | — | ✅ | 页面加载后执行 JS（主世界） |
| `--iso-eval` | ✅ | — | — | ✅ | 页面加载后执行 JS（隔离世界） |
| `--profile` | ✅ | ✅ | 继承 | — | 浏览器指纹，启动时设定 |
| `--no-stealth` | ✅ | ✅ | ✅ | — | 禁用 Stealth 模式（默认启用） |
| `--with-head` | — | ✅ | — | — | 显示浏览器窗口 |
| `--url` | — | ✅ | — | — | 直接打开 URL（Target.createTarget，绕过 CDP navigate） |
| `--ntrace` | ✅ | — | ✅ | ✅ | 导航时网络追踪 |
| `--ntrace-first` | ✅ | — | ✅ | ✅ | 提取首个匹配的请求+响应+body（配合 --ntrace） |
| `--wait` | ✅ | — | ✅ | — | 导航等待控制 |
| `--debug` | ✅ | ✅ | ✅ | — | Console 捕获 + init 错误监控（Runtime.enable） |
| `--session` | — | ✅ | ✅ | ✅ | 会话名称 |

---

## `browse <url>` — 一次性模式

启动临时浏览器，请求完成后自动关闭。

**参数：** `<url>`（必填）、`--code`、`--js`、`--iso-code`、`--iso-js`、`--eval`/`-e`、`--iso-eval`、`--ntrace`、`--ntrace-first`、`--wait`/`-w`、`--profile`、`--no-stealth`、`--debug`

```nu
browse https://example.com
browse https://example.com --eval "document.title"
browse https://example.com --code "window.__x = 1;" --eval "window.__x"
browse https://example.com --ntrace '.*'
browse https://example.com --profile windows-gamer
# 管道输入 → 自动作为 --iso-eval 执行
"document.title" | browse https://example.com
```

**返回 record：**

```nu
{
    status:      string,           # "success"（灾难性故障才非 success）
    url:         string,           # 请求的 URL
    message:     record,           # {pre, post, console} 始终存在
    network:     list<record>,     # 始终存在，无 --ntrace 时为 []
    idle_reason: record,           # {type: "skipped"|"normal"|"timeout"|"binding"|"deadline", reason?}
}
```

### `message` 结构（browse / goto / ready 统一）

所有 key 始终存在，`output` 无值时是 `Nothing`（null），`errors` 无错误时是 `[]`，`console` 始终为 `list<string>`：

```nu
{
    pre: {                         # Init 脚本（--code/--js）结果
        output: any|null,          # binding data（JSON→Nu 类型）或 Nothing
        errors: list<record>,      # [] 无错误
    },
    post: {                        # Eval（--eval/--iso-eval）结果
        output: any|null,          # eval 返回值或 Nothing
        errors: list<record>,      # [] 无错误
    },
    console: list<string>,         # console.log/warn/error 捕获输出
}
```

### `console` 捕获（`--debug`）

设置 `--debug` 时，插件启用 `Runtime.enable` 并捕获 JS 的 `console.log/warn/error` 等输出（通过 CDP `Runtime.consoleAPICalled` 事件）和 init 脚本异常（通过 `EventExceptionThrown`）。`Runtime.enable` 是可检测的反爬信号，仅在需要 console 输出或错误捕获时使用。

格式为 `[Log] message`。

- **有 `--debug` 时：** `console` 包含捕获的条目；`message.pre.errors` 包含 init 脚本异常；无输出时为 `[]`
- **无 `--debug` 时：** `console` 始终为 `[]`，init 脚本异常不被捕获（最大隐蔽性）
- 固定 key 契约：`message.console` **始终存在**，不会缺失

### 错误记录结构

```nu
{ message: string, line?: int, column?: int, kind: string }
# kind: "js_exception" — JS 异常（含行列号）
# kind: "error"       — 通用错误
```

### `network` 条目

```nu
# 请求：
{type: "request", method: string, url: string, headers: record}

# 响应：
{type: "response", id: string, status: int, url: string, mime: string, headers: record, body: any}
```

### `idle_reason`

| type | 含义 |
|------|------|
| `"skipped"` | 无等待 / 仅固定 `--wait` |
| `"normal"` | 网络空闲确认（`--ntrace`） |
| `"timeout"` | 网络空闲超时（10s，`--ntrace`） |
| `"binding"` | Init 脚本通过 `__browse_done()` 提前终止等待 |
| `"deadline"` | `--wait` 全局截止时间到期 |

---

## `browse open` — 启动会话

启动持久浏览器。默认预热导航到 `https://example.com`（建立 TLS/DNS/网络栈就绪状态）。可设置 `--profile` 指纹和 stealth 设置。

**`--url <target>`** 通过 `browser.new_page(url)` 直接打开目标站点，绕过 CDP `Page.navigate`。适用于 x.com 等检测并阻止 CDP 程序化导航的站点。

如果同名 session 已存在，会先关闭旧浏览器进程再重新启动。

**参数：** `--session`/`-s`、`--with-head`、`--no-stealth`、`--profile`、`--url`、`--init-js`、`--init-code`、`--iso-init-js`、`--iso-init-code`、`--debug`

```nu
browse open                                    # 默认会话，预热到 example.com
browse open --session grok                     # 命名会话
browse open --session grok --with-head          # 显示浏览器窗口
browse open --profile windows-gamer             # 指纹预设
browse open --url https://x.com/home            # 直接打开目标 URL
browse open --session twitter --with-head --url https://x.com/home
browse open --session grok --with-head --init-js ./browse-sdk.min.js --url https://grok.com
```

**返回 record：**

```nu
{session: string, status: "ready", url: "https://example.com" | <target_url>, message?: record}
```

当使用 `--url` + init scripts 时，包含 `message`（SDK 服务的 done 信号 + 状态数据）。

---

## `browse goto <url>` — 导航 + 注入

连接已有会话，导航到新 URL。Init 脚本从 session 文件自动加载（通过 `browse open --init-js/--init-code` 注册）。

**参数：** `<url>`（必填）、`--session`/`-s`、`--ntrace`、`--ntrace-first`、`--wait`/`-w`、`--no-stealth`、`--debug`

```nu
browse goto https://example.com
browse goto https://example.com --ntrace 'response:api\.'
browse goto https://example.com --debug --wait 10sec
```

**返回 record：** 字段与 `browse <url>` 相同，外加 `session`：

```nu
{
    session:     string,           # 会话名称
    status:      string,           # "ready"（错误查看 message.*.errors）
    url:         string,           # 导航的 URL
    message:     record,           # {pre, post, console}（始终存在）
    network:     list<record>,     # 始终存在，无 --ntrace 时为 []
    idle_reason: record,           # {type, reason?}
}
```

---

## `browse ready` — 执行 JS

在已打开会话的当前页面上执行 JavaScript，不导航。

**参数：** `--eval`、`--iso-eval`、`--session`/`-s`、`--ntrace`、`--ntrace-first`

`--eval` 和 `--iso-eval` 可同时使用，主世界先执行。

```nu
browse ready --eval "document.title"
browse ready --iso-eval "document.querySelectorAll('a').length"
browse ready --eval "document.title" --iso-eval "1+1" --session grok
```

**返回 record：** 与 `browse <url>` 同构的 `message`：

```nu
{
    session:  string,              # 会话名称
    status:   string,              # "ready"（错误查看 message.post.errors）
    url:      string,              # 当前页面 URL
    message: {
        pre:  { output: null, errors: [] },  # 无 init-js 阶段，固定为空
        post: { output: any|null, errors: list<record> },
        console: list<string>,               # [] （ready 不导航，无 console 捕获）
    },
    network:  list<record>,        # 始终存在，无 --ntrace 时为 []
}
```

必须指定 `--eval` 或 `--iso-eval` 其中之一，否则抛出 LabeledError。

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

扫描所有 `.nu_browse_profile*` 目录，逐个通过 CDP 连接检查状态。

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
| `--init-code <js>` | 主世界 | 内联 JS | `--init-code "window.__x = 1;"` |
| `--init-js <path>` | 主世界 | 文件路径 | `--init-js ./hook.js` |
| `--iso-init-code <js>` | 隔离世界 | 内联 JS | `--iso-init-code "var x = 1;"` |
| `--iso-init-js <path>` | 隔离世界 | 文件路径 | `--iso-init-js ./hook.js` |

### 一次性模式（`browse <url>`）

仅在当前导航生效，不存储在 session 中。

| Flag | 世界 | 输入 | 示例 |
|------|------|------|------|
| `--code <js>` | 主世界 | 内联 JS | `--code "window.__x = 1;"` |
| `--js <path>` | 主世界 | 文件路径 | `--js ./hook.js` |
| `--iso-code <js>` | 隔离世界 | 内联 JS | `--iso-code "var x = 1;"` |
| `--iso-js <path>` | 隔离世界 | 文件路径 | `--iso-js ./hook.js` |

- CDP 稳定注入（页面导航/iframe 自动重新注入）
- 自动注入 `window.__browse_done()` 函数，用于提前终止等待并传递数据（DOM meta tag 通道，零 `Runtime.enable`）
- 错误捕获需要 `--debug`（启用 `Runtime.enable`）
- 路径相对于 nushell CWD 解析

---

## `--eval` / `--iso-eval` — JS 执行

页面加载后执行 JavaScript。可同时使用 — 主世界优先，然后隔离世界。

**参数值支持两种输入方式：**
- **内联 JS：** 直接传入 JavaScript 代码字符串
- **文件路径：** 传入 `.js` 文件路径（相对 nushell PWD 解析），插件自动读取文件内容执行

| Flag | 世界 | 访问范围 | Stealth | 机制 |
|------|------|---------|---------|------|
| `--eval` | 主世界 | 页面全局变量、框架状态 | 可被页面脚本检测 | `raw_page.evaluate` |
| `--iso-eval` | 隔离世界 | 仅 DOM（通过 `grantUniversalAccess`） | 安全 | `chaser.evaluate` |

返回值自动解析：JSON → Nushell 类型。错误进入 `message.post.errors`，不会抛出异常。

```nu
# 主世界 — 访问页面框架状态
browse ready --eval "window.__NEXT_DATA__"

# 隔离世界 — 访问 DOM，stealth 安全
browse ready --iso-eval "document.querySelectorAll('a').length"

# 从文件执行
browse ready --iso-eval ./extract-links.js
```

---

## `--ntrace <pattern>` — 网络追踪

追踪网络请求和响应，启用网络空闲等待（最多 10s）。请求和响应的 URL 都用同一个 pattern 匹配。

| Pattern | 收集请求 | 收集响应 | URL 过滤 |
|---------|---------|---------|----------|
| `".*"` 或纯正则 | ✅ | ✅ | regex 匹配 |
| `"request"` | ✅ | — | 不过滤 |
| `"response"` | — | ✅ | 不过滤 |
| `"request:<regex>"` | ✅ | — | regex 匹配 |
| `"response:<regex>"` | — | ✅ | regex 匹配 |

```nu
browse https://example.com --ntrace '.*'
browse https://example.com --ntrace 'response:json'
browse https://example.com --ntrace 'request:api\.example\.com'
```

**`--ntrace-first`：** 配合 `--ntrace` 使用，将首个匹配的请求和响应提取到 `message.pre.output`，返回 record `{request: {...}, response: {...}, body: ...}`。`body` 尝试 JSON 解析为结构化数据。优先级低于 binding 数据。

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

`--wait N`（默认 15s）是全局硬截止时间，从导航开始到返回的整个流程中生效。所有内部机制（binding、ntrace、goto）正常运作，但 `--wait` 作为最外层硬截止优先。`--wait` **不是 sleep** — 不传时，无 binding/ntrace 的 wait phase 直接返回 `skipped`。

| 参数 | 内部机制 | `idle_reason.type` |
|------|----------|-------------------|
| 仅 `--ntrace` | 网络空闲循环（最多 10s） | `"normal"` / `"timeout"` |
| `--ntrace` + init scripts | 同上 + binding 捷径 | + `"binding"` |
| 仅 init scripts | Binding 等待（上限为 `--wait` 或 15s） | `"binding"` / `"skipped"` |
| 仅 `--wait` | 固定等待（goto 时间从 N 中扣除） | `"skipped"` |
| 无任何参数 | goto 完成后立即返回 | `"skipped"` |
| `--wait` 到期 | 全局截止覆盖所有机制 | `"deadline"` |

### Binding 协议（done 信号）

Init 脚本可通过 `window.__browse_done()` 提前终止等待并传递数据。插件自动注入 `__browse_done` 函数，它将数据写入 DOM `<meta name="__browse_done">` 标签，Rust 端通过隔离世界轮询读取（零 `Runtime.enable`，完全隐蔽）。

```js
// 在 --code 或 --js 文件中：
setTimeout(() => {
    window.__browse_done({reason: 'scraped', data: {title: document.title}});
}, 100);
```

`data` 从 JSON 解析为 Nushell Value，放入 `message.pre.output`。

---

## 错误处理

`status` 字段反映的是**浏览器/连接健康状态**，不作 per-phase 错误判断。
JS 层面的错误全部放在 `message.*.errors` 中，`status` 仅在灾难性故障时为非 `"success"`/`"ready"`。

| 错误 | 处理方式 | 示例 |
|------|----------|------|
| Eval JS 错误 | `message.post.errors`（`status` 仍为 `"ready"`） | `ReferenceError` |
| Init 脚本错误 | `message.pre.errors`（需 `--debug`，否则不捕获） | `SyntaxError` |
| 无效 URL | 抛出 `LabeledError` | `browse baidu.com` |
| 会话未打开 | 抛出 `LabeledError` | 已关闭 session 上 `browse goto` |
| 非法会话名 | 抛出 `LabeledError` | `--session "bad.name"` |
| ready 无 --eval/--iso-eval | 抛出 `LabeledError` | 单独的 `browse ready` |
| close 同时指定 --all + session 名 | 抛出 `LabeledError` | `browse close grok --all` |

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
$result.message.pre.errors | where kind == "js_exception"

# 取第一条错误消息
$result.message.post.errors | first | get message
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

## SDK

浏览器端 TypeScript SDK，通过 CDP `addScriptToEvaluateOnNewDocument` 注入。提供 DOM 查询、HTTP 拦截、跨世界通信、持久存储和服务集成（Grok、Twitter/X、Google）。

### 构建

```bash
cd sdk && bun install
bun run build      # 生成 IIFE bundle → 各 skill 目录
bun x tsc --noEmit # 类型检查
bun test           # 单元测试
```

### Bundle

| Bundle | 内容 | 用途 |
|--------|------|------|
| `browse-sdk.min.js` | runtime（dom + http + channel + storage） | 轻量 DOM 操作 |
| `grok.min.js` | runtime + grok 服务 | Grok 对话 |
| `twitter.min.js` | runtime + twitter 服务 | Twitter/X API |
| `google.min.js` | runtime + google 服务 | Google 搜索提取 |

每个 bundle 是独立自包含 IIFE，对应 Nu skill 脚本放在同一目录。

---

## 测试

```bash
cargo build --release

# 快速测试（无浏览器，<1s）
nu -c 'plugin rm browse; plugin add target/release/nu_plugin_browse.exe; plugin use browse; source tests/test_fast.nu'

# 全部测试（99 项：fast 19 + slow 80）
nu -c 'plugin rm browse; plugin add target/release/nu_plugin_browse.exe; plugin use browse; source tests/test_error.nu; source tests/test_fast.nu; source tests/test_basic.nu; source tests/test_js_worlds.nu; source tests/test_persistent.nu; source tests/test_network.nu; source tests/test_stealth.nu'
```

| 测试文件 | 数量 | 速度 | 覆盖 |
|----------|------|------|------|
| `test_fast.nu` | 7 | fast | 参数验证 |
| `test_error.nu` | 12 | fast | 错误路径边界 |
| `test_basic.nu` | 18 | slow | 导航、eval、init script、profile |
| `test_js_worlds.nu` | 22 | slow | 跨世界隔离、binding 协议、console |
| `test_persistent.nu` | 16 | slow | session 生命周期、cookie、多 session |
| `test_network.nu` | 12 | slow | ntrace、ntrace-first、JSON body |
| `test_stealth.nu` | 12 | slow | 反检测（bot.sannysoft.com） |

支持 `$env.TEST_FROM = N` 从指定编号开始跳过前面的测试。

---

## License

MIT
