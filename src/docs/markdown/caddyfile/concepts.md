---
title: Caddyfile Concepts
---

# Caddyfile 概念

本文档将帮助您详细了解 HTTP Caddyfile。

1. [结构](#结构)
	- [代码块](#代码块)
	- [指令](#指令)
	- [令牌与引号](#令牌与引号)
2. [全局选项](#全局选项)
3. [地址](#地址)
4. [匹配器](#匹配器)
5. [占位符](#占位符)
6. [片段](#片段)
7. [命名路由](#命名路由)
8. [注释](#注释)
9. [环境变量](#环境变量)

## 结构

Caddyfile 的结构可以通过视觉方式描述：

<style>
	:root {
		--struct-border-global: #e74c3c;
		--struct-border-snippet: #2ecc71;
		--struct-border-site: #3498db;
		--struct-border-matcher: #d453d4;
		--struct-bg-1: #edf5fd;
		--struct-bg-2: #f8fbfd;
		--struct-bg-end: 100%;
		--struct-fg: #254048;
		--struct-opt-name-bg: #ffd9dd;
		--struct-opt-name-fg: #7a2a39;
		--struct-opt-value-bg: #f4dec6;
		--struct-opt-value-fg: #5a3723;
		--struct-comment-bg: #d2d7d8;
		--struct-comment-fg: #495456;
		--struct-site-addr-bg: #cbe4f2;
		--struct-site-addr-fg: #1f6f9a;
		--struct-directive-bg: #c8f7d6;
		--struct-directive-fg: #14663a;
		--struct-matcher-token-bg: #ffd6ff;
		--struct-matcher-token-fg: #6f2070;
		--struct-arg-bg: #ded0ff;
		--struct-arg-fg: #4b2f7a;
		--struct-subdir-bg: #dbbca2;
		--struct-subdir-fg: #5b3a25;
	}
	html.dark {
		--struct-border-global: #e74c3c;
		--struct-border-snippet: #2ecc71;
		--struct-border-site: #3498db;
		--struct-border-matcher: #d453d4;
		--struct-bg-1: #0d313c;
		--struct-bg-2: transparent;
		--struct-bg-end: 120%;
		--struct-fg: #cbd6da;
		--struct-opt-name-bg: #6b2630;
		--struct-opt-name-fg: #ffd9dd;
		--struct-opt-value-bg: #68412b;
		--struct-opt-value-fg: #f4dec6;
		--struct-comment-bg: #2f424d;
		--struct-comment-fg: #e8eef0;
		--struct-site-addr-bg: #204d59;
		--struct-site-addr-fg: #d6f0ff;
		--struct-directive-bg: #1f4e36;
		--struct-directive-fg: #c8f7d6;
		--struct-matcher-token-bg: #65305a;
		--struct-matcher-token-fg: #ffd6ff;
		--struct-arg-bg: #3b2e46;
		--struct-arg-fg: #ded0ff;
		--struct-subdir-bg: #6a4a2e;
		--struct-subdir-fg: #ebc095;
	}
	/* color variables - easy to tweak */
	.struct-caddyfile-visual-repl {
		display: block;
		margin: 0;
		padding: 0;
	}
	/* default (light) visual background */
	.struct-caddyfile-visual-repl .struct-visual {
		box-sizing: border-box;
		margin: 0 0 1.25rem;
		padding: 14px;
		border-radius: 14px;
		background: linear-gradient(to bottom, var(--struct-bg-1) 0%, var(--struct-bg-2) var(--struct-bg-end));
		color: var(--struct-fg);
		font-family: Inter, 'Source Sans Pro', Arial, system-ui, sans-serif;
		line-height: 1.2;
	}
	/* layout */
	.struct-caddyfile-visual-repl .struct-panel {
		display: flex;
		gap: 18px;
		align-items: flex-start;
		flex-wrap: wrap;
	}
	.struct-caddyfile-visual-repl .struct-diagram {
		flex: 1;
		padding: 8px 8px;
	}
	.struct-caddyfile-visual-repl .struct-legend {
		width: 310px;
		padding: 12px 4px;
	}
	/* code-like box: use normal whitespace so HTML pretty-printing won't leak source indentation */
	.struct-caddyfile-visual-repl .struct-code-box {
		background: transparent;
		border-radius: 8px;
		padding: 6px 6px !important;
		font-family: var(--monospace-fonts);
		font-size: 90%;
		white-space: normal;
	}
	.struct-block {
		border-radius: 8px;
		padding: 10px;
		margin: 0 0 10px 0;
	}
	.struct-block.global {
		border: 4px solid var(--struct-border-global);
	}
	.struct-block.snippet {
		border: 4px solid var(--struct-border-snippet);
	}
	.struct-block.site {
		border: 4px solid var(--struct-border-site);
	}
	.struct-block.matcher {
		border: 4px solid var(--struct-border-matcher);
		margin: 8px 8px 10px 10px;
		padding: 8px;
		border-radius: 6px;
	}
	.struct-token, .struct-opt-name, .struct-opt-value, .struct-comment, .struct-site-addr, .struct-directive, .struct-matcher-token, .struct-arg, .struct-subdir {
		display: inline !important;
		padding: .03rem .18rem !important;
		border-radius: 6px;
		font-family: var(--monospace-fonts);
		font-size: 95%;
		vertical-align: middle;
	}
	.struct-opt-name {
		background: var(--struct-opt-name-bg);
		color: var(--struct-opt-name-fg);
	}
	.struct-opt-value {
		background: var(--struct-opt-value-bg);
		color: var(--struct-opt-value-fg);
	}
	.struct-comment {
		background: var(--struct-comment-bg);
		color: var(--struct-comment-fg);
	}
	.struct-site-addr {
		background: var(--struct-site-addr-bg);
		color: var(--struct-site-addr-fg);
	}
	.struct-directive {
		background: var(--struct-directive-bg);
		color: var(--struct-directive-fg);
	}
	.struct-matcher-token {
		background: var(--struct-matcher-token-bg);
		color: var(--struct-matcher-token-fg);
	}
	.struct-arg {
		background: var(--struct-arg-bg);
		color: var(--struct-arg-fg);
	}
	.struct-subdir {
		background: var(--struct-subdir-bg);
		color: var(--struct-subdir-fg);
	}
	.struct-legend .struct-legend-title {
		font-weight: 700;
		font-size: 1.6rem;
	}
	.struct-legend .struct-item {
		display: flex;
		align-items: center;
		gap: 10px;
		margin: 16px 0;
	}
	.struct-legend .struct-item-spacer {
		height: 8px;
	}
	/* swatch for border-based legend items (blocks) */
	.struct-legend .struct-swatch-border {
		width: 42px;
		height: 24px;
		border-radius: 6px;
		box-sizing: border-box;
		border: 4px solid transparent;
		background: transparent;
	}
	/* swatch for filled legend items (text backgrounds) */
	.struct-legend .struct-swatch-fill {
		width: 42px;
		height: 24px;
		border-radius: 6px;
		box-sizing: border-box;
		background: transparent;
	}
	.struct-legend .struct-label {
		font-size: 90%;
		color: inherit;
	}
	.struct-caddyfile-visual-repl .struct-visual, .struct-caddyfile-visual-repl .struct-panel, .struct-caddyfile-visual-repl .struct-diagram, .struct-caddyfile-visual-repl .struct-legend, .struct-caddyfile-visual-repl .struct-code-box {
		margin: 0;
	}
	/* force compact vertical rhythm and explicit indenting so global CSS can't leak in
		NOTE: use normal whitespace so server-side HTML formatting doesn't create visible gaps */
	.struct-line {
		display: block !important;
		margin: 0 !important;
		padding: 2px 0 !important;
		line-height: 1.2 !important;
		white-space: normal !important;
	}
	/* helper to visually indent lines (do not rely on source file whitespace)
		use an explicit spacer element so HTML formatting won't affect alignment */
	.struct-line.struct-indent {
		padding-left: 0 !important;
	}
	.struct-indent-spacer {
		display: inline-block;
		width: 1.2rem;
		height: 1px;
		margin-right: 0.18rem;
	}
	/* smaller spacer for sub-directive / nested lines */
	.struct-subindent-spacer {
		display: inline-block;
		width: 0.9rem;
		height: 1px;
		margin-right: 0.12rem;
	}
</style>

<div class="struct-caddyfile-visual-repl fullwidth">
	<div class="struct-visual">
		<div class="struct-panel">
			<div class="struct-diagram">
				<div class="struct-code-box">
					<div class="struct-block global">
						<div class="struct-line">{</div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-opt-name">email</span> <span class="struct-opt-value">you@yours.com</span></div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-opt-name">servers</span> {</div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-indent-spacer"></span><span class="struct-subdir">trusted_proxies</span> <span class="struct-arg">static</span> <span class="struct-arg">private_ranges</span></div>
						<div class="struct-line"><span class="struct-indent-spacer"></span>}</div>
						<div class="struct-line">}</div>
					</div>
					<div class="struct-block snippet">
						<div class="struct-line">(snippet) {</div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-comment"># this is a reusable snippet</span></div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-directive">log</span> {</div>
						<div class="struct-line"><span class="struct-subindent-spacer"></span><span class="struct-indent-spacer"></span><span class="struct-subdir">output</span> <span class="struct-arg">file</span> <span class="struct-arg">/var/log/access.log</span></div>
						<div class="struct-line"><span class="struct-indent-spacer"></span>}</div>
						<div class="struct-line">}</div>
					</div>
					<div class="struct-block site">
						<div class="struct-line"><span class="struct-site-addr">example.com</span> {</div>
						<div class="struct-block matcher">
							<div class="struct-line"><span class="struct-matcher-token">@post</span> {</div>
							<div class="struct-line"><span class="struct-subindent-spacer"></span><span class="struct-matcher-token">method</span> <span class="struct-arg">POST</span></div>
							<div class="struct-line">}</div>
						</div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-directive">reverse_proxy</span> <span class="struct-matcher-token">@post</span> <span class="struct-arg">localhost:9001</span> <span class="struct-arg">localhost:9002</span> {</div>
						<div class="struct-line"><span class="struct-subindent-spacer"></span><span class="struct-indent-spacer"></span><span class="struct-subdir">lb_policy</span> <span class="struct-arg">first</span></div>
						<div class="struct-line"><span class="struct-indent-spacer"></span>}</div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-directive">file_server</span> <span class="struct-matcher-token">/static</span></div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-directive">import</span> <span class="struct-arg">snippet</span></div>
						<div class="struct-line">}</div>
					</div>
					<div class="struct-block site">
						<div class="struct-line struct-indent"><span class="struct-site-addr">www.example.com</span> {</div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-directive">redir</span> <span class="struct-arg">https://example.com{uri}</span></div>
						<div class="struct-line struct-indent"><span class="struct-indent-spacer"></span><span class="struct-directive">import</span> <span class="struct-arg">snippet</span></div>
						<div class="struct-line">}</div>
					</div>
				</div>
			</div>
			<div class="struct-legend" aria-hidden="false">
				<div class="struct-legend-title">图例</div>
				<div class="struct-item-spacer"></div>
				<div class="struct-item"><div class="struct-swatch-border" style="border-color:var(--struct-border-global)"></div><div class="struct-label">全局选项块</div></div>
				<div class="struct-item"><div class="struct-swatch-border" style="border-color:var(--struct-border-snippet)"></div><div class="struct-label">片段</div></div>
				<div class="struct-item"><div class="struct-swatch-border" style="border-color:var(--struct-border-site)"></div><div class="struct-label">站点块</div></div>
				<div class="struct-item"><div class="struct-swatch-border" style="border-color:var(--struct-border-matcher)"></div><div class="struct-label">匹配器定义</div></div>
				<div class="struct-item-spacer"></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-opt-name-bg)"></div><div class="struct-label">选项名称</div></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-opt-value-bg)"></div><div class="struct-label">选项值</div></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-comment-bg)"></div><div class="struct-label">注释</div></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-site-addr-bg)"></div><div class="struct-label">站点地址</div></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-directive-bg)"></div><div class="struct-label">指令</div></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-matcher-token-bg)"></div><div class="struct-label">匹配器令牌</div></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-arg-bg)"></div><div class="struct-label">参数</div></div>
				<div class="struct-item"><div class="struct-swatch-fill" style="background:var(--struct-subdir-bg)"></div><div class="struct-label">子指令</div></div>
			</div>
		</div>
	</div>
</div>

关键点：

- 可选的 [**全局选项块**](#全局选项) 可以是文件中最开始的内容。

- 接下来可以出现[片段](#片段)或[命名路由](#命名路由)。

- 否则，Caddyfile 的第一行**始终**是要服务的站点[地址](#地址)。

- 所有[指令](#指令)和[匹配器](#匹配器)**必须**放在站点块内。没有全局作用域或跨站点块的继承。

- 如果只有一个站点块，其花括号 `{ }` 是可选的。

Caddyfile 由一个或多个站点块组成，每个站点块始终以站点的一个或多个[地址](#地址)开头。出现在地址之前的任何指令都会让解析器困惑。

### 代码块

使用花括号打开和关闭**代码块**：

```
... {
	...
}
```

- 左花括号 `{` 必须位于其行的末尾，并且前面有一个空格。

- 右花括号 `}` 必须独占一行。

当只有一个站点块时，花括号（和缩进）是可选的。这是为了方便快速定义单个站点，例如：

```caddy
localhost

reverse_proxy /api/* localhost:9001
file_server
```

等价于：

```caddy
localhost {
	reverse_proxy /api/* localhost:9001
	file_server
}
```

当你只有一个站点块时，这取决于个人喜好。

如果要用同一个 Caddyfile 配置多个站点，**必须**在每个站点周围使用花括号来分隔其配置：

```caddy
example1.com {
	root /www/example.com
	file_server
}

example2.com {
	reverse_proxy localhost:9000
}
```

如果请求匹配多个站点块，则选择地址最具体的站点块。请求不会级联到其他站点块。

### 指令

[**指令**](/docs/caddyfile/directives) 是自定义站点服务方式的功能性关键字。它们**必须**出现在站点块内。例如，一个完整的文件服务器配置可能如下所示：

```caddy
localhost {
	file_server
}
```

或反向代理：

```caddy
localhost {
	reverse_proxy localhost:9000
}
```

在这些示例中，[`file_server`](/docs/caddyfile/directives/file_server) 和 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) 是指令。指令是站点块中一行的第一个单词。

在第二个示例中，`localhost:9000` 是一个**参数**，因为它出现在指令之后的同一行。

有时指令可以打开自己的块。**子指令**出现在指令块内每行的开头：

```caddy
localhost {
	reverse_proxy localhost:9000 localhost:9001 {
		lb_policy first
	}
}
```

这里，`lb_policy` 是 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) 的一个子指令（它设置后端之间的负载均衡策略）。

**除非另有说明，否则指令不能在其他指令块内使用。** 例如，[`basic_auth`](/docs/caddyfile/directives/basic_auth) 不能在 [`file_server`](/docs/caddyfile/directives/file_server) 内使用，因为文件服务器不知道如何进行身份验证；但您可以在 [`route`](/docs/caddyfile/directives/route)、[`handle`](/docs/caddyfile/directives/handle) 和 [`handle_path`](/docs/caddyfile/directives/handle_path) 块内使用指令，因为它们专门设计用于将指令分组。

请注意，当 HTTP Caddyfile 被适配时，除非在 [`route`](/docs/caddyfile/directives/route) 块内，否则 HTTP 处理程序指令将按照特定的默认[指令顺序](/docs/caddyfile/directives#directive-order)排序，因此指令的出现顺序无关紧要，除非在 `route` 块内。

### 令牌与引号

Caddyfile 在解析之前会被词法分析为令牌。Caddyfile 中的空白符是有意义的，因为令牌由空白符分隔。

通常，指令期望一定数量的参数；如果单个参数的值包含空白符，它会被词法分析为两个独立的令牌：

```caddy-d
directive abc def
```

这可能会导致问题并返回错误或意外行为。

如果 `abc def` 应该是单个参数的值，则需要用引号括起来：

```caddy-d
directive "abc def"
```

如果需要，引号可以被转义：

```caddy-d
directive "\"abc def\""
```

为避免转义引号，您可以使用反引号 <code>\` \`</code> 来括起令牌；例如：

```caddy-d
directive `{"foo": "bar"}`
```

在引号括起的令牌内部，所有其他字符都被视为字面量，包括空格、制表符和换行符。因此，多行令牌是可能的：

```caddy-d
directive "first line
	second line"
```

Heredoc <span id="heredocs"/> 也受支持：

```caddy
example.com {
	respond <<HTML
		<html>
		  <head><title>Foo</title></head>
		  <body>Foo</body>
		</html>
		HTML 200
}
```

开头的 heredoc 标记必须以 `<<` 开头，后跟任意文本（建议使用大写字母）。结尾的 heredoc 标记必须是相同的文本（在上面的示例中是 `HTML`）。开头的标记可以用 `\<<` 转义，以防止 heredoc 解析（如果需要）。

结尾标记可以缩进，这会使每一行文本都剥离相应缩进（受 [PHP](https://www.php.net/manual/en/language.types.string.php#language.types.string.syntax.heredoc) 启发），这对[块](#代码块)内的可读性很好，同时可以很好地控制令牌文本中的空白符。末尾的换行符也会被剥离，但可以通过在结尾标记前添加一个额外的空行来保留。

在结尾标记之后可以跟其他令牌作为指令的参数（如上例中的状态码 `200`）。

## 全局选项

Caddyfile 可以选择以一个没有键的特殊块开头，称为[全局选项块](/docs/caddyfile/options)：

```caddy
{
	...
}
```

如果存在，它必须是配置中的第一个块。

它用于设置全局适用的选项，或不特定于某个站点的选项。内部只能设置全局选项；不能在其中使用常规站点指令。

例如，启用 `debug` 全局选项，常用于生成详细日志以进行故障排除：

```caddy
{
	debug
}
```

**[阅读全局选项页面](/docs/caddyfile/options)以了解更多。** 

## 地址

地址始终出现在站点块的顶部，通常是 Caddyfile 中的第一件事。

以下是有效地址的示例：

| 地址                 | 效果                            |
|----------------------|-----------------------------------|
| `example.com`        | HTTPS，使用受管理的[公开信任证书](/docs/automatic-https#hostname-requirements) |
| `*.example.com`      | HTTPS，使用受管理的[通配符公开信任证书](/docs/caddyfile/patterns#wildcard-certificates) |
| `localhost`          | HTTPS，使用受管理的[本地信任证书](/docs/automatic-https#local-https) |
| `http://`            | HTTP 全匹配，受 [`http_port`](/docs/caddyfile/options#http-port) 影响 |
| `https://`           | HTTPS 全匹配，受 [`https_port`](/docs/caddyfile/options#http-port) 影响 |
| `http://example.com` | 显式 HTTP，带 `Host` 匹配器 |
| `example.com:443`    | HTTPS，因为匹配了默认的 [`https_port`](/docs/caddyfile/options#http-port) |
| `:443`               | HTTPS 全匹配，因为匹配了默认的 [`https_port`](/docs/caddyfile/options#http-port) |
| `:8080`              | HTTP 在非标准端口，无 `Host` 匹配器 |
| `localhost:8080`     | HTTPS 在非标准端口，因为具有有效域名 |
| `https://example.com:443` | HTTPS，但同时使用 `https://` 和 `:443` 是多余的 |
| `127.0.0.1` | HTTPS，使用本地信任的 IP 证书 |
| `http://127.0.0.1` | HTTP，带 IP 地址 `Host` 匹配器（拒绝 `localhost`） |


<aside class="tip">

[自动 HTTPS](/docs/automatic-https) 在站点地址包含主机名或 IP 地址时启用。然而，这种行为纯粹是隐式的，因此它永远不会覆盖任何显式配置。

例如，如果站点地址是 `http://example.com`，自动 HTTPS 不会激活，因为方案是显式的 `http://`。

</aside>


从地址中，Caddy 可以推断出站点的方案、主机和端口。如果地址没有端口，Caddyfile 将选择与指定方案匹配的端口，或者假设为默认端口 443。

如果指定了主机名，则仅会处理具有匹配 `Host` 头的请求。换句话说，如果站点地址是 `localhost`，则 Caddy 不会匹配对 `127.0.0.1` 的请求。

可以使用通配符 (`*`)，但只能用于表示主机名的一个标签。例如，`*.example.com` 匹配 `foo.example.com` 但不匹配 `foo.bar.example.com`，而 `*` 匹配 `localhost` 但不匹配 `example.com`。请参阅[通配符证书模式](/docs/caddyfile/patterns#wildcard-certificates)以了解实际示例。

要匹配所有主机，请省略地址的主机部分，例如简单地使用 `https://`。这在不知道域名的 [按需 TLS](/docs/automatic-https#on-demand-tls) 场景下很有用。

如果多个站点共享相同的定义，您可以一次列出所有站点，用空格和逗号分隔（至少需要一个空格）。以下三个示例是等价的：

```caddy
# 逗号分隔的站点地址
localhost:8080, example.com, www.example.com {
	...
}
```

或者

```caddy
# 空格分隔的站点地址
localhost:8080 example.com www.example.com {
	...
}
```

或者

```caddy
# 逗号和新行分隔的站点地址
localhost:8080,
example.com,
www.example.com {
	...
}
```

地址必须唯一；您不能多次指定相同的地址。

[占位符](#占位符) **不能**在地址中使用，但您可以包含 Caddyfile 风格的[环境变量](#环境变量)：

```caddy
{$DOMAIN:localhost} {
	...
}
```

默认情况下，站点绑定到所有网络接口。如果要覆盖此设置，请使用 [`bind` 指令](/docs/caddyfile/directives/bind)或 [`default_bind` 全局选项](/docs/caddyfile/options#default-bind)。

## 匹配器

HTTP 处理程序[指令](#指令)默认应用于所有请求（除非另有说明）。

[请求匹配器](/docs/caddyfile/matchers)可用于按给定条件对请求进行分类。使用匹配器，您可以精确指定某个指令应用于哪些请求。

对于支持匹配器的指令，指令后的第一个参数是**匹配器令牌**。以下是一些示例：

```caddy-d
root *           /var/www  # 匹配器令牌: *
root /index.html /var/www  # 匹配器令牌: /index.html
root @post       /var/www  # 匹配器令牌: @post
```

匹配器令牌可以完全省略以匹配所有请求；例如，如果下一个参数看起来不像路径匹配器，则不需要给出 `*`。

**[阅读请求匹配器页面](/docs/caddyfile/matchers)以了解更多。** 

## 占位符

[占位符](/docs/conventions#placeholders) 是一种将动态值注入静态配置的简单方式。它们可以用作指令和子指令的参数。

占位符由花括号 `{ }` 包围，内部包含标识符，例如：`{foo.bar}`。可以用 `\{like.this}` 转义开头的占位符花括号以防止替换。占位符标识符通常使用点命名空间，以避免跨模块冲突。

哪些占位符可用取决于上下文。并非所有占位符在配置的所有部分都可用。例如，[HTTP 应用程序设置了占位符](/docs/json/apps/http/#docs)，这些占位符仅在配置中与处理 HTTP 请求相关的区域可用（即 HTTP 处理程序[指令](#指令)和[匹配器](#匹配器)，但**不能**在 [`tls` 配置](/docs/caddyfile/directives/tls)中使用）。某些指令或匹配器也可能设置自己的占位符，这些占位符可以被后面的任何内容使用。一些占位符是[全局可用的](/docs/conventions#placeholders)。

您可以在 Caddyfile 中使用任何占位符，但为了方便，您也可以使用一些等价缩写，这些缩写会在 Caddyfile 解析时展开：

| 缩写             | 替换为                            |
|------------------|-------------------------------------|
| `{cookie.*}`     | `{http.request.cookie.*}`           |
| `{client_ip}`    | `{http.vars.client_ip}`             |
| `{dir}`          | `{http.request.uri.path.dir}`       |
| `{err.*}`        | `{http.error.*}`                    |
| `{file_match.*}` | `{http.matchers.file.*}`            |
| `{file.base}`    | `{http.request.uri.path.file.base}` |
| `{file.ext}`     | `{http.request.uri.path.file.ext}`  |
| `{file}`         | `{http.request.uri.path.file}`      |
| `{header.*}`     | `{http.request.header.*}`           |
| `{host}`         | `{http.request.host}`               |
| `{hostport}`     | `{http.request.hostport}`           |
| `{labels.*}`     | `{http.request.host.labels.*}`      |
| `{method}`       | `{http.request.method}`             |
| `{orig_method}`  | `{http.request.orig_method}`        |
| `{orig_uri}`     | `{http.request.orig_uri}`           |
| `{orig_path}`    | `{http.request.orig_uri.path}`      |
| `{orig_dir}`     | `{http.request.orig_uri.path.dir}`  |
| `{orig_file}`    | `{http.request.orig_uri.path.file}` |
| `{orig_query}`   | `{http.request.orig_uri.query}`     |
| `{orig_?query}`  | `{http.request.orig_uri.prefixed_query}` |
| `{path.*}`       | `{http.request.uri.path.*}`         |
| `{path}`         | `{http.request.uri.path}`           |
| `{%path}`        | `{http.request.uri.path_escaped}`   |
| `{port}`         | `{http.request.port}`               |
| `{query.*}`      | `{http.request.uri.query.*}`        |
| `{query}`        | `{http.request.uri.query}`          |
| `{%query}`       | `{http.request.uri.query_escaped}`  |
| `{?query}`       | `{http.request.uri.prefixed_query}` |
| `{re.*}`         | `{http.regexp.*}`                   |
| `{remote_host}`  | `{http.request.remote.host}`        |
| `{remote_port}`  | `{http.request.remote.port}`        |
| `{remote}`       | `{http.request.remote}`             |
| `{rp.*}`         | `{http.reverse_proxy.*}`            |
| `{resp.*}`       | `{http.intercept.*}`                |
| `{scheme}`       | `{http.request.scheme}`             |
| `{tls_cipher}`   | `{http.request.tls.cipher_suite}`   |
| `{tls_client_certificate_der_base64}` | `{http.request.tls.client.certificate_der_base64}` |
| `{tls_client_certificate_pem}`        | `{http.request.tls.client.certificate_pem}` |
| `{tls_client_fingerprint}`            | `{http.request.tls.client.fingerprint}`     |
| `{tls_client_issuer}`                 | `{http.request.tls.client.issuer}`          |
| `{tls_client_serial}`                 | `{http.request.tls.client.serial}`          |
| `{tls_client_subject}`                | `{http.request.tls.client.subject}`         |
| `{tls_version}`       | `{http.request.tls.version}`             |
| `{upstream_hostport}` | `{http.reverse_proxy.upstream.hostport}` |
| `{uri}`               | `{http.request.uri}`                     |
| `{%uri}`              | `{http.request.uri_escaped}`             |
| `{vars.*}`            | `{http.vars.*}`                          |

并非所有配置字段都支持占位符，但大多数您期望支持的地方都支持。需要在这些字段中显式添加占位符支持。插件作者可以[阅读此文章](/docs/extending-caddy/placeholders) 以了解如何在自己的模块中添加占位符支持。

## 片段

您可以通过给特殊块一个用括号括起来的名称来定义片段：

```caddy
(logging) {
	log {
		output file /var/log/caddy.log
		format json
	}
}
```

然后，您可以在任何需要的地方使用特殊的 [`import`](/docs/caddyfile/directives/import) 指令重用它：

```caddy
example.com {
	import logging
}

www.example.com {
	import logging
}
```

[`import`](/docs/caddyfile/directives/import) 指令也可以用于在其位置包含其他文件。如果参数不匹配已定义的片段，则会尝试作为文件。它还支持使用 glob 导入多个文件。作为一种特殊情况，它可以出现在 Caddyfile 中的任何位置（除了作为另一个指令的参数），包括站点块之外：

```caddy
{
	email admin@example.com
}

import sites/*
```

您可以向导入的配置（片段或文件）传递参数，并按如下方式使用：

```caddy
(snippet) {
	respond "Yahaha! You found {args[0]}!"
}

a.example.com {
	import snippet "Example A"
}

b.example.com {
	import snippet "Example B"
}
```

⚠️ <i>实验性</i> <span style='white-space: pre;'> | </span> <span>v2.9.x+</span>

您还可以向导入的片段传递一个可选的块，并按如下方式使用。

```caddy
(snippet) {
	{block}
	respond "OK"
}

a.example.com {
	import snippet {
		header +foo bar
	}
}

b.example.com {
	import snippet {
		header +bar foo
	}
}
```

**[阅读 `import` 指令页面](/docs/caddyfile/directives/import)以了解更多。** 

## 命名路由

⚠️ <i>实验性</i>

命名路由使用与[片段](#片段)类似的语法；它们是在站点块之外定义的、前缀为 `&(` 并以 `)` 结尾的特殊块，名称位于中间。

```caddy
&(app-proxy) {
	reverse_proxy app-01:8080 app-02:8080 app-03:8080
}
```

然后，您可以在任何站点中重用此命名路由：

```caddy
example.com {
	invoke app-proxy
}

www.example.com {
	invoke app-proxy
}
```

如果同一个路由需要出现在多个站点中，或者需要多个不同的匹配条件来调用同一个路由，这尤其有助于减少内存使用。

**[阅读 `invoke` 指令页面](/docs/caddyfile/directives/invoke)以了解更多。** 

## 注释

注释以 `#` 开头，持续到行尾：

```caddy-d
# 注释可以在一行开头
directive  # 也可以放在行尾
```

注释的井号 `#` 不能出现在令牌中间（即必须前面有空格或出现在行首）。这允许在 URI 或其他值中使用井号而无需加引号。

## 环境变量

如果您的配置依赖环境变量，可以在 Caddyfile 中使用它们：

```caddy
{$ENV}
```

这种形式的环境变量在 **Caddyfile 解析开始之前**被替换，因此它们可以展开为空值（即 `""`）、部分令牌、完整令牌，甚至多个令牌和行。

例如，环境变量 `UPSTREAMS="app1:8080 app2:8080 app3:8080"` 将展开为多个[令牌](#令牌与引号)：

```caddy
example.com {
	reverse_proxy {$UPSTREAMS}
}
```

可以指定默认值，用于环境变量未找到的情况，使用 `:` 作为变量名和默认值之间的分隔符：

```caddy
{$DOMAIN:localhost} {

}
```

如果您想**延迟环境变量的替换**直到运行时，可以使用标准的 [`{env.*}` 占位符](/docs/conventions#placeholders)。但请注意，并非所有配置参数都支持这些占位符，因为模块开发人员需要添加一行代码来执行替换。如果似乎不起作用，请提交问题以请求支持。

例如，如果您安装了 [`caddy-dns/cloudflare` 插件 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddy-dns/cloudflare) 并希望配置 [DNS 挑战](/docs/automatic-https#dns-challenge)，您可以像这样将您的 `CLOUDFLARE_API_TOKEN` 环境变量传递给插件：

```caddy
{
	acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}
```

如果您将 Caddy 作为 systemd 服务运行，请参阅[这些说明](/docs/running#overrides)来设置服务覆盖以定义您的环境变量。