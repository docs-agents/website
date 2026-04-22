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

一个静态文件服务器，支持真实和虚拟文件系统。它通过将请求的 URI 路径附加到 [站点的根路径](root) 来形成文件路径。

默认情况下，它会强制执行规范 URI；这意味着对于没有以斜杠结尾的目录请求（以添加斜杠），或有以斜杠结尾的文件请求（以移除斜杠），将发出 HTTP 重定向。但是，如果内部重写修改了路径的最后一个元素（文件名），则不会发出重定向。

大多数情况下，`file_server` 指令与 [`root`](root) 指令搭配使用，为整个站点设置文件根目录。该指令还有一个 `root` 子指令（见下文），仅为此处理器设置根目录（不推荐）。请注意，站点根目录不带有沙箱保证：文件服务器可以防止来自路径组件的目录遍历，但根目录内的符号链接仍然可以允许访问根目录之外的内容。

当发生错误时（例如文件未找到 `404`、权限被拒 `403`），将调用错误路由。使用 [`handle_errors`](handle_errors) 指令定义错误路由，并显示自定义错误页面。

使用 `browse` 时，输出由 HTML 模板生成。客户端可以使用 `Accept: application/json` 或 `Accept: text/plain` 头将目录列表作为 JSON 或纯文本请求。JSON 输出可用于脚本编写，纯文本输出可用于人工终端使用。


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

- **fs** <span id="fs"/> 指定要使用的替代（或许是虚拟）文件系统。`caddy.fs` 命名空间中的任何 Caddy 模块都可以在这里使用。任何根路径/前缀仍将应用于替代文件系统模块。默认情况下使用本地磁盘。

	[`xcaddy`](/docs/build#xcaddy) v0.4.0 引入了 [`--embed` 标志](https://github.com/caddyserver/xcaddy#custom-builds)，将文件系统树嵌入到自定义 Caddy 构建中，并注册一个名为 `embedded` 的 `fs` 模块，允许您的静态站点作为 Caddy 可执行文件分发。

- **root** <span id="root"/> 设置站点根目录的路径。它与 [`root`](root) 指令类似，但它仅适用于此文件服务器实例，并覆盖可能已定义的任何其他站点根目录。默认：`{http.vars.root}` 或当前工作目录。注意：此子指令仅更改此处理程序的根目录。要让其他指令（如 [`try_files`](try_files) 或 [`templates`](templates)）知道相同的站点根目录，请使用 [`root`](root) 指令。

- **hide** <span id="hide"/> 是要隐藏的文件或文件夹列表；如果请求，文件服务器将假装它们不存在。接受占位符和 glob 模式。请注意，这些是*文件系统*路径，而不是请求路径。换句话说，相对路径以当前工作目录为基准，而不是站点根目录；并且所有路径在比较之前都转换为它们的绝对形式（如果可能）。指定不带路径分隔符的文件名或模式将隐藏所有具有匹配名称的文件，无论其位置如何；否则，将尝试路径前缀匹配，然后是 glob 匹配。由于这是 Caddyfile 配置，活动配置文件将默认添加。隐藏比较是区分大小写的；在不区分大小写的文件系统上，不同大小写的请求路径可能仍会解析为相同的磁盘路径，因此 `hide` 不应被视为敏感路径的安全边界。

- **index** <span id="index"/> 是要作为索引文件查找的文件名列表。默认：`index.html index.txt`

- **browse** <span id="browse"/> 为没有索引文件的目录请求启用文件列表。

  - **<template_file>** <span id="template_file"/> 是用于目录列表的可选自定义模板文件。默认使用可以通过命令 `caddy file-server export-template` 提取的模板，该命令将默认模板打印到 stdout。嵌入模板也可以在此处的[源代码中找到 ![外部链接](/old/resources/images/external-link.svg)](https://github.com/caddyserver/caddy/blob/master/modules/caddyhttp/fileserver/browse.html)。浏览模板可以使用[标准模板模块](/docs/modules/http.handlers.templates#docs)中的操作。

  - **reveal_symlinks** <span id="reveal_symlinks"/> 启用在目录列表中显示符号链接的目标。默认情况下，符号链接目标被隐藏，仅显示链接文件本身。

  - **sort** <span id="sort"/> 更改目录列表的默认排序。第一个参数是要排序的字段/列：`name`、`namedirfirst`、`size` 或 `time`。第二个参数是可选的方向：`asc` 或 `desc`。例如，`sort name desc` 将按名称降序排序。

  - **file_limit** <span id="file_limit"/> 设置目录列表中显示的最大文件数。默认：`10000`。如果文件数超过此限制，将仅显示前 N 个文件，其中 N 是指定限制。

- **precompressed** <span id="precompressed"/> 是要搜索预压缩附属文件的编码格式列表。参数是用于搜索预压缩 [附属文件](https://en.wikipedia.org/wiki/Sidecar_file) 的编码格式有序列表。支持的格式包括 `gzip` (`.gz`)、`zstd` (`.zst`) 和 `br` (`.br`)。如果省略格式，则默认为 `br zstd gzip`（按此顺序）。

	所有文件查找将首先查找未压缩文件的存在。找到后，Caddy 将查找每个启用格式的文件扩展名的附属文件。如果找到预压缩附属文件，Caddy 将响应预压缩文件，并适当设置 `Content-Encoding` 响应头。否则，Caddy 将像往常一样响应未压缩文件。如果启用了 [`encode` 指令](encode)，则如果未预压缩，它可能会即时压缩响应。

- **status** <span id="status"/> 是用于写入响应时的可选状态码覆盖。特别适用于使用 [自定义错误页面](handle_errors) 响应请求。可以是 3 位状态码，例如：`404`。支持占位符。默认情况下，写入的状态码通常为 `200`，或部分内容的 `206`。

- **disable_canonical_uris** <span id="disable_canonical_uris"/> 禁用默认的重定向行为（如果请求路径是目录则添加尾随斜杠，如果请求路径是文件则移除尾随斜杠）。请注意，默认情况下，如果请求路径的最后一个元素（文件名）经过了内部重写，则不会发生规范化，以避免用隐式行为覆盖显式重写。

- **pass_thru** <span id="pass_thru"/> 启用透传模式，如果未找到请求的文件，则继续到路由中的下一个 HTTP 处理器，而不是触发 `404` 错误（调用 [`handle_errors`](handle_errors) 路由）。实际上，这仅在 [`route`](route) 块内与其他跟在 `file_server` 后面的处理器指令一起使用时才有用，因为此指令实际上[排在最后](/docs/caddyfile/directives#directive-order)。


## 示例

从当前目录提供静态文件服务器：

```caddy-d
file_server
```

启用文件列表：

```caddy-d
file_server browse
```

仅在 `/static` 文件夹内提供静态文件：

```caddy-d
file_server /static/*
```

`file_server` 指令通常与 [`root` 指令](root) 搭配使用，以设置提供文件的根路径：

```caddy
example.com {
	root /srv
	file_server
}
```

<aside class="tip">

如果您将 Caddy 作为 systemd 服务运行，从 `/home` 读取文件将无法工作，因为 `caddy` 用户在 `/home` 目录上没有"可执行"权限（遍历所必需）。建议将文件放在 `/srv` 或 `/var/www/html` 中。

</aside>


隐藏所有 `.git` 文件夹及其内容：

```caddy-d
file_server {
	hide .git
}
```

如果客户端支持（`Accept-Encoding` 头），检查请求文件旁边是否存在预压缩文件。因此如果请求 `/path/to/file`，它将按顺序检查 `/path/to/file.br`、`/path/to/file.zst` 和 `/path/to/file.gz`，并提供第一个可用的文件及其对应的 `Content-Encoding`：

```caddy-d
file_server {
	precompressed
}
```
