---
title: handle (Caddyfile 指令)
---

# handle

在同一嵌套层级中，与其他 `handle` 块互斥地评估一组指令。

换句话说，当多个 `handle` 指令依次出现时，只有第一个 _匹配_ 的 `handle` 块会被评估。没有匹配器的 handle 块充当 _回退_ 路由。

`handle` 指令按其匹配器根据[指令排序算法](/docs/caddyfile/directives#sorting-algorithm)进行排序。`handle_path` 指令是一个特例，其排序优先级与带有路径匹配器的 `handle` 相同。

Handle 块可以根据需要嵌套。Handle 块内只能使用 HTTP 处理指令。

## 语法

```caddy-d
handle [<matcher>] {
	<directives...>
}
```

- **<directives...>** 是一系列 HTTP 处理指令或指令块，每行一个，就像在 handle 块外部使用的那样。

## 相似指令

还有其他指令可以包装 HTTP 处理指令，但每个指令的用途取决于您想要表达的行为：

- [`handle_path`](handle_path) 与 `handle` 功能相同，但在运行其处理器之前会从请求中去除前缀。

- [`handle_errors`](handle_errors) 类似于 `handle`，但仅在 Caddy 处理请求时遇到错误时才会被调用。

- [`route`](route) 像 `handle` 一样包装其他指令，但有两个区别：
  1. route 块之间不互斥，
  2. route 内的指令不会[重新排序](/docs/caddyfile/directives#directive-order)，如果需要，您可以获得更多控制权。

## 示例

使用静态文件服务器处理对 `/foo/` 的请求，并使用反向代理处理其他请求：

```caddy
example.com {
	handle /foo/* {
		file_server
	}

	handle {
		reverse_proxy 127.0.0.1:8080
	}
}
```

您可以在同一站点中混用 `handle` 和 `handle_path`，它们仍然会互斥：

```caddy
example.com {
	handle_path /foo/* {
		# 路径已被去除 "/foo" 前缀
	}

	handle /bar/* {
		# 路径仍然保留 "/bar"
	}
}
```

您可以嵌套 `handle` 块以创建更复杂的路由逻辑：

```caddy
example.com {
	handle /foo* {
		handle /foo/bar* {
			# 此块仅匹配 /foo/bar 下的路径
		}

		handle {
			# 此块匹配 /foo/ 下的所有其他路径
		}
	}

	handle {
		# 此块匹配所有其他内容（充当回退）
	}
}