---
标题: HTTPS 快速入门
---

# HTTPS 快速入门

本指南将向你展示如何在短时间内启动并运行[完全托管的 HTTPS](/docs/automatic-https)。

<aside class="tip">
	默认情况下，只要配置中提供了主机名，Caddy 就会为所有站点使用 HTTPS。本教程假设你希望让你的站点通过 HTTPS 获得公开信任（即不是 "localhost"），因此我们将使用公共域名和外部端口。
</aside>

**前提条件：**
- 基本的终端/命令行技能
- 对 DNS 的基本理解
- 一个已注册的公共域名
- 对端口 80 和 443 的外部访问权限
- 在 PATH 中可执行 `caddy` 和 `curl`

---

在本教程中，请将 `example.com` 替换为你实际的域名。

将域名的 A/AAAA 记录指向你的服务器。你可以登录 DNS 提供商并管理你的域名来完成此操作。

在继续之前，通过权威查询验证记录是否正确。将 `example.com` 替换为你的域名，如果你使用 IPv6，则将 `type=A` 替换为 `type=AAAA`：

<pre><code class="cmd bash">curl "https://cloudflare-dns.com/dns-query?name=example.com&type=A" \
  -H "accept: application/dns-json"</code></pre>

同时确保你的服务器可以通过公共接口从外部访问 80 和 443 端口。

<aside class="tip">
	如果你在家用网络或其他受限网络中，可能需要转发端口或调整防火墙设置。
</aside>

我们需要做的就是在配置中包含你的域名并启动 Caddy。有几种方法可以实现。

## Caddyfile

这是获取 HTTPS 最常用的方式。

创建一个名为 `Caddyfile`（无扩展名）的文件，第一行是你的域名，例如：

```caddy
example.com

respond "Hello, privacy!"
```

然后从同一目录运行：

<pre><code class="cmd bash">caddy run</code></pre>

你会看到 Caddy 配置 TLS 证书并通过 HTTPS 提供你的站点服务。这是可行的，因为 Caddyfile 中站点的地址包含了域名。

## `file-server` 命令

如果你只需要通过 HTTPS 提供静态文件服务，运行此命令（替换你的域名）：

<pre><code class="cmd bash">caddy file-server --domain example.com</code></pre>

你会看到 Caddy 配置 TLS 证书并通过 HTTPS 提供你的站点服务。

## `reverse-proxy` 命令

如果你只需要一个简单的通过 HTTPS 的反向代理（作为 TLS 终止器），运行此命令（替换你的域名和实际后端地址）：

<pre><code class="cmd bash">caddy reverse-proxy --from example.com --to localhost:9000</code></pre>

你会看到 Caddy 配置 TLS 证书并通过 HTTPS 提供你的站点服务。

## JSON 配置

一般的经验法则是，任何[主机匹配器](/docs/json/apps/http/servers/routes/match/host/)都会触发自动 HTTPS。

因此，像下面这样的 JSON 配置将启用可用于生产环境的[自动 HTTPS](/docs/automatic-https)：

```json
{
	"apps": {
		"http": {
			"servers": {
				"hello": {
					"listen": [":443"],
					"routes": [
						{
							"match": [{
								"host": ["example.com"]
							}],
							"handle": [{
								"handler": "static_response",
								"body": "Hello, privacy!"
							}]
						}
					]
				}
			}
		}
	}
}