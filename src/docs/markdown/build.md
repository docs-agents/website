---
title: "从源代码构建"
---

# 从源代码构建

如果您需要自定义构建（例如包含插件），有多种构建 Caddy 的选项：
- [Git](#git)：从 Git 仓库构建
- [`xcaddy`](#xcaddy)：使用 `xcaddy` 构建
- [Docker](#docker)：构建自定义 Docker 镜像

要求：

- 安装 [Go](https://golang.org/doc/install) 1.20 或更新版本

[包支持文件](#适用于-debianubunturaspbian-自定义构建的包支持文件) 部分包含为那些使用 APT 命令在 Debian 衍生系统上安装了 Caddy，但仍需自定义构建可执行文件进行操作的用户的说明。

## Git

要求：

- 安装 Go（见上文）

克隆仓库：

<pre><code class="cmd bash">git clone "https://github.com/caddyserver/caddy.git"</code></pre>

如果没有 git，可以从 [GitHub](https://github.com/caddyserver/caddy) 下载源代码档案。每个[发行版](https://github.com/caddyserver/caddy/releases)也包含源代码快照。

构建：

<pre><code class="cmd"><span class="bash">cd caddy/cmd/caddy/</span>
<span class="bash">go build</span></code></pre>

<aside class="tip">

由于 [Go 的一个 bug](https://github.com/golang/go/issues/29228)，这些基本步骤不会嵌入版本信息。如果您需要版本信息（`caddy version`），则需要将 Caddy 作为依赖项而非主模块来编译。相关说明在 Caddy 的 [main.go](https://github.com/caddyserver/caddy/blob/master/cmd/caddy/main.go) 文件中。或者，您可以使用自动完成此操作的 [`xcaddy`](#xcaddy)。

</aside>

Go 程序很容易为其他平台编译。只需设置不同的 `GOOS`、`GOARCH` 和/或 `GOARM` 环境变量即可。（[详见 Go 文档](https://golang.org/doc/install/source#environment)）

例如，在非 Windows 系统上为 Windows 编译 Caddy：

<pre><code class="cmd bash">GOOS=windows go build</code></pre>

或者在非 Linux 或非 ARMv6 系统上为 Linux ARMv6 编译：

<pre><code class="cmd bash">GOOS=linux GOARCH=arm GOARM=6 go build</code></pre>

## xcaddy

[`xcaddy` 命令](https://github.com/caddyserver/xcaddy) 是构建带有版本信息和/或插件的 Caddy 的最简单方法。

要求：

- 安装 Go（见上文）
- 确保 [`xcaddy`](https://github.com/caddyserver/xcaddy/releases) 在您的 `PATH` 中

您**无需**下载 Caddy 源代码（它会自动为您下载）。

然后，构建 Caddy（包含版本信息）只需：

<pre><code class="cmd bash">xcaddy build</code></pre>

要包含插件构建，使用 `--with`：

<pre><code class="cmd bash">xcaddy build \
    --with github.com/caddyserver/nginx-adapter
	--with github.com/caddyserver/ntlm-transport@v0.1.1</code></pre>

如您所见，您可以使用 `@` 语法自定义插件的版本。版本可以是标签名、提交 SHA 或分支。

使用 `xcaddy` 进行跨平台编译与使用 `go` 命令相同。例如，为 macOS 交叉编译：

<pre><code class="cmd bash">GOOS=darwin xcaddy build</code></pre>

## Docker

您可以使用 `:builder` 镜像作为快捷方式，构建包含自定义模块的新 Caddy 二进制文件：

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

请确保将 `<version>` 替换为要使用的 Caddy 最新版本。

注意第二个 `FROM` 指令——通过简单地将新构建的二进制文件覆盖到常规的 `caddy` 镜像之上，可以生成更小的镜像。

Builder 使用 `xcaddy` 构建带有提供的模块的 Caddy，类似于[上文所述的](#xcaddy)过程。`--mount=type=cache,target=/go/pkg/mod` 和 `--mount=type=cache,target=/root/.cache/go-build` 选项分别用于缓存 Go 模块依赖和构建工件，以加速后续构建。该标志是 [Docker 的功能](https://docs.docker.com/build/cache/optimize/#use-cache-mounts)，而非 `xcaddy` 的功能。

要使用 Docker Compose，请参阅我们推荐的 [`compose.yml`](/docs/running#docker-compose) 和使用说明。

## 适用于 Debian/Ubuntu/Raspbian 自定义构建的包支持文件

此过程旨在简化运行自定义 `caddy` 二进制文件的操作，同时保留 `caddy` 包中的支持文件。

此过程允许用户利用官方包中的默认配置、systemd 服务文件和 bash-completion。

要求：
- 按照[这些说明](/docs/install#debian-ubuntu-raspbian)安装 `caddy` 包
- 构建您的自定义 `caddy` 二进制文件（参见上文章节），或[下载](/download)自定义构建
- 您的自定义 `caddy` 二进制文件应位于当前目录

操作步骤：
<pre><code class="cmd"><span class="bash">sudo dpkg-divert --divert /usr/bin/caddy.default --rename /usr/bin/caddy</span>
<span class="bash">sudo mv ./caddy /usr/bin/caddy.custom</span>
<span class="bash">sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.default 10</span>
<span class="bash">sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.custom 50</span>
<span class="bash">sudo systemctl restart caddy</span>
</code></pre>

说明：

- `dpkg-divert` 会将 `/usr/bin/caddy` 二进制文件移动到 `/usr/bin/caddy.default`，并设置一个转移，以防任何软件包想要在此位置安装文件。

- `update-alternatives` 会从所需的 caddy 二进制文件创建指向 `/usr/bin/caddy` 的符号链接。

- `systemctl restart caddy` 将关闭默认版本的 Caddy 服务器并启动自定义版本。

您可以通过执行以下命令并在屏幕信息指引下，在自定义和默认 `caddy` 二进制文件之间切换。然后，重启 Caddy 服务。

<pre><code class="cmd bash">update-alternatives --config caddy</code></pre>

此后如需升级 Caddy，您可以运行 [`caddy upgrade`](/docs/command-line#caddy-upgrade)。此命令会尝试[下载](/download)与当前构建具有相同插件的最新版本 Caddy 构建，然后用新二进制文件替换当前二进制文件。