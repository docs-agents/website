---
title: request_body（Caddyfile 指令）
---

# request_body

操作或设置传入请求正文的限制。

## 语法

```caddy-d
request_body [<matcher>] {
	max_size <value>
	set <body_content>
}
```

- **max_size** 是允许的请求正文最大字节数。它接受 [go-humanize](https://pkg.go.dev/github.com/dustin/go-humanize#pkg-constants) 支持的所有大小值。读取超过此字节数的数据将返回 HTTP 状态码 `413` 错误。

⚠️ <i>实验性</i> <span style='white-space: pre;'> | </span> <span>v2.10.0+</span>
- **set** 允许将请求正文设置为特定内容。内容可以包含占位符以动态插入数据。

## 示例

将请求正文大小限制为 10 MB：

```caddy
example.com {
	request_body {
		max_size 10MB
	}
	reverse_proxy localhost:8080
}
```

设置包含 SQL 查询的 JSON 结构作为请求正文：

```caddy
example.com {
	handle /jazz {
		request_body {
			set `\{"statementText":"SELECT name, genre, debut_year FROM artists WHERE genre = 'Jazz'"}`
		}

		reverse_proxy localhost:8080 {
			header_up Content-Type application/json
			method POST
			rewrite /execute-sql
		}
	}
}