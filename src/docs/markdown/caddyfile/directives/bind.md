---
title: bind (Caddyfile 指令)
---

# bind

覆盖服务器套接字应绑定的网络接口。

通常情况下，监听器会绑定到空（通配符）接口。但你可以强制监听器绑定到其他主机名或 IP。此指令仅接受主机，不接受端口。端口由[站点地址](/docs/caddyfile/concepts#addresses)决定（默认为 `443`）。

请注意，不一致地绑定站点可能会导致意外后果。例如，如果同一端口上的两个站点都解析到 `127.0.0.1`，但只有其中一个站点配置了 `bind 127.0.0.1`，那么只有一个站点可访问，因为另一个站点将绑定到该端口但未指定具体主机；操作系统将选择更具体的匹配套接字。（虚拟主机不会在不同的监听器之间共享。）

`bind` 接受[网络地址](/docs/conventions#network-addresses)，但不能包含端口。

## 语法

```caddy-d
bind <hosts...>
```

- **&lt;hosts...&gt;** 是要绑定监听器的主机接口列表。

## 示例

要使套接字仅可在当前机器上访问，请绑定到回环接口（localhost）：

```caddy
example.com {
	bind 127.0.0.1
}
```

包含 IPv6：

```caddy
example.com {
	bind 127.0.0.1 [::1]
}
```

绑定到 `10.0.0.1:8080`：

```caddy
example.com:8080 {
	bind 10.0.0.1
}
```

绑定到 Unix 域套接字 `/run/caddy`：

```caddy
example.com {
	bind unix//run/caddy
}
```

将文件权限更改为所有用户可写（[默认值](/docs/conventions#network-addresses)为 `0200`，仅所有者可写）：

```caddy
example.com {
	bind unix//run/caddy|0222
}
```

将一个域名绑定到两个不同的接口，并返回不同的响应：

```caddy
example.com {
	bind 10.0.0.1
	respond "One"
}

example.com {
	bind 10.0.0.2
	respond "Two"
}