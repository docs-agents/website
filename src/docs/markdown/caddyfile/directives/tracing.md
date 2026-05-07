---
title: tracing（Caddyfile 指令）
---

# tracing

通过使用 [`opentelemetry-go` <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/open-telemetry/opentelemetry-go)，启用与 OpenTelemetry 追踪设施的集成。

启用后，它将传播现有的追踪上下文或初始化一个新的上下文。

它使用 [gRPC](https://github.com/grpc/) 作为导出器协议，并使用 W3C [tracecontext](https://www.w3.org/TR/trace-context/) 和 [baggage](https://www.w3.org/TR/baggage/) 作为传播器。

追踪 ID 和跨度 ID 会以标准的 `traceID` 和 `spanID` 字段添加到 [访问日志](/docs/caddyfile/directives/log) 中。此外，还可以使用 `{http.vars.trace_id}` 和 `{http.vars.span_id}` 占位符；例如，您可以在 [`request_header`](request_header) 中使用它们将 ID 传递给您的应用。

## 语法

```caddy-d
tracing {
	span <span_name>
	span_attributes {
		<attr1> <value1>
		<attr2> <value2>
	}
}
```

- **&lt;span_name&gt;** 是跨度名称。请参阅跨度 [命名指南](https://github.com/open-telemetry/opentelemetry-specification/blob/v1.7.0/specification/trace/api.md)。
- **&lt;span_attributes&gt;** 是附加到每个记录跨度上的额外属性。根据 OTEL [HTTP 跨度的语义约定](https://opentelemetry.io/docs/specs/semconv/http/http-spans/)，许多跨度属性默认已设置，例如有关请求、响应和客户端的详细信息。

  跨度名称和属性中可以使用 [占位符](/docs/caddyfile/concepts#placeholders)。请注意，跨度名称是在请求转发之前设置的，因此只能使用请求占位符。跨度属性中可以使用所有占位符。

## 配置

### 环境变量

可以使用由 [OpenTelemetry 环境变量规范](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/configuration/sdk-environment-variables.md) 定义的环境变量进行配置。

有关导出器配置的详细信息，请参阅 [规范](https://github.com/open-telemetry/opentelemetry-specification/blob/v1.7.0/specification/protocol/exporter.md)。

例如：

```bash
export OTEL_EXPORTER_OTLP_HEADERS="myAuthHeader=myToken,anotherHeader=value"
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://my-otlp-endpoint:55680
```

## 示例

以下是一个 **Caddyfile** 示例：

```caddy
example.com {
	handle /api* {
		tracing {
			span api
		}
		request_header X-Trace-Id {http.vars.trace_id}
		reverse_proxy localhost:8081
	}

	handle {
		tracing {
			span app
			span_attributes {
				user_id {http.request.cookie.user-id}
			}
		}
		reverse_proxy localhost:8080
	}
}