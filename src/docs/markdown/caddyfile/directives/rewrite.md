---
title: rewrite（Caddyfile 指令）
---

# rewrite

在内部重写请求 URI。

重写会更改请求 URI 的部分或全部内容。请注意，URI 不包含协议或权限部分（主机和端口），并且客户端通常不会发送片段。因此，该指令主要用于**路径**和**查询字符串**的操作。

`rewrite` 指令表示接受该请求的意图，但会对其进行修改。

它在同一个块中与其他 `rewrite` 指令互斥，因此定义可能相互级联的重写规则是安全的，因为只会执行第一个匹配的重写规则。

在 `rewrite` 之前匹配请求的[请求匹配器](/docs/caddyfile/matchers)可能不会在 `rewrite` 之后匹配同一个请求。如果希望 `rewrite` 与其他处理器共享路由，请使用 [`route`](route) 或 [`handle`](handle) 指令。

## 语法

```caddy-d
rewrite [<matcher>] <to>
```

- **&lt;to&gt;** 是要重写请求的 URI。仅操作重写中指定的 URI 组件（路径或查询字符串）。URI 路径是 `?` 之前的任何子字符串。如果省略 `?`，则整个令牌被视为路径。

在 v2.8.0 之前，如果 `<to>` 参数以 `/` 开头，解析器可能会将其误认为是[匹配器令牌](/docs/caddyfile/matchers#syntax)，因此需要指定通配符匹配器令牌（`*`）。

## 类似指令

还有其他执行重写的指令，但表示不同的意图，或者在不完全替换 URI 的情况下进行重写：

- [`uri`](uri) 操作 URI（去除前缀、后缀或替换子字符串）。

- [`try_files`](try_files) 根据文件是否存在来重写请求。

## 示例

将所有请求重写为 `index.html`，保留任何查询字符串不变：

```caddy
example.com {
	rewrite /index.html
}
```

<aside class="tip">

请注意，在 v2.8.0 之前，这里需要一个[通配符匹配器](/docs/caddyfile/matchers#wildcard-matchers)，因为第一个参数与[路径匹配器](/docs/caddyfile/matchers#path-matchers)存在歧义，即 `rewrite * /foo`，但现在可以简化为 `rewrite /foo`。

</aside>

在所有请求前添加 `/api` 前缀，保留 URI 的其余部分，然后反向代理到应用程序：

```caddy
api.example.com {
	rewrite /api{uri}
	reverse_proxy localhost:8080
}
```

将 API 请求的查询字符串替换为 `a=b`，保持路径不变：

```caddy
example.com {
	rewrite ?a=b
}
```

仅针对 `/api/` 的请求，保留现有查询字符串并添加一个键值对：

```caddy
example.com {
	rewrite /api/* ?{query}&a=b
}
```

同时更改路径和查询字符串，保留原始查询字符串，同时将原始路径作为 `p` 参数添加：

```caddy
example.com {
	rewrite /index.php?{query}&p={path}
}