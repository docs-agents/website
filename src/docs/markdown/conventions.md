---
title: 约定
---

# 约定

Caddy 生态系统遵循一些约定，以确保跨平台的一致性和直观性。


- [网络地址](#网络地址)
- [占位符](#占位符)
- [文件位置](#文件位置)
  - [数据目录](#数据目录)
  - [配置目录](#配置目录)
- [持续时间](#持续时间)



## 网络地址

指定用于连接或绑定的网络地址时，Caddy 接受以下格式的字符串：

```
network/address
```

网络部分是可选的（默认为 `tcp`），可以是 [Go 的 `net.Dial` 函数](https://pkg.go.dev/net#Dial) 识别的任何内容。如果指定了网络，则必须使用单个正斜杠 `/` 分隔网络和地址部分。

网络可以是以下任一类型；后缀为 `4` 或 `6` 的分别仅限 IPv4 或 IPv6：

- TCP：`tcp`、`tcp4`、`tcp6`
- UDP：`udp`、`udp4`、`udp6`
- IP：`ip`、`ip4`、`ip6`
- Unix：`unix`、`unixgram`、`unixpacket`

地址部分可以是以下任何形式：

- `host`
- `host:port`
- `:port`
- `[ipv6%zone]:port`
- `/path/to/unix/socket`
- `/path/to/unix/socket|0200`

主机可以是任何主机名、可解析域名或 IP 地址。

对于 IPv6 地址，地址必须括在方括号 `[]` 中。区域标识符（以 `%` 开头）是可选的（通常用于链路本地地址）。

端口可以是单个值（`:8080`）或包含范围的端口（`:8080-8085`）。端口范围将被扩展为多个单个地址。并非所有配置字段都接受端口范围。特殊端口 `:0` 表示任何可用端口。

仅在网络类型为 `unix*` 时才接受 Unix 套接字路径。分隔网络和地址的正斜杠不被视为路径的一部分。

当 Unix 套接字用作绑定地址时，您可以选择在路径后使用竖线 `|` 指定文件权限模式。默认为 `0200`（八进制），即 `u=w,g=,o=`（符号表示）。前导 `0` 是可选的。

有效示例：

```
:8080
127.0.0.1:8080
localhost:8080
localhost:8080-8085
tcp/localhost:8080
tcp/localhost:8080-8085
udp/localhost:9005
[::1]:8080
tcp6/[fe80::1%eth0]:8080
unix//path/to/socket
unix//path/to/socket|0200
```

<aside class="tip">

Caddy 的网络地址不是 URL。URL 将 [OSI 模型 <img src="/old/resources/images/external-link.svg" class="external-link">](https://en.wikipedia.org/wiki/OSI_model#Layer_architecture) 的较低层和较高层耦合在一起，但 Caddy 通常独立于特定应用使用网络地址，因此将它们组合起来会有问题。在 Caddy 中，网络地址精确地指代可以在 L3-L5 层连接或绑定的资源，而 URL 则组合了 L3-L7 层，层次过多。网络地址要求主机+端口和路径互斥，但 URL 不要求。网络地址有时支持端口范围，但 URL 不支持。

</aside>




## 占位符

Caddy 的配置支持使用*占位符*。使用占位符是将动态值注入静态配置的一种简单方法。

<aside class="tip">

占位符类似于其他软件中的变量。例如，[nginx 有变量 <img src="/old/resources/images/external-link.svg" class="external-link">](https://nginx.org/en/docs/varindex.html) 如 `$uri` 和 `$document_root`，而 Caddy 中的对应项是 [`{http.request.uri}`](/docs/json/apps/http/#docs) 和 [`{http.vars.root}`](/docs/caddyfile/directives/root)。

</aside>


占位符由花括号 `{ }` 括起，内部包含标识符，例如：`{foo.bar}`。开头的占位符花括号可以通过 `\{like.this}` 转义以避免替换。占位符标识符通常用点号命名空间，以避免模块间冲突。

哪些占位符可用取决于上下文。并非所有占位符在配置的所有部分都可用。例如，[HTTP 应用设置的占位符](/docs/json/apps/http/#docs) 仅在与处理 HTTP 请求相关的配置区域中可用。当请求通过 [`reverse_proxy` 处理器](/docs/json/apps/http/servers/routes/handle/reverse_proxy/#docs) 时，处理器会设置几个特定于代理的占位符。这些占位符可以在代理期间以及之后（在 `handle_response` 中）被引用，例如在设置响应头或丰富访问日志时。

以下占位符始终可用（全局）：

占位符 | 描述
------------|-------------
`{env.*}` | 环境变量；示例：`{env.HOME}`
`{file.*}` | 文件内容；示例：`{file./path/to/secret.txt}`
`{system.hostname}` | 系统的本地主机名
`{system.slash}` | 系统的文件路径分隔符
`{system.os}` | 系统的操作系统
`{system.arch}` | 系统架构
`{system.wd}` | 当前工作目录
`{time.now}` | 当前时间（Go Time 结构体）
`{time.now.http}` | [HTTP 标头 <img src="/old/resources/images/external-link.svg" class="external-link">](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified) 中使用的格式的当前时间
`{time.now.unix}` | 当前时间的 Unix 时间戳（秒）
`{time.now.unix_ms}` | 当前时间的 Unix 时间戳（毫秒）
`{time.now.common_log}` | 通用日志格式的当前时间
`{time.now.year}` | 当前年份（YYYY 格式）

并非所有配置字段都支持占位符，但大多数您期望的地方都支持。占位符的支持需要显式添加到这些字段中。插件作者可以[阅读本文](/docs/extending-caddy/placeholders)了解如何在自己的模块中添加对占位符的支持。




## 文件位置

本节包含如何查找各种文件的信息。此处描述的文件和目录路径最多只是默认值；有些可以被覆盖。

### 您的配置文件

没有单个、约定俗成的位置存放您的配置文件。将它们放在您认为最合理的地方。

<aside class="tip">

唯一的例外可能是当前工作目录中名为 `Caddyfile` 的文件，如果未指定其他配置文件，caddy 命令会为了方便而尝试使用它。

</aside>


附带默认配置文件的发行版应记录此配置文件的位置，即使对于包/发行版维护者来说可能很明显。对于大多数 Linux 安装，Caddyfile 位于 `/etc/caddy/Caddyfile`。


### 数据目录

Caddy 将 TLS 证书和其他重要资产存储在数据目录中，该目录由[配置的存储模块](/docs/json/storage/)（默认：本地文件系统）支持。

如果设置了 `XDG_DATA_HOME` 环境变量，则为 `$XDG_DATA_HOME/caddy`。

否则，其路径因平台而异，遵守操作系统的约定：

操作系统 | 数据目录路径
---|---------------------
**Linux、BSD** | `$HOME/.local/share/caddy`
**Windows** | `%AppData%\Caddy`
**macOS** | `$HOME/Library/Application Support/Caddy`
**Plan 9** | `$HOME/lib/caddy`
**Android** | `$HOME/caddy`（或 `/sdcard/caddy`）

所有其他操作系统都使用 Linux/BSD 的目录路径。

**数据目录绝不能被视为缓存。** 其内容**不是**临时性的或仅仅为了性能。Caddy 将 TLS 证书、私钥、OCSP 装订以及其他必要信息存储到数据目录。不经理解后果不应清除它。

此目录必须持久化且可由 Caddy 写入，这一点至关重要。


### 配置目录

这是 Caddy 可能将某些配置存储到磁盘的地方。最值得注意的是，它（默认情况下）将最后的活动配置持久化到此文件夹，以便稍后使用 [`caddy run --resume`](/docs/command-line#caddy-run) 轻松恢复。

<aside class="tip">

配置目录*不是*您需要存放[配置文件](#您的配置文件)的地方。（不过，您也可以存放。）

</aside>


如果设置了 `XDG_CONFIG_HOME` 环境变量，则为 `$XDG_CONFIG_HOME/caddy`。

否则，其路径因平台而异，遵守操作系统的约定：


操作系统 | 配置目录路径
---|---------------------
**Linux、BSD** | `$HOME/.config/caddy`
**Windows** | `%AppData%\Caddy`
**macOS** | `$HOME/Library/Application Support/Caddy`
**Plan 9** | `$HOME/lib/caddy`

所有其他操作系统都使用 Linux/BSD 的目录路径。

此目录必须持久化且可由 Caddy 写入，这一点至关重要。


## 持续时间

持续时间字符串在 Caddy 的配置中广泛使用。它们采用与 [Go 的 `time.ParseDuration` 语法](https://golang.org/pkg/time/#ParseDuration) 相同的格式，但您还可以使用 `d` 表示天（为简单起见，我们假设 1 天 = 24 小时）。有效单位是：

- `ns`（纳秒）
- `us`/`µs`（微秒）
- `ms`（毫秒）
- `s`（秒）
- `m`（分钟）
- `h`（小时）
- `d`（天）

示例：

- `250ms`
- `5s`
- `1.5h`
- `2h45m`
- `90d`

在 [JSON 配置](/docs/json/)中，持续时间值也可以是整数，表示纳秒。