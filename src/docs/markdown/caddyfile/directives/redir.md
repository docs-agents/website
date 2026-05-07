---
title: redir (Caddyfile 指令)
---

# redir

向客户端发出 HTTP 重定向。

此指令意味着匹配到的请求将被原样拒绝，客户端应尝试另一个 URL。因此，它的[指令顺序](/docs/caddyfile/directives#directive-order)非常靠前。


## 语法

```caddy-d
redir [<matcher>] <to> [<code>]
```

- **&lt;to&gt;** 是目标位置。将成为响应的 [`Location` 头 <img src="/old/resources/images/external-link.svg" class="external-link">](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Location)。

- **&lt;code&gt;** 是用于重定向的 HTTP 状态码。可以是：

	- `3xx` 范围内的正整数，或 `401`
	
	- `temporary` 表示临时重定向（`302`，此为默认值）
	
	- `permanent` 表示永久重定向（`301`）
	
	- `html` 使用 HTML 文档执行重定向（对浏览器有效，但对 API 客户端无效）
	
	- 包含状态码值的占位符



## 示例

将所有请求重定向到 `https://example.com`：

```caddy
www.example.com {
	redir https://example.com
}
```

同样，但通过附加 [`{uri}` 占位符](/docs/caddyfile/concepts#placeholders) 保留原始 URI：

```caddy
www.example.com {
	redir https://example.com{uri}
}
```

同样，但改为永久重定向：

```caddy
www.example.com {
	redir https://example.com{uri} permanent
}
```

将旧的 `/about-us` 页面重定向到新的 `/about` 页面：

```caddy
example.com {
	redir /about-us /about
	reverse_proxy localhost:9000
}