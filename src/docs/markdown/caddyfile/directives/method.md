---
title: method（Caddyfile 指令）
---

# method

更改请求的 HTTP 方法。

## 语法

```caddy-d
method [<matcher>] <method>
```

- **&lt;method&gt;** 是要将请求更改为的 HTTP 方法。

## 示例

将所有 `/api` 路径下请求的方法更改为 `POST`：

```caddy-d
method /api* POST