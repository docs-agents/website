---
title: route（Caddyfile 指令）
---

# route

按字面顺序作为一个整体来评估一组指令。

位于 `route` 块内的指令将**不会**被[内部重排](/docs/caddyfile/directives#directive-order)。只有 HTTP 处理程序指令（即向链中添加处理程序或中间件的指令）可以在 `route` 块中使用。

该指令是一个特例，因为它的子指令也是常规指令。

## 语法

```caddy-d
route [<matcher>] {
	<directives...>
}
```

- **<directives...>** 是一系列指令或指令块，每行一个，就像在 `route` 块外部一样；不同之处在于这些指令不会被打乱顺序。只能使用 HTTP 处理程序指令。

## 用途

`route` 指令在某些高级用例或边缘情况下很有帮助，可以使你对 HTTP 处理程序链的某些部分拥有绝对控制权。

由于 HTTP 中间件的评估顺序很重要，Caddyfile 通常会在解析后对指令进行重排，以简化使用；你无需担心输入的顺序。

虽然[内置顺序](/docs/caddyfile/directives#directive-order)适用于大多数站点，但有时你可能需要手动控制整个站点或其中一部分的顺序。这正是 `route` 指令的用途。

为了说明这一点，考虑两个终止型处理程序：[`redir`](redir) 和 [`file_server`](file_server)。两者都会向客户端写入响应，并且不会调用链中的下一个处理程序，因此对于某个请求，只会执行其中一个。那么谁先执行？通常情况下，`redir` 先于 `file_server` 执行，因为通常你只希望在特定情况下发出重定向，而在一般情况下提供文件服务。

然而，有时第一个指令（`file_server`）可能比第二个指令（`redir`）具有更具体的匹配器。换句话说，你希望在一般情况下重定向，而只提供特定文件。

因此，你可能会尝试这样写 Caddyfile（但这不会按预期工作！）：

```caddy
example.com {
	file_server /specific.html
	redir https://anothersite.com{uri}
}
```

问题在于，[指令排序](/docs/caddyfile/directives#sorting-algorithm)后，`redir` 会排在 `file_server` 之前。

但在本例中，`redir` 的匹配器（隐式的 [`*`](/docs/caddyfile/matchers#wildcard-matchers)）是 `file_server` 匹配器（`*` 是 `/specific.html` 的超集）的超集。

幸运的是，解决方案很简单：只需将这两个指令包裹在一个 `route` 块中，以确保 `file_server` 在 `redir` 之前执行：

```caddy
example.com {
	route {
		file_server /specific.html
		redir https://anothersite.com{uri}
	}
}
```

<aside class="tip">

另一种方法是使两个匹配器互斥，但如果存在多个条件，这很快就会变得复杂。使用 `route` 指令，两个处理程序的互斥性是隐式的，因为它们都是终止型处理程序。

</aside>

现在 `file_server` 将先于 `redir` 被链接到链中，因为顺序是逐字逐句采用的。

## 类似指令

还有其他指令可以包裹 HTTP 处理程序指令，但每种都有其用途，取决于你想要表达的行为：

- [`handle`](handle) 像 `route` 一样包裹其他指令，但有两个区别：1) `handle` 块之间彼此互斥；2) `handle` 内的指令会正常地[重新排序](/docs/caddyfile/directives#directive-order)。

- [`handle_path`](handle_path) 与 `handle` 相同，但在运行其处理程序之前会从请求中剥离一个前缀。

- [`handle_errors`](handle_errors) 类似于 `handle`，但仅在 Caddy 处理请求时遇到错误时才会被调用。

## 示例

按原样将 `/api` 的请求代理，并根据是否匹配磁盘上的文件重写所有其他请求；如果不匹配，则重写到 `/index.html`，然后提供该文件。

由于 [`try_files`](try_files) 的指令顺序高于 [`reverse_proxy`](reverse_proxy)，正常情况下它会排得更前并且先运行；这将导致所有 API 请求都被重写为 `/index.html`，无法匹配 `/api*`，因此它们都不会被代理，反而会由 [`file_server`](file_server) 返回 `404` 错误。将其全部包裹在 `route` 中可以确保 `reverse_proxy` 总是在请求被重写之前运行。

```caddy
example.com {
	root /srv
	route {
		reverse_proxy /api* localhost:9000

		try_files {path} /index.html
		file_server
	}
}
```

<aside class="tip">

这并不是解决此问题的唯一方法。你也可以使用一对 [`handle`](handle) 块，第一个匹配 `/api*` 并代理到 `reverse_proxy`，第二个作为后备并提供文件。请参阅[SPA 示例](/docs/caddyfile/patterns#single-page-apps-spas)。

</aside>