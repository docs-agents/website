---
title: forward_auth (Caddyfile 指令)
---

<script>
ready(function() {
	// 修正代码块中的 >
	$$_('pre.chroma .k').forEach(item => {
		if (item.innerText.includes('>')) {
			// 如果以 > 结尾则跳过
			if (item.innerText.trim().endsWith('>')) return;
			// 将 > 替换为 <span class="p">&gt;</span>
			item.innerHTML = item.innerHTML.replace(/&gt;/g, '<span class="p">&gt;</span>');
		}
	});

	// 修复 uri 子指令，由于存在 "uri" 指令，它会被解析为匹配器参数
	$$_('.k').forEach(item => {
		if (item.innerText.includes('uri') && item.nextElementSibling && item.nextElementSibling.classList.contains('nd')) {
			const next = item.nextElementSibling;
			next.classList.remove('nd');
			next.classList.add('s');
			next.textContent = next.textContent;
		}
	});
});
</script>

# forward_auth

这是一个有主见的（opinionated）指令，用于将请求的副本代理到身份验证网关，由该网关决定是继续处理，还是需要重定向到登录页面。

- [语法](#syntax)
- [展开形式](#expanded-form)
- [示例](#examples)
  - [Authelia](#authelia)
  - [Tailscale](#tailscale)

Caddy 的 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) 能够对外部服务执行“预检查请求”，但本指令专门针对身份验证场景进行了定制。实际上，本指令只是使用更冗长、更通用的配置（见下文）的一种便捷方式。

该指令向配置的上游服务发送一个 `GET` 请求，并将 `uri` 重写：
- 如果上游响应状态码为 `2xx`，则允许访问，并将 `copy_headers` 中的头部字段复制到原始请求中，继续处理。
- 否则，如果上游返回任何其他状态码，则上游的响应会被原样返回给客户端。该响应通常应包含一个重定向到身份验证网关登录页面的指令。

如果这种行为并非您所需，您可以参考下面的[展开形式](#expanded-form)进行定制。

所有 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) 的子指令均受支持，并向下传递给底层的 `reverse_proxy` 处理器。

## 语法

```caddy-d
forward_auth [<匹配器>] [<上游...>] {
	uri          <目标URI>
	copy_headers <字段...> {
		<字段...>
	}
}
```

- **&lt;上游...&gt;**：一个上游（后端）列表，身份验证请求将发送至这些后端。

- **uri**：设置发送给上游的请求的 URI（路径和查询）。这通常是身份验证网关的验证端点。

- **copy_headers**：一个 HTTP 头部字段列表，当请求成功（状态码为 2xx）时，将这些字段从上游响应复制到原始请求中。

  字段可以通过使用 `>` 后跟新名称来重命名，例如 `Before>After`。

  如果您倾向于更易读的格式，可以使用代码块逐行列示所有字段，每行一个。

由于本指令是对反向代理的有主见封装，您可以使用任何 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy#syntax) 的子指令来自定义它。

## 展开形式

`forward_auth` 指令等同于以下配置。像 [Authelia](https://www.authelia.com/) 这样的身份验证网关可以很好地与此预设配合使用。如果您的网关不兼容，请参考此配置并根据需要自定义，而非使用 `forward_auth` 快捷方式。

```caddy-d
reverse_proxy <上游...> {
	# 始终使用 GET 方法，以免消耗传入请求的请求体
	method GET

	# 将 URI 重写为身份验证网关的验证端点
	rewrite <目标URI>

	# 转发原始的 HTTP 方法和 URI，
	# 因为它们在上述重写中被修改；
	# 这是除 reverse_proxy 已设置的
	# X-Forwarded-* 头部之外的额外操作
	header_up X-Forwarded-Method {method}
	header_up X-Forwarded-Uri {uri}

	# 在成功响应时，复制响应头部
	@good status 2xx
	handle_response @good {
		# 例如，对于每个 copy_headers 字段...
		request_header Remote-User {rp.header.Remote-User}
		request_header Remote-Email {rp.header.Remote-Email}
	}
}
```

## 示例

### Authelia

在通过反向代理服务您的应用之前，将身份验证委托给 [Authelia](https://www.authelia.com/)：

```caddy
# 为身份验证网关本身提供服务
auth.example.com {
	reverse_proxy authelia:9091
}

# 为您的应用提供服务
app1.example.com {
	forward_auth authelia:9091 {
		uri /api/authz/forward-auth
		copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
	}

	reverse_proxy app1:8080
}
```

更多信息，请参阅 Authelia 关于与 Caddy 集成的[文档](https://www.authelia.com/integration/proxies/caddy/)。

### Tailscale

将身份验证委托给 [Tailscale](https://tailscale.com/)（当前名为 [`nginx-auth`](https://tailscale.com/blog/tailscale-auth-nginx/)，但仍可与 Caddy 配合使用），并使用 `copy_headers` 的替代语法来*重命名*复制的头部（注意每个头部中的 `>`）：

```caddy-d
forward_auth unix//run/tailscale.nginx-auth.sock {
	uri /auth
	header_up Remote-Addr {remote_host}
	header_up Remote-Port {remote_port}
	header_up Original-URI {uri}
	copy_headers {
		Tailscale-User>X-Webauth-User
		Tailscale-Name>X-Webauth-Name
		Tailscale-Login>X-Webauth-Login
		Tailscale-Tailnet>X-Webauth-Tailnet
		Tailscale-Profile-Picture>X-Webauth-Profile-Picture
	}
}