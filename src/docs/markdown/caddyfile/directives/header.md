---
title: header (Caddyfile 指令)
---

# header

操控 HTTP 响应头部字段。可以设置、添加、删除头部值，或使用正则表达式进行替换。

默认情况下，头部操作会立即执行，除非正在删除头部（`-` 前缀）或设置默认值（`?` 前缀）。在这些情况下，头部操作会自动推迟至写入客户端时执行。

要操控 HTTP 请求头部，可以使用 [`request_header`](request_header) 指令。

## 语法

```caddy-d
header [<matcher>] [[+|-|?|>]<field> [<value>|<find>] [<replace>]] {
	# 添加
	+<field> <value>

	# 设置
	<field> <value>

	# 设置（延时执行）
	><field> <value>

	# 删除
	-<field>

	# 替换
	<field> <find> <replace>

	# 替换（延时执行）
	><field> <find> <replace>

	# 默认值
	?<field> <value>

	[defer]

	match <inline_response_matcher>
}
```

- **&lt;field&gt;** 是头部字段的名称。

  不加前缀时为设置（覆盖）该字段。

  使用 `+` 前缀则添加字段而不是覆盖（如果字段已存在）；响应中头部字段可以出现多次。

  使用 `-` 前缀则删除该字段。该字段可以使用前缀或后缀 `*` 通配符来删除所有匹配的字段。

  使用 `?` 前缀时为字段设置默认值。仅当字段尚未存在时才写入该字段。

  使用 `>` 前缀则设置字段，并同时启用 `defer`，作为快捷方式。

- **&lt;value&gt;** 是添加或设置字段时的头部字段值。

- **&lt;find&gt;** 是要搜索的正则表达式。可以使用占位符动态输入搜索模式。使用的正则表达式语言是 Go 中的 RE2。请参考 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax) 和 [Go 正则表达式语法概述](https://pkg.go.dev/regexp/syntax)。

- **&lt;replace&gt;** 是替换值；如果执行查找替换则必须提供。使用 `$1`、`$2` 等引用搜索模式中的捕获组。如果替换值为 `""`，则从值中移除匹配的文本。详情请参阅 [Go 文档](https://golang.org/pkg/regexp/#Regexp.Expand)。

- **defer** 将头部操作的执行推迟到向客户端发送响应时。以下情况会自动启用此选项：
	- 使用 `-` 删除任何头部字段时。
	- 使用 `?` 设置默认值时。
	- 在设置或替换操作中使用 `>` 前缀时。
	- 存在一个或多个 `match` 条件时。

- **match** <span id="match"/> 是内联 [响应匹配器](/docs/caddyfile/response-matchers)。头部操作仅应用于满足指定条件的响应。

对于多个头部操作，可以打开一个块，每行指定一个操作，方式相同。

使用 `?` 前缀设置默认头部值时，如果它位于包含多个头部操作的 `header` 块中，它会被自动分隔到单独的 `header` 处理器中。[幕后机制](/docs/modules/http.handlers.headers#response/require) 是，使用 `?` 会配置一个 [响应匹配器](/docs/caddyfile/response-matchers)，该匹配器应用于指令的整个处理器，仅当字段尚未设置时才应用头部操作（如 `defer`）。

## 示例

在所有响应上设置自定义头部字段：

```caddy-d
header Custom-Header "My value"
```

移除 "Hidden" 头部字段：

```caddy-d
header -Hidden
```

在 Location 头部中将 `http://` 替换为 `https://`：

```caddy-d
header Location http:// https://
```

在所有页面上设置安全与隐私头部：（**警告：** 只有在你了解后果时才使用！）

```caddy-d
header {
	# 禁用 FLoC 跟踪
	Permissions-Policy interest-cohort=()

	# 启用 HSTS
	Strict-Transport-Security max-age=31536000;

	# 禁用客户端嗅探媒体类型
	X-Content-Type-Options nosniff

	# 点击劫持保护
	X-Frame-Options DENY
}
```

多个互斥的头部指令：

```caddy-d
route {
	header           Cache-Control max-age=3600
	header /static/* Cache-Control max-age=31536000
}
```

如果上游未定义缓存过期时间，则设置默认值：

```caddy-d
header ?Cache-Control "max-age=3600"
reverse_proxy upstream:443
```

将所有对 GET 请求的成功响应标记为可缓存一小时：

```caddy-d
@GET method GET
header @GET Cache-Control "max-age=3600" {
	match status 2xx
}
reverse_proxy upstream:443
```

在发生上游服务器异常时，防止缓存错误响应：

```caddy-d
header {
	-Cache-Control
	-CDN-Cache-Control
	match status 500
}
reverse_proxy upstream:443
```

如果上游服务器支持客户端提示，则将浅色模式响应与深色模式响应分别标记为可缓存：
```caddy-d
header {
	Cache-Control "max-age=3600"
	Vary "Sec-CH-Prefers-Color-Scheme"
	match {
		header Accept-CH "*Sec-CH-Prefers-Color-Scheme*"
		header Critical-CH "Sec-CH-Prefers-Color-Scheme"
	}
}
reverse_proxy upstream:443
```

通过将通配符值替换为特定域名，防止过于宽松的 CORS 头部：
```caddy-d
header >Access-Control-Allow-Origin "\*" "allowed-partner.com"
reverse_proxy upstream:443
```
**注意**：在替换操作中，`<find>` 值被解释为正则表达式。要匹配 `*` 字符，必须如上例所示用反斜杠转义。

或者，可以使用 [响应匹配器](/docs/caddyfile/response-matchers) 来逐字匹配头部值：
```caddy-d
header Access-Control-Allow-Origin "allowed-partner.com" {
	match header Access-Control-Allow-Origin *
}
reverse_proxy upstream:443
```

要覆盖代理上游为以 `/no-cache` 开头的路径设置的缓存过期时间；需要启用 `defer` 以确保在代理写入其头部 _之后_ 才设置该头部：

```caddy-d
header /no-cache* >Cache-Control no-cache
reverse_proxy upstream:443
```

要延时更新 `Set-Cookie` 头部以添加 `SameSite=None`；使用正则表达式捕获现有值，然后通过 `$1` 将其重新插入开头并附加额外选项：

```caddy-d
header >Set-Cookie (.*) "$1; SameSite=None;"