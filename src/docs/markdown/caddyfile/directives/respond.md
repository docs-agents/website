---
title: respond (Caddyfile 指令)
---

# respond

向客户端写入一个硬编码/静态的响应。

如果响应体非空，该指令会在未设置 `Content-Type` 头的情况下自动设置。默认值为 `text/plain; utf-8`，除非响应体是有效的 JSON 对象或数组，在这种情况下会设置为 `application/json`。对于其他类型的内容，请使用 [`header` 指令](/docs/caddyfile/directives/header) 显式设置正确的 Content-Type。

## 语法

```caddy-d
respond [<matcher>] <status>|<body> [<status>] {
	body <text>
	close
}
```

- **&lt;status&gt;** 是要写入的 HTTP 状态码。

  如果是 `103`（Early Hints），响应将不带响应体写入，并且处理链将继续。（HTTP `1xx` 响应是信息性的，不是最终的。）
  
  默认值：`200`

- **&lt;body&gt;** 是要写入的响应体。

- **body** 是提供响应体的另一种方式；适用于多行文本。

- **close** 会在写入响应后关闭客户端到服务器的连接。

说明：第一个非匹配器参数可以是 3 位状态码或响应体字符串。如果是响应体，下一个参数可以是状态码。

<aside class="tip">

使用错误状态码响应与在处理链中返回错误不同，后者会内部调用错误处理器。

</aside>

## 示例

对所有健康检查写入一个空体的 200 状态，对所有其他请求写入一个简单的响应体：

```caddy
example.com {
	respond /health-check 200
	respond "Hello, world!"
}
```

写入一个错误响应并关闭连接：

<aside class="tip">

你可能更倾向于使用 [`error` 指令](error)，它会触发一个错误，该错误可以通过 [`handle_errors` 指令](handle_errors) 处理。

</aside>

```caddy
example.com {
	respond /secret/* "Access denied" 403 {
		close
	}
}
```

写入一个 HTML 响应，使用 [heredoc 语法](/docs/caddyfile/concepts#heredocs) 控制空白，并设置 `Content-Type` 头以匹配响应体：

```caddy
example.com {
	header Content-Type text/html
	respond <<HTML
		<html>
			<head><title>Foo</title></head>
			<body>Foo</body>
		</html>
		HTML 200
}