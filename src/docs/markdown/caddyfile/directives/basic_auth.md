---
title: basic_auth（Caddyfile 指令）
---

# basic_auth

启用 HTTP 基本认证，可用于使用用户名和哈希密码保护目录和文件。

**请注意，基本认证在纯 HTTP 上不安全。** 在决定用什么进行基本认证保护时请自行斟酌。

当用户请求受保护的资源时，如果尚未提供用户名和密码，浏览器将提示用户输入。如果 Authorization 头中存在正确的凭据，服务器将授予对该资源的访问权限。如果头信息缺失或凭据不正确，服务器将返回 HTTP 401 未授权响应。

Caddy 配置不接受明文密码；在将其放入配置之前，您**必须**先对其进行哈希处理。[`caddy hash-password`](/docs/command-line#caddy-hash-password) 命令可以帮助您完成此操作。

成功认证后，将可使用 `{http.auth.user.id}` 占位符，其中包含已认证的用户名。

在 v2.8.0 之前，此指令名为 `basicauth`，但为与其他指令保持一致而进行了重命名。


## 语法

```caddy-d
basic_auth [<matcher>] [<hash_algorithm> [<realm>]] {
	<username> <hashed_password>
	...
}
```

- **&lt;hash_algorithm&gt;** 指定用于此配置中哈希的密码哈希算法（或密钥推导函数）。可用选项包括 `argon2id`，默认为 `bcrypt`。

- **&lt;realm&gt;** 是自定义的realm名称。

- **&lt;username&gt;** 是用户名或用户 ID。

- **&lt;hashed_password&gt;** 是密码哈希值。


## 示例

要求对 `example.com` 的所有请求进行身份验证：

```caddy
example.com {
	basic_auth {
		# Username "Bob", password "hiccup"
		Bob $2a$14$Zkx19XLiW6VYouLHR5NmfOFU0z2GTNmpkT/5qqR7hx4IjWJPDhjvG
	}
	respond "Welcome, {http.auth.user.id}" 200
}
```

保护 `/secret/` 中的文件，使只有 `Bob` 能够访问（其他人可以查看其他路径）：

```caddy
example.com {
	root /srv

	basic_auth /secret/* {
		# Username "Bob", password "hiccup"
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
		# Username "Bob", password "hiccup"
		Bob $argon2id$v=19$m=47104,t=1,p=1$zJPvVe48N64JUa9MFlVhiw$b5Tznu0PxnA4TciY6qYe2BFPxncF1ePQaeNukHhH1cU
	}

	file_server
}
```
