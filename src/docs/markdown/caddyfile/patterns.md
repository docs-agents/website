---
title: 常见 Caddyfile 模式
---

# 常见 Caddyfile 模式

本页展示了一些常见用例的完整且精简的 Caddyfile 配置。这些配置可以作为您自己 Caddyfile 文档的起点。

它们并非直接可用的解决方案；您需要自定义域名、端口/套接字、目录路径等。它们旨在说明一些最常见的配置模式。

- [静态文件服务器](#静态文件服务器)
- [反向代理](#反向代理)
- [PHP](#php)
- [重定向 `www.` 子域名](#重定向-www-子域名)
- [尾部斜杠](#尾部斜杠)
- [通配符证书](#通配符证书)
- [单页应用 (SPA)](#单页应用-spa)
- [Caddy 代理到另一个 Caddy](#caddy-代理到另一个-caddy)

## 静态文件服务器

```caddy
example.com {
	root /var/www
	file_server
}
```

与往常一样，第一行是站点地址。[`root` 指令](/docs/caddyfile/directives/root) 指定站点根目录的路径（`*` 表示匹配所有请求，以便与[路径匹配器](/docs/caddyfile/matchers#path-matchers) 区分）——如果您的站点不是当前工作目录，请将路径改为您的站点。最后，我们启用[静态文件服务器](/docs/caddyfile/directives/file_server)。

## 反向代理

代理所有请求：

```caddy
example.com {
	reverse_proxy localhost:5000
}
```

仅代理路径以 `/api/` 开头的请求，其他请求则提供静态文件：

```caddy
example.com {
	root /var/www
	reverse_proxy /api/* localhost:5000
	file_server
}
```

这里使用[请求匹配器](/docs/caddyfile/matchers#syntax) 仅匹配以 `/api/` 开头的请求，并将其代理到后端。所有其他请求将使用站点 [`root`](/docs/caddyfile/directives/root) 和[静态文件服务器](/docs/caddyfile/directives/file_server) 提供服务。这还依赖于 `reverse_proxy` 在[指令顺序](/docs/caddyfile/directives#directive-order) 中优先级高于 `file_server` 的事实。

[这里有更多 `reverse_proxy` 示例](/docs/caddyfile/directives/reverse_proxy#examples)。

## PHP

### PHP-FPM

在运行 PHP FastCGI 服务时，以下配置适用于大多数现代 PHP 应用：

```caddy
example.com {
	root /srv/public
	encode
	php_fastcgi localhost:9000
	file_server
}
```

根据情况自定义站点根目录；此示例假定您的 PHP 应用的 web 根目录位于 `public` 目录中——对磁盘上已存在文件的请求将由 [`file_server`](/docs/caddyfile/directives/file_server) 处理，其他所有请求将被路由到 `index.php`，由 PHP 应用处理。

有时您可能会使用 Unix 套接字连接到 PHP-FPM：

```caddy-d
php_fastcgi unix//run/php/php8.2-fpm.sock
```

[`php_fastcgi` 指令](/docs/caddyfile/directives/php_fastcgi) 实际上是[几个配置项](/docs/caddyfile/directives/php_fastcgi#expanded-form) 的快捷方式。

### FrankenPHP

另一种选择是使用 [FrankenPHP](https://frankenphp.dev/)，它是 Caddy 的一个发行版，通过 CGO（Go 到 C 绑定）直接调用 PHP。其速度可比 PHP-FPM 快 4 倍，如果使用工作进程模式则更佳。

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

**添加** `www.` 子域名并通过 HTTP 重定向：

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

为**多个域名**同时移除 `www.`；这里使用了 `{labels.*}` 占位符，它们是主机名的组成部分，从右侧以 `0` 开始索引（例如 `0`=`com`，`1`=`example-one`，`2`=`www`）：

```caddy
www.example-one.com, www.example-two.com {
	redir https://{labels.1}.{labels.0}{uri}
}

example-one.com, example-two.com {
}
```

## 尾部斜杠

通常您不需要自己配置；[`file_server` 指令](/docs/caddyfile/directives/file_server) 会自动通过 HTTP 重定向为请求添加或移除尾部斜杠，具体取决于请求的资源是目录还是文件。

但如果需要，您仍然可以通过配置来强制尾部斜杠。有两种方法：内部强制或外部强制。

### 内部强制

这使用了 [`rewrite`](/docs/caddyfile/directives/rewrite) 指令。Caddy 会在内部重写 URI 以添加或移除尾部斜杠：

```caddy
example.com {
	rewrite /add     /add/
	rewrite /remove/ /remove
}
```

使用重写后，带或不带尾部斜杠的请求将视为相同。

### 外部强制

这使用了 [`redir`](/docs/caddyfile/directives/redir) 指令。Caddy 会要求浏览器更改 URI 以添加或移除尾部斜杠：

```caddy
example.com {
	redir /add     /add/
	redir /remove/ /remove
}
```

使用重定向，客户端必须重新发出请求，从而为资源强制执行单一可接受的 URI。

## 通配符证书

对于包括 Let's Encrypt 在内的大多数颁发机构，您必须启用 [ACME DNS 挑战](/docs/automatic-https#dns-challenge) 才能让 Caddy 自动管理通配符证书。

在启用 DNS 挑战后，自 Caddy 2.10 起，Caddy 会优先使用已配置或管理的适用通配符证书，而不是为子域名单独管理证书。

如果需要用同一张通配符证书服务多个子域名，最佳处理方式是使用如下 Caddyfile，利用 [`handle` 指令](/docs/caddyfile/directives/handle) 和 [`host` 匹配器](/docs/caddyfile/matchers#host)：

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

	# 未处理的域名回退
	handle {
		abort
	}
}
```

必须启用 [ACME DNS 挑战](/docs/automatic-https#dns-challenge) 才能让 Caddy 自动管理通配符证书。

## 单页应用 (SPA)

当网页自行处理路由时，服务器可能会收到大量请求，这些页面在服务器端并不存在，但只要提供唯一的索引文件，就可以在客户端渲染。这种架构的 Web 应用称为 SPA，即单页应用。

主要思路是让服务器“尝试文件”，查看请求的文件在服务器端是否存在，如果不存在，则回退到索引文件，由客户端（通常使用客户端 JavaScript）进行路由。

典型的 SPA 配置通常如下所示：

```caddy
example.com {
	root /srv
	encode
	try_files {path} /index.html
	file_server
}
```

如果您的 SPA 与 API 或其他仅服务器端的端点配合使用，您需要使用 `handle` 块来单独处理：

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

如果您的 `index.html` 中包含对带有哈希文件名的 JS/CSS 资产的引用，您可能需要添加一个 `Cache-Control` 头，指示客户端不要缓存它（这样当资产更改时，浏览器会获取新版本）。由于 `try_files` 重写用于从任何不匹配磁盘上其他文件的路径提供 `index.html`，您可以将 `try_files` 包装在 `route` 中，以便 `header` 处理器在重写之后运行（正常情况下由于[指令顺序](/docs/caddyfile/directives#directive-order)，它会先运行）：

```caddy-d
route {
	try_files {path} /index.html
	header /index.html Cache-Control "public, max-age=0, must-revalidate"
}
```

## Caddy 代理到另一个 Caddy

如果您有一个公网可访问的 Caddy 实例（我们称之为“前端”），以及另一个在私有网络中运行实际应用的 Caddy 实例（我们称之为“后端”），您可以使用 [`reverse_proxy` 指令](/docs/caddyfile/directives/reverse_proxy) 将请求传递过去。

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

- 此示例服务两个不同的域名，并将两者都代理到同一个后端 Caddy 实例的 80 端口。您的后端实例以不同方式服务这两个域名，因此配置了两个独立的站点块。
- 在后端，使用 [`http://`](/docs/caddyfile/concepts#addresses) 来接受 80 端口的 HTTP 流量。前端实例终止 TLS，前端和后端之间的流量在私有网络上，因此无需重新加密。
- 如果需要，您可以在后端实例上使用不同的端口，例如 `8080`；只需在后端配置的每个站点地址后附加 `:8080`，或者将 [`http_port` 全局选项](/docs/caddyfile/options#http_port) 设置为 `8080`。
- 在后端，使用 [`trusted_proxies` 全局选项](/docs/caddyfile/options#trusted_proxies) 告诉 Caddy 信任前端实例作为代理。这确保了真实的客户端 IP 得以保留。
- 更进一步，您可以在多个后端实例之间进行[负载均衡](/docs/caddyfile/directives/reverse_proxy#load-balancing)。您可以使用前端实例上的 [`acme_server`](/docs/caddyfile/directives/acme_server) 设置 mTLS（双向 TLS），使其充当后端实例的 CA（如果前端和后端之间的流量跨越不可信网络，这会很有用）。