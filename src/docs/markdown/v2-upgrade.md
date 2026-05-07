---
title: 升级至 Caddy 2
---

升级指南
=======

Caddy 2 是一个全新的代码库，从头编写，旨在改进 Caddy 1。Caddy 2 不向后兼容 Caddy 1。但别担心，对于大多数基本配置，差异并不大。本指南将帮助您尽可能轻松地完成过渡。

本指南不会深入介绍新功能——其实它们非常酷，您应该[了解一下](/docs/getting-started)——这里的目标是让您快速上手 Caddy 2。

- [重要提示](#重要提示)
- [步骤](#步骤)
- [HTTPS 与端口](#https-与端口)
- [命令行](#命令行)
- [Caddyfile](#caddyfile)
	- [主要变化](#主要变化)
	- [basicauth](#basicauth)
	- [browse](#browse)
	- [errors](#errors)
	- [ext](#ext)
	- [fastcgi](#fastcgi)
	- [gzip](#gzip)
	- [header](#header)
	- [log](#log)
	- [proxy](#proxy)
	- [redir](#redir)
	- [rewrite](#rewrite)
	- [root](#root)
	- [status](#status)
	- [templates](#templates)
	- [tls](#tls)
- [服务文件](#服务文件)
- [插件](#插件)
- [获取帮助](#获取帮助)

## 重要提示

- “Caddy 2” 仍简称为 `caddy`。我们可能会用“Caddy 2”来明确版本，以减少混乱。
- 大多数用户只需替换 `caddy` 二进制文件和更新后的 `Caddyfile` 配置（测试通过后）。
- 最好带着空杯心态进入 Caddy 2，不要沿用 Caddy 1 的假设。
- 您可能无法在 v2 中完美复现 v1 的特定配置，通常这是有充分理由的。
- 命令行不再用于服务器配置。
- 环境变量不再需要用于配置。
- 为 Caddy 2 提供配置的主要方式是通过其 [API](/docs/api)，但也可以使用 [`caddy` 命令](/docs/command-line)。
- 您应该知道 Caddy 2 的原生配置语言是 [JSON](/docs/json/)，而 Caddyfile 只是一个[配置适配器](/docs/config-adapters)，为您转换成 JSON。非常定制/高级的用例可能需要 JSON，因为 Caddyfile 无法表达所有可能的配置。
- Caddyfile 大致相同，但更强大；指令已经改变。

## 步骤

1. 通过我们的[入门](/docs/getting-started)教程熟悉 Caddy 2。
2. 如果还没做，请先完成第 1 步。说真的——至少要知道如何使用 Caddy 2 有多重要，我们怎么强调都不为过。（而且更有趣！）
3. 使用下面的指南转换您的 `caddy` 命令。
4. 使用下面的指南转换您的 Caddyfile。
5. 在本地或预发布环境中测试您的新配置。
6. 测试、再测试、再再测试。
7. 部署并享受吧！

## HTTPS 与端口

Caddy 的默认端口不再是 `:2015`。Caddy 2 的默认端口是 `:443`，或者如果没有已知的主机名/IP，则使用端口 `:80`。您可以在配置中随时自定义端口。

Caddy 2 的默认协议是：如果已知主机名或 IP，则 _始终_ 使用 HTTPS](/docs/automatic-https#overview)。这与 Caddy 1 不同，在 Caddy 1 中只有看起来公开的域名默认使用 HTTPS。现在，_每个_ 站点都使用 HTTPS（除非您通过明确指定端口 `:80` 或 `http://` 来禁用它）。

IP 地址和 localhost 域名将从[本地信任的内置 CA](/docs/automatic-https#local-https) 获取证书。所有其他域名将使用 ZeroSSL 或 Let's Encrypt。（这些都是可配置的。）

证书和 ACME 资源的存储结构已经改变。Caddy 2 可能会为您的站点获取新证书；但如果您有很多证书，并且 Caddy 没有自动迁移，您可以手动迁移。详情请参见 issue [#2955](https://github.com/caddyserver/caddy/issues/2955) 和 [#3124](https://github.com/caddyserver/caddy/issues/3124)。

## 命令行

`caddy` 命令现在变成了 `caddy run`。

所有命令行标志都已改变。请移除它们；所有服务器配置现在都存在于实际配置文档中（通常是 Caddyfile 或 JSON）。您可能会在 [JSON 结构](/docs/json/) 或 [Caddyfile 全局选项](/docs/caddyfile/options) 中找到替换 v1 命令行标志所需的内容。

像 `caddy -conf ../Caddyfile` 这样的命令将变为 `caddy run --config ../Caddyfile`。

和以前一样，如果您的 Caddyfile 在当前文件夹中，Caddy 会自动找到并使用它；在这种情况下，您不需要使用 `--config` 标志。

信号基本保持不变，但不再支持 USR1 和 USR2。请改用 [`caddy reload`](/docs/command-line#caddy-reload) 命令或 [API](/docs/api) 来加载新配置。

不带任何配置运行 `caddy` 曾经是一个简单的文件服务器。在 Caddy 2 中，对应的命令是 [`caddy file-server`](/docs/command-line#caddy-file-server)。

环境变量不再相关，除了 `HOME`（以及您设置的任意 `XDG_*` 变量）。`CADDYPATH` 已被[操作系统惯例](/docs/conventions#file-locations)取代。

## Caddyfile

[v2 Caddyfile](/docs/caddyfile/concepts) 与您已经熟悉的非常相似。您主要需要做的是更改指令。

⚠️ **请务必阅读新指令的文档！** 特别是如果您的配置比较复杂，会有很多细微之处需要考虑。这些提示能让您快速上手，但请阅读每个指令的完整文档，以便理解升级的含义。当然，在投入生产前，一定要全面测试配置。

### 主要变化

- 如果您要提供静态文件服务，需要添加 [`file_server` 指令](/docs/caddyfile/directives/file_server)，因为 Caddy 2 默认不会这样做。出于安全原因，Caddy 2 默认也不会嗅探 MIME 类型；如果缺少 Content-Type，您可能需要使用 [header](/docs/caddyfile/directives/header) 指令自行设置。
- 在 v1 中，您只能通过请求路径过滤（或“匹配”）指令。在 v2 中，[请求匹配](/docs/caddyfile/matchers) 更加强大。任何向 HTTP 处理链添加中间件或以任何方式操作 HTTP 请求/响应的 v2 指令都利用了这种新的匹配功能。[阅读更多关于 v2 请求匹配器。](/docs/caddyfile/matchers) 您需要了解它们才能理解 v2 Caddyfile。
- 尽管许多[占位符](/docs/conventions#placeholders)相同，但许多已经改变，并且现在有[许多新的占位符](/docs/modules/http#docs)，包括 [Caddyfile 的简写](/docs/caddyfile/concepts#placeholders)。
- Caddy 2 日志都是结构化的，默认格式为 JSON。所有日志级别都可以发送到同一个日志进行处理（但您可以根据需要自定义）。
- 在 Caddy 1 中，您使用请求路径前缀进行匹配；在 Caddy 2 中，路径匹配默认是精确的。如果您想匹配像 `/foo/` 这样的前缀，需要在 Caddy 2 中使用 `/foo/*`。

我们在这里列出一些最常见的 v1 指令，并描述如何转换为 v2 Caddyfile 中使用。

⚠️ **本页缺少某个 v1 指令并不意味着 v2 无法实现！** 某些 v1 指令已不再需要、不易转换，或在 v2 中以其他方式实现。对于某些高级定制，您可能需要降级到 JSON 来获得所需功能。探索[我们的文档](/docs/caddyfile)以找到您需要的内容！

### basicauth

HTTP 基本认证仍然通过 [`basic_auth`](/docs/caddyfile/directives/basic_auth) 指令配置。但是，Caddy 2 配置不接受明文密码。您必须对密码进行哈希处理，可以使用 [`caddy hash-password`](/docs/command-line#caddy-hash-password) 命令。

- **v1:**
```
basicauth /secret/ Bob hiccup
```

- **v2:**
```caddy-d
basic_auth /secret/* {
	Bob JDJhJDEwJEVCNmdaNEg2Ti5iejRMYkF3MFZhZ3VtV3E1SzBWZEZ5Q3VWc0tzOEJwZE9TaFlZdEVkZDhX
}
```

### browse

文件浏览现在通过 [`file_server`](/docs/caddyfile/directives/file_server) 指令启用。

- **v1:**
```
browse /subfolder/
```
- **v2:**
```caddy-d
file_server /subfolder/* browse
```

### errors

自定义错误页面可以通过 [`handle_errors`](/docs/caddyfile/directives/handle_errors) 实现。

- **v1:**
```
errors {
	404 404.html
	500 500.html
}
```

- **v2:**
```
handle_errors {
	rewrite /{err.status_code}.html
	file_server
}
```

### ext

隐含的文件扩展名可以通过 [`try_files`](/docs/caddyfile/directives/try_files) 实现。

- **v1:** `ext .html`
- **v2:** `try_files {path}.html {path}`

### fastcgi

假设您正在为 PHP 提供服务，v2 的对应指令是 [`php_fastcgi`](/docs/caddyfile/directives/php_fastcgi)。

- **v1:**
```
fastcgi / localhost:9005 php
```
- **v2:**
```caddy-d
php_fastcgi localhost:9005
```

请注意，v1 的 `fastcgi` 指令在幕后做了很多事情，包括尝试磁盘上的文件、重写请求甚至重定向。v2 的 `php_fastcgi` 指令也会为您做这些事情，但其文档提供了[展开形式](/docs/caddyfile/directives/php_fastcgi#expanded-form)，如果您的需求不同，可以修改。

v2 中不需要 `php` 预设，因为 `php_fastcgi` 指令默认假定 PHP。像 `php_fastcgi 127.0.0.1:9000 php` 这样的行会使反向代理认为存在一个名为 `php` 的后端，导致连接错误。

v2 中的子指令不同——您可能不需要为 PHP 设置任何子指令。

### gzip

单个指令 [`encode`](/docs/caddyfile/directives/encode) 现在用于所有响应编码，包括多种压缩格式。

- **v1:**
```
gzip
```
- **v2:**
```caddy-d
encode gzip
```

有趣的事实：Caddy 2 也支持 `zstd`（但还没有浏览器支持）。

### header

[基本不变](/docs/caddyfile/directives/header)，但在 v2 中更强大，因为它可以进行子字符串替换。

- **v1:**
```
header / Strict-Transport-Security max-age=31536000;
```
- **v2:**
```caddy-d
header Strict-Transport-Security max-age=31536000;
```

### log

启用访问日志；[`log`](/docs/caddyfile/directives/log) 指令在 v2 中仍可使用，但所有日志默认是结构化 JSON 编码。

启用访问日志的推荐方式简单如下：

```caddy-d
log
```

它将结构化日志输出到 stderr。（您也可以输出到文件或网络套接字；请参阅 [`log`](/docs/caddyfile/directives/log) 指令文档。）

默认情况下，日志将采用[结构化](/docs/logging) JSON 格式。如果出于遗留原因仍需要通用日志格式（CLF），您可以使用 [`transform-encoder`](https://github.com/caddyserver/transform-encoder) 插件。

### proxy

v2 的对应指令是 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy)。

值得注意的子指令变化包括 `header_upstream` 和 `header_downstream` 分别变更为 `header_up` 和 `header_down`；与负载均衡相关的子指令前缀为 `lb_`。

另一个显著区别是 v2 代理默认通过所有传入头部（包括 `Host` 头部）并设置 `X-Forwarded-For` 头部。换句话说，v1 的“透明”模式基本上是 v2 的默认模式（但如果您需要其他头部如 X-Real-IP，则需要自行设置）。您仍可以使用 `header_up` 子指令覆盖/自定义 `Host` 头部。

WebSocket 代理在 v2 中“开箱即用”；无需像 v1 那样“启用” WebSocket。

`without` 子指令已被移除，因为 v2 中改进的匹配器支持不再需要[重写技巧](#rewrite)。

- **v1:**
```
proxy / localhost:9005
```
- **v2:**
```caddy-d
reverse_proxy localhost:9005
```

### redir

[基本不变](/docs/caddyfile/directives/redir)，除了可选状态码参数的一些细节。大多数配置无需更改。

- **v1:** `redir https://example.com{uri}`
- **v2:** `redir https://example.com{uri}`

### rewrite

请求重写（“内部重定向”）的语义略有变化。如果您在 v1 中使用所谓的“重写技巧”来匹配除简单路径前缀以外的请求，这在 v2 中完全不需要。

[新的 `rewrite` 指令](/docs/caddyfile/directives/rewrite) 非常简单但功能强大，因为其大部分复杂性由 v2 中的[匹配器](/docs/caddyfile/matchers)处理：

- **v1:**
```
rewrite {
	if {>User-Agent} has mobile
	to /mobile{uri}
}
```
- **v2:**
```caddy-d
@mobile {
	header User-Agent *mobile*
}
rewrite @mobile /mobile{uri}
```

注意我们只是使用 Caddy 2 常用的[匹配器标记](/docs/caddyfile/matchers)；它不再是指令的特殊情况。

从移除所有重写技巧开始；将它们转换为[命名匹配器](/docs/caddyfile/concepts#named-matchers) 代替。评估每个 v1 `rewrite` 是否在 v2 中确实需要。提示：一个 v1 Caddyfile 使用 `rewrite` 添加路径前缀，然后使用 `proxy` 配合 `without` 移除相同前缀，这是重写技巧，可以消除。

您可能会发现新的 [`route`](/docs/caddyfile/directives/route) 和 [`handle`](/docs/caddyfile/directives/handle) 指令在更好地控制高级路由逻辑方面很有用。

### root

[基本不变](/docs/caddyfile/directives/root)。

如果提供静态文件，请记得添加 [`file_server` 指令](/docs/caddyfile/directives/file_server)，因为 Caddy 2 默认不假定这样做，而 v1 始终启用它。

### status

v2 的对应指令是 [`respond`](/docs/caddyfile/directives/respond)，它还可以写入响应体。

- **v1:**
```
status 404 /secrets/
```
- **v2:**
```caddy-d
respond /secrets/* 404
```

### templates

[`templates`](/docs/caddyfile/directives/templates) 指令的总体语法不变，但实际的模板动作/函数已不同且改进很大。例如，模板能够包含文件、渲染 Markdown、执行内部子请求、解析前言元数据等等！

[查看文档](/docs/modules/http.handlers.templates)了解新函数的详细信息。

- **v1:** `templates`
- **v2:** `templates`

### tls

[`tls`](/docs/caddyfile/directives/tls) 指令的基础没有改变，例如指定您自己的证书和密钥：

- **v1:** `tls cert.pem key.pem`
- **v2:** `tls cert.pem key.pem`

但 Caddy 的[自动 HTTPS 逻辑](/docs/automatic-https) _已经_改变，请注意！

密码套件名称也已改变。

Caddy 2 中一个常见的配置是使用 `tls internal` 来为不是 `localhost` 或 IP 地址的开发主机名提供本地信任的证书。

大多数站点根本不需要这个指令。

## 服务文件

对于 Caddy 部署，我们建议使用[我们官方的 systemd 服务文件之一](/docs/running#linux-service)。

如果您需要自定义服务文件，请以我们提供的为基础。它们经过精心调整，有充分的理由！如果需要，请务必自定义您的文件。

## 插件

为 v1 编写的插件不会自动与 v2 兼容。许多 v1 插件在 v2 中甚至不需要。另一方面，v2 比 v1 更容易扩展和灵活！

如果您想为 Caddy 2 编写插件，请[学习如何编写 Caddy 模块](/docs/extending-caddy)。

### 使用插件构建 Caddy 2

您可以在[交互式下载页面](/download)下载带插件的 Caddy 2。或者，您也可以使用 `xcaddy` [自己构建 Caddy](/docs/build)，并选择要包含的插件。`xcaddy` 可以自动化 Caddy 的 [main.go](https://github.com/caddyserver/caddy/blob/master/cmd/caddy/main.go) 文件中的指令。

## 获取帮助

如果您在让 Caddy 正常工作时遇到困难，请先浏览我们的网站文档。花时间尝试新事物并理解发生了什么——v2 在很多方面与 v1 非常不同（但也有很多相似之处）！

如果您仍需要帮助，请加入[我们的社区](https://caddy.community)！您可能会发现帮助他人也是帮助自己的最佳方式。