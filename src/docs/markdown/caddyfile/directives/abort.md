---
title: abort (Caddyfile 指令)
---

# abort

通过立即中止 HTTP 处理链并关闭连接，阻止向客户端发送任何响应。同一连接上的任何并发、活跃的 HTTP 流也会被中断。

## 语法

```caddy-d
abort [<matcher>]
```

## 示例

当使用通配符证书时，强制关闭针对未知域名的连接：

```caddy
*.example.com {
    @foo host foo.example.com
    handle @foo {
        respond "This is foo!" 200
    }

    handle {
		# 未处理的域名会落到这里，
		# 但我们不希望接受它们的请求
        abort
    }
}