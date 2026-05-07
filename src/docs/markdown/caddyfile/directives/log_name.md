---
title: log_name (Caddyfile 指令)
---

# log_name

覆盖在使用 [`log` 指令](log) 写入访问日志时为请求使用的日志器名称。

当你希望根据某些条件（例如请求路径或方法）将请求记录到不同文件时，此指令非常有用。

可以指定多个日志器名称，这样请求的日志会被推送到多个匹配的日志器。

该指令通常与 `log` 指令的 [`no_hostname`](log#no_hostname) 选项配合使用，该选项可防止日志器与站点块的任何主机名关联，从而只有设置了 `log_name` 的请求才会将日志推送到该日志器。

## 语法

```caddy-d
log_name [<匹配器>] <名称...>
```

## 示例

你可能希望将请求记录到不同文件，例如，你可能希望将健康检查日志与主访问日志分开记录。

在 `log` 中使用 `no_hostname` 可防止日志器与站点块的任何主机名关联（即此处的 `localhost`），从而只有将 `log_name` 设置为该日志器名称的请求才会收到日志。

```caddy
localhost {
	log {
		output file ./caddy.access.log
	}

	log health_check_log {
		output file ./caddy.access.health.log
		no_hostname
	}

	handle /healthz* {
		log_name health_check_log
		respond "Healthy"
	}

	handle {
		respond "Hello World"
	}
}