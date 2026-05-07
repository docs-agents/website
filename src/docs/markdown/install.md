---
title: "安装"
---

# 安装

本页面介绍了在您的系统上安装 Caddy 的各种方法。

**官方提供：**

- [静态二进制文件](#static-binaries)
- [Debian、Ubuntu、Raspbian 软件包](#debian-ubuntu-raspbian)
- [Fedora、RedHat、CentOS 软件包](#fedora-redhat-centos)
- [Arch Linux、Manjaro、Parabola 软件包](#arch-linux-manjaro-parabola)
- [Docker 镜像](#docker)
- [Railway 模板](#railway)

<aside class="tip">

[官方软件包](https://github.com/caddyserver/dist)仅包含标准模块。如果您需要第三方插件，请[使用 `xcaddy` 从源码构建](/docs/build#xcaddy)，使用[我们的下载页面](/download)，或者[在 Railway 上部署](#railway)。

</aside>

**社区维护：**

- [Gentoo](#gentoo)
- [Homebrew (Mac)](#homebrew-mac)
- [Chocolatey (Windows)](#chocolatey-windows)
- [Scoop (Windows)](#scoop-windows)
- [Webi](#webi)
- [Ansible](#ansible)
- [Termux](#termux)
- [Nix/Nixpkgs/NixOS](#nixnixpkgsnixos)
- [Unikraft](#unikraft)
- [OPNsense](#opnsense)
- [Mise](#mise)


## 静态二进制文件

**如果安装到生产系统，我们建议使用下方可用的官方发行版软件包。**

1. 获取 Caddy 二进制文件：
	- [从 GitHub 发布页面](https://github.com/caddyserver/caddy/releases)（展开 "Assets"）
		- 请参考[验证资产签名](/docs/signature-verification)了解如何验证资产签名
	- [从我们的下载页面](/download)
	- [从源码构建](/docs/build)（使用 `go` 或 `xcaddy`）
2. [将 Caddy 安装为系统服务。](/docs/running#manual-installation) 强烈建议这样做，尤其是生产服务器。

将二进制文件放在 `$PATH`（Windows 下为 `%PATH%`）的一个目录中，这样您就可以直接运行 `caddy` 而无需输入完整路径。（运行 `echo $PATH` 查看符合条件的目录列表。）

您可以通过将静态二进制文件替换为新版本并重启 Caddy 来升级。 [`caddy upgrade` 命令](/docs/command-line#caddy-upgrade) 可以简化此操作。


## Debian、Ubuntu、Raspbian

安装此软件包会自动将 Caddy 作为名为 `caddy` 的 [systemd 服务](/docs/running#linux-service) 启动并运行。它还附带一个可选的 `caddy-api` 服务，该服务**默认未启用**，但如果您主要通过 API 而不是配置文件来配置 Caddy，则应使用该服务。

安装后，请阅读[服务使用说明](/docs/running#using-the-service)。

**稳定版：**

<pre><code class="cmd"><span class="bash">sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl</span>
<span class="bash">curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg</span>
<span class="bash">curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list</span>
<span class="bash">sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg</span>
<span class="bash">sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list</span>
<span class="bash">sudo apt update</span>
<span class="bash">sudo apt install caddy</span></code></pre>

**测试版**（包括 Beta 版和候选发布版）：

<pre><code class="cmd"><span class="bash">sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl</span>
<span class="bash">curl -1sLf 'https://dl.cloudsmith.io/public/caddy/testing/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-testing-archive-keyring.gpg</span>
<span class="bash">curl -1sLf 'https://dl.cloudsmith.io/public/caddy/testing/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-testing.list</span>
<span class="bash">sudo chmod o+r /usr/share/keyrings/caddy-testing-archive-keyring.gpg</span>
<span class="bash">sudo chmod o+r /etc/apt/sources.list.d/caddy-testing.list</span>
<span class="bash">sudo apt update</span>
<span class="bash">sudo apt install caddy</span></code></pre>

[**查看 Cloudsmith 仓库**](https://cloudsmith.io/~caddy/repos/)

如果您希望将打包的支持文件（systemd 服务、bash 补全和默认配置）用于自定义 Caddy 构建，说明可[在此处查看](/docs/build#package-support-files-for-custom-builds-for-debianubunturaspbian)。


## Fedora、RedHat、CentOS

此软件包包含 Caddy 的两个 [systemd 服务](/docs/running#linux-service) 单元文件，但默认不启用它们。建议使用该服务。如果使用，请阅读[服务使用说明](/docs/running#using-the-service)。

Fedora：

<pre><code class="cmd"><span class="bash">dnf install dnf5-plugins</span>
<span class="bash">dnf copr enable @caddy/caddy</span>
<span class="bash">dnf install caddy</span></code></pre>

CentOS/RHEL：

<pre><code class="cmd"><span class="bash">dnf install dnf-plugins-core</span>
<span class="bash">dnf copr enable @caddy/caddy</span>
<span class="bash">dnf install caddy</span></code></pre>

[**查看 Caddy COPR**](https://copr.fedorainfracloud.org/coprs/g/caddy/caddy/)


## Arch Linux、Manjaro、Parabola

此软件包包含经过大幅修改的 Caddy 两个 [systemd 服务](/docs/running#linux-service) 单元文件，但默认不启用它们。
这些修改包括自定义启动/停止行为以及额外的沙箱标志，这些标志在 [systemd 的 exec 文档](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#Sandboxing) 中有说明，可能导致某些主机目录无法被 Caddy 进程访问。

<pre><code class="cmd"><span class="bash">pacman -Syu caddy</span></code></pre>

[**查看 Arch Linux 仓库中的 Caddy**](https://archlinux.org/packages/extra/x86_64/caddy/) 以及 [**Arch Linux 维基**](https://wiki.archlinux.org/title/Caddy)

## Docker

<pre><code class="cmd bash">docker pull caddy</code></pre>

[**在 Docker Hub 上查看**](https://hub.docker.com/_/caddy)

请参阅我们推荐的 [Docker Compose 配置](/docs/running#docker-compose) 和使用说明。


## Railway

通过 [Railway](https://railway.com) 的赞助，我们正式支持此模板：

[![在 Railway 上部署](https://railway.com/button.svg)](https://railway.com/deploy/caddy?referralCode=YOPtw9&utm_medium=integration&utm_source=template&utm_campaign=generic)


## Gentoo

_注意：这是一种社区维护的安装方法。_

<pre><code class="cmd">emerge www-servers/caddy</code></pre>

[**查看 Gentoo 软件包**](https://packages.gentoo.org/packages/www-servers/caddy)


## Homebrew (Mac)

_注意：这是一种社区维护的安装方法。_

<pre><code class="cmd bash">brew install caddy</code></pre>

[**查看 Homebrew 配方**](https://formulae.brew.sh/formula/caddy)


## Chocolatey (Windows)

_注意：这是一种社区维护的安装方法。_

<pre><code class="cmd">choco install caddy</code></pre>

[**查看 Chocolatey 软件包**](https://chocolatey.org/packages/caddy)


## Scoop (Windows)

_注意：这是一种社区维护的安装方法。_

<pre><code class="cmd">scoop install caddy</code></pre>

[**查看 Scoop 清单**](https://github.com/ScoopInstaller/Main/blob/master/bucket/caddy.json)


## Webi

_注意：这是一种社区维护的安装方法。_

Linux 和 macOS：

<pre><code class="cmd bash">curl -sS https://webi.sh/caddy | sh</code></pre>

Windows：

<pre><code class="cmd">curl.exe https://webi.ms/caddy | powershell</code></pre>

您可能需要调整 Windows 防火墙规则以允许非本地主机的传入连接。

[**在 Webi 上查看**](https://webinstall.dev/caddy)


## Ansible

_注意：这是一种社区维护的安装方法。_

<pre><code class="cmd bash">ansible-galaxy install nvjacobo.caddy</code></pre>

[**查看 Ansible 角色仓库**](https://github.com/nvjacobo/caddy)


## Termux

_注意：这是一种社区维护的安装方法。_

<pre><code class="cmd">pkg install caddy</code></pre>

[**查看 Termux 的 build.sh 文件**](https://github.com/termux/termux-packages/blob/master/packages/caddy/build.sh)


## Nix/Nixpkgs/NixOS

_注意：这是一种社区维护的安装方法。_

- 软件包名称：[`caddy`](https://search.nixos.org/packages?channel=unstable&show=caddy&query=caddy)
- NixOS 模块：[`services.caddy`](https://search.nixos.org/options?channel=unstable&show=services.caddy.enable&query=services.caddy)

[**在 Nixpkgs 搜索中查看 Caddy**](https://search.nixos.org/packages?channel=unstable&show=caddy&query=caddy) 和 [**NixOS 选项搜索**](https://search.nixos.org/options?channel=unstable&show=services.caddy.enable&query=services.caddy)


## Unikraft

_注意：这是一种社区维护的安装方法。_

首先安装 Unikraft 的配套工具 [`kraft`](https://unikraft.org/docs/cli)：

<pre><code class="cmd">curl --proto '=https' --tlsv1.2 -sSf https://get.kraftkit.sh | sh</code></pre>

然后使用 Unikraft 运行 Caddy：

<pre><code class="cmd">kraft run --rm -p 2015:2015 --plat qemu --arch x86_64 -M 256M caddy:2.7</code></pre>

要允许非本地主机的传入连接，您需要[将 unikernel 实例连接到网络](https://unikraft.org/docs/cli/running#connecting-a-unikernel-instance-to-a-network)。

[**查看 Unikraft 应用程序目录**](https://github.com/unikraft/catalog/tree/main/examples/caddy) 和 [**KraftCloud 平台示例（由 Unikraft 驱动）**](https://github.com/kraftcloud/examples/tree/main/caddy)。


## OPNsense

_注意：这是一种社区维护的安装方法。_

<pre><code class="cmd">pkg install os-caddy</code></pre>

[**查看 FreeBSD caddy-custom makefile**](https://github.com/opnsense/ports/blob/master/www/caddy-custom/Makefile) 和 [**os-caddy 插件源码**](https://github.com/opnsense/plugins/tree/master/www/caddy)

## Mise

_注意：这是一种社区维护的安装方法。_

如果您正在使用 [mise](https://github.com/jdx/mise)（多语言工具版本管理器），可以使用如下命令安装最新版本：

<pre><code class="cmd">mise use -g caddy@latest</code></pre>