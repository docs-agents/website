---
title: metrics（Caddyfile 指令）
---

# metrics

配置 Prometheus 指标暴露端点，使收集到的指标能被抓取。**必须首先在[全局选项](/docs/caddyfile/options#metrics)中启用指标。**

注意 `/metrics` 端点也会附加到[管理 API](/docs/api) 上，该端点不可配置，并且在管理 API 禁用时不可用。

此端点将返回 [Prometheus 暴露格式](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format) 的指标，或者（如果协商后）返回 [OpenMetrics 暴露格式](https://pkg.go.dev/github.com/prometheus/client_golang@v1.9.0/prometheus/promhttp#HandlerOpts)（`application/openmetrics-text`）的指标。

另请参阅[使用 Prometheus 指标监控 Caddy](/docs/metrics)。

## 语法

```caddy-d
metrics [<匹配器>] {
	disable_openmetrics
}
```

- **disable_openmetrics** 禁用 OpenMetrics 协商。通常非必要，除非需要解决解析错误。

## 示例

在默认 `/metrics` 路径暴露指标：

```caddy-d
metrics /metrics
```

在其他路径暴露指标：

```caddy-d
metrics /foo/bar/baz
```

在单独的子域名上提供指标：

```caddy
metrics.example.com {
	metrics
}
```

禁用 OpenMetrics 协商：

```caddy-d
metrics /metrics {
	disable_openmetrics
}