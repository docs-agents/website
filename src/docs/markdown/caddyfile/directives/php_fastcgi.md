---
title: php_fastcgi (Caddyfile 指令)
---

<script>
ready(function() {
	// 如果页面上找到匹配的锚点标签，则向所有子指令添加链接。
	addLinksToSubdirectives();
});
</script>

# php_fastcgi

一个带有明确预设的指令，用于将请求代理到 PHP FastCGI 服务器（如 php-fpm）。

- [语法](#syntax)
- [展开形式](#expanded-form)
  - [说明](#explanation)
- [示例](#examples)

Caddy 的 [`reverse_proxy`](reverse_proxy) 能够处理任何 FastCGI 应用，但本指令专为 PHP 应用量身定制。本指令是一个便捷的快捷方式，可替代[更长的配置](#expanded-form)。

它假设站点根目录下的 `index.php` 充当路由器。如果不需要此行为，可以重新配置 [`try_files` 子指令](#try_files) 修改默认的重写行为，或者以[展开形式](#expanded-form)为基础进行自定义。

除了下面列出的子指令外，本指令还支持 [`reverse_proxy`](reverse_proxy#syntax) 的所有子指令。例如，您可以启用负载均衡和健康检查。

**大多数现代 PHP 应用无需额外的子指令或定制即可正常工作。** 子指令通常仅在特定边缘情况或与旧版 PHP 应用一起使用时才需要。

## 语法

```caddy-d
php_fastcgi [<匹配器>] <php-fpm_网关...> {
	root <路径>
	split <子字符串...>
	index <文件名>|off
	try_files <文件...>
	env [<键> <值>]
	resolve_root_symlink
	capture_stderr
	dial_timeout  <持续时间>
	read_timeout  <持续时间>
	write_timeout <持续时间>

	<any other reverse_proxy subdirectives...>
}
```

- **<php-fpm_网关...>** 是 FastCGI 服务器的[地址](/docs/conventions#network-addresses)。通常是 TCP 套接字或 Unix 套接字文件。

- **root** <span id="root"/> 设置站点的根文件夹。建议始终将 [`root` 指令](root) 与 `php_fastcgi` 结合使用，但覆盖此设置可在您的 PHP-FPM 上游使用与 Caddy 不同的根目录时有用（参见[示例](#docker)）。如果使用了 [`root` 指令](root)，则默认为其值；否则默认为 Caddy 的当前工作目录。

- **split** <span id="split"/> 设置用于将 URI 拆分为两部分的子字符串。第一个匹配的子字符串将用于从路径中拆分出“路径信息”。第一部分以匹配的子字符串结尾，将被假定为实际资源（CGI 脚本）名称。第二部分将设置为 PATH_INFO，供 CGI 脚本使用。默认值：`.php`

- **index** <span id="index"/> 指定被视为目录索引文件的文件名。这会影响[展开形式](#expanded-form)中的文件匹配器。默认值：`index.php`。可以设置为 `off`，以在未找到匹配文件时禁用回退到 `index.php` 的重写。

- **try_files** <span id="try_files"/> 指定替代默认尝试文件重写的设置。详细信息请参见 [`try_files` 指令](try_files)。默认值：`{path} {path}/index.php index.php`。

- **env** <span id="env"/> 设置一个额外的环境变量及其值。可多次指定以设置多个环境变量。默认情况下，所有相关的 FastCGI 环境变量（包括 HTTP 头部）已经设置，但您可以根据需要添加或覆盖变量。

- **resolve_root_symlink** <span id="resolve_root_symlink"/> 当 [`root`](#root) 目录是符号链接时，启用解析为其实际值。这有时用作部署策略，简单地通过交换符号链接来指向另一个目录中的新版本。默认禁用，以避免重复的系统调用。

- **capture_stderr** <span id="capture_stderr"/> 启用捕获并记录上游 FastCGI 服务器在 `stderr` 上发送的任何消息。默认情况下，以 `WARN` 级别记录。如果响应具有 `4xx` 或 `5xx` 状态码，则使用 `ERROR` 级别。默认情况下，忽略 `stderr`。

- **dial_timeout** <span id="dial_timeout"/> 是一个[持续时间值](/docs/conventions#durations)，设置连接上游套接字时等待的时间。默认值：`3s`。

- **read_timeout** <span id="read_timeout"/> 是一个[持续时间值](/docs/conventions#durations)，设置从 FastCGI 上游读取时等待的时间。默认值：无超时。

- **write_timeout** <span id="write_timeout"/> 是一个[持续时间值](/docs/conventions#durations)，设置向 FastCGI 上游发送时等待的时间。默认值：无超时。

由于本指令是反向代理的预设封装，您可以使用 [`reverse_proxy`](reverse_proxy#syntax) 的任何子指令进行自定义。

## 展开形式

`php_fastcgi` 指令（不带子指令）等同于以下配置。大多数现代 PHP 应用在此预设下运行良好。如果您的应用不适用，可以引用此配置并根据需要进行自定义，而不是使用 `php_fastcgi` 快捷方式。

```caddy-d
route {
	# 为目录请求添加尾部斜杠
	# 如果 "{http.request.uri.path}/index.php" 未出现在 try_files 列表中，
	# 则此重定向自动禁用
	@canonicalPath {
		file {path}/index.php
		not path */
	}
	redir @canonicalPath {http.request.orig_uri.path}/ 308

	# 如果请求的文件不存在，尝试索引文件并假设 index.php 始终存在
	@indexFiles file {
		try_files {path} {path}/index.php index.php
		try_policy first_exist_fallback
		split_path .php
	}
	rewrite @indexFiles {file_match.relative}

	# 将 PHP 文件代理到 FastCGI 响应器
	@phpFiles path *.php
	reverse_proxy @phpFiles <php-fpm_网关> {
		transport fastcgi {
			split .php
		}
	}
}
```

### 说明

- 第一部分处理请求路径的规范化。目标确保针对磁盘上目录的请求具有尾部斜杠 `/`，从而使得对该目录的请求只有一个有效 URL。

  仅当 `try_files` 子指令包含 `{path}/index.php`（默认情况）时，才执行此规范化。

  这是通过使用一个请求匹配器来实现的，该匹配器仅匹配不以斜杠结尾的请求，并且这些请求映射到磁盘上包含 `index.php` 文件的目录。如果匹配，则执行 HTTP 308 重定向并附加尾部斜杠。例如，如果 `/foo/index.php` 存在于磁盘上，它将对路径为 `/foo` 的请求重定向到 `/foo/`（附加 `/`，以规范化目录的路径）。

- 下一部分根据磁盘上是否存在匹配的文件来执行路径重写。这还会记住路径中 `.php` 之后的部分（如果请求路径中包含 `.php`）。这对于 Caddy 正确设置 FastCGI 环境变量非常重要。

  - 首先，检查 `{path}` 是否为磁盘上存在的文件。如果是，则重写为该路径。这实际上会短路其余步骤，并确保对磁盘上确实存在的文件的请求不会被其他方式重写（请参见下面的步骤）。例如，如果磁盘上有一个 `/js/app.js` 文件，则对该路径的请求将保持不变。

  - 其次，检查 `{path}/index.php` 是否为磁盘上存在的文件。如果是，则重写为该路径。对于像 `/foo/` 这样的目录请求，它将查找 `/foo//index.php`（被规范化为 `/foo/index.php`），如果存在则重写为该路径。当您在 Web 根目录的子目录中运行另一个 PHP 应用时，此行为有时很有用。

  - 最后，它将始终重写为 `index.php`（对于现代 PHP 应用，它几乎总是存在的）。这允许您的 PHP 应用通过使用 `index.php` 脚本作为入口点来处理任何不映射到磁盘上文件的路径的请求。

- 最后一部分是将请求代理到您的 PHP FastCGI（或 PHP-FPM）服务以实际运行 PHP 代码。请求匹配器仅匹配以 `.php` 结尾的请求。因此，任何不是 PHP 脚本并且确实存在于磁盘上的文件将不由本指令处理，并将继续传递。

通常，仅靠 `php_fastcgi` 指令是不够的。它几乎总是需要与 [`root` 指令](root) 配对，以设置磁盘上文件的位置（对于现代 PHP 应用，这可能是 `/var/www/html/public`，其中 `public` 目录包含您的 `index.php`），以及与 [`file_server` 指令](file_server) 配对，以提供本指令未处理且继续传递的静态文件（JS、CSS、图像等）。

## 示例

将所有 PHP 请求代理到监听在 `127.0.0.1:9000` 的 FastCGI 响应器：

```caddy-d
php_fastcgi 127.0.0.1:9000
```

相同，但仅针对 `/blog/` 下的请求：

```caddy-d
php_fastcgi /blog/* localhost:9000
```

当使用通过 Unix 套接字监听的 PHP-FPM 时：

```caddy-d
php_fastcgi unix//run/php/php8.2-fpm.sock
```

[`root` 指令](root) 几乎总是用于指定包含 PHP 脚本的目录，[`file_server` 指令](file_server) 用于服务静态文件：

```caddy
example.com {
	root /var/www/html/public
	php_fastcgi 127.0.0.1:9000
	file_server
}
```

<span id="docker"/> 当使用 Caddy 服务多个 PHP 应用时，每个应用的 Web 根目录必须不同，以便 Caddy 可以分别读取和服务静态文件，并检测 PHP 文件是否存在。

如果您使用 Docker，通常您的 PHP-FPM 容器会将文件挂载到相同的根目录。在这种情况下，解决方案是将文件挂载到 Caddy 容器中的不同目录，然后使用 [`root` 子指令](#root) 为每个容器设置根目录：

```caddy
app1.example.com {
	root /srv/app1/public
	php_fastcgi app1:9000 {
		root /var/www/html/public
	}
	file_server
}

app2.example.com {
	root /srv/app2/public
	php_fastcgi app2:9000 {
		root /var/www/html/public
	}
	file_server
}
```

对于不使用 `index.php` 作为入口点的 PHP 站点，您可以回退到发出 `404` 错误。可以使用 [`handle_errors` 指令](handle_errors) 捕获并处理该错误：

```caddy
example.com {
	php_fastcgi localhost:9000 {
		try_files {path} {path}/index.php =404
	}

	handle_errors {
		respond "{err.status_code} {err.status_text}"
	}
}