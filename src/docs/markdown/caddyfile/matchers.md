---
title: 请求匹配器（Caddyfile）
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
		item.innerHTML = `<a href="#${anchor}" style="color: inherit;" title="Matcher token">${text}</a>`;
	});
});
</script>

# 请求匹配器

**请求匹配器**可用于根据各种条件过滤（或分类）请求。

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

在 Caddyfile 中，紧跟在指令后面的**匹配器标记**可以限制该指令的作用范围。匹配器标记可以是以下形式之一：

1. [**`*`**](#通配符匹配器) 匹配所有请求（通配符；默认）。
2. [**`/路径`**](#路径匹配器) 以斜杠开头，匹配请求路径。
3. [**`@名称`**](#命名匹配器) 指定一个_命名匹配器_。

如果某个指令支持匹配器，则在其语法文档中会以 `[<匹配器>]` 的形式出现。匹配器标记[通常是可选的](/docs/caddyfile/directives#syntax)，用 `[ ]` 表示。如果省略匹配器标记，则等同于通配符匹配器（`*`）。


#### 示例

以下指令适用于[所有](#通配符匹配器) HTTP 请求：

```caddy-d
reverse_proxy localhost:9000
```

这也是一样的（此处 `*` 不是必需的）：

```caddy-d
reverse_proxy * localhost:9000
```

但以下指令仅适用于路径以 `/api/` 开头的请求：

```caddy-d
reverse_proxy /api/* localhost:9000
```

要匹配路径以外的其他内容，请定义一个[命名匹配器](#命名匹配器)，并使用 `@名称` 引用它：

```caddy-d
@postfoo {
	method POST
	path /foo/*
}
reverse_proxy @postfoo localhost:9000
```




### 通配符匹配器

通配符（或“捕获所有”）匹配器 `*` 匹配所有请求，仅在需要匹配器标记时才使用。例如，如果您想为指令提供的第一个参数恰好也是一个路径，那么它看起来就像一个路径匹配器！因此，您可以使用通配符匹配器来消除歧义，例如：

```caddy-d
root * /home/www/mysite
```

否则，这个匹配器不常用。我们通常建议在语法不要求时省略它。


### 路径匹配器

按 URI 路径匹配是最常见的匹配请求方式，因此匹配器可以内联，如下所示：

```caddy-d
redir /old.html /new.html
```

路径匹配器标记必须以斜杠 `/` 开头。

**[路径匹配](#path)默认是精确匹配，而不是前缀匹配。** 您必须附加一个 `*` 才能进行快速前缀匹配。请注意，`/foo*` 会匹配 `/foo`、`/foo/` 以及 `/foobar`；您可能实际上想要的是 `/foo/*`。


### 命名匹配器

所有不是路径或通配符匹配器的匹配器都必须是命名匹配器。此类匹配器在任何特定指令之外定义，并且可以重复使用。

使用唯一名称定义匹配器能提供更大的灵活性，允许您将[任何可用的匹配器](#标准匹配器)组合成一个集合：

```caddy-d
@名称 {
	...
}
```

或者，如果集合中只有一个匹配器，可以将其放在同一行：

```caddy-d
@名称 ...
```

然后，您可以像这样使用匹配器，将其作为指令的第一个参数：

```caddy-d
指令 @名称
```

例如，此配置将 HTTP/1.1 WebSocket 请求代理到 `localhost:6001`，其他请求代理到 `localhost:8080`。它匹配具有 `Connection` 标头字段_包含_ `Upgrade`，**并且**另一个 `Upgrade` 标头字段精确为 `websocket` 的请求：

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

如果匹配器集合仅包含一个匹配器，也可以使用单行语法：

```caddy-d
@post method POST
reverse_proxy @post localhost:6001
```

作为特例，[`expression` 匹配器](#expression)可以在不指定其名称的情况下使用，只要在匹配器名称后跟一个[引用的](/docs/caddyfile/concepts#tokens-and-quotes)参数（即 CEL 表达式本身）：

```caddy-d
@not-found `{err.status_code} == 404`
```

与指令一样，命名匹配器定义必须放在使用它们的[站点块](/docs/caddyfile/concepts#structure)内部。

一个命名匹配器定义构成一个_匹配器集合_。集合中的匹配器是“与”的关系，即所有匹配器都必须匹配。例如，如果集合中同时包含 [`header`](#header) 和 [`path`](#path) 匹配器，则两者都必须匹配。

同一类型的多个匹配器可以使用布尔代数（AND/OR）进行合并（例如同一集合中的多个 [`path`](#path) 匹配器），具体在其各自的章节中描述。

对于更复杂的布尔匹配逻辑，建议使用 [`expression` 匹配器](#expression)编写 CEL 表达式，它支持 **与** `&&`、**或** `||` 以及**括号** `( )`。





## 标准匹配器

完整的匹配器文档可以在[各个匹配器模块的文档](/docs/json/apps/http/servers/routes/match/)中找到。

请求可以通过以下方式匹配：



### client_ip

```caddy-d
client_ip <范围...>

expression client_ip('<范围...>')
```

按客户端 IP 地址匹配。接受精确 IP 或 CIDR 范围。支持 IPv6 区域。

此匹配器最好在配置了 [`trusted_proxies`](/docs/caddyfile/options#trusted-proxies) 全局选项时使用，否则其行为与 [`remote_ip`](#remote-ip) 匹配器相同。只有来自受信任代理的请求才会在请求开始时解析其客户端 IP；未受信任的请求将使用直接对端的远程 IP 地址或通过 [PROXY 协议](/docs/caddyfile/options#proxy-protocol)设置的地址。

作为快捷方式，可以使用 `private_ranges` 来匹配所有私有 IPv4 和 IPv6 范围。这等同于指定以下所有范围：`192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8 fd00::/8 ::1`

每个命名匹配器中可以有多个 `client_ip` 匹配器，它们的作用范围将合并并通过“或”逻辑连接。

#### 示例：

匹配来自私有 IPv4 地址的请求：

```caddy-d
@private-ipv4 client_ip 192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8
```

此匹配器通常与 [`not`](#not) 匹配器配合使用，以反转匹配。例如，要中止所有来自_公共_ IPv4 和 IPv6 地址的连接（即所有私有范围的反向）：

```caddy
example.com {
	@denied not client_ip private_ranges
	abort @denied

	respond "Hello, you must be from a private network!"
}
```

在 [CEL 表达式](#expression)中，它将如下所示：

```caddy-d
@my-friends `client_ip('12.23.34.45', '23.34.45.56')`
```



### expression

```caddy-d
expression <cel...>
```

按任意返回 `true` 或 `false` 的 [CEL（通用表达式语言）](https://github.com/google/cel-spec)表达式匹配。

大多数其他请求匹配器也可以在表达式中作为函数使用，这为布尔逻辑提供了比外部表达式更大的灵活性。有关 CEL 表达式中支持的语法，请参阅每个匹配器的文档。

Caddy [占位符](/docs/conventions#placeholders)（或 [Caddyfile 简写](/docs/caddyfile/concepts#placeholders)）可以在这些 CEL 表达式中使用，因为它们会在被 CEL 环境解释之前进行预处理并转换为常规的 CEL 函数调用。如果占位符应作为字符串参数传递给匹配器函数，则开头的 `{` 应该用反斜杠 `\` 转义，以免被预处理，例如 `file('\{path}.md')`。

为方便起见，如果定义的命名匹配器仅由一个 CEL 表达式组成，则可以省略匹配器名称。CEL 表达式必须[引用](/docs/caddyfile/concepts#tokens-and-quotes)（推荐使用反引号或 heredoc）。这看起来非常简洁：

```caddy-d
@mutable `{method}.startsWith("P")`
```

在这种情况下，默认使用 CEL 匹配器。

#### 示例：

匹配方法以 `P` 开头的请求，例如 `PUT` 或 `POST`：

```caddy-d
@methods expression {method}.startsWith("P")
```

匹配处理器返回错误状态码 `404` 的请求，通常与 [`handle_errors` 指令](/docs/caddyfile/directives/handle_errors)一起使用：

```caddy-d
@404 expression {err.status_code} == 404
```

匹配路径匹配两个不同正则表达式之一的请求；这只能使用表达式编写，因为 [`path_regexp`](#path-regexp) 匹配器通常每个命名匹配器只能存在一次：

```caddy-d
@user expression path_regexp('^/user/(\w*)') || path_regexp('^/(\w*)')
```

或者相同，省略匹配器名称，并包裹在[反引号](/docs/caddyfile/concepts#tokens-and-quotes)中，使其被解析为单个标记：

```caddy-d
@user `path_regexp('^/user/(\w*)') || path_regexp('^/(\w*)')`
```

您可以使用 [heredoc 语法](/docs/caddyfile/concepts#heredocs)编写多行 CEL 表达式：

```caddy-d
@api <<CEL
	{method} == "GET"
	&& {path}.startsWith("/api/")
	CEL
respond @api "Hello, API!"
```


---
### file

```caddy-d
file {
	root       <路径>
	try_files  <文件...>
	try_policy first_exist|first_exist_fallback|smallest_size|largest_size|most_recently_modified
	split_path <分隔符...>
}
file <文件...>

expression `file({
	'root': '<路径>',
	'try_files': ['<文件...>'],
	'try_policy': 'first_exist|first_exist_fallback|smallest_size|largest_size|most_recently_modified',
	'split_path': ['<分隔符...>']
})`
expression file('<文件...>')
```

按文件匹配。

- `root` 定义查找文件的目录。默认为当前工作目录，或者如果设置了 [变量](/docs/modules/http.handlers.vars) `root`（可通过 [`root` 指令](/docs/caddyfile/directives/root)设置），则为 `{http.vars.root}`。

- `try_files` 检查列表中与 try_policy 匹配的文件。

  要匹配目录，请在路径后附加一个尾部斜杠 `/`。所有文件路径都相对于站点[根目录](/docs/caddyfile/directives/root)，并且[全局模式](https://pkg.go.dev/path/filepath#Match)将被展开。

  如果 `try_policy` 是 `first_exist`（默认值），则列表中的最后一项可以是一个以 `=` 为前缀的数字（例如 `=404`），作为后备，它将发出带有该代码的错误；该错误可以被 [`handle_errors`](/docs/caddyfile/directives/handle_errors) 捕获和处理。

- `try_policy` 指定如何选择文件。默认值为 `first_exist`。

  - `first_exist` 检查文件是否存在。选择第一个存在的文件。
  - `first_exist_fallback` 类似于 `first_exist`，但假定列表中的最后一个元素始终存在，以防止磁盘访问。
  - `smallest_size` 选择大小最小的文件。
  - `largest_size` 选择大小最大的文件。
  - `most_recently_modified` 选择最近修改的文件。

- `split_path` 将导致路径在列表中找到的第一个分隔符处进行拆分，每个要尝试的文件路径都会这样处理。对于每个拆分值，包括分隔符本身的拆分左侧部分将是尝试的文件路径。例如，`/remote.php/dav/` 使用分隔符 `.php` 将尝试文件 `/remote.php`。每个分隔符必须出现在 URI 路径组件的末尾才能用作拆分分隔符。这是一个小众设置，主要用于服务 PHP 站点。

因为 `try_files` 与 `first_exist` 策略非常常用，所以有一个单行快捷方式：

```caddy-d
file <文件...>
```

一个空的 `file` 匹配器（后面没有列出文件）将检查请求的文件——直接取自 URI，相对于[站点根目录](/docs/caddyfile/directives/root)——是否存在。这实际上等同于 `file {path}`。


<aside class="tip">

由于基于磁盘上文件是否存在进行重写非常常见，还有一个 [`try_files` 指令](/docs/caddyfile/directives/try_files)，它是 `file` 匹配器和 [`rewrite` 处理器](/docs/caddyfile/directives/rewrite)的快捷方式。

</aside>


匹配成功后，将提供四个新的占位符：

- `{file_match.relative}` 文件相对于根目录的路径。这在重写请求时通常很有用。
- `{file_match.absolute}` 匹配文件的绝对路径，包括根目录。
- `{file_match.type}` 文件类型，`file` 或 `directory`。
- `{file_match.remainder}` 拆分文件路径后剩余的部分（如果配置了 `split_path`）。


#### 示例：

匹配路径是存在的文件的请求：

```caddy-d
@file file
```

匹配路径后跟 `.html` 是存在的文件的请求，或者如果不存在，则匹配路径本身是存在的文件的请求：

```caddy-d
@html file {
	try_files {path}.html {path} 
}
```

与上面相同，但使用单行快捷方式，并在未找到文件时回退到发出 404 错误：

```caddy-d
@html-or-error file {path}.html {path} =404
```

更多使用 [CEL 表达式](#expression) 的示例。请记住，占位符会被预处理并转换为常规的 CEL 函数调用，然后再由 CEL 环境解释，因此这里使用了拼接。此外，由于当前解析限制，如果与占位符拼接，必须使用长格式：

```caddy-d
@file `file()`
@first `file({'try_files': [{path}, {path} + '/', 'index.html']})`
@smallest `file({'try_policy': 'smallest_size', 'try_files': ['a.txt', 'b.txt']})`
```


---
### header

```caddy-d
header <字段> [<值> ...]

expression header({'<字段>': '<值>'})
```

按请求标头字段匹配。

- `<字段>` 是要检查的 HTTP 标头字段的名称。
  - 如果以 `!` 为前缀，则字段必须不存在才能匹配（省略值参数）。
- `<值>` 是字段必须具有才能匹配的值。可以指定一个或多个。
  - 如果以 `*` 为前缀，则执行快速后缀匹配（出现在末尾）。
  - 如果以 `*` 为后缀，则执行快速前缀匹配（出现在开头）。
  - 如果两边都有 `*`，则执行快速子字符串匹配（出现在任意位置）。
  - 否则，是快速精确匹配。

同一集合中的不同标头字段是“与”的关系。每个字段的多个值是“或”的关系。

请注意，标头字段可以重复并具有不同的值。后端应用程序必须考虑到标头字段值是数组，而不是单个值，Caddy 不会解释这种困境中的含义。

#### 示例：

匹配 `Connection` 标头包含 `Upgrade` 的请求：

```caddy-d
@upgrade header Connection *Upgrade*
```

匹配 `Foo` 标头包含 `bar` 或 `baz` 的请求：

```caddy-d
@foo {
	header Foo bar
	header Foo baz
}
```

匹配完全没有 `Foo` 标头字段的请求：

```caddy-d
@not_foo header !Foo
```

使用 [CEL 表达式](#expression)，通过检查 `Connection` 标头包含 `Upgrade` 且 `Upgrade` 标头等于 `websocket` 来匹配 WebSocket 请求（HTTP/2 有 `:protocol` 标头用于此目的）：

```caddy-d
@websockets `header({'Connection':'*Upgrade*','Upgrade':'websocket'}) || header({':protocol': 'websocket'})`
```


---
### header_regexp

```caddy-d
header_regexp [<名称>] <字段> <正则表达式>

expression header_regexp('<名称>', '<字段>', '<正则表达式>')
expression header_regexp('<字段>', '<正则表达式>')
```

类似于 [`header`](#header)，但支持正则表达式。

使用的正则表达式语言是 Go 中包含的 RE2。请参阅 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax)和 [Go 正则表达式语法概述](https://pkg.go.dev/regexp/syntax)。

从 v2.8.0 开始，如果_没有_提供 `名称`，则名称将从命名匹配器的名称中获取。例如，命名匹配器 `@foo` 将使此匹配器命名为 `foo`。指定名称的主要优点是，如果在同一个命名匹配器中使用了多个正则表达式匹配器（例如 `header_regexp` 和 [`path_regexp`](#path-regexp)，或多个不同的标头字段）时。

捕获组可以在匹配后通过[占位符](/docs/caddyfile/concepts#placeholders)在指令中访问：
- `{re.<名称>.<捕获组>}`，其中：
  - `<名称>` 是正则表达式的名称，
  - `<捕获组>` 是表达式中捕获组的名称或编号。

- `{re.<捕获组>}`（不带名称）也会为了方便而填充。需要注意的是，如果多个正则表达式匹配器顺序使用，则占位符值将被下一个匹配器覆盖。

捕获组 `0` 是整个正则表达式匹配，`1` 是第一个捕获组，`2` 是第二个捕获组，依此类推。因此 `{re.foo.1}` 或 `{re.1}` 都将包含第一个捕获组的值。

每个标头字段只支持一个正则表达式，因为正则表达式模式无法合并；如果需要更多，请考虑使用 [`expression` 匹配器](#expression)。针对多个不同标头字段的匹配将是“与”的关系。

#### 示例：

匹配 Cookie 标头包含 `login_` 后跟十六进制字符串的请求，捕获组可通过 `{re.login.1}` 或 `{re.1}` 访问。

```caddy-d
@login header_regexp login Cookie login_([a-f0-9]+)
```

可以通过省略名称来简化，名称将从命名匹配器中推断：

```caddy-d
@login header_regexp Cookie login_([a-f0-9]+)
```

或者相同，使用 [CEL 表达式](#expression)：

```caddy-d
@login `header_regexp('login', 'Cookie', 'login_([a-f0-9]+)')`
```



---
### host

```caddy-d
host <主机...>

expression host('<主机...>')
```

按请求的 `Host` 标头字段匹配请求。

由于大多数站点块已经在站点地址中指明了主机，因此此匹配器更常用于使用通配符主机名的站点块（请参阅[通配符证书模式](/docs/caddyfile/patterns#wildcard-certificates)），但需要特定于主机名的逻辑。

多个 `host` 匹配器将通过“或”逻辑连接。

#### 示例：

匹配一个子域名：

```caddy-d
@sub host sub.example.com
```

匹配根域名和一个子域名：

```caddy-d
@site host example.com www.example.com
```

使用 [CEL 表达式](#expression)匹配多个子域名：

```caddy-d
@app `host('app1.example.com', 'app2.example.com')`
```



---
### method

```caddy-d
method <动词...>

expression method('<动词...>')
```

按 HTTP 请求的方法（动词）匹配。动词应为大写，如 `POST`。可以匹配一个或多个方法。

多个 `method` 匹配器将通过“或”逻辑连接。

#### 示例：

匹配 `GET` 方法的请求：

```caddy-d
@get method GET
```

匹配 `PUT` 或 `DELETE` 方法的请求：

```caddy-d
@put-delete method PUT DELETE
```

使用 [CEL 表达式](#expression)匹配只读方法：

```caddy-d
@read `method('GET', 'HEAD', 'OPTIONS')`
```



---
### not

```caddy-d
not <匹配器>
```

或者，要否定多个匹配器（这些匹配器通过“与”逻辑连接），请打开一个块：

```caddy-d
not {
	<匹配器...>
}
```

封闭匹配器的结果将被否定。

#### 示例：

匹配路径不以 `/css/` 或 `/js/` 开头的请求。

```caddy-d
@not-assets {
	not path /css/* /js/*
}
```

匹配**既不包含**以下两者的请求：
- 路径前缀 `/api/`，也不
- `POST` 请求方法

即必须都不匹配才能匹配：

```caddy-d
@with-neither {
	not path /api/*
	not method POST
}
```

匹配**不同时包含**以下两者的请求：
- 路径前缀 `/api/`，且
- `POST` 请求方法

即必须两个都不匹配或只匹配其中一个才能匹配：

```caddy-d
@without-both {
	not {
		path /api/*
		method POST
	}
}
```

此匹配器没有对应的 [CEL 表达式](#expression)，因为您可以使用 `!` 运算符进行否定。例如：

```caddy-d
@without-both `!path('/api*') && !method('POST')`
```

这与以下使用括号的表达式相同：

```caddy-d
@without-both `!(path('/api*') || method('POST'))`
```




---
### path

```caddy-d
path <路径...>

expression path('<路径...>')
```

按请求路径（请求 URI 的路径部分）匹配。路径匹配是精确的，但不区分大小写。可以使用通配符 `*`：

- 仅在末尾，用于前缀匹配（`/前缀/*`）
- 仅在开头，用于后缀匹配（`*.后缀`）
- 仅两侧，用于子字符串匹配（`*/包含/*`）
- 仅在中间，用于全局匹配（`/accounts/*/info`）

斜杠是重要的。例如，`/foo*` 将匹配 `/foo`、`/foobar`、`/foo/` 和 `/foo/bar`，但 `/foo/*` 将_不_匹配 `/foo` 或 `/foobar`。

请求路径会被清理以在匹配前解析目录遍历点。此外，多个斜杠会被合并，除非匹配模式本身包含多个斜杠。换句话说，`/foo` 将匹配 `/foo` 和 `//foo`，但 `//foo` 仅匹配 `//foo`。

由于任何给定的 URI 都有多种转义形式，请求路径会被规范化（URL 解码、取消转义），但那些在匹配模式中也存在转义序列的位置除外。例如，`/foo/bar` 匹配 `/foo/bar` 和 `/foo%2Fbar`，但 `/foo%2Fbar` 仅匹配 `/foo%2Fbar`，因为配置中明确给出了转义序列。

特殊通配符转义 `%*` 也可以用来代替 `*`，以保持其匹配的跨度转义。例如，`/bands/*/*` 不会匹配 `/bands/AC%2FDC/T.N.T`，因为路径会在规范化空间中进行比较，看起来像 `/bands/AC/DC/T.N.T`，这与模式不匹配；然而，`/bands/%*/*` 将匹配 `/bands/AC%2FDC/T.N.T`，因为由 `%*` 表示的跨度将在不解码转义序列的情况下进行比较。

多个路径将通过“或”逻辑连接。

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
path_regexp [<名称>] <正则表达式>

expression path_regexp('<名称>', '<正则表达式>')
expression path_regexp('<正则表达式>')
```

类似于 [`path`](#path)，但支持正则表达式。针对 URI 解码/取消转义后的路径运行。

使用的正则表达式语言是 Go 中包含的 RE2。请参阅 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax)和 [Go 正则表达式语法概述](https://pkg.go.dev/regexp/syntax)。

从 v2.8.0 开始，如果_没有_提供 `名称`，则名称将从命名匹配器的名称中获取。例如，命名匹配器 `@foo` 将使此匹配器命名为 `foo`。指定名称的主要优点是，如果在同一个命名匹配器中使用了多个正则表达式匹配器（例如 `path_regexp` 和 [`header_regexp`](#header-regexp)）时。

捕获组可以在匹配后通过[占位符](/docs/caddyfile/concepts#placeholders)在指令中访问：
- `{re.<名称>.<捕获组>}`，其中：
  - `<名称>` 是正则表达式的名称，
  - `<捕获组>` 是表达式中捕获组的名称或编号。

- `{re.<捕获组>}`（不带名称）也会为了方便而填充。需要注意的是，如果多个正则表达式匹配器顺序使用，则占位符值将被下一个匹配器覆盖。

捕获组 `0` 是整个正则表达式匹配，`1` 是第一个捕获组，`2` 是第二个捕获组，依此类推。因此 `{re.foo.1}` 或 `{re.1}` 都将包含第一个捕获组的值。

每个命名匹配器中只能有一个 `path_regexp` 模式，因为此匹配器无法与自身合并；如果需要更多，请考虑使用 [`expression` 匹配器](#expression)。

#### 示例：

匹配路径以 6 个十六进制字符后跟 `.css` 或 `.js` 作为文件扩展名结尾的请求，捕获组（括号 `( )` 内的部分）可分别通过 `{re.static.1}` 和 `{re.static.2}`（或 `{re.1}` 和 `{re.2}`）访问：

```caddy-d
@static path_regexp static \.([a-f0-9]{6})\.(css|js)$
```

可以通过省略名称来简化，名称将从命名匹配器中推断：

```caddy-d
@static path_regexp \.([a-f0-9]{6})\.(css|js)$
```

或者相同，使用 [CEL 表达式](#expression)，同时验证文件在磁盘上存在：

```caddy-d
@static `path_regexp('\.([a-f0-9]{6})\.(css|js)$') && file()`
```



---
### protocol

```caddy-d
protocol http|https|grpc|http/<版本>[+]

expression protocol('http|https|grpc|http/<版本>[+]')
```

按请求协议匹配。可以使用广泛的协议名称，如 `http`、`https` 或 `grpc`；也可以使用特定或最低的 HTTP 版本，如 `http/1.1` 或 `http/2+`。

每个命名匹配器中只能有一个 `protocol` 匹配器。

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
query <键>=<值>...
query ""

expression query({'<键>': '<值>'})
expression query({'<键>': ['<值...>']})
```

按查询字符串参数匹配。应该是一系列 `键=值` 对，或一个空字符串 `""`。键是精确匹配（区分大小写），但也支持 `*` 匹配任何值。值可以使用占位符。空字符串匹配没有查询参数的 HTTP 请求。

每个命名匹配器中可以有多个 `query` 匹配器，相同键的对将通过“或”逻辑连接。不同键将通过“与”逻辑连接。因此，匹配器中所有键必须至少有一个匹配的值。

非法的查询字符串（语法错误、未转义的分号等）将无法解析，因此不会匹配。

**注意：** 查询字符串参数是数组，而不是单个值。这是因为重复的键在查询字符串中是有效的，每个键可能有不同的值。如果查询字符串中分配了该键的任何一个配置值，此匹配器将匹配该键。使用查询字符串的后端应用程序必须考虑到查询字符串值是数组，并且可以有多个值。

#### 示例：

匹配带有任意值的 `q` 查询参数：

```caddy-d
@search query q=*
```

匹配 `sort` 查询参数的值为 `asc` 或 `desc`：

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
remote_ip <范围...>

expression remote_ip('<范围...>')
```

按远程 IP 地址（即直接对端的 IP 地址或通过 [PROXY 协议](/docs/caddyfile/options#proxy-protocol)设置的地址）匹配。接受精确 IP 或 CIDR 范围。支持 IPv6 区域。

作为快捷方式，可以使用 `private_ranges` 来匹配所有私有 IPv4 和 IPv6 范围。这等同于指定以下所有范围：`192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8 fd00::/8 ::1`

如果您希望匹配从 HTTP 标头解析的客户端的“真实 IP”，请改用 [`client_ip`](#client-ip) 匹配器。

每个命名匹配器中可以有多个 `remote_ip` 匹配器，它们的作用范围将合并并通过“或”逻辑连接。

#### 示例：

匹配来自私有 IPv4 地址的请求：

```caddy-d
@private-ipv4 remote_ip 192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8
```

此匹配器通常与 [`not`](#not) 匹配器配合使用，以反转匹配。例如，要中止所有来自_公共_ IPv4 和 IPv6 地址的连接（即所有私有范围的反向）：

```caddy
example.com {
	@denied not remote_ip private_ranges
	abort @denied

	respond "Hello, you must be from a private network!"
}
```

在 [CEL 表达式](#expression)中，它将如下所示：

```caddy-d
@my-friends `remote_ip('12.23.34.45', '23.34.45.56')`
```



---
### vars

```caddy-d
vars <变量> <值...>

expression vars({'<变量>': '<值>'})
expression vars({'<变量>': ['<值...>']})
```

按请求上下文中的变量值或占位符的值匹配。可以指定多个值来匹配其中的任何一个（“或”关系）。

**&lt;变量&gt;** 参数可以是变量名或花括号 `{ }` 中的占位符。（第一个参数中的占位符不会被展开。）

此匹配器在与设置输出的 [`map` 指令](/docs/caddyfile/directives/map)、在路由中使用的 [`vars` 指令](/docs/caddyfile/directives/vars)或设置请求上下文信息的插件一起使用时最有用。

#### 示例：

匹配名为 `magic_number` 的 [`map` 指令](/docs/caddyfile/directives/map)输出，值为 `3` 或 `5`：

```caddy-d
vars {magic_number} 3 5
```

匹配任意占位符的值，例如已验证用户的 ID，`Bob` 或 `Alice`：

```caddy-d
vars {http.auth.user.id} Bob Alice
```

一个完整示例，使用 [`vars` 指令](/docs/caddyfile/directives/vars)设置一个变量，然后使用 [`vars` 匹配器](#vars)进行匹配。这里我们将两个请求标头组合成一个变量，并匹配该变量：

```caddy
example.com {
	vars combined_header "{header.Foo}_{header.Bar}"
	@special vars {vars.combined_header} "123_456"
	handle @special {
		respond "You sent Foo=123 and Bar=456!"
	}
	handle {
		respond "Foo and Bar were not special."
	}
}
```

在 [CEL 表达式](#expression)中，它将如下所示：

```caddy-d
@magic `vars({'magic_number': ['3', '5']})`
```


---
### vars_regexp

```caddy-d
vars_regexp [<名称>] <变量> <正则表达式>

expression vars_regexp('<名称>', '<变量>', '<正则表达式>')
expression vars_regexp('<变量>', '<正则表达式>')
```

类似于 [`vars`](#vars)，但支持正则表达式。

使用的正则表达式语言是 Go 中包含的 RE2。请参阅 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax)和 [Go 正则表达式语法概述](https://pkg.go.dev/regexp/syntax)。

从 v2.8.0 开始，如果_没有_提供 `名称`，则名称将从命名匹配器的名称中获取。例如，命名匹配器 `@foo` 将使此匹配器命名为 `foo`。指定名称的主要优点是，如果在同一个命名匹配器中使用了多个正则表达式匹配器（例如 `vars_regexp` 和 [`header_regexp`](#header-regexp)）时。

捕获组可以在匹配后通过[占位符](/docs/caddyfile/concepts#placeholders)在指令中访问：
- `{re.<名称>.<捕获组>}`，其中：
  - `<名称>` 是正则表达式的名称，
  - `<捕获组>` 是表达式中捕获组的名称或编号。

- `{re.<捕获组>}`（不带名称）也会为了方便而填充。需要注意的是，如果多个正则表达式匹配器顺序使用，则占位符值将被下一个匹配器覆盖。

捕获组 `0` 是整个正则表达式匹配，`1` 是第一个捕获组，`2` 是第二个捕获组，依此类推。因此 `{re.foo.1}` 或 `{re.1}` 都将包含第一个捕获组的值。

每个变量名只支持一个正则表达式，因为正则表达式模式无法合并；如果需要更多，请考虑使用 [`expression` 匹配器](#expression)。针对不同变量的匹配将是“与”的关系。

#### 示例：

匹配名为 `magic_number` 的 [`map` 指令](/docs/caddyfile/directives/map)输出，值以 `4` 开头，捕获组可通过 `{re.magic.1}` 或 `{re.1}` 访问：

```caddy-d
@magic vars_regexp magic {magic_number} ^(4.*)
```

可以通过省略名称来简化，名称将从命名匹配器中推断：

```caddy-d
@magic vars_regexp {magic_number} ^(4.*)
```

在 [CEL 表达式](#expression)中，它将如下所示：

```caddy-d
@magic `vars_regexp('magic_number', '^(4.*)')`