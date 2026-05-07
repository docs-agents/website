---
title: Caddyfile 教程
---

# Caddyfile 教程

本教程将教你 [HTTP Caddyfile](/docs/caddyfile) 的基础知识，让你能快速、轻松地编写美观且功能完善的站点配置。

**目标：**
- 🔲 第一个站点
- 🔲 静态文件服务器
- 🔲 模板
- 🔲 压缩
- 🔲 多站点
- 🔲 匹配器
- 🔲 环境变量
- 🔲 注释

**前提条件：**
- 基本的终端/命令行技能
- 基本的文本编辑器技能
- `caddy` 已在 PATH 中

---

创建一个名为 `Caddyfile` 的新文本文件（无扩展名）。

首先，你需要输入站点的 [地址](/docs/caddyfile/concepts#addresses)：

```caddy
localhost
```

<aside class="tip">

如果你的操作系统上 HTTP 和 HTTPS 端口（分别为 80 和 443）是特权端口，那么你需要以提升的权限运行，或者使用更高的端口号。要使用更高的端口号，只需将地址改为类似 `localhost:2015` 的形式，并通过 [http_port](/docs/caddyfile/options) Caddyfile 选项更改 HTTP 端口。

</aside>

然后按回车，输入你想要它执行的操作。对于本教程，请将你的 Caddyfile 配置成这样：

```caddy
localhost

respond "Hello, world!"
```

保存后运行 Caddy（由于是培训教程，我们将使用 `--watch` 标志，这样对 Caddyfile 的修改会自动生效）：

```bash
caddy run --watch
```

<aside class="tip">

如果遇到权限错误，可以尝试在地址中使用更高的端口（如 `localhost:2015`）并 [更改 HTTP 端口](/docs/caddyfile/options)，或者以提升的权限运行。

</aside>

第一次运行时，会要求你输入密码。这是因为 Caddy 需要通过 HTTPS 提供你的站点服务。

<aside class="tip">

只要站点的地址中包含主机名或 IP，Caddy 默认就会通过 HTTPS 提供所有站点服务。可以通过在地址前显式添加 `http://` 来禁用 [自动 HTTPS](/docs/automatic-https)。

</aside>

<aside class="complete">第一个站点</aside>

在浏览器中打开 [localhost](https://localhost)，你将看到你的 Web 服务器正在运行，并且启用了 HTTPS！

<aside class="tip">
如果第一次出现证书错误，你可能需要重启浏览器。
</aside>

这没什么特别的，所以让我们将静态响应改为启用目录列表的 [文件服务器](/docs/caddyfile/directives/file_server)：

```caddy
localhost

file_server browse
```

保存 Caddyfile，然后刷新浏览器页面。如果当前目录中存在索引文件，你将看到文件列表或一个 HTML 页面。

<aside class="complete">静态文件服务器</aside>

## 添加功能

让我们为文件服务器添加一些有趣的功能：提供一个模板页面。创建一个新文件并粘贴以下内容：

```html
<!DOCTYPE html>
<html>
	<head>
		<title>Caddy 教程</title>
	</head>
	<body>
		页面加载时间：{{`{{`}}now | date "Mon Jan 2 15:04:05 MST 2006"{{`}}`}}
	</body>
</html>
```

将其保存为当前目录下的 `caddy.html`，然后在浏览器中加载：[https://localhost/caddy.html](https://localhost/caddy.html)

输出结果是：

```
页面加载时间：{{`{{`}}now | date "Mon Jan 2 15:04:05 MST 2006"{{`}}`}}
```

稍等，我们本应看到当前日期。为什么没生效？因为服务器尚未配置为解析模板！这很容易修复，只需在 Caddyfile 中添加一行，使其如下所示：

```caddy
localhost

templates
file_server browse
```

保存后，刷新浏览器页面。你应该看到：

```
页面加载时间：{{now | date "Mon Jan 2 15:04:05 MST 2006"}}
```

利用 Caddy 的 [模板模块](/docs/modules/http.handlers.templates)，你可以对静态文件做很多有用的事情，例如包含其他 HTML 文件、发起子请求、设置响应头、处理数据结构等！

<aside class="complete">模板</aside>

使用快速且现代的压缩算法来压缩响应是一种良好的实践。让我们使用 [`encode`](/docs/caddyfile/directives/encode) 指令启用 Gzip 和 Zstandard 支持：

```caddy
localhost

encode
templates
file_server browse
```

<aside class="complete">压缩</aside>

以上就是让一个半高级、生产就绪的站点运行起来的基本流程！

当你准备好启用 [自动 HTTPS](/docs/automatic-https) 时，只需将站点地址（本教程中的 `localhost`）替换为你的域名即可。更多信息请参阅我们的 [HTTPS 快速入门指南](/docs/quick-starts/https)。

## 多站点

使用目前的 Caddyfile，我们只能定义一个站点！只有第一行可以是站点的地址，文件其余部分都必须是指令。

但我们可以轻松地添加更多站点！

我们目前使用的 Caddyfile：

```caddy
localhost

encode
templates
file_server browse
```

等价于以下这种形式：

```caddy
localhost {
	encode
	templates
	file_server browse
}
```

第二种形式允许我们添加更多站点。

通过用花括号 `{ }` 包裹站点块，我们可以在同一个 Caddyfile 中定义多个不同的站点。

例如：

```caddy
:8080 {
	respond "I am 8080"
}

:8081 {
	respond "I am 8081"
}
```

当用花括号包裹站点块时，只有 [地址](/docs/caddyfile/concepts#addresses) 出现在花括号外部，只有 [指令](/docs/caddyfile/directives) 出现在花括号内部。

对于共享相同配置的多个站点，你可以添加更多地址，例如：

```caddy
:8080, :8081 {
	...
}
```

然后你可以根据需要定义任意多个不同的站点，只要每个地址是唯一的。

<aside class="complete">多站点</aside>

## 匹配器

我们可能希望仅对某些请求应用某些指令。例如，假设我们既想要文件服务器又想要反向代理，但显然不能在每个请求上都同时执行两者！要么文件服务器提供静态文件响应，要么反向代理将请求传递给后端并返回其响应。

以下配置不会按我们预期的方式工作（由于 [指令顺序](/docs/caddyfile/directives#directive-order)，`reverse_proxy` 会优先执行）：

```caddy
localhost

file_server
reverse_proxy 127.0.0.1:9005
```

实际上，我们可能只想对 API 请求使用反向代理，即路径以 `/api/` 开头的请求。通过添加一个 [匹配器标记](/docs/caddyfile/matchers#syntax)，很容易实现这一点：

```caddy
localhost

reverse_proxy /api/* 127.0.0.1:9005
file_server
```

这样一来，所有以 `/api/` 开头的请求都会优先被反向代理处理。

我们刚刚添加的 `/api/*` 部分称为 **匹配器标记**。你可以通过它以正斜杠 `/` 开头、且紧跟在指令后面来识别它（不过你也可以查阅 [指令文档](/docs/caddyfile/directives) 来确认）。

匹配器非常强大。你可以声明具名匹配器，并使用类似 `@name` 的方式匹配更多内容，而不仅仅是请求路径！在继续之前，请花点时间 [了解更多关于匹配器的信息](/docs/caddyfile/matchers)。

<aside class="complete">匹配器</aside>

## 环境变量

Caddyfile 适配器允许在解析 Caddyfile 之前替换 [环境变量](/docs/caddyfile/concepts#environment-variables)。

首先，设置一个环境变量（在与运行 Caddy 相同的 shell 中）：

```bash
export SITE_ADDRESS=localhost:9055
```

然后在 Caddyfile 中这样使用它：

```caddy
{$SITE_ADDRESS}

file_server
```

在解析 Caddyfile 之前，它会展开为：

```caddy
localhost:9055

file_server
```

你可以在 Caddyfile 中的任何位置、为任意数量的标记使用环境变量。

<aside class="complete">环境变量</aside>

## 注释

最后一件非常有帮助的事情：如果你想在 Caddyfile 中添加备注或注释，可以使用以 `#` 开头的注释：

```caddy
# 这是一条注释
```

<aside class="complete">注释</aside>

## 延伸阅读

- [Caddyfile 概念](/docs/caddyfile/concepts)
- [指令](/docs/caddyfile/directives)
- [常见模式](/docs/caddyfile/patterns)