---
title: 请求匹配器 (Caddyfile)
---

<script>
ready(function() {
	// We'll add links on the matchers in the code blocks
	// to their associated anchor tags.
	let headers = Array.from($$_('article h3')).map(el => el.id.replace(/-/g, "_"));

	$$_('pre.chroma .k').forEach(item => {
		if (headers.includes(item.innerText)) {
			let text = item.innerText.replace(/</g, '&lt;').replace(/>/g, '&gt;');
			let url = '#' + item.innerText.replace(/_/g, "-");
			item.innerHTML = `<a href="${url}" style="color: inherit;" title="${text}">${text}</a>`;
		}
	});

	// Link matcher tokens based on their contents to the syntax section
	$$_('pre.chroma .nd').forEach(item => {
		let text = item.innerText.replace(/</g, '&lt;').replace(/>/g, '&gt;');
		let anchor = "named-matchers";
		if (text == "*") anchor = "wildcard-matchers";
		if (text.startsWith('/')) anchor = "path-matchers";
		item.innerHTML = `<a href="#${anchor}" style="color: inherit;" title="匹配器令牌">${text}</a>`;
	});
});
</script>

# 请求匹配器

**请求匹配器**可用于根据各种标准来过滤（或分类）请求。

- [语法](#语法)
	- [示例](#示例)
	- [通配符匹配器](#通配符匹配器)
	- [路径匹配器](#路径匹配器)
	- [命名匹配器](#命名匹配器)
- [标准匹配器](#标准匹配器)
	- [client_ip](#client-ip)
	- [expression](#expression)
	- [file](#file)
	- [header](#header)
	- [header_regexp](#header-regexp)
	- [host](#host)
	- [method](#method)
	- [not](#not)
	- [path](#path)
	- [path_regexp](#path-regexp)
	- [protocol](#protocol)
	- [query](#query)
	- [remote_ip](#remote-ip)
	- [vars](#vars)
	- [vars_regexp](#vars-regexp)


## 语法

在 Caddyfile 中，紧跟在指令后的**匹配器令牌**可以限制该指令的范围。匹配器令牌可以是以下形式之一：

1. [**`*`**](#通配符匹配器) 匹配所有请求（通配符；默认值）。
2. [**`/path`**](#路径匹配器) 以正斜杠开头以匹配请求路径。
3. [**`@name`**](#命名匹配器) 指定一个_命名匹配器_。

如果某指令支持匹配器，它将在其语法文档中显示为 `[<匹配器>]`。匹配器令牌通常在语法中是 [可选的](/docs/caddyfile/directives#syntax)，用 `[ ]` 表示。如果省略匹配器令牌，则等同于通配符匹配器（`*`）。


#### 示例

此指令应用于 [所有](#通配符匹配器) HTTP 请求：

```caddy-d
reverse_proxy localhost:9000
```

这与以下内容相同（此处不需要 `*`）：

```caddy-d
reverse_proxy * localhost:9000
```

但此指令仅应用于 [路径](#路径匹配器) 以 `/api/` 开头的请求：

```caddy-d
reverse_proxy /api/* localhost:9000
```

若要匹配路径以外的内容，请定义一个 [命名匹配器](#命名匹配器) 并使用 `@name` 引用它：

```caddy-d
@postfoo {
	method POST
	path /foo/*
}
reverse_proxy @postfoo localhost:9000
```




### 通配符匹配器

通配符（或"捕获所有"）匹配器 `*` 匹配所有请求，仅当需要匹配器令牌时才需要它。例如，如果您想给指令的第一个参数恰好是一个路径，它会看起来正像一个路径匹配器！因此您可以使用通配符匹配器来消除歧义，例如：

```caddy-d
root * /home/www/mysite
```

否则，此匹配器并不常使用。如果语法不需要，我们通常建议省略它。


### 路径匹配器

通过 URI 路径匹配是最常用的匹配请求方式，因此匹配器可以内联，如下所示：

```caddy-d
redir /old.html /new.html
```

路径匹配器令牌必须以正斜杠 `/` 开头。

**[路径匹配](#路径) 默认是精确匹配，而不是前缀匹配。** 您必须附加一个 `*` 才能进行快速前缀匹配。注意 `/foo*` 将匹配 `/foo`、`/foo/` 和 `/foobar`；您可能实际上想要的是 `/foo/*`。


### 命名匹配器

所有不是路径或通配符的匹配器都必须是命名匹配器。这是在任意特定指令之外定义的匹配器，可以重复使用。

使用唯一名称定义匹配器可为您提供更多灵活性，允许您将 [任何可用的匹配器](#标准匹配器) 组合成一个集合：

```caddy-d
@name {
	...
}
```

或者，如果集合中只有一个匹配器，您可以将其放在同一行：

```caddy-d
@name ...
```

然后您可以像这样使用匹配器，将名称指定为指令的第一个参数：

```caddy-d
directive @name
```

例如，此代理将 HTTP/1.1 websocket 请求转发到 `localhost:6001`，其他请求转发到 `localhost:8080`。它匹配具有名为 `Connection` 的字段_包含_ `Upgrade` 的请求，**并且**另一个名为 `Upgrade` 的字段恰好为 `websocket`：

```caddy
example.com {
	@websockets {
		header Connection *Upgrade*
		header Upgrade    websocket
	}
	reverse_proxy @websockets localhost:6001

	reverse_proxy localhost:8080
}
```

如果匹配器集合仅包含一个匹配器，则单行语法也适用：

```caddy-d
@post method POST
reverse_proxy @post localhost:6001
```

作为特殊情况，[`expression` 匹配器](#expression) 可以在不指定其名称的情况下使用，只要有一个 [引用](/docs/caddyfile/concepts#tokens-and-quotes) 参数（CEL 表达式本身）跟在匹配器名称之后：

```caddy-d
@not-found `{err.status_code} == 404`
```

与指令一样，命名匹配器定义必须在 [站点块](/docs/caddyfile/concepts#structure) 内使用它们。

命名匹配器定义构成一个_匹配器集合_。集合中的匹配器是 AND（与）关系；即必须全部匹配。例如，如果集合中有 [`header`](#header) 和 [`path`](#path) 匹配器，则两者都必须匹配。

同一类型的多个匹配器可以合并（例如，同一集合中的多个 [`path`](#path) 匹配器），使用布尔代数（AND/OR），如下节所述。

对于更复杂的布尔匹配逻辑，建议使用 [`expression` 匹配器](#expression) 编写 CEL 表达式，它支持 **and** `&&`、**or** `||` 和 **括号** `( )`。





## 标准匹配器

完整的匹配器文档可在 [每个相应匹配器模块的文档](/docs/json/apps/http/servers/routes/match/) 中找到。

请求可以通过以下方式匹配：



### client_ip

```caddy-d
client_ip <ranges...>

expression client_ip('<ranges...>')
```

按客户端 IP 地址匹配。接受确切 IP 或 CIDR 范围。支持 IPv6 区域。

此匹配器在配置 [`trusted_proxies`](/docs/caddyfile/options#trusted-proxies) 全局选项时最适合使用，否则其作用与 [`remote_ip`](#remote-ip) 匹配器相同。只有受信任代理的请求才会在请求开始时解析其客户端 IP；不受信任的请求将使用直接对等点的远程 IP 地址或通过 [PROXY 协议](/docs/caddyfile/options#proxy-protocol) 设置的地址。

作为快捷方式，可以使用 `private_ranges` 匹配所有私有 IPv4 和 IPv6 范围。它与指定所有这些范围相同：`192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8 fd00::/8 ::1`

每个命名匹配器可以有多个 `client_ip` 匹配器，它们的范围将合并并使用 OR（或）连接。

#### 示例：

匹配来自私有 IPv4 地址的请求：

```caddy-d
@private-ipv4 client_ip 192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8
```

此匹配器常与 [`not`](#not) 匹配器配对使用以反转匹配。例如，终止来自_公共_ IPv4 和 IPv6 地址（这是所有私有范围的反面）的所有连接：

```caddy
example.com {
	@denied not client_ip private_ranges
	abort @denied

	respond "您好，您必须来自私有网络！"
}
```

在 [CEL 表达式](#expression) 中，它将如下所示：

```caddy-d
@my-friends `client_ip('12.23.34.45', '23.34.45.56')`
```



### expression

```caddy-d
expression <cel...>
```

根据任何 [CEL（公共表达式语言）](https://github.com/google/cel-spec) 表达式，该表达式返回 `true` 或 `false`。

大多数其他请求匹配器也可以用作表达式中的函数，这允许在表达式之外进行更灵活的布尔逻辑。有关在 CEL 表达式中支持的语法的文档，请参阅每个匹配器的文档。

Caddy [占位符](/docs/conventions#placeholders)（或 [Caddyfile 简写](/docs/caddyfile/concepts#placeholders)）可用于这些 CEL 表达式，因为它们会被预处理并转换为常规的 CEL 函数调用，然后再由 CEL 环境解释。如果占位符应作为字符串参数传递给匹配器函数，则前导 `{` 应使用反斜杠 `\` 转义，这样它就不会被预处理，例如 `file('\{path}.md')`。

为方便起见，如果定义的名命名匹配器完全由 CEL 表达式组成，则可以省略匹配器名称。CEL 表达式必须 [引用](/docs/caddyfile/concepts#tokens-and-quotes)（建议反引号或 heredocs）。这读起来很不错：

```caddy-d
@mutable `{method}.startsWith("P")`
```

在这种情况下，假定是 CEL 匹配器。

#### 示例：

匹配其方法以 `P` 开头的请求，例如 `PUT` 或 `POST`：

```caddy-d
@methods expression {method}.startsWith("P")
```

匹配处理器返回错误状态码 `404` 的请求，将与 [`handle_errors` 指令](/docs/caddyfile/directives/handle_errors) 一起使用：

```caddy-d
@404 expression {err.status_code} == 404
```

匹配路径匹配两个不同正则表达式的请求；这只能使用表达式编写，因为 [`path_regexp`](#path-regexp) 匹配器通常每个命名匹配器只能存在一次：

```caddy-d
@user expression path_regexp('^/user/(\w*)') || path_regexp('^/(\w*)')
```

或者相同，省略匹配器名称，并用 [反引号](/docs/caddyfile/concepts#tokens-and-quotes) 包装以便将其解析为单个令牌：

```caddy-d
@user `path_regexp('^/user/(\w*)') || path_regexp('^/(\w*)')`
```

您可以使用 [heredoc 语法](/docs/caddyfile/concepts#heredocs) 编写多行 CEL 表达式：

```caddy-d
@api <<CEL
	{method} == "GET"
	&& {path}.startsWith("/api/")
	CEL
respond @api "您好，API！"
```


---
### file

```caddy-d
file {
	root       <path>
	try_files  <files...>
	try_policy first_exist|first_exist_fallback|smallest_size|largest_size|most_recently_modified
	split_path <delims...>
}
file <files...>

expression `file({
	'root': '<path>',
	'try_files': ['<files...>'],
	'try_policy': 'first_exist|first_exist_fallback|smallest_size|largest_size|most_recently_modified',
	'split_path': ['<delims...>']
})`
expression file('<files...>')
```

按文件匹配。

- `root` 定义在其中查找文件的目录。默认为当前工作目录，或设置的 `root` [变量](/docs/modules/http.handlers.vars) (`{http.vars.root}`)（可通过 [`root` 指令](/docs/caddyfile/directives/root) 设置）。

- `try_files` 检查其列表中匹配 try_policy 的文件。

  要匹配目录，在路径后附加尾随正斜杠 `/`。所有文件路径相对于站点 [根目录](/docs/caddyfile/directives/root)，并且 [通配符模式](https://pkg.go.dev/path/filepath#Match) 将被展开。

  如果 `try_policy` 是 `first_exist`（默认值），则列表中的最后一项可以是前缀为 `=` 的数字（例如 `=404`），作为回退，将发出带有该代码的错误；错误可以 [`handle_errors`](/docs/caddyfile/directives/handle_errors) 捕获和处理。



- `try_policy` 指定如何选择文件。默认为 `first_exist`。

	- `first_exist` 检查文件是否存在。选择第一个存在的文件。

	- `first_exist_fallback` 类似于 `first_exist`，但假设列表中的最后一个元素始终存在以防止磁盘访问。

	- `smallest_size` 选择最小尺寸的文件。

	- `largest_size` 选择最大尺寸的文件。

	- `most_recently_modified` 选择最近修改的文件。

- `split_path` 将在每个尝试的文件名中找到的列表中的第一个分隔符处拆分路径。对于每个拆分值，拆分包括分隔符本身在内的左侧部分将是尝试的文件名。例如，使用分隔符 `.php`，`/remote.php/dav/` 将尝试文件 `/remote.php`。每个分隔符必须出现在 URI 路径组件的末尾才能用作拆分分隔符。这是一个特殊设置，主要用于服务 PHP 站点。

由于 `try_files` 使用策略 `first_exist` 非常常见，因此有一个单行快捷方式：

```caddy-d
file <files...>
```

空的 `file` 匹配器（未列出任何文件后）将检查请求的文件——来自 URI 的 verbatim，相对于 [站点根目录](/docs/caddyfile/directives/root)&mdash;是否存在。这实际上与 `file {path}` 相同。


<aside class="tip">

由于基于磁盘上文件的存在进行重写非常常见，因此还有一个 [`try_files` 指令](/docs/caddyfile/directives/try_files)，它是 `file` 匹配器和 [`rewrite` 处理器](/docs/caddyfile/directives/rewrite) 的快捷方式。

</aside>


匹配后，将提供四个新的占位符：

- `{file_match.relative}` 文件的根相对路径。这通常在重写请求时很有用。
- `{file_match.absolute}` 匹配文件的绝对路径，包括根目录。
- `{file_match.type}` 文件类型，`file` 或 `directory`。
- `{file_match.remainder}` 拆分文件路径后的剩余部分（如果配置了 `split_path`）


#### 示例：

匹配路径是存在的文件的请求：

```caddy-d
@file file
```

匹配路径后跟 `.html` 是存在的文件，或者如果不是，则路径是存在的文件的请求：

```caddy-d
@html file {
	try_files {path}.html {path} 
}
```

与上面相同，但使用单行快捷方式，如果在找不到文件时回退到发出 404 错误：

```caddy-d
@html-or-error file {path}.html {path} =404
```

更多使用 [CEL 表达式](#expression) 的示例。请记住，占位符会被预处理并转换为常规的 CEL 函数调用，然后再由 CEL 环境解释，因此此处使用连接。此外，由于当前解析限制，如果使用占位符进行连接必须使用长格式：

```caddy-d
@file `file()`
@first `file({'try_files': [{path}, {path} + '/', 'index.html']})`
@smallest `file({'try_policy': 'smallest_size', 'try_files': ['a.txt', 'b.txt']})`
```


---
### header

```caddy-d
header <field> [<value> ...]

expression header({'<field>': '<value>'})
```

按请求头字段匹配。

- `<field>` 是要检查的 HTTP 头字段名称。
	- 如果前缀为 `!`，则字段必须不存在才能匹配（省略值参数）。
- `<value>` 是字段必须具有的匹配值。可以指定一个或多个值。
	- 如果前缀为 `*`，则执行快速后缀匹配（出现在末尾）。
	- 如果后缀为 `*`，则执行快速前缀匹配（出现在开头）。
	- 如果由 `*` 包围，则执行快速子字符串匹配（出现在任何位置）。
	- 否则，是快速精确匹配。

同一集合中的不同头字段是 AND（与）关系。每个字段的多个值使用 OR（或）连接。

请注意，头字段可以重复并具有不同的值。后端应用程序必须考虑头字段值是数组，而不是单个值，Caddy 不在此类情况中解释含义。

#### 示例：

匹配具有包含 `Upgrade` 的 `Connection` 头的请求：

```caddy-d
@upgrade header Connection *Upgrade*
```

匹配 `Foo` 头包含 `bar` 或 `baz` 的请求：

```caddy-d
@foo {
	header Foo bar
	header Foo baz
}
```

匹配完全不具有 `Foo` 头字段的请求：

```caddy-d
@not_foo header !Foo
```

使用 [CEL 表达式](#expression)，通过检查包含 `Upgrade` 的 `Connection` 头和等于 `websocket` 的 `Upgrade` 头来匹配 websocket 请求（HTTP/2 有用于此的 `:protocol` 头）：

```caddy-d
@websockets `header({'Connection':'*Upgrade*','Upgrade':'websocket'}) || header({':protocol': 'websocket'})`
```


---
### header_regexp

```caddy-d
header_regexp [<name>] <field> <regexp>

expression header_regexp('<name>', '<field>', '<regexp>')
expression header_regexp('<field>', '<regexp>')
```

类似于 [`header`](#header)，但支持正则表达式。

使用的正则表达式语言是 RE2，包含在 Go 中。请参见 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax) 和 [Go regexp 语法概述](https://pkg.go.dev/regexp/syntax)。

从 v2.8.0 开始，如果_未_提供 `name`，名称将取自命名匹配器的名称。例如，命名匹配器 `@foo` 将使此匹配器命名为 `foo`。指定名称的主要优势是如果在同一命名匹配器中使用多个 regexp 匹配器（例如 `header_regexp` 和 [`path_regexp`](#path-regexp)，或不同头字段的多个匹配器）。

捕获组可以通过匹配后的指令中的 [占位符](/docs/caddyfile/concepts#placeholders) 访问：
- `{re.<name>.<capture_group>}` 其中：
  - `<name>` 是正则表达式的名称，
  - `<capture_group>` 是表达式中的捕获组的名称或编号。

- `{re.<capture_group>}` 无名称，也用于方便填充。缺点是如果在序列中使用多个 regexp 匹配器，则占位符值将被下一个匹配器覆盖。

捕获组 `0` 是完整的 regexp 匹配，`1` 是第一个捕获组，`2` 是第二个捕获组，依此类推。因此 `{re.foo.1}` 或 `{re.1}` 都将保存第一个捕获组的值。

每个头字段仅支持一个正则表达式，因为 regexp 模式无法合并；如果需要更多，请考虑使用 [`expression` 匹配器](#expression)。对多个不同头字段的匹配将进行 AND（与）连接。

#### 示例：

匹配 Cookie 头包含 `login_` 后跟十六进制字符串的请求，其中捕获组可以使用 `{re.login.1}` 或 `{re.1}` 访问。

```caddy-d
@login header_regexp login Cookie login_([a-f0-9]+)
```

可以通过省略名称来简化，该名称将从命名匹配器推断：

```caddy-d
@login header_regexp Cookie login_([a-f0-9]+)
```

或相同，使用 [CEL 表达式](#expression)：

```caddy-d
@login `header_regexp('login', 'Cookie', 'login_([a-f0-9]+)')`
```



---
### host

```caddy-d
host <hosts...>

expression host('<hosts...>')
```

按请求的 `Host` 头字段匹配请求。

由于大多数站点块已经在站点地址指示中指示主机，因此此匹配器更常用于使用通配符主机名的站点块中（参见 [通配符证书模式](/docs/caddyfile/patterns#wildcard-certificates)），但需要主机名特定逻辑。

多个 `host` 匹配器将一起使用 OR（或）连接。

#### 示例：

匹配一个子域：

```caddy-d
@sub host sub.example.com
```

匹配根域和子域：

```caddy-d
@site host example.com www.example.com
```

使用 [CEL 表达式](#expression) 匹配多个子域：

```caddy-d
@app `host('app1.example.com', 'app2.example.com')`
```



---
### method

```caddy-d
method <verbs...>

expression method('<verbs...>')
```

按 HTTP 请求的方法（动词）匹配。动词应为大写，如 `POST`。可以匹配一个或多个方法。

多个 `method` 匹配器将一起使用 OR（或）连接。

#### 示例：

匹配具有 `GET` 方法的请求：

```caddy-d
@get method GET
```

匹配具有 `PUT` 或 `DELETE` 方法的请求：

```caddy-d
@put-delete method PUT DELETE
```

使用 [CEL 表达式](#expression) 匹配只读方法：

```caddy-d
@read `method('GET', 'HEAD', 'OPTIONS')`
```



---
### not

```caddy-d
not <matcher>
```

或者，要否定多个 AND（与）连接的匹配器，打开一个块：

```caddy-d
not {
	<matchers...>
}
```

所封闭匹配器的结果将被否定。

#### 示例：

匹配路径_不_以 `/css/` 或 `/js/` 开头的请求。

```caddy-d
@not-assets {
	not path /css/* /js/*
}
```

匹配_既无_：
- `/api/` 路径前缀，也_无_
- `POST` 请求方法

即必须都不具有这些才能匹配：

```caddy-d
@with-neither {
	not path /api/*
	not method POST
}
```

匹配_不同时具有_：
- `/api/` 路径前缀，_和_
- `POST` 请求方法

即必须不具有或仅具有其中一个才能匹配：

```caddy-d
@without-both {
	not {
		path /api/*
		method POST
	}
}
```

[CEL 表达式](#expression) 中对此没有匹配器，因为您可以使用 `!` 运算符进行否定。例如：

```caddy-d
@without-both `!path('/api*') && !method('POST')`
```

这与以下相同，使用括号：

```caddy-d
@without-both `!(path('/api*') || method('POST'))`
```




---
### path

```caddy-d
path <paths...>

expression path('<paths...>')
```

按请求路径（请求 URI 的路径组件）。路径匹配是精确但不区分大小写的。可以使用通配符 `*`：

- 仅结尾处，用于前缀匹配（`/prefix/*`）
- 仅开头处，用于后缀匹配（`*.suffix`）
- 仅在两侧，用于子字符串匹配（`*/contains/*`）
- 仅在中间，用于 globular 匹配（`/accounts/*/info`）

斜杠是重要的。例如，`/foo*` 将匹配 `/foo`、`/foobar`、`/foo/` 和 `/foo/bar`，但 `/foo/*` 将_不_匹配 `/foo` 或 `/foobar`。

在匹配之前，请求路径会被清理以解析目录遍历点。此外，多个斜杠会合并，除非匹配模式具有多个斜杠。换句话说，`/foo` 将匹配 `/foo` 和 `//foo`，但 `//foo` 仅匹配 `//foo`。

由于任何给定的 URI 都有多个转义形式，请求路径会被标准化（URL 解码，取消转义），除了匹配模式中存在的转义序列中的位置。例如，`/foo/bar` 匹配 `/foo/bar` 和 `/foo%2Fbar`，但 `/foo%2Fbar` 仅匹配 `/foo%2Fbar`，因为配置中明确给出了转义序列。

特殊通配符转义 `%*` 也可以代替 `*` 使用，以保留其匹配跨度未转义。例如，`/bands/*/*` 将不匹配 `/bands/AC%2FDC/T.N.T`，因为路径将在标准化空间中比较，看起来像 `/bands/AC/DC/T.N.T`，这与模式不匹配；然而，`/bands/%*/*` 将匹配 `/bands/AC%2FDC/T.N.T`，因为 `%*` 表示的跨度将在不解码转义序列的情况下比较。

多个路径将一起使用 OR（或）连接。

#### 示例：

匹配多个目录及其内容：

```caddy-d
@assets path /js/* /css/* /images/*
```

匹配特定文件：

```caddy-d
@favicon path /favicon.ico
```

匹配文件扩展名：

```caddy-d
@extensions path *.js *.css
```

使用 [CEL 表达式](#expression)：

```caddy-d
@assets `path('/js/*', '/css/*', '/images/*')`
```



---
### path_regexp

```caddy-d
path_regexp [<name>] <regexp>

expression path_regexp('<name>', '<regexp>')
expression path_regexp('<regexp>')
```

类似于 [`path`](#path)，但支持正则表达式。运行在 URI 解码/取消转义的路径上。

使用的正则表达式语言是 RE2，包含在 Go 中。请参见 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax) 和 [Go regexp 语法概述](https://pkg.go.dev/regexp/syntax)。

从 v2.8.0 开始，如果_未_提供 `name`，名称将取自命名匹配器的名称。例如，命名匹配器 `@foo` 将使此匹配器命名为 `foo`。指定名称的主要优势是如果在同一命名匹配器中使用多个 regexp 匹配器（例如 `path_regexp` 和 [`header_regexp`](#header-regexp)）。

捕获组可以通过匹配后的指令中的 [占位符](/docs/caddyfile/concepts#placeholders) 访问：
- `{re.<name>.<capture_group>}` 其中：
  - `<name>` 是正则表达式的名称，
  - `<capture_group>` 是表达式中的捕获组的名称或编号。

- `{re.<capture_group>}` 无名称，也用于方便填充。缺点是如果在序列中使用多个 regexp 匹配器，则占位符值将被下一个匹配器覆盖。

捕获组 `0` 是完整的 regexp 匹配，`1` 是第一个捕获组，`2` 是第二个捕获组，依此类推。因此 `{re.foo.1}` 或 `{re.1}` 都将保存第一个捕获组的值。

每个命名匹配器只能有一个 `path_regexp` 模式，因为此匹配器无法与自身合并；如果需要更多，请考虑使用 [`expression` 匹配器](#expression)。

#### 示例：

匹配路径以 6 字符十六进制字符串后跟 `.css` 或 `.js` 作为文件扩展名的请求，其中捕获组（括号中的部分），可以使用 `{re.static.1}` 和 `{re.static.2}`（或 `{re.1}` 和 `{re.2}`）访问：

```caddy-d
@static path_regexp static \.([a-f0-9]{6})\.(css|js)$
```

可以通过省略名称来简化，该名称将从命名匹配器推断：

```caddy-d
@static path_regexp \.([a-f0-9]{6})\.(css|js)$
```

或相同，使用 [CEL 表达式](#expression)，同时也验证 [`file`](#file) 在磁盘上存在：

```caddy-d
@static `path_regexp('\.([a-f0-9]{6})\.(css|js)$') && file()`
```



---
### protocol

```caddy-d
protocol http|https|grpc|http/<version>[+]

expression protocol('http|https|grpc|http/<version>[+]')
```

按请求协议。可以使用广义协议名称，如 `http`、`https` 或 `grpc`；也可以使用特定的或最小 HTTP 版本，如 `http/1.1` 或 `http/2+`。

每个命名匹配器只能有一个 `protocol` 匹配器。

#### 示例：

匹配使用 HTTP/2 的请求：

```caddy-d
@http2 protocol http/2+
```

使用 [CEL 表达式](#expression)：

```caddy-d
@http2 `protocol('http/2+')`
```



---
### query

```caddy-d
query <key>=<val>...
query ""

expression query({'<key>': '<val>'})
expression query({'<key>': ['<vals...>']})
```

按查询字符串参数。应为一组 `key=value` 对序列，或空字符串 ""。键精确匹配（区分大小写）但也支持 `*` 以匹配任何值。值可以使用占位符。空字符串匹配没有查询参数的 HTTP 请求。

每个命名匹配器可以有多个 `query` 匹配器，具有相同键的对将一起使用 OR（或）连接。不同键将一起使用 AND（与）连接。因此，匹配器中的所有键必须至少有一个匹配值。

非法查询字符串（语法错误、未转义的分号等）将解析失败，因此不匹配。

**注意：** 查询字符串参数是数组，而不是单个值。这是因为重复的键在查询字符串中是有效的，每个键可能具有不同的值。此匹配器在查询字符串中分配了配置的任何值时将匹配键。使用查询字符串的后端应用程序必须考虑查询字符串值是数组并可以具有多个值。

#### 示例：

匹配具有任何值的 `q` 查询参数：

```caddy-d
@search query q=*
```

匹配具有值 `asc` 或 `desc` 的 `sort` 查询参数：

```caddy-d
@sorted query sort=asc sort=desc
```

同时匹配 `q` 和 `sort`，使用 [CEL 表达式](#expression)：

```caddy-d
@search-sort `query({'sort': ['asc', 'desc'], 'q': '*'})`
```



---
### remote_ip

```caddy-d
remote_ip <ranges...>

expression remote_ip('<ranges...>')
```

按远程 IP 地址（即直接对等点的 IP 地址或通过 [PROXY 协议](/docs/caddyfile/options#proxy-protocol) 设置的地址）。接受确切 IP 或 CIDR 范围。支持 IPv6 区域。

作为快捷方式，可以使用 `private_ranges` 匹配所有私有 IPv4 和 IPv6 范围。它与指定所有这些范围相同：`192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8 fd00::/8 ::1`

如果您希望匹配客户端的"真实 IP"，即从 HTTP 头解析的 IP，请使用 [`client_ip`](#client-ip) 匹配器。

每个命名匹配器可以有多个 `remote_ip` 匹配器，它们的范围将合并并使用 OR（或）连接。

#### 示例：

匹配来自私有 IPv4 地址的请求：

```caddy-d
@private-ipv4 remote_ip 192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8
```

此匹配器常与 [`not`](#not) 匹配器配对使用以反转匹配。例如，终止来自_公共_ IPv4 和 IPv6 地址（这是所有私有范围的反面）的所有连接：

```caddy
example.com {
	@denied not remote_ip private_ranges
	abort @denied

	respond "您好，您必须来自私有网络！"
}
```

在 [CEL 表达式](#expression) 中，它将如下所示：

```caddy-d
@my-friends `remote_ip('12.23.34.45', '23.34.45.56')`
```



---
### vars

```caddy-d
vars <variable> <values...>

expression vars({'<variable>': '<value>'})
expression vars({'<variable>': ['<values...>']})
```

按请求上下文中的变量值或占位符值。可以指定多个值以匹配这些可能值中的任何一个（OR 或）。

**&lt;variable&gt;** 参数可以是变量名或花括号中的占位符 `{ }`。（第一个参数中的占位符不会展开。）

此匹配器在以下情况下最有用：与设置输出的 [`map` 指令](/docs/caddyfile/directives/map) 配对，在路由内使用 [`vars` 指令](/docs/caddyfile/directives/vars)，或与插件在请求上下文中设置某些信息。

#### 示例：

匹配 [`map` 指令](/docs/caddyfile/directives/map) 名为 `magic_number` 的输出，值为 `3` 或 `5`：

```caddy-d
vars {magic_number} 3 5
```

匹配任意占位符的值，即已认证用户的 ID，`Bob` 或 `Alice`：

```caddy-d
vars {http.auth.user.id} Bob Alice
```

使用 [`vars` 指令](/docs/caddyfile/directives/vars) 设置变量，然后使用 [`vars` 匹配器](#vars) 进行匹配的完整示例。在此，我们将两个请求头合并为一个变量，并匹配该变量：

```caddy
example.com {
	vars combined_header "{header.Foo}_{header.Bar}"
	@special vars {vars.combined_header} "123_456"
	handle @special {
		respond "您发送了 Foo=123 和 Bar=456！"
	}
	handle {
		respond "Foo 和 Bar 不特殊。"
	}
}
```

在 [CEL 表达式](#expression) 中，它将如下所示：

```caddy-d
@magic `vars({'magic_number': ['3', '5']})`
```


---
### vars_regexp

```caddy-d
vars_regexp [<name>] <variable> <regexp>

expression vars_regexp('<name>', '<variable>', '<regexp>')
expression vars_regexp('<variable>', '<regexp>')
```

类似于 [`vars`](#vars)，但支持正则表达式。

使用的正则表达式语言是 RE2，包含在 Go 中。请参见 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax) 和 [Go regexp 语法概述](https://pkg.go.dev/regexp/syntax)。

从 v2.8.0 开始，如果_未_提供 `name`，名称将取自命名匹配器的名称。例如，命名匹配器 `@foo` 将使此匹配器命名为 `foo`。指定名称的主要优势是如果在同一命名匹配器中使用多个 regexp 匹配器（例如 `vars_regexp` 和 [`header_regexp`](#header-regexp)）。

捕获组可以通过匹配后的指令中的 [占位符](/docs/caddyfile/concepts#placeholders) 访问：
- `{re.<name>.<capture_group>}` 其中：
  - `<name>` 是正则表达式的名称，
  - `<capture_group>` 是表达式中的捕获组的名称或编号。

- `{re.<capture_group>}` 无名称，也用于方便填充。缺点是如果在序列中使用多个 regexp 匹配器，则占位符值将被下一个匹配器覆盖。

捕获组 `0` 是完整的 regexp 匹配，`1` 是第一个捕获组，`2` 是第二个捕获组，依此类推。因此 `{re.foo.1}` 或 `{re.1}` 都将保存第一个捕获组的值。

每个变量名仅支持一个正则表达式，因为 regexp 模式无法合并；如果需要更多，请考虑使用 [`expression` 匹配器](#expression)。对多个不同变量的匹配将进行 AND（与）连接。

#### 示例：

匹配 [`map` 指令](/docs/caddyfile/directives/map) 名为 `magic_number` 的输出，值为以 `4` 开头的值，其中捕获组可以使用 `{re.magic.1}` 或 `{re.1}` 访问：

```caddy-d
@magic vars_regexp magic {magic_number} ^(4.*)
```

可以通过省略名称来简化，该名称将从命名匹配器推断：

```caddy-d
@magic vars_regexp {magic_number} ^(4.*)
```

在 [CEL 表达式](#expression) 中，它将如下所示：

```caddy-d
@magic `vars_regexp('magic_number', '^(4.*)')`
```