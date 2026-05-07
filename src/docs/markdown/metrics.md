---
title: 使用 Prometheus 指标监控 Caddy
---

# 使用 Prometheus 指标监控 Caddy

无论你是在云端运行数千个 Caddy 实例，还是在嵌入式设备上运行单个 Caddy 服务器，你很可能会在某个时刻想要了解 Caddy 正在做什么、花费了多长时间。换句话说，你将希望能够**监控** Caddy。

## 启用指标

你需要开启指标功能。

如果使用 Caddyfile，请在[全局选项](/docs/caddyfile/options#metrics)中启用指标：

```caddy
{
	metrics
}
```

如果使用 JSON，请在你的 [`apps > http > servers` 配置](/docs/json/apps/http/servers/)中添加 `"metrics": {}`。

要添加按主机统计的指标，可以插入 `per_host` 选项。主机特定的指标现在将带有 Host 标签。

```caddy
{
	metrics {
		per_host
	}
}
```

此配置将会观测已配置的主机。如果配置了 HTTPS 服务器，即使没有显式配置（例如按需 TLS 设置），该主机也会被观测。如果禁用了 HTTPS，则由于存在无限的基数风险，只启用已配置的主机。要观测 HTTP 设置中的所有主机，包括未配置的，请使用 `observe_catchall_hosts` 选项。

```caddy
{
	metrics {
		per_host
		observe_catchall_hosts
	}
}
```

## Prometheus

[Prometheus](https://prometheus.io) 是一个监控平台，它通过抓取被监控目标的指标 HTTP 端点来收集这些目标的指标。除了帮助你使用像 [Grafana](https://grafana.com/docs/grafana/latest/introduction/) 这样的仪表板工具显示指标之外，Prometheus 还用于[告警](https://prometheus.io/docs/alerting/latest/overview/)。

和 Caddy 一样，Prometheus 也是用 Go 编写的，并以单个二进制文件分发。要安装它，请参阅 [Prometheus 安装文档](https://prometheus.io/docs/prometheus/latest/installation/)，或者在 MacOS 上只需运行 `brew install prometheus`。

如果你对 Prometheus 完全陌生，请阅读 [Prometheus 文档](https://prometheus.io/docs/introduction/first_steps/)，否则请继续阅读！

要配置 Prometheus 从 Caddy 抓取数据，你需要一个类似下面这样的 YAML 配置文件：

```yaml
# prometheus.yaml
global:
  scrape_interval: 15s # 默认是 1 分钟

scrape_configs:
  - job_name: caddy
    static_configs:
      - targets: ['localhost:2019']
```

然后你可以这样启动 Prometheus：

```console
$ prometheus --config.file=prometheus.yaml
```

## Caddy 的指标

与任何用 Prometheus 监控的进程一样，Caddy 暴露一个 HTTP 端点，该端点以 [Prometheus 展示格式](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format) 响应。Caddy 的 Prometheus 客户端也配置为在协商后以 [OpenMetrics 展示格式](https://pkg.go.dev/github.com/prometheus/client_golang@v1.7.1/prometheus/promhttp#HandlerOpts) 响应（即如果 `Accept` 头设置为 `application/openmetrics-text; version=0.0.1`）。

默认情况下，在[管理 API](/docs/api)（即 http://localhost:2019/metrics）上有一个 `/metrics` 端点。但如果管理 API 被禁用，或者你希望监听不同的端口或路径，你可以使用 [`metrics` 处理器](/docs/caddyfile/directives/metrics) 进行配置。

你可以用任何浏览器或 HTTP 客户端（如 `curl`）查看这些指标：

```console
$ curl http://localhost:2019/metrics
# HELP caddy_admin_http_requests_total Counter of requests made to the Admin API's HTTP endpoints.
# TYPE caddy_admin_http_requests_total counter
caddy_admin_http_requests_total{code="200",handler="metrics",method="GET",path="/metrics"} 2
# HELP caddy_http_request_duration_seconds Histogram of round-trip request durations.
# TYPE caddy_http_request_duration_seconds histogram
caddy_http_request_duration_seconds_bucket{code="308",handler="static_response",method="GET",server="remaining_auto_https_redirects",le="0.005"} 1
caddy_http_request_duration_seconds_bucket{code="308",handler="static_response",method="GET",server="remaining_auto_https_redirects",le="0.01"} 1
caddy_http_request_duration_seconds_bucket{code="308",handler="static_response",method="GET",server="remaining_auto_https_redirects",le="0.025"} 1
...
```

你会看到许多指标，大致分为 4 类：

- 运行时指标
- 管理 API 指标
- HTTP 中间件指标
- 反向代理指标

### 运行时指标

这些指标涵盖 Caddy 进程的内部机制，由 Prometheus Go 客户端自动提供。它们以 `go_*` 和 `process_*` 为前缀。

注意：`process_*` 指标仅在 Linux 和 Windows 上收集。

请参阅 [Go 收集器](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus#NewGoCollector)、[进程收集器](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus#NewProcessCollector) 和 [构建信息收集器](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus#NewBuildInfoCollector) 的文档。

### 管理 API 指标

这些指标有助于监控 Caddy 管理 API。每个管理端点都被检测以跟踪请求计数和错误。

这些指标以 `caddy_admin_*` 为前缀。

例如：

```console
$ curl -s http://localhost:2019/metrics | grep ^caddy_admin
caddy_admin_http_requests_total{code="200",handler="admin",method="GET",path="/config/"} 1
caddy_admin_http_requests_total{code="200",handler="admin",method="GET",path="/debug/pprof/"} 2
caddy_admin_http_requests_total{code="200",handler="admin",method="GET",path="/debug/pprof/cmdline"} 1
caddy_admin_http_requests_total{code="200",handler="load",method="POST",path="/load"} 1
caddy_admin_http_requests_total{code="200",handler="metrics",method="GET",path="/metrics"} 3
```

#### `caddy_admin_http_requests_total`

管理端点处理的请求数量的计数器，包括 `admin.api.*` 命名空间中的模块。

标签  | 描述
-------|------------
`code` | HTTP 状态码
`handler` | 处理器或模块名称
`method` | HTTP 方法
`path` | 管理端点挂载的 URL 路径

#### `caddy_admin_http_request_errors_total`

管理端点中遇到的错误数量的计数器，包括 `admin.api.*` 命名空间中的模块。

标签  | 描述
-------|------------
`handler` | 处理器或模块名称
`method` | HTTP 方法
`path` | 管理端点挂载的 URL 路径

### HTTP 中间件指标

所有 Caddy HTTP 中间件处理器都会自动被检测，以确定请求延迟、首字节时间、错误以及请求/响应体大小。

<aside class="tip">
	由于所有中间件处理器都被检测，并且许多请求由多个处理器处理，请确保不要简单地将所有计数器相加。
</aside>

对于下面的直方图指标，桶目前不可配置。对于持续时间，使用默认（[`prometheus.DefBuckets`](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus#pkg-variables) 桶集（5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s）。对于大小，桶为 256b, 1kiB, 4kiB, 16kiB, 64kiB, 256kiB, 1MiB, 4MiB。

#### `caddy_http_requests_in_flight`

当前由此服务器处理的请求数量的仪表盘。

标签  | 描述
-------|------------
`server` | 服务器名称
`handler` | 处理器或模块名称

#### `caddy_http_request_errors_total`

处理请求时遇到的中间件错误计数器。

标签  | 描述
-------|------------
`server` | 服务器名称
`handler` | 处理器或模块名称

#### `caddy_http_requests_total`

HTTP(S) 请求数量的计数器。

标签  | 描述
-------|------------
`server` | 服务器名称
`handler` | 处理器或模块名称

#### `caddy_http_request_duration_seconds`

往返请求持续时间的直方图。

标签  | 描述
-------|------------
`server` | 服务器名称
`handler` | 处理器或模块名称
`code` | HTTP 状态码
`method` | HTTP 方法

#### `caddy_http_request_size_bytes`

请求总大小（估计值）的直方图。包括 body。

标签  | 描述
-------|------------
`server` | 服务器名称
`handler` | 处理器或模块名称
`code` | HTTP 状态码
`method` | HTTP 方法

#### `caddy_http_response_size_bytes`

返回的响应体大小的直方图。

标签  | 描述
-------|------------
`server` | 服务器名称
`handler` | 处理器或模块名称
`code` | HTTP 状态码
`method` | HTTP 方法

#### `caddy_http_response_duration_seconds`

响应首字节时间的直方图。

标签  | 描述
-------|------------
`server` | 服务器名称
`handler` | 处理器或模块名称
`code` | HTTP 状态码
`method` | HTTP 方法

### 反向代理指标

#### `caddy_reverse_proxy_upstreams_healthy`

反向代理上游健康状态的仪表盘。

值 `0` 表示上游不健康，而 `1` 表示上游健康。

标签  | 描述
-------|------------
`upstream` | 上游地址

## 示例查询

一旦你让 Prometheus 抓取了 Caddy 的指标，你就可以开始看到一些关于 Caddy 性能的有趣指标。

<aside class="tip">

如果你已经按照上面的配置启动了一个 Prometheus 服务器来抓取 Caddy，请尝试在 Prometheus UI（[http://localhost:9090/graph](http://localhost:9090/graph)）中粘贴这些查询。

</aside>

例如，要查看每秒钟的请求速率（5 分钟平均）：

```
rate(caddy_http_requests_total{handler="file_server"}[5m])
```

要查看超出 100ms 延迟阈值的速率：

```
sum(rate(caddy_http_request_duration_seconds_count{server="srv0"}[5m])) by (handler)
-
sum(rate(caddy_http_request_duration_seconds_bucket{le="0.100", server="srv0"}[5m])) by (handler)
```

要找出 `file_server` 处理器上请求持续时间的第 95 百分位数，可以使用如下查询：

```
histogram_quantile(0.95, sum(caddy_http_request_duration_seconds_bucket{handler="file_server"}) by (le))
```

或者查看 `file_server` 处理器上成功的 `GET` 请求的中位数响应大小（以字节为单位）：

```
histogram_quantile(0.5, caddy_http_response_size_bytes_bucket{method="GET", handler="file_server", code="200"})