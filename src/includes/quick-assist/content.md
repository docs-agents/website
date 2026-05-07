<!--
	所有 Markdown 内容默认隐藏，通过 ID 加载。
	HTML ID 应以 qa-content- 开头，后接状态 ID。
	确保在 div 开头和结尾处留有空白行，否则 Markdown 解析将无法正常工作。
-->

<div id="qa-content-install_dpkg">

<pre><code class="cmd"><span class="bash">sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https</span>
<span class="bash">curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg</span>
<span class="bash">curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list</span>
<span class="bash">sudo apt update</span>
<span class="bash">sudo apt install caddy</span></code></pre>

</div>
<div id="qa-content-install_rpm">

<pre><code class="cmd"><span class="bash">dnf install 'dnf-command(copr)'</span>
<span class="bash">dnf copr enable @caddy/caddy</span>
<span class="bash">dnf install caddy</span></code></pre>

</div>
<div id="qa-content-install_arch">

<pre><code class="cmd"><span class="bash">pacman -Syu caddy</span></code></pre>

</div>
<div id="qa-content-install_mac">

<pre><code class="cmd bash">brew install caddy</code></pre>

</div>
<div id="qa-content-install_windows">

<p>Chocolatey:</p> <pre><code class="cmd">choco install caddy</code></pre>
<p>Scoop:</p> <pre><code class="cmd">scoop install caddy</code></pre>

</div>
<div id="qa-content-install_nix">

- 包名：[`caddy`](https://search.nixos.org/packages?channel=unstable&show=caddy&query=caddy)
- NixOS 模块：[`services.caddy`](https://search.nixos.org/options?channel=unstable&show=services.caddy.enable&query=services.caddy)

</div>
<div id="qa-content-install_android">

在 Termux 中：<pre><code class="cmd">pkg install caddy</code></pre>

</div>
<div id="qa-content-install_other">

<h4>Webi</h2>
<p>Linux 和 macOS：</p>
<pre><code class="cmd bash">curl -sS https://webi.sh/caddy | sh</code></pre>
<p>Windows：</p>
<pre><code class="cmd">curl.exe https://webi.ms/caddy | powershell</code></pre>
<h4>Ansible</h4>
<pre><code class="cmd bash">ansible-galaxy install nvjacobo.caddy</code></pre>

</div>
<div id="qa-content-install_docker">

<pre><code class="cmd bash">docker pull caddy</code></pre>

</div>
<div id="qa-content-install_build">

请确保已安装 `git` 和 [Go](https://go.dev) 的最新版本。

<pre><code class="cmd"><span class="bash">git clone "https://github.com/caddyserver/caddy.git"</span>
<span class="bash">cd caddy/cmd/caddy/</span>
<span class="bash">go build</span></code></pre>

</div>
<div id="qa-content-install_with_plugins">


[`xcaddy`](https://github.com/caddyserver/xcaddy) 是一个命令行工具，可帮助你构建带插件的 Caddy。一个基本的构建命令如下：

<pre><code class="cmd bash">xcaddy build</code></pre>

要使用插件构建，请使用 `--with` 参数：

<pre><code class="cmd bash">xcaddy build \
	--with github.com/caddyserver/nginx-adapter
	--with github.com/caddyserver/ntlm-transport@v0.1.1</code></pre>

</div>
<div id="qa-content-install_binary">

1. 获取 Caddy 二进制文件：
	- 从 [GitHub 上的发布页面](https://github.com/caddyserver/caddy/releases) 获取（展开“Assets”）
		- 请参阅 [验证资产签名](/docs/signature-verification) 了解如何验证资产签名
	- 从 [我们的下载页面](/download) 获取
	- 通过 [从源码构建](/docs/build)（使用 `go` 或 `xcaddy`）
2. [将 Caddy 安装为系统服务。](/docs/running#manual-installation) 强烈建议这样做，尤其是对于生产服务器。

将二进制文件放入 `$PATH`（Windows 上为 `%PATH%`）目录之一，这样你就可以在不输入可执行文件完整路径的情况下运行 `caddy` 命令。（运行 `echo $PATH` 查看符合条件的目录列表。）

你可以通过将静态二进制文件替换为新版本并重启 Caddy 来进行升级。[`caddy upgrade` 命令](/docs/command-line#caddy-upgrade) 可以简化此过程。

</div>
<div id="qa-content-cfg_ondemand_smallscale">

按需 TLS 适用于以下场景：你无法控制域名，或者拥有太多证书无法在服务器启动时一次性加载。对于其他所有用例，标准的 TLS 自动化可能更合适。

</div>
<div id="qa-content-cfg_ondemand_caddyfile">


为了防止滥用，你必须首先配置一个 `ask` 端点，以便 Caddy 能够检查是否应获取证书。在顶部的全局选项中添加以下内容：

```caddy
{
	on_demand_tls {
		ask http://localhost:5555/check
	}
}
```

将该端点更改为你已设置的内容，如果 `domain=` 查询参数中给定的域名允许获取证书，则返回 HTTP 200。

然后创建一个站点块，为 TLS 端口上的所有站点/主机提供服务：

```caddy
https:// {
	tls {
		on_demand
	}
}
```

这是启用 Caddy 接受并为任意主机提供 TLS 服务所需的最小配置。此配置不会调用任何处理器。通常你还需要 [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) 到你的后端应用程序。

</div>