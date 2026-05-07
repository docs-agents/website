---
title: error (Caddyfile 指令)
---

# error

在 HTTP 处理器链中触发一个错误，可附带可选的消息和推荐的 HTTP 状态码。

此处理器不会写入响应。相反，它旨在与 [`handle_errors`](handle_errors) 指令配合使用，以调用自定义的错误处理逻辑。

## 语法

```caddy-d
error [<matcher>] <status>|<message> [<status>] {
    message <text>
}
```

- **&lt;status&gt;** 是要写入的 HTTP 状态码。默认为 `500`。
- **&lt;message&gt;** 是错误消息。默认为无错误消息。
- **message** 是提供错误消息的另一种方式；当消息跨越多行时很方便。

需要明确的是，第一个非匹配器的参数可以是 3 位数的状态码，也可以是错误消息字符串。如果是错误消息，则下一个参数可以是状态码。

## 示例

在特定请求路径上触发错误，并使用 [`handle_errors`](handle_errors) 写入响应：

```caddy
example.com {
	root /srv

	# 为特定路径触发错误
    error /private* "Unauthorized" 403
	error /hidden* "Not found" 404

    # 通过提供 HTML 页面来处理错误
    handle_errors {
        rewrite /{err.status_code}.html
		file_server
    }

	file_server
}