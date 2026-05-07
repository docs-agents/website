---
title: request_header (Caddyfile 指令)
---

# request_header

操作请求中的 HTTP 头部字段。可以设置、添加和删除头部值，或使用正则表达式执行替换。

如果打算操作用于代理的头部，请改用 `reverse_proxy` 的 [`header_up` 子指令](/docs/caddyfile/directives/reverse_proxy#header_up)，因为这些操作是代理感知的。

要操作 HTTP 响应头部，可以使用 [`header`](header) 指令。


## 语法

```caddy-d
request_header [<匹配器>] [[+|-]<字段> [<值>|<查找>] [<替换>]]
```

- **&lt;字段&gt;** 是头部字段的名称。

  无前缀时，字段被设置（覆盖）。

  前缀为 `+` 时，如果字段已存在，则添加字段而不是覆盖（设置）；在一个请求中，头部字段可以出现多次。

  前缀为 `-` 时，删除该字段。字段名可以使用前缀或后缀 `*` 通配符来删除所有匹配的字段。

- **&lt;值&gt;** 是头部字段的值，用于添加或设置字段时。

- **&lt;查找&gt;** 是要搜索的子字符串或正则表达式。

- **&lt;替换&gt;** 是替换值；如果执行查找替换，则需要此参数。


## 示例

从请求中删除 Referer 头部：

```caddy-d
request_header -Referer
```

从请求中删除所有包含下划线的头部：

```caddy-d
request_header -*_*