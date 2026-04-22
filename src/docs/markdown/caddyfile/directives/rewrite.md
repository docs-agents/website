---
title: rewrite（Caddyfile 指令）
---

# rewrite

在内部重写请求 URI。

`rewrite` 指令会修改请求 URI 的部分或全部内容。请注意，URI 不包含方案（scheme）或权威部分（主机和端口），并且客户端通常不发送片段。因此，该指令主要用于**路径**和**查询**字符串操作。

`rewrite` 指令隐含了接受请求的意图，但会进行一些修改。

它与同一块中的其他 `rewrite` 指令互斥，因此可以安全地定义 otherwise 会级联的重写规则，因为只有第一个匹配的重写会被执行。

在 `rewrite` 之前匹配的 [请求匹配器](/docs/caddyfile/matchers) 可能在 `rewrite` 之后无法匹配相同的请求。如果您希望 `rewrite` 与其他处理器共享路由，请使用 [`route`](route) 或 [`handle`](handle) 指令。


## 语法

```caddy-d
rewrite [<matcher>] <to>
```

- **&lt;to&gt;** 是要重写的 URI。只有重写中指定的 URI 组件（路径或查询字符串）会被操作。URI 路径是 `?` 之前的任何子字符串。如果省略了 `?`，则整个令牌被视为路径。

在 v2.8.0 之前，如果 `<to>` 参数以 `/` 开头，解析器可能会将其误认为是 [匹配器令牌](/docs/caddyfile/matchers#syntax)，因此有必要指定一个通配符匹配器令牌（`*`）。


## 类似指令

还有其他执行重写的指令，但它们隐含不同的意图或以不完整替换 URI 的方式执行重写：

- [`uri`](uri) 操作 URI（去除前缀、后缀或子字符串替换）。

- [`try_files`](try_files) 根据文件的存在性重写请求。



## 示例

将所有请求重写为 `index.html`，保留任何查询字符串不变：

```caddy
example.com {
	rewrite /index.html
}
```

<aside class="tip">

请注意，在 v2.8.0 之前，此处需要一个 [通配符匹配器](/docs/caddyfile/matchers#wildcard-matchers)，因为第一个参数与 [路径匹配器](/docs/caddyfile/matchers#path-matchers) 存在歧义，即 `rewrite * /foo`，但现在可以简化为 `rewrite /foo`。

</aside>

在所有请求前添加 `/api` 前缀，保留 URI 的其余部分，然后反向代理到应用程序：

```caddy
api.example.com {
	rewrite /api{uri}
	reverse_proxy localhost:8080
}
```

将 API 请求的查询字符串替换为 `a=b`，路径保持不变：

```caddy
example.com {
	rewrite ?a=b
}
```

对于仅发往 `/api/` 的请求，保留现有查询字符串并添加键值对：

```caddy
example.com {
	rewrite /api/* ?{query}&a=b
}
```

同时更改路径和查询字符串，保留原始查询字符串，同时将原始路径添加为 `p` 参数：

```caddy
example.com {
	rewrite /index.php?{query}&p={path}
}
```
