---
title: uri (Caddyfile 指令)
---

# uri

操作请求的 URI。可以去除路径前缀/后缀，或者在整个 URI 中替换子字符串。

该指令与 [`rewrite`](rewrite) 不同之处在于，`uri` 是 _差异性地_ 修改 URI，而不是像 `rewrite` 那样将其完全重置为另一个值。`rewrite` 被视为特殊的内置重定向，而 `uri` 只是一个普通中间件。

## 语法

支持多种不同的操作：

```caddy-d
uri [<匹配器>] strip_prefix <目标>
uri [<匹配器>] strip_suffix <目标>
uri [<匹配器>] replace      <目标> <替换> [<限制>]
uri [<匹配器>] path_regexp  <目标> <替换>
uri [<匹配器>] query        [-|+]<参数> [<值>]
uri [<匹配器>] query {
	<参数> [<值>] [<替换>]
	...
}
```

第一个（非匹配器）参数指定操作类型：

- **strip_prefix** 从路径中去除前缀。

- **strip_suffix** 从路径中去除后缀。

- **replace** 对整个 URI 执行子字符串替换。

	- **&lt;目标&gt;** 是前缀、后缀或搜索字符串/正则表达式。如果是前缀，开头的斜杠可以省略，因为路径总是以斜杠开头。

	- **&lt;替换&gt;** 是替换字符串。支持使用 `$name` 或 `${name}` 语法引用捕获组，或使用数字索引如 `$1`。详情参见 [Go 文档](https://golang.org/pkg/regexp/#Regexp.Expand)。如果替换值为 `""`，则匹配的文本会被移除。

	- **&lt;限制&gt;** 是可选的替换次数上限。

- **path_regexp** 对 URI 的路径部分执行正则表达式替换。

	- **&lt;目标&gt;** 是前缀、后缀或搜索字符串/正则表达式。如果是前缀，开头的斜杠可以省略，因为路径总是以斜杠开头。

	- **&lt;替换&gt;** 是替换字符串。支持使用 `$name` 或 `${name}` 语法引用捕获组，或使用数字索引如 `$1`。详情参见 [Go 文档](https://golang.org/pkg/regexp/#Regexp.Expand)。如果替换值为 `""`，则匹配的文本会被移除。

- **query** 对 URI 查询执行操作，模式取决于参数名的前缀或参数数量。可以使用块语法一次性指定多个操作，它们按以下顺序分组执行：重命名 🡒 设置 🡒 追加 🡒 替换 🡒 删除。

	- 无前缀时，使用给定值在查询中设置参数。
	
	  例如，`uri query foo bar` 会将 `foo` 参数的值设置为 `bar`。

	- 使用 `-` 作为前缀，从查询中移除该参数。
	
	  例如，`uri query -foo` 会从查询中删除 `foo` 参数。

	- 使用 `+` 作为前缀，将参数及给定值追加到查询中。这 _不会_ 覆盖已有的同名参数（省略 `+` 可覆盖）。
	
	  例如，`uri query +foo bar` 会将 `foo=bar` 追加到查询中。

	- 在参数名中使用 `>` 作为中缀，可以将参数重命名为 `>` 后的值。
	
	  例如，`uri query foo>bar` 会将 `foo` 参数重命名为 `bar`。

	- 使用三个参数时，执行查询值的正则替换：第一个参数是查询参数名，第二个是搜索值，第三个是替换值。第一个参数（参数名）可为 `*`，表示对所有查询参数执行替换。
	
	  支持使用 `$name` 或 `${name}` 语法引用捕获组，或使用数字索引如 `$1`。详情参见 [Go 文档](https://golang.org/pkg/regexp/#Regexp.Expand)。如果替换值为 `""`，则匹配的文本会被移除。
	
	  例如，`uri query foo ^(ba)r $1z` 会替换 `foo` 参数的值，若值以 `bar` 开头，则结果变为 `baz`。

URI 的变更发生在归一化或未转义形式的 URI 上。但前缀或后缀模式中可以使用转义序列，仅匹配请求路径中对应位置的字面转义字符。例如，`uri strip_prefix /a/b` 会将 `/a/b/c` 和 `/a%2Fb/c` 都重写为 `/c`；而 `uri strip_prefix /a%2Fb` 会将 `/a%2Fb/c` 重写为 `/c`，但不会匹配 `/a/b/c`。

URI 路径会在修改前清理掉目录遍历的点号。此外，除非 `<目标>` 也包含多个斜杠，否则多个斜杠（如 `//`）会被合并。

## 类似指令

其他一些指令也可以操作请求的 URI。

- [`rewrite`](rewrite) 会将整个路径和查询更改为一个新值，而不是部分修改。
- [`handle_path`](handle_path) 与 [`handle`](handle) 相同，但在运行处理器之前会从请求中去掉一个前缀。在许多情况下，可用它代替 `uri strip_prefix` 来减少一行配置。

## 示例

从所有请求路径的开头去除 `/api`：

```caddy-d
uri strip_prefix /api
```

从所有请求路径的末尾去除 `.php`：

```caddy-d
uri strip_suffix .php
```

在任何请求 URI 中将 "/docs/" 替换为 "/v1/docs/"：

```caddy-d
uri replace /docs/ /v1/docs/
```

将请求路径（而非请求查询）中所有连续的斜杠折叠为一个斜杠：

```caddy-d
uri path_regexp /{2,} /
```

将 `foo` 查询参数的值设置为 `bar`：

```caddy-d
uri query foo bar
```

从查询中移除 `foo` 参数：

```caddy-d
uri query -foo
```

将 `foo` 查询参数重命名为 `bar`：

```caddy-d
uri query foo>bar
```

将 `bar` 参数追加到查询：

```caddy-d
uri query +foo bar
```

替换 `foo` 查询参数的值，若值以 `bar` 开头则替换为 `baz`：

```caddy-d
uri query foo ^(ba)r $1z
```

一次执行多个查询操作：

```caddy-d
uri query {
	+foo bar
	-baz
	qux test
	renamethis>renamed
}