---
title: invoke (Caddyfile 指令)
---

# invoke

<i>⚠️ 实验性功能</i>

调用一个[命名路由](/docs/caddyfile/concepts#named-routes)。

当与拥有自身内存状态的 HTTP 处理指令搭配使用时，或者这些指令在加载时开销较大时，此功能非常有用。如果你有数百个或更多站点，调用命名路由有助于减少内存使用。

<aside class="tip">
	
与 [`import`](/docs/caddyfile/directives/import) 不同，`invoke` 不支持参数，但你可以使用 [`vars`](/docs/caddyfile/directives/vars) 定义可在命名路由中使用的变量。

</aside>

## 语法

```caddy-d
invoke [<匹配器>] <路由名称>
```

- **&lt;路由名称&gt;** 是先前定义的要被调用的路由的名称。如果未找到该路由，则会触发错误。

## 示例

定义一个包含 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) 的[命名路由](/docs/caddyfile/concepts#named-routes)，该路由可在多个站点中重用，并且每个站点共享相同的内存负载均衡状态。

```caddy
&(app-proxy) {
	reverse_proxy app-01:8080 app-02:8080 app-03:8080 {
		lb_policy least_conn
		health_uri /healthz
		health_interval 5s
	}
}

# 顶级域名允许通过 /app 子路径访问应用，
# 否则访问主站点。
example.com {
	handle_path /app* {
		invoke app-proxy
	}

	handle {
		root /srv
		file_server
	}
}

# 应用也可以通过子域名访问。
app.example.com {
	invoke app-proxy
}