---
title: 常见 Caddyfile 模式
---

# 常见 Caddyfile 模式

本文展示了一些常见用法的完整且精简的 Caddyfile 配置示例。这些可以作为您编写自己的 Caddyfile 文档的起始参考。

这些不是即插即用的解决方案；您需要自定义域名、端口/套接字、目录路径等。它们旨在说明一些最常见的配置模式。

- [静态文件服务器](#静态文件服务器)
- [反向代理](#反向代理)
- [PHP](#php)
- [重定向 `www.` 子域名](#重定向-www-子域名)
- [尾随斜杠](#尾随斜杠)
- [通配符证书](#通配符证书)
- [单页应用 (SPAs)](#单页应用-spas)
- [Caddy 代理到另一个 Caddy](#caddy-代理到另一个-caddy)


## 静态文件服务器

```caddy
example.com {
	root /var/www
	file_server
}
```

和往常一样，第一行是站点地址。[`root` 指令](/docs/caddyfile/directives/root) 指定站点根目录的路径（`*` 表示匹配所有请求，以便与 [路径匹配器](/docs/caddyfile/matchers#path-matchers) 区分开）——如果不是当前工作目录，请更改为您的站点路径。最后，我们启用 [静态文件服务器](/docs/caddyfile/directives/file_server)。



## 反向代理

代理所有请求：

```caddy
example.com {
	reverse_proxy localhost:5000
}
```

仅代理路径以 `/api/` 开头的请求，并为其他所有请求提供静态文件：

```caddy
example.com {
	root /var/www
	reverse_proxy /api/* localhost:5000
	file_server
}
```

此配置使用 [请求匹配器](/docs/caddyfile/matchers#syntax) 仅匹配以 `/api/` 开头的请求，并将它们代理到后端。所有其他请求将通过站点 [`root`](/docs/caddyfile/directives/root) 由 [静态文件服务器](/docs/caddyfile/directives/file_server) 服务。这也依赖于 `reverse_proxy` 在 [指令顺序](/docs/caddyfile/directives#directive-order) 上比 `file_server` 更高这一事实。

此处有大量的 [`reverse_proxy` 示例](/docs/caddyfile/directives/reverse_proxy#examples)。



## PHP

### PHP-FPM

当 PHP FastCGI 服务运行时，以下配置适用于大多数现代 PHP 应用：

```caddy
example.com {
	root /srv/public
	encode
	php_fastcgi localhost:9000
	file_server
}
```

相应地定制站点根目录；此示例假设您的 PHP 应用的网络根目录位于 `public` 目录内——磁盘上存在的文件将使用 [`file_server`](/docs/caddyfile/directives/file_server) 服务，其他内容将路由到 `index.php` 由 PHP 应用处理。

您有时也可以使用 Unix 套接字连接到 PHP-FPM：

```caddy-d
php_fastcgi unix//run/php/php8.2-fpm.sock
```

[`php_fastcgi` 指令](/docs/caddyfile/directives/php_fastcgi) 实际上是 [多个配置片段](/docs/caddyfile/directives/php_fastcgi#expanded-form) 的快捷方式。


### FrankenPHP

或者，您可以使用 [FrankenPHP](https://frankenphp.dev/)，这是一个使用 CGO（Go 到 C 绑定）直接调用 PHP 的 Caddy 发行版。这比 PHP-FPM 快多达 4 倍，如果您能使用 worker 模式效果会更好。

```caddy
{
    frankenphp
    order php_server before file_server
}

example.com {
	root /srv/public
    encode zstd br gzip
    php_server
}
```


## 重定向 `www.` 子域名

**添加** `www.` 子域名并执行 HTTP 重定向：

```caddy
example.com {
	redir https://www.{host}{uri}
}

www.example.com {
}
```


**移除** 它：

```caddy
www.example.com {
	redir https://example.com{uri}
}

example.com {
}
```


同时为 **多个域名** 移除它；此示例使用 `{labels.*}` 占位符，这些是主机名的各部分，从右侧开始 0 索引（例如 `0`=`com`，`1`=`example-one`，`2`=`www`）：

```caddy
www.example-one.com, www.example-two.com {
	redir https://{labels.1}.{labels.0}{uri}
}

example-one.com, example-two.com {
}
```



## 尾随斜杠

您通常不需要自行配置此内容；[`file_server` 指令](/docs/caddyfile/directives/file_server) 会通过 HTTP 重定向自动为请求添加或删除尾随斜杠，具体取决于所请求的资源是目录还是文件。

但是，如果您需要，您仍然可以通过配置强制执行尾随斜杠。有两种方式：内部或外部。

### 内部强制执行

此方法使用 [`rewrite`](/docs/caddyfile/directives/rewrite) 指令。Caddy 将内部重写 URI 以添加或删除尾随斜杠：

```caddy
example.com {
	rewrite /add     /add/
	rewrite /remove/ /remove
}
```

使用重写，带或不带尾随斜杠的请求将是相同的。


### 外部强制执行

此方法使用 [`redir`](/docs/caddyfile/directives/redir) 指令。Caddy 将要求浏览器更改 URI 以添加或删除尾随斜杠：

```caddy
example.com {
	redir /add     /add/
	redir /remove/ /remove
}
```

使用重定向，客户端必须重新发出请求，从而为资源强制执行单个可接受的 URI。



## 通配符证书

对于包括 Let's Encrypt 在内的多数颁发机构，您必须启用 [ACME DNS 挑战](/docs/automatic-https#dns-challenge) 才能使 Caddy 自动化管理通配符证书。

启用 DNS 挑战后，从 Caddy 2.10 开始，如果已配置或管理了适用的通配符证书，Caddy 将优先使用该证书，而不是为子域名单独管理证书。



如果您需要使用相同的通配符证书服务多个子域名，处理它们最好的方式是使用类似以下的 Caddyfile，使用 [`handle` 指令](/docs/caddyfile/directives/handle) 和 [`host` 匹配器](/docs/caddyfile/matchers#host)：

```caddy
*.example.com {
	tls {
		dns <provider_name> [<params...>]
	}

	@foo host foo.example.com
	handle @foo {
		respond "Foo!"
	}

	@bar host bar.example.com
	handle @bar {
		respond "Bar!"
	}

	# 否则未处理的域名回退
	handle {
		abort
	}
}
```

您必须启用 [ACME DNS 挑战](/docs/automatic-https#dns-challenge) 才能使 Caddy 自动管理通配符证书。



## 单页应用 (SPAs)

当网页进行自身路由时，服务器可能会收到大量不存在的页面请求，但只要提供单个 index 文件，这些页面就可以作为客户端可渲染的页面。采用这种架构的 Web 应用被称为 SPA，即单页应用。

主要思路是让服务器尝试文件，查看请求的文件是否在服务器端存在，如果不存在，则回退到 index 文件，由客户端进行路由（通常使用客户端 JavaScript）。

典型的 SPA 配置通常如下所示：

```caddy
example.com {
	root /srv
	encode
	try_files {path} /index.html
	file_server
}
```

如果您的 SPA 与 API 或其他仅服务器端端点耦合，您可能希望使用 `handle` 块来专门处理它们：

```caddy
example.com {
	encode

	handle /api/* {
		reverse_proxy backend:8000
	}

	handle {
		root /srv
		try_files {path} /index.html
		file_server
	}
}
```

如果您的 `index.html` 包含使用哈希文件名的 JS/CSS 资产引用，您可能希望考虑添加 `Cache-Control` 标头以指示客户端_不要_缓存它（以便如果资产发生变化，浏览器会获取新的）。由于 `try_files` 重写用于从任何不匹配磁盘上其他文件的路径服务您的 `index.html`，您可以使用 `route` 包装 `try_files`，使得 `header` 处理程序在重写_之后_运行（由于 [指令顺序](/docs/caddyfile/directives#directive-order)，它通常会在之前运行）：

```caddy-d
route {
	try_files {path} /index.html
	header /index.html Cache-Control "public, max-age=0, must-revalidate"
}
```


## Caddy 代理到另一个 Caddy

如果您有一个公开可访问的 Caddy 实例（我们称之为"前端"），以及另一个位于您私有网络中的 Caddy 实例（我们称之为"后端"）用于服务您的实际应用，您可以使用 [`reverse_proxy` 指令](/docs/caddyfile/directives/reverse_proxy) 将请求转发过去。

前端实例：

```caddy
foo.example.com, bar.example.com {
	reverse_proxy 10.0.0.1:80
}
```

后端实例：

```caddy
{
	servers {
		trusted_proxies static private_ranges
	}
}

http://foo.example.com {
	reverse_proxy foo-app:8080
}

http://bar.example.com {
	reverse_proxy bar-app:9000
}
```

- 此示例提供两个不同的域名，将两者都代理到同一后端 Caddy 实例的 `80` 端口。您的后端实例以不同方式服务这两个域名，因此它配置为两个单独的网站块。

- 在 Backend 上，使用 [`http://`](/docs/caddyfile/concepts#addresses) 在 `80` 端口接受 HTTP。前端实例终止 TLS，前端和后端之间的流量在私有网络上，因此无需重新加密。

- 您可以在后端实例上使用不同的端口（如 `8080`；只需在每个站点地址后附加 `:8080`，或将 [`http_port` 全局选项](/docs/caddyfile/options#http_port) 设置为 `8080`）。

- 在 Backend 上，[`trusted_proxies` 全局选项](/docs/caddyfile/options#trusted_proxies) 用于告诉 Caddy 信任前端实例作为代理。这确保保留真实的客户端 IP。

- 更进一步，您可以有多个后端实例，在它们之间 [负载均衡](/docs/caddyfile/directives/reverse_proxy#load-balancing)。您可以使用前端实例上的 [`acme_server`](/docs/caddyfile/directives/acme_server) 设置 mTLS（双向 TLS），使其充当后端实例的 CA（如果前端和后端之间的流量穿越不可信网络，这将非常有用）。
