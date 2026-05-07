---
title: handle_errors (Caddyfile 指令)
---

# handle_errors

设置错误处理程序。

当普通的 HTTP 请求处理器返回错误时，正常处理停止，并调用错误处理程序。错误处理程序形成一个类似普通路由的路由，并且可以执行普通路由能做的任何操作。这使您在 HTTP 请求期间处理错误时拥有极大的控制力和灵活性。例如，您可以提供静态错误页面、模板化错误页面，或将反向代理到另一个后端来处理错误。

该指令可以通过不同的状态码重复使用，以不同方式处理不同的错误。如果未指定状态码，则将匹配任何错误，并在没有其他错误处理程序匹配时作为后备。

请求的上下文会携带到错误路由中，因此在请求上下文上设置的任何值（例如[站点根目录](root)或[变量](vars)）也会保留在错误处理程序中。此外，在处理错误时，还可以使用[新的占位符](#placeholders)。

请注意，某些指令（例如 [`reverse_proxy`](reverse_proxy)）可能会写入一个被归类为错误的 HTTP 状态码的响应，但*不会*触发错误路由。

您可以使用 [`error`](error) 指令根据您自己的路由决策显式触发错误。

## 语法

```caddy-d
handle_errors [<status_codes...>] {
	<directives...>
}
```

- **<status_codes...>** 是一个或多个 HTTP 状态码，用于匹配正在处理的错误。状态码可以是三位数数字，也可以是对应匹配 400-499 或 500-599 范围内所有状态码的特殊情况 `4xx` 或 `5xx`。如果未指定状态码，则将匹配任何错误，并在没有其他错误处理程序匹配时作为后备。

- **<directives...>** 是一系列 HTTP 处理程序[指令](/docs/caddyfile/directives)和[匹配器](/docs/caddyfile/matchers)，每行一个。

## 占位符

处理错误时，可以使用以下占位符。它们是完整占位符的 [Caddyfile 简写](/docs/caddyfile/concepts#placeholders)，完整占位符可在 [HTTP 服务器错误路由的 JSON 文档](/docs/json/apps/http/servers/errors/#routes)中找到。

| 占位符              | 描述                     |
|---------------------|--------------------------|
| `{err.status_code}` | 推荐的 HTTP 状态码       |
| `{err.status_text}` | 与推荐状态码关联的状态文本 |
| `{err.message}`     | 错误消息                 |
| `{err.trace}`       | 错误的来源               |
| `{err.id}`          | 此错误实例的标识符       |

## 示例

基于状态码的自定义错误页面（例如，对于 `404` 错误，使用名为 `404.html` 的页面）。请注意，[`file_server`](file_server) 在 `handle_errors` 中运行时，会保留错误的 HTTP 状态码（假设您事先在站点中设置了[站点根目录](root)）：

```caddy-d
handle_errors {
	rewrite /{err.status_code}.html
	file_server
}
```

一个使用 [`templates`](templates) 来编写自定义错误消息的单一错误页面：

```caddy-d
handle_errors {
	rewrite /error.html
	templates
	file_server
}
```

如果您只想为某些错误代码提供自定义错误页面，可以使用 [`file`](/docs/caddyfile/matchers#file) 匹配器预先检查自定义错误文件是否存在：

```caddy-d
handle_errors {
	@custom_err file /err-{err.status_code}.html /err.html
	handle @custom_err {
		rewrite {file_match.relative}
		file_server
	}
	respond "{err.status_code} {err.status_text}"
}
```

反向代理到能够专业地处理 HTTP 错误并改善您的一天 😸 的专业服务器：

```caddy-d
handle_errors {
	rewrite /{err.status_code}
	reverse_proxy https://http.cat {
		replace_status {err.status_code}
	}
}
```

只需使用 [`respond`](respond) 返回错误代码和名称：

```caddy-d
handle_errors {
	respond "{err.status_code} {err.status_text}"
}
```

针对特定错误代码进行不同处理：

```caddy-d
handle_errors 404 410 {
	respond "It's a 404 or 410 error!"
}

handle_errors 5xx {
	respond "It's a 5xx error."
}

handle_errors {
	respond "It's another error"
}
```

上述行为与下面的示例相同，下面的示例使用 [`expression`](/docs/caddyfile/matchers#expression) 匹配器对状态码进行匹配，并使用 [`handle`](handle) 实现互斥：

```caddy-d
handle_errors {
	@404-410 `{err.status_code} in [404, 410]`
	handle @404-410 {
		respond "It's a 404 or 410 error!"
	}

	@5xx `{err.status_code} >= 500 && {err.status_code} < 600`
	handle @5xx {
		respond "It's a 5xx error."
	}

	handle {
		respond "It's another error"
	}
}