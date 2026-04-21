---
title: "从源代码构建"
---

# 从源代码构建

如果您需要自定义构建（例如带插件），有多种 Caddy 构建选项：
- [Git](#git): 从 Git 仓库构建
- [`xcaddy`](#xcaddy): 使用 `xcaddy` 构建
- [Docker](#docker): 构建自定义 Docker 镜像

要求：

- [Go](https://golang.org/doc/install) 1.20 或更高版本

[自定义构建的包支持文件](#debianubunturaspbian-自定义构建的包支持文件) 部分包含已使用 APT 命令在 Debian 衍生系统上安装 Caddy 但需要自定义构建可执行文件进行操作的用户的说明。



## Git

要求：

- 已安装 Go（见上文）

克隆仓库：

<pre><code class="cmd bash">git clone "https://github.com/caddyserver/caddy.git"</code></pre>

如果您没有 git，可以从 [GitHub](https://github.com/caddyserver/caddy) 下载源代码作为文件压缩包。每个 [发布](https://github.com/caddyserver/caddy/releases) 也有源代码快照。

构建：

<pre><code class="cmd"><span class="bash">cd caddy/cmd/caddy/</span>
<span class="bash">go build</span></code></pre>


<aside class="tip">

由于 [Go 中的 bug](https://github.com/golang/go/issues/29228)，这些基本步骤不嵌入版本信息。如果您想要版本信息（`caddy version`），您需要将 Caddy 作为依赖项编译，而不是作为主模块。此说明在 Caddy 的 [main.go](https://github.com/caddyserver/caddy/blob/master/cmd/caddy/main.go) 文件中。或者，您可以使用 [`xcaddy`](#xcaddy) 来自动化此过程。

</aside>

Go 程序很容易为其他平台编译。只需设置不同的 `GOOS`、`GOARCH` 和/或 `GOARM` 环境变量。（[参见 go 文档了解详情。](https://golang.org/doc/install/source#environment)）

例如，当您不在 Windows 上时为 Windows 编译 Caddy：

<pre><code class="cmd bash">GOOS=windows go build</code></pre>

或类似地，当您不在 Linux 或不在 ARMv6 上时为 Linux ARMv6 编译：

<pre><code class="cmd bash">GOOS=linux GOARCH=arm GOARM=6 go build</code></pre>



## xcaddy

[`xcaddy` 命令](https://github.com/caddyserver/xcaddy) 是使用版本信息和/或插件构建 Caddy 的最简单方式。

要求：

- 已安装 Go（见上文）
- 确保 [`xcaddy`](https://github.com/caddyserver/xcaddy/releases) 在您的 `PATH` 中

您**不需要**下载 Caddy 源代码（它会自动为您下载）。

然后构建 Caddy（带版本信息）非常简单：

<pre><code class="cmd bash">xcaddy build</code></pre>

要使用插件构建，使用 `--with`：

<pre><code class="cmd bash">xcaddy build \
    --with github.com/caddyserver/nginx-adapter
	--with github.com/caddyserver/ntlm-transport@v0.1.1</code></pre>

如您所见，您可以使用 `@` 语法自定义插件的版本。版本可以是标签名、提交 SHA 或分支。

`xcaddy` 的跨平台编译与 `go` 命令相同。例如，为 macOS 交叉编译：

<pre><code class="cmd bash">GOOS=darwin xcaddy build</code></pre>



## Docker

您可以使用 `:builder` 镜像作为快捷方式来构建带有自定义模块的新 Caddy 二进制文件：

```Dockerfile
FROM caddy:<version>-builder AS builder

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    xcaddy build \
    --with github.com/caddyserver/nginx-adapter \
    --with github.com/hairyhenderson/caddy-teapot-module@v0.0.3-0

FROM caddy:<version>

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

确保用最新的 Caddy 版本替换 `<version>`。

注意第二个 `FROM` 指令——这通过将新构建的二进制文件直接覆盖在常规 `caddy` 镜像上来生成更小的镜像。

构建器使用 `xcaddy` 以提供模块构建 Caddy，类似于[上述流程](#xcaddy)。`--mount=type=cache,target=/go/pkg/mod` 和 `--mount=type=cache,target=/root/.cache/go-build` 选项用于缓存 Go 模块依赖项和构建产物，分别加快后续构建。该标志是 [Docker 的特性](https://docs.docker.com/build/cache/optimize/#use-cache-mounts)，而非 `xcaddy` 的。

要使用 Docker Compose，请参阅我们推荐的 [`compose.yml`](/docs/running#docker-compose) 和使用说明。



## Debian/Ubuntu/Raspbian 自定义构建的包支持文件

此过程旨在简化运行自定义 `caddy` 二进制文件的同时保留来自 `caddy` 包的支持文件。

此过程允许用户利用官方包的默认配置、systemd 服务文件和 bash 补全。

要求：
- 按照 [这些说明](/docs/install#debian-ubuntu-raspbian) 安装 `caddy` 包
- 构建您的自定义 `caddy` 二进制文件（见上述部分），或 [下载](/download) 自定义构建
- 您的自定义 `caddy` 二进制文件应位于当前目录

流程：
<pre><code class="cmd"><span class="bash">sudo dpkg-divert --divert /usr/bin/caddy.default --rename /usr/bin/caddy</span>
<span class="bash">sudo mv ./caddy /usr/bin/caddy.custom</span>
<span class="bash">sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.default 10</span>
<span class="bash">sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.custom 50</span>
<span class="bash">sudo systemctl restart caddy</span>
</code></pre>

解释：

- `dpkg-divert` 将移动 `/usr/bin/caddy` 二进制文件到 `/usr/bin/caddy.default` 并放置一个防止任何包将文件安装到此位置的转换。

- `update-alternatives` 将创建从期望的 caddy 二进制文件到 `/usr/bin/caddy` 的符号链接

- `systemctl restart caddy` 将关闭默认版本的 Caddy 服务器并启动自定义版本。

您可以通过执行以下操作在自定义和默认 `caddy` 二进制文件之间切换，并遵循屏幕信息。然后，重启 Caddy 服务。

<pre><code class="cmd bash">update-alternatives --config caddy</code></pre>

从此点升级 Caddy，您可以运行 [`caddy upgrade`](/docs/command-line#caddy-upgrade)。这尝试 [下载](/download) 与您当前构建相同插件的构建，使用最新版本的 Caddy，然后用新二进制文件替换当前二进制文件。