---
title: 反向代理快速入门
---

# 反向代理快速入门

本指南将向您展示如何快速搭建一个适用于生产环境的反向代理，支持或不支持HTTPS。

**前提条件：**
- 基本的终端/命令行技能
- `caddy` 已在 PATH 环境变量中
- 一个正在运行的后端进程，用于代理转发

---

本教程假设您有一个在 `127.0.0.1:9000` 运行的后端 HTTP 服务。以下命令适用于 Linux，但相同原理也适用于其他操作系统。

您可以在没有配置文件的情况下启动一个简单的反向代理，也可以使用配置文件以获得更高的灵活性和控制力。

## 命令行

要在您的机器上启动一个从端口 2080 到端口 9000 的明文 HTTP 代理：

<pre><code class="cmd bash">caddy reverse-proxy --from :2080 --to :9000</code></pre>

然后尝试：

<pre><code class="cmd bash">curl -v 127.0.0.1:2080</code></pre>

[`reverse-proxy` 命令](/docs/command-line#reverse-proxy) 旨在快速简单地配置反向代理。（如果您的需求简单，可以在生产环境中使用。）

## Caddyfile

在当前工作目录中，创建一个名为 `Caddyfile` 的文件，内容如下：

```caddy
:2080

reverse_proxy :9000
```

这个配置文件大致等同于上面的 `caddy reverse-proxy` 命令。

然后，在同一目录下运行：

<pre><code class="cmd bash">caddy run</code></pre>

然后测试您的代理：

<pre><code class="cmd bash">curl -v 127.0.0.1:2080</code></pre>

如果您修改了 Caddyfile，请务必 [重载](/docs/command-line#caddy-reload) Caddy。

这是一个简单的示例。您可以使用 [`reverse_proxy` 指令](/docs/caddyfile/directives/reverse_proxy) 实现更多功能。

## 从客户端到代理的 HTTPS

如果 Caddy 知道主机名（域名），它将 [自动且默认](/docs/automatic-https) 通过 HTTPS 提供代理服务。如果您省略 `--from` 标志，`caddy reverse-proxy` 命令将默认使用 `localhost`，或者您可以将 Caddyfile 的第一行替换为代理的域名。

- 如果您使用 `localhost` 或以 `.localhost` 结尾的任何域名，Caddy 将使用自动续期的自签名证书。首次执行时，Caddy 会尝试将其 CA 的根证书安装到您的信任存储中，您可能需要输入密码。
- 如果您使用任何其他域名，Caddy 将尝试获取公开信任的证书；请确保您的 DNS 记录指向您的机器，并且端口 80 和 443 对公网开放并指向 Caddy。

如果您不指定端口，Caddy 在 HTTPS 中默认使用 443。在这种情况下，您还需要拥有绑定低端口的权限。在 Linux 上有几种方法可以实现：

- 以 root 身份运行（例如 `sudo -E`）。
- 或者运行 `sudo setcap cap_net_bind_service=+ep $(which caddy)` 赋予 Caddy 此特定能力。

以下是提供 HTTPS 的最基本的 `caddy reverse-proxy` 命令：

<pre><code class="cmd bash">caddy reverse-proxy --to :9000</code></pre>

然后测试：

<pre><code class="cmd bash">curl -v https://localhost</code></pre>

您可以使用 `--from` 标志自定义主机名：

<pre><code class="cmd bash">caddy reverse-proxy --from example.com --to :9000</code></pre>

如果您没有绑定低端口的权限，可以从较高端口代理：

<pre><code class="cmd bash">caddy reverse-proxy --from example.com:8443 --to :9000</code></pre>

如果您使用 Caddyfile，只需将第一行改为您的域名，例如：

```caddy
example.com

reverse_proxy :9000
```

## 从代理到后端的 HTTPS

如果后端支持 TLS，Caddy 也可以在其和后端之间使用 HTTPS 进行代理。只需在后端地址中使用 `https://`：

<pre><code class="cmd bash">caddy reverse-proxy --from :2080 --to https://localhost:9000</code></pre>

这要求后端证书被 Caddy 运行的系统信任。（除非明确配置，否则 Caddy 不信任自签名证书。）

当然，您也可以两端都使用 HTTPS：

<pre><code class="cmd bash">caddy reverse-proxy --from example.com --to https://example.com:9000</code></pre>

这将在客户端到代理以及代理到后端之间提供 HTTPS。

如果您要代理的主机名与代理来源的主机名不同，则需要使用 `--change-host-header` 标志：

<pre><code class="cmd bash">caddy reverse-proxy \
	--from example.com \
	--to https://localhost:9000 \
	--change-host-header</code></pre>

默认情况下，Caddy 会原封不动地传递所有 HTTP 头，包括 `Host` 头，并且 Caddy 会根据 Host 头派生 TLS ServerName。`--change-host-header` 将 Host 头重置为后端的 Host 头，以便 TLS 握手能够成功完成。在上面的示例中，它将被从 `example.com` 改为 `localhost:9000`（并且在 TLS 握手中会使用 `localhost`）。