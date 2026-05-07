---
title: root (Caddyfile 指令)
---

# root

设置站点的根路径，由访问文件系统的各种匹配器和指令使用。如果未设置，默认站点根路径为当前工作目录。

具体来说，此指令会设置 `{http.vars.root}` 占位符。它在同一个块中与其他 `root` 指令互斥，因此使用相交的匹配器定义多个根路径是安全的：它们不会相互级联覆盖。

此指令不会自动启用静态文件服务，因此通常与 [`file_server` 指令](file_server) 或 [`php_fastcgi` 指令](php_fastcgi) 配合使用。

## 语法

```caddy-d
root [<匹配器>] <路径>
```

- **&lt;路径&gt;** 是用于站点根路径的路径。

在 v2.8.0 之前，如果 `<路径>` 参数以 `/` 开头，解析器可能会将其误认为是[匹配器令牌](/docs/caddyfile/matchers#syntax)，因此需要指定一个通配符匹配器令牌（`*`）。

## 示例

将站点根路径设置为 `/home/bob/public_html`（假设 Caddy 以用户 `bob` 身份运行）：

<aside class="tip">

如果你将 Caddy 作为 systemd 服务运行，从 `/home` 读取文件将无法工作，因为 `caddy` 用户没有 `/home` 目录的“可执行”权限（遍历目录所需）。建议将文件放在 `/srv` 或 `/var/www/html` 中。

</aside>

```caddy-d
root /home/bob/public_html
```

<aside class="tip">

注意，在 v2.8.0 之前，这里需要一个[通配符匹配器](/docs/caddyfile/matchers#wildcard-matchers)，因为第一个参数与[路径匹配器](/docs/caddyfile/matchers#path-matchers)存在歧义，即 `root * /srv`，但现在可以简化为 `root /srv`。

</aside>

对所有请求将站点根路径设置为 `public_html`（相对于当前工作目录）：

```caddy-d
root public_html
```

仅对 `/foo/*` 中的请求更改站点根路径：

```caddy-d
root /foo/* /home/user/public_html/foo
```

`root` 指令通常与 [`file_server`](file_server) 配对以提供静态文件服务，或与 [`php_fastcgi`](php_fastcgi) 配对以提供 PHP 站点：

```caddy
example.com {
	root /srv
	file_server
}