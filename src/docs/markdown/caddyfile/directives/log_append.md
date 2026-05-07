---
title: log_append (Caddyfile 指令)
---

# log_append

向当前请求的访问日志中追加一个字段。

此指令应与 [`log` 指令](log) 一同使用，后者是启用访问日志的前提。

值可以是静态字符串，也可以是[占位符](/docs/caddyfile/concepts#placeholders)，后者将在请求时被替换为占位符的值。

## 语法

```caddy-d
log_append [<匹配器>] [<]<键> <值>
```

默认情况下，日志字段在中间件链返回途中（即“后期”）添加，在所有后续处理器完成之后（例如在 [`reverse_proxy`](reverse_proxy)、[`respond`](respond) 或 [`file_server`](file_server) 等写入响应的处理器之后），因此它捕获了请求和响应的最终状态。

如果键的前缀为 `<`，则标记为“早期”，这意味着日志字段将在调用链中的下一个处理器之前添加到日志中，以便在后续处理器修改请求之前读取请求。

仅用于调试目的（不适用于生产环境），当值是以下占位符之一时，该处理器有专门的处理方式：`{http.request.body}`、`{http.request.body_base64}`、`{http.response.body}` 或 `{http.response.body_base64}`。如果使用了请求体占位符，则会隐式启用“早期”模式，并缓冲请求体。如果使用了响应体占位符，则会启用响应缓冲以捕获响应体，并在写入响应时“后期”将该字段添加到日志中。

## 示例

在日志中显示请求服务的站点区域（`static` 或 `dynamic`）：

```caddy
example.com {
	log

	handle /static* {
		log_append area "static"
		respond "Static response!"
	}

	handle {
		log_append area "dynamic"
		reverse_proxy localhost:9000
	}
}
```

在日志中显示实际使用的反向代理上游（`node1`、`node2` 或 `node3`）、代理到上游所花费的毫秒数以及上游写入响应头所花费的时间：

```caddy
example.com {
	log

	handle {
		reverse_proxy node1:80 node2:80 node3:80 {
			lb_policy random_choose 2 
		}
		log_append upstream_host {rp.upstream.host}
		log_append upstream_duration_ms {rp.upstream.duration_ms}
		log_append upstream_latency_ms {rp.upstream.latency_ms}
	}
}
```

通过在键前加上 `<`，可以在“早期”向日志添加一个字段。这允许你在后续处理器修改请求之前捕获请求的状态。例如，记录重写之前的原始请求路径（尽管这是一个刻意设计的例子，因为原始请求路径本身已经会被记录，但有助于说明这一点）：

```caddy
example.com {
	log
	log_append <original_path {http.request.uri.path}
	rewrite /new-base{uri}
	reverse_proxy localhost:9000
}
```

用于调试目的，将请求体和响应体添加到日志中（不适用于生产环境，因为这会影响性能并导致日志非常嘈杂）。如果你预期主体是包含不可打印字符的二进制数据，可以改用占位符的 base64 变体（例如 `{http.request.body_base64}` 和 `{http.response.body_base64}`），这将更易于复制和检查：

```caddy
example.com {
	log
	log_append req_body {http.request.body}
	log_append resp_body {http.response.body}

	reverse_proxy localhost:9000
}