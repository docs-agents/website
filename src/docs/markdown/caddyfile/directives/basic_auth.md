---
title: basic_auth (Caddyfile 指令)
---

# basic_auth

启用 HTTP 基本认证，可用于使用用户名和哈希密码保护目录和文件。

**请注意，基本认证在纯 HTTP 下是不安全的。** 在使用 HTTP 基本认证保护内容时请审慎决定。

当用户请求受保护的资源时，如果尚未提供凭据，浏览器会提示用户输入用户名和密码。如果 `Authorization` 头中包含正确的凭据，服务器将授予对该资源的访问权限。如果缺少该头或凭据不正确，服务器将返回 HTTP 401 Unauthorized 响应。

Caddy 配置不接受明文密码；您必须在将其放入配置之前对密码进行哈希处理。[`caddy hash-password`](/docs/command-line#caddy-hash-password) 命令可以帮助完成此操作。

认证成功后，将提供 `{http.auth.user.id}` 占位符，其中包含已认证的用户名。

在 v2.8.0 之前，此指令名为 `basicauth`，但为了与其他指令保持一致而重命名。

## 语法

```caddy-d
basic_auth [<匹配器>] [<哈希算法> [<领域>]] {
	<用户名> <已哈希密码>
	...
}
```

- **&lt;哈希算法&gt;** 指定用于此配置中密码哈希的哈希算法（或密钥派生函数）。可用选项包括 `argon2id`，默认为 `bcrypt`。

- **&lt;领域&gt;** 是一个自定义的领域名称。

- **&lt;用户名&gt;** 是用户名或用户 ID。

- **&lt;已哈希密码&gt;** 是密码哈希值。

## 示例

要求对 `example.com` 的所有请求进行身份验证：

```caddy
example.com {
	basic_auth {
		# 用户名 "Bob"，密码 "hiccup"
		Bob $2a$14$Zkx19XLiW6VYouLHR5NmfOFU0z2GTNmpkT/5qqR7hx4IjWJPDhjvG
	}
	respond "欢迎, {http.auth.user.id}" 200
}
```

保护 `/secret/` 目录下的文件，只有 `Bob` 可以访问（其他人可以访问其他路径）：

```caddy
example.com {
	root /srv

	basic_auth /secret/* {
		# 用户名 "Bob"，密码 "hiccup"
		Bob $2a$14$Zkx19XLiW6VYouLHR5NmfOFU0z2GTNmpkT/5qqR7hx4IjWJPDhjvG
	}

	file_server
}
```

`argon2id` 示例

```caddy
example.com {
	root /srv

	basic_auth /secret/* argon2id {
		# 用户名 "Bob"，密码 "hiccup"
		Bob $argon2id$v=19$m=47104,t=1,p=1$zJPvVe48N64JUa9MFlVhiw$b5Tznu0PxnA4TciY6qYe2BFPxncF1ePQaeNukHhH1cU
	}

	file_server
}