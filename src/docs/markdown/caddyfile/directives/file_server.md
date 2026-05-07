---
title: file_server (Caddyfile 指令)
---

<script>
ready(function() {
	// Fix inline browse arg
	for (let item of $$_('pre.chroma .s')) {
		if (item.innerText.includes('browse')) {
			const span = document.createElement('span');
			span.className = 'k';
			item.parentNode.insertBefore(span, item);
			span.appendChild(item);
			span.innerHTML = '<a href="#browse" style="color: inherit;" title="browse">browse</a>';
			break;
		}
	}

	// We'll add links to all the subdirectives if a matching anchor tag is found on the page.
	addLinksToSubdirectives();
});
</script>

# file_server

一个静态文件服务器，支持真实和虚拟文件系统。它通过将请求的 URI 路径附加到[站点根路径](root)来构建文件路径。

默认情况下，它强制执行规范化 URI；这意味着对于不以尾部斜杠结尾的目录请求（以添加斜杠），或对于具有尾部斜杠的文件请求（以移除斜杠），会发出 HTTP 重定向。但是，如果内部重写修改了路径的最后一个元素（文件名），则不会发出重定向。

通常，`file_server` 指令与 [`root`](root) 指令配对使用，为整个站点设置文件根目录。该指令也有一个 `root` 子指令（见下文）仅为该处理器设置根目录（不推荐）。注意，站点根目录不提供沙箱保证：文件服务器确实可以防止路径遍历中的目录跳转，但根目录内的符号链接仍可能允许访问根目录之外的内容。

发生错误时（例如文件未找到 `404`、权限被拒绝 `403`），将调用错误路由。使用 [`handle_errors`](handle_errors) 指令定义错误路由并显示自定义错误页面。

使用 `browse` 时，默认输出由 HTML 模板生成。客户端可以通过分别使用 `Accept: application/json` 或 `Accept: text/plain` 头部，请求目录列表为 JSON 或纯文本格式。JSON 输出适用于脚本编程，纯文本输出则适合人类终端使用。


## 语法

```caddy-d
file_server [<matcher>] [browse] {
	fs            <backend...>
	root          <path>
	hide          <files...>
	index         <filenames...>
	browse        [<template_file>] {
		reveal_symlinks
		sort <sort_field> [<direction>]
		file_limit <number>
	}
	precompressed [<formats...>]
	status        <status>
	disable_canonical_uris
	pass_thru
}
```

- **fs** <span id="fs"/> 指定一个替代（可能是虚拟的）文件系统。可以使用 `caddy.fs` 命名空间中的任何 Caddy 模块。任何根路径/前缀仍将应用于替代文件系统模块。默认使用本地磁盘。

	[`xcaddy`](/docs/build#xcaddy) v0.4.0 引入了 [`--embed` 标志](https://github.com/caddyserver/xcaddy#custom-builds) 将文件系统树嵌入到自定义 Caddy 构建中，并注册了一个名为 `embedded` 的 `fs` 模块，允许您的静态站点以 Caddy 可执行文件的形式分发。

- **root** <span id="root"/> 设置站点根目录的路径。它类似于 [`root`](root) 指令，但仅适用于此文件服务器实例，并覆盖可能已定义的任何其他站点根目录。默认值：`{http.vars.root}` 或当前工作目录。注意：此子指令仅更改此处理器的根目录。为了使其他指令（如 [`try_files`](try_files) 或 [`templates`](templates)）知道相同的站点根目录，请改用 [`root`](root) 指令。

- **hide** <span id="hide"/> 是要隐藏的文件或文件夹列表；如果请求这些文件或文件夹，文件服务器将假装它们不存在。接受占位符和 glob 模式。注意，这些是_文件系统_路径，而非请求路径。换句话说，相对路径使用当前工作目录作为基础，而非站点根目录；所有路径在比较之前（如果可能）都会转换为绝对形式。指定不带路径分隔符的文件名或模式将隐藏所有匹配该名称的文件，无论其位置如何；否则，将尝试路径前缀匹配，然后进行 glob 匹配。由于这是 Caddyfile 配置，默认会添加活动的配置文件。隐藏比较区分大小写；在不区分大小写的文件系统上，大小写不同的请求路径可能仍解析到同一磁盘路径，因此对于敏感路径，不应将 `hide` 视为安全边界。

- **index** <span id="index"/> 是要作为索引文件查找的文件名列表。默认值：`index.html index.txt`

- **browse** <span id="browse"/> 为没有索引文件的目录请求启用文件列表。

  - **<template_file>** <span id="template_file"/> 是用于目录列表的可选自定义模板文件。默认可通过命令 `caddy file-server export-template` 提取的模板，该命令会将默认模板输出到 stdout。嵌入的模板也可以在[源代码中此处找到 ![external link](/old/resources/images/external-link.svg)](https://github.com/caddyserver/caddy/blob/master/modules/caddyhttp/fileserver/browse.html)。浏览模板也可以使用[标准模板模块](/docs/modules/http.handlers.templates#docs)中的操作。

  - **reveal_symlinks** <span id="reveal_symlinks"/> 允许在目录列表中显示符号链接的目标。默认情况下，隐藏符号链接目标，仅显示链接文件本身。

  - **sort** <span id="sort"/> 更改目录列表的默认排序方式。第一个参数是要排序的字段/列：`name`、`namedirfirst`、`size` 或 `time`。第二个参数是可选的方向：`asc` 或 `desc`。例如，`sort name desc` 将按名称降序排序。

  - **file_limit** <span id="file_limit"/> 设置目录列表中显示的最大文件数。默认值：`10000`。如果文件数超过此限制，则只显示前 N 个文件，其中 N 是指定的限制值。

- **precompressed** <span id="precompressed"/> 是要搜索预压缩 sidecar 文件的编码格式列表。参数是一个有序列表，指定要搜索的预压缩 [sidecar 文件](https://en.wikipedia.org/wiki/Sidecar_file) 的编码格式。支持的格式有 `gzip` (`.gz`)、`zstd` (`.zst`) 和 `br` (`.br`)。如果省略格式，则默认使用 `br zstd gzip`（按此顺序）。

  所有文件查找首先会查找未压缩文件是否存在。一旦找到，Caddy 将查找带有每个已启用格式文件扩展名的 sidecar 文件。如果找到预压缩的 sidecar 文件，Caddy 将使用该预压缩文件响应，并适当设置 `Content-Encoding` 响应头。否则，Caddy 将正常返回未压缩文件。如果启用了 [`encode` 指令](encode)，那么如果未预压缩，它可能会即时压缩响应。

- **status** <span id="status"/> 是一个可选的响应状态码覆盖值，用于写入响应。在响应请求时，例如使用[自定义错误页面](handle_errors)时特别有用。可以是 3 位状态码，例如：`404`。支持占位符。默认情况下，写入的状态码通常为 `200`，对于部分内容则为 `206`。

- **disable_canonical_uris** <span id="disable_canonical_uris"/> 禁用默认的重定向行为（如果请求路径是目录则添加尾部斜杠，如果请求路径是文件则移除尾部斜杠）。请注意，默认情况下，如果请求路径的最后一个元素（文件名）经过了内部重写，则不会执行规范化，以避免隐式行为覆盖显式重写。

- **pass_thru** <span id="pass_thru"/> 启用直通模式，如果请求的文件未找到，则继续执行路由中的下一个 HTTP 处理器，而不是触发 `404` 错误（调用 [`handle_errors`](handle_errors) 路由）。实际上，这仅在 [`route`](route) 块内与其他在 `file_server` 之后的处理器指令一起使用时才有用，因为该指令在[顺序上处于最后](/docs/caddyfile/directives#directive-order)。


## 示例

从当前目录提供静态文件：

```caddy-d
file_server
```

启用文件列表：

```caddy-d
file_server browse
```

仅提供 `/static` 文件夹内的静态文件：

```caddy-d
file_server /static/*
```

`file_server` 指令通常与 [`root` 指令](root) 配对使用，以设置提供文件的根路径：

```caddy
example.com {
	root /srv
	file_server
}
```

<aside class="tip">

如果您将 Caddy 作为 systemd 服务运行，则无法读取 `/home` 下的文件，因为 `caddy` 用户没有 `/home` 目录的“可执行”权限（遍历所需）。建议将文件放置在 `/srv` 或 `/var/www/html` 中。

</aside>


隐藏所有 `.git` 文件夹及其内容：

```caddy-d
file_server {
	hide .git
}
```

如果客户端支持（通过 `Accept-Encoding` 头部），检查请求文件旁边是否存在预压缩文件。因此，如果请求 `/path/to/file`，它会按顺序检查 `/path/to/file.br`、`/path/to/file.zst` 和 `/path/to/file.gz`，并返回第一个可用文件，同时设置相应的 `Content-Encoding`：

```caddy-d
file_server {
	precompressed
}