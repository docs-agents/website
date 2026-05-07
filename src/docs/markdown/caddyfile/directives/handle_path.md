---
title: handle_path (Caddyfile 指令)
---

<script>
ready(function() {
	// Add a link to [<path_matcher>] as a special case for this directive.
	// The matcher text includes <> characters which are parsed as HTML,
	// so we must use text() to change the link text.
	$$_('pre.chroma .s').forEach(item => {
		if (item.innerText.includes('<path_matcher>')) {
			let text = item.innerText.replace(/</g, '&lt;').replace(/>/g, '&gt;');
			item.innerHTML = `<a href="/docs/caddyfile/matchers#path-matchers" style="color: inherit;" title="Matcher token">${text}</a>`;
			item.classList.remove('s');
			item.classList.add('nd');
		}
	});
});
</script>

# handle_path

工作方式与 [`handle` 指令](handle)相同，但会隐式使用 [`uri strip_prefix`](uri) 来去除匹配的路径前缀。

处理匹配特定路径的请求（并从请求URI中去除该路径）是一个常见的用例，因此专门有一个指令来方便使用。

## 语法

```caddy-d
handle_path <path_matcher> {
	<directives...>
}
```

- **<directives...>** 是一系列HTTP处理器指令或指令块，每行一个，就像在 `handle_path` 块外使用一样。

只接受单个[路径匹配器](/docs/caddyfile/matchers#path-matchers)，且为必需；你不能在 `handle_path` 中使用命名匹配器。

## 示例

以下配置：

```caddy-d
handle_path /prefix/* {
	...
}
```

👆 实际上等同于以下 👇，但 `handle_path` 形式 👆 更为简洁：

```caddy-d
handle /prefix/* {
	uri strip_prefix /prefix
	...
}
```

一个完整的 Caddyfile 示例，其中 `handle_path` 和 `handle` 是互斥的；但请注意[子文件夹问题 <img src="/old/resources/images/external-link.svg" class="external-link">](https://caddy.community/t/the-subfolder-problem-or-why-cant-i-reverse-proxy-my-app-into-a-subfolder/8575)：

```caddy
example.com {
	# 提供你的 API 服务，去除 /api 前缀
	handle_path /api/* {
		reverse_proxy localhost:9000
	}

	# 提供你的静态站点
	handle {
		root /srv
		file_server
	}
}