---
title: push（Caddyfile 指令）
---

# push

配置服务器使用 HTTP/2 服务器推送功能，预先向客户端发送资源。

可以通过指定响应的 Link 头来链接用于服务器推送的资源。该指令会自动推送上游 Link 头中描述的资源，支持的格式如下：

- `<resource>; as=script`
- `<resource>; as=script,<resource>; as=style`
- `<resource>; nopush`
- `<resource>;<resource2>;...`

其中 `<resource>` 以斜杠 `/` 开头（即与相同主机的 URI 路径）。只有同主机资源才能被推送。如果链接的资源是外部资源或带有 `nopush` 属性，则不会被推送。

默认情况下，推送请求会包含一些被认为可以安全地从原始请求中复制的头部：

- Accept-Encoding
- Accept-Language
- Accept
- Cache-Control
- User-Agent

因为如果没有这些头部，许多请求可能会失败；这些不需要手动配置。

推送请求在内部是虚拟化的，因此非常轻量。

## 语法

```caddy-d
push [<matcher>] [<resource>] {
	[GET|HEAD] <resource>
	headers {
		[+]<field> [<value|regexp> [<replacement>]]
		-<field>
	}
}
```

- **&lt;resource&gt;** 是要推送的目标 URI 路径。如果在块内使用，可以选择在前面加上方法（GET 或 POST；默认为 GET）。
- **&lt;headers&gt;** 使用与 [`header` 指令](/docs/caddyfile/directives/header) 相同的语法来操作推送请求的头部。某些头部默认已继承，无需显式配置（见上文）。

## 示例

推送响应中由 `Link` 头描述的所有资源：

```caddy-d
push
```

相同，但同时对所有请求推送 `/resources/style.css`：

```caddy-d
push * /resources/style.css
```

仅在客户端请求 `/foo.html` 时推送 `/foo.jpg`：

```caddy-d
push /foo.html /foo.jpg