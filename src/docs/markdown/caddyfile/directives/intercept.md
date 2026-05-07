---
title: intercept（Caddyfile 指令）
---

<script>
ready(function() {
	// 修正响应匹配器的颜色渲染，
	// 并链接到响应匹配器章节
	$$_('pre.chroma .k').forEach(item => {
		if (item.innerText.includes('@')) {
			let text = item.innerText.replace(/</g, '&lt;').replace(/>/g, '&gt;');
			let url = '#' + item.innerText.replace(/_/g, "-");
			item.classList.add('nd');
			item.classList.remove('k');
			item.innerHTML = `<a href="#response-matcher" style="color: inherit;" title="响应匹配器">${text}</a>`;
		}
	});

	// 响应匹配器
	const nameMatchers = Array.from($$_('pre.chroma .nd')).filter(item => item.innerText.includes('@name'));
	if (nameMatchers.length > 0) {
		const first = nameMatchers[0];
		const span = document.createElement('span');
		span.className = 'nd';
		first.parentNode.insertBefore(span, first);
		span.appendChild(first);
		span.innerHTML = '<a href="/docs/caddyfile/response-matchers" style="color: inherit;">@name</a>';
	}
	
	$$_('pre.chroma .k').forEach(item => {
		if (item.innerText === 'status') {
			item.innerHTML = '<a href="/docs/caddyfile/response-matchers#status" style="color: inherit;">status</a>';
		}
	});
	
	const headerElements = $$_('pre.chroma .k');
	for (let item of headerElements) {
		if (item.innerText.includes('header')) {
			item.innerHTML = '<a href="/docs/caddyfile/response-matchers#header" style="color: inherit;">header</a>';
			break;
		}
	}

	// 如果页面上找到匹配的锚标签，我们将为所有子指令添加链接。
	addLinksToSubdirectives();
});
</script>

# intercept

这是 [`reverse_proxy` 指令](reverse_proxy) 中[响应拦截](reverse_proxy#intercepting-responses)功能的通用抽象。它可以与任何生成响应的处理器一起使用，包括来自插件（如 [FrankenPHP](https://frankenphp.dev/) 的 `php_server`）的处理器。

该指令允许您[匹配响应](/docs/caddyfile/response-matchers)，并将调用第一个匹配的 `handle_response` 路由或 `replace_status`。当被调用时，原始响应体会被暂存，从而使该路由有机会写入不同的响应体，并附带新的状态码或任何必要的响应头操作。如果该路由**没有**写入新的响应体，则会写入原始响应体。


## 语法

```caddy-d
intercept [<匹配器>] {
	@name {
		status <状态码...>
		header <字段> [<值>]
	}

	replace_status [<响应匹配器>] <状态码>

	handle_response [<响应匹配器>] {
		<指令...>
	}
}
```

- **@name** 是一个命名的[响应匹配器](/docs/caddyfile/response-matchers)块。只要每个响应匹配器拥有唯一名称，就可以定义多个匹配器。可以根据状态码以及响应头的存在或值来匹配响应。

- **replace_status** <span id="replace_status"/> 在匹配给定匹配器时，简单地将响应的状态码更改为指定的值。

- **handle_response** <span id="handle_response"/> 定义当原始响应被给定响应匹配器匹配时执行的路由。如果省略匹配器，则所有响应都会被拦截。当定义了多个 `handle_response` 块时，将应用第一个匹配的块。在块内部，可以使用所有其他[指令](/docs/caddyfile/directives)。

在 `handle_response` 路由中，可以使用以下占位符来获取原始响应中的信息：

- `{resp.status_code}` 原始响应的状态码。

- `{resp.header.*}` 原始响应中的头部。


## 示例

当使用 [FrankenPHP](https://frankenphp.dev/) 的 `php_server` 时，您可以使用 `intercept` 来实现 `X-Accel-Redirect` 支持，按 PHP 应用请求提供静态文件：

```caddy
localhost {
	root /srv

	intercept {
		@accel header X-Accel-Redirect *
		handle_response @accel {
			root /path/to/private/files
			rewrite {resp.header.X-Accel-Redirect}
			method GET
			file_server
		}
	}

	php_server
}