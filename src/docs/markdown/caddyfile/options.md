---
title: 全局选项 (Caddyfile)
---

<script>
ready(function() {
	// 为顶部代码块中的选项添加指向相应锚点的链接
	let headers = Array.from($$_('article h5')).map(el => el.id.replace(/-/g, "_"));
	
	$$_('pre.chroma .k').forEach(item => {
		if (headers.includes(item.innerText)) {
			let text = item.innerText.replace(/</g, '&lt;').replace(/>/g, '&gt;');
			let url = '#' + item.innerText.replace(/_/g, "-");
			item.innerHTML = `<a href="${url}" style="color: inherit;" title="${text}">${text}</a>`;
		}
	});
	
	// 为注释添加指向相应部分的链接
	$$_('pre.chroma .c1').forEach(item => {
		if (item.innerText.includes('#')) {
			let text = item.innerText;
			let before = text.slice(0, text.indexOf('#')); // 前导空白
			text = text.slice(text.indexOf('#')); // 仅注释部分
			let url = '#' + text.replace(/#/g, '').trim().toLowerCase().replace(/ /g, "-");
			item.innerHTML = `${before}<a href="${url}" style="color: inherit;" title="${text}">${text}</a>`;
		}
	});
	
	// 修复重复链接；'name'在两个不同部分出现，因此将第二个改为#name-1
	const caLine = Array.from($$_('pre.chroma .line'))
		.find(line => line.innerText.includes('ca [<id>]'));
	if (caLine && caLine.nextElementSibling) {
		const nameLink = caLine.nextElementSibling.querySelector('a');
		if (nameLink && nameLink.innerText.includes('name')) {
			nameLink.href = '#name-1';
		}
	}

	// 修复`renewal_window_ratio`，它在两个不同部分出现，因此将第二个改为#renewal_window_ratio-1
	const renewalLine = Array.from($$_('pre.chroma .line'))
		.find(line => line.innerText.includes('renewal_window_ratio'));
	if (renewalLine && renewalLine.nextElementSibling) {
		const renewalLink = renewalLine.nextElementSibling.querySelector('a');
		if (renewalLink && renewalLink.innerText.includes('renewal_window_ratio')) {
			renewalLink.href = '#renewal_window_ratio-1';
		}
	}
});
</script>


# 全局选项

Caddyfile 提供了一种方式让你指定全局适用的选项。某些选项作为默认值；其他选项用于自定义 HTTP 服务器，并不只适用于某一个站点；还有一些选项用于自定义 Caddyfile [适配器](/docs/config-adapters) 的行为。

Caddyfile 的最顶部可以是**全局选项块**。这是一个没有键的块：

```caddy
{
	...
}
```

最多只能有一个全局选项块，并且它必须是 Caddyfile 的第一个块。

可用的选项如下（点击每个选项可跳转到其文档）：

```caddy
{
	# 通用选项
	debug
	http_port    <端口>
	https_port   <端口>
	default_bind <主机...>
	order <指令1> first|last|[before|after <指令2>]
	storage <模块名称> {
		<选项...>
	}
	storage_clean_interval <时长>
	admin   off|<地址> {
		origins <来源...>
		enforce_origin
	}
	persist_config off
	log [名称] {
		output  <写入器模块> ...
		format  <编码器模块> ...
		level   <级别>
		include <命名空间...>
		exclude <命名空间...>
	}
	grace_period   <时长>
	shutdown_delay <时长>
	metrics {
		per_host
		observe_catchall_hosts
	}

	# TLS 选项
	auto_https off|disable_redirects|ignore_loaded_certs|disable_certs
	email <你的邮箱>
	default_sni <名称>
	fallback_sni <名称>
	local_certs
	skip_install_trust
	acme_ca <目录 URL>
	acme_ca_root <PEM 文件>
	acme_eab {
		key_id <密钥 ID>
		mac_key <MAC 密钥>
	}
	acme_dns <提供商> ...
	dns <提供商> ...
	ech <公共名称...> {
		dns <提供商> ...
	}
	on_demand_tls {
		ask        <端点>
		permission <模块>
	}
	key_type ed25519|p256|p384|rsa2048|rsa4096
	cert_issuer <名称> ...
	renew_interval <时长>
	cert_lifetime  <时长>
	ocsp_interval  <时长>
	ocsp_stapling off
	renewal_window_ratio <比率>
	preferred_chains [smallest] {
		root_common_name <通用名称...>
		any_common_name  <通用名称...>
	}

	# 服务器选项
	servers [<监听器地址>] {
		name <名称>
		listener_wrappers {
			<监听器包装器...>
		}
		timeouts {
			read_body   <时长>
			read_header <时长>
			write       <时长>
			idle        <时长>
		}
		keepalive_interval <时长>
		keepalive_idle     <时长>
		keepalive_count	   <数字>
		0rtt off

		trusted_proxies <模块> ...
		trusted_proxies_strict
		trusted_proxies_unix
		client_ip_headers <标头...>

		trace
		max_header_size <大小>
		enable_full_duplex
		log_credentials
		protocols [h1|h2|h2c|h3]
		strict_sni_host [on|insecure_off]
	}

	# 文件系统
	filesystem <名称> <模块> {
		<选项...>
	}

	# PKI 选项
	pki {
		ca [<ID>] {
			name                  <名称>
			root_cn               <名称>
			intermediate_cn       <名称>
			intermediate_lifetime <时长>
			maintenance_interval  <时长>
			renewal_window_ratio  <比率>
			root {
				format <格式>
				cert   <路径>
				key    <路径>
			}
			intermediate {
				format <格式>
				cert   <路径>
				key    <路径>
			}
		}
	}

	# 事件选项
	events {
		on <事件> <处理器...>
	}
}
```


## 通用选项

##### `debug`
启用调试模式，将[默认日志器](#log)的日志级别设置为 `DEBUG`。这会显示更多详细信息，对故障排除很有用（在生产环境中非常冗长）。我们建议你在[社区论坛](https://caddy.community)寻求帮助之前启用此选项。例如，在 Caddyfile 顶部，如果没有其他全局选项：

```caddy
{
	debug
}
```


##### `http_port`
服务器用于 HTTP 的端口。

**仅供内部使用**；不会更改客户端的 HTTP 端口。通常用于在你的内部网络中，你需要在将 `80` 端口转发到不同端口（例如 `8080`）之后才到达 Caddy，以便进行路由。

默认值：`80`


##### `https_port`
服务器用于 HTTPS 的端口。

**仅供内部使用**；不会更改客户端的 HTTPS 端口。通常用于在你的内部网络中，你需要在将 `443` 端口转发到不同端口（例如 `8443`）之后才到达 Caddy，以便进行路由。

默认值：`443`


##### `default_bind`
所有站点使用的默认绑定地址，如果在站点中没有使用 [`bind` 指令](/docs/caddyfile/directives/bind)。默认值：空，即绑定到所有接口。

<aside class="tip">

请注意，这仅适用于由 Caddyfile 生成的服务器；这意味着由[自动 HTTPS](/docs/automatic-https) 创建的用于 HTTP 到 HTTPS 重定向的 HTTP 服务器不会继承这些绑定地址。要解决此问题，请确保声明一个 `http://` 站点（可以为空，不含任何指令），这样在适配 Caddyfile 时它就会存在，从而接收这些绑定地址。

</aside>

```caddy
{
	default_bind 10.0.0.1
}
```



##### `order`
为 HTTP 处理程序指令分配顺序。由于 HTTP 处理程序按顺序链执行，因此必须以正确的顺序执行。标准指令有[预定义的顺序](/docs/caddyfile/directives#directive-order)，但如果使用第三方 HTTP 处理程序模块，则需要使用此选项或将该指令放在 [`route` 块](/docs/caddyfile/directives/route)中来显式定义顺序。排序可以绝对描述（`first` 或 `last`），也可以相对于另一个指令描述（`before` 或 `after`）。

例如，要使用 [`replace-response` 插件](https://github.com/caddyserver/replace-response)，你需要确保其指令在 `encode` 之后排序，以便在响应被编码之前执行替换（因为响应是向上流经处理程序链，而不是向下）：

```caddy
{
	order replace after encode
}
```


##### `storage`
配置 Caddy 的存储机制。默认值为 [`file_system`](/docs/json/storage/file_system/)。还有许多其他可用的[存储模块](/docs/json/storage/)作为插件提供。

例如，更改文件系统的存储位置：

```caddy
{
	storage file_system /path/to/custom/location
}
```

当跨多个 Caddy 实例同步 Caddy 的存储以确保它们使用相同的证书和密钥时，通常需要自定义存储模块。有关更多详细信息，请参阅[自动 HTTPS 中关于存储的说明](/docs/automatic-https#storage)。


##### `storage_clean_interval`
多久扫描一次存储单元以查找旧资产或过期资产并删除它们。这些扫描会在存储模块上产生大量读取（和列表操作），因此对于大型部署，请选择较长的间隔。接受[时长值](/docs/conventions#durations)。

进程首次启动时始终会清理存储。然后，如果上次清理在不到该间隔一半的时间内完成，则在上次清理开始后经过此间隔后开始新的清理（否则将跳过一次）。

默认值：`24h`

```caddy
{
	storage_clean_interval 7d
}
```




##### `admin`
自定义[管理 API 端点](/docs/api)。接受占位符。接受[网络地址](/docs/conventions#network-addresses)。

默认值：`localhost:2019`，除非设置了 `CADDY_ADMIN` 环境变量。

如果设置为 `off`，则管理端点将被禁用。禁用后，**无法更改配置**，除非停止并重新启动服务器，因为 [`caddy reload` 命令](/docs/command-line#caddy-reload) 使用管理 API 将新配置推送到运行中的服务器。

如果运行中的服务器的地址已从默认值更改，请记得使用兼容[命令](/docs/command-line)的 `--address` CLI 标志指定当前管理端点。

还支持以下子选项：

- **origins** 配置允许连接到端点的来源列表。

  默认会智能选择：
  - 如果监听地址是回环地址（例如 `localhost` 或回环 IP，或 Unix 套接字），则允许的来源为 `localhost`、`::1` 和 `127.0.0.1`，并与监听地址端口连接（因此 `localhost:2019` 是有效来源）。
  - 如果监听地址不是回环地址，则允许的来源与监听地址相同。

  如果监听地址主机不是通配符接口（通配符包括：空字符串、`0.0.0.0` 或 `[::]`），则执行 `Host` 标头强制检查。实际上，默认情况下，由于接口是 `localhost`，因此会验证 `Host` 标头是否在 `origins` 中。但对于像 `:2020` 这样具有通配符接口的地址，则不执行 `Host` 标头验证。

- **enforce_origin** 启用对 `Origin` 请求标头的强制检查。

  当监听地址是通配符接口时（因为不验证 `Host`），并且管理 API 暴露于公网时，这最有用。它启用 CORS 预检检查，并确保 `Origin` 标头针对 `origins` 列表进行验证。仅当你在开发机器上运行 Caddy 并需要从 Web 浏览器访问管理 API 时才使用此选项。

例如，要在所有接口上以不同端口公开管理 API — ⚠️ 此端口**不应公开暴露**，否则任何人都可以控制你的服务器；如果你需要公开，请考虑启用来源强制检查：

```caddy
{
	admin :2020
}
```

要关闭管理 API — ⚠️ 这会导致**无法重新加载配置**，除非停止并重新启动服务器：

```caddy
{
	admin off
}
```

要使用 [Unix 套接字](/docs/conventions#network-addresses) 作为管理 API，从而通过文件权限控制访问：

```caddy
{
	admin unix//run/caddy-admin.sock
}
```

要仅允许具有匹配 `Origin` 标头的请求：

```caddy
{
	admin :2019 {
		origins localhost
		enforce_origin
	}
}
```



##### `persist_config`

控制是否应将当前 JSON 配置持久化到[配置目录](/docs/conventions#configuration-directory)，以避免丢失通过管理 API 进行的配置更改。目前，仅支持 `off` 选项。默认情况下，配置会持久化。

```caddy
{
	persist_config off
}
```



##### `log`
配置命名的日志器。

可以传入名称以指示自定义行为的特定日志器。如果未指定名称，则修改 `default` 日志器的行为。你可以阅读更多关于 `default` 日志器以及 [Caddy 中日志记录的工作原理](/docs/logging) 的说明。

可以通过多次使用 `log` 来配置多个不同名称的日志器。

这与 [`log` 指令](/docs/caddyfile/directives/log)不同，后者仅配置 HTTP 请求日志记录（也称为访问日志）。`log` 全局选项与指令共享其配置结构（除了 `include` 和 `exclude`），完整的文档可在该指令的页面上找到。

- **output** 配置日志写入位置。

  有关完整文档，请参阅 [`log` 指令](/docs/caddyfile/directives/log#output-modules)。

- **format** 描述如何编码或格式化日志。

  有关完整文档，请参阅 [`log` 指令](/docs/caddyfile/directives/log#format-modules)。

- **level** 是要记录的日志条目的最低级别。

  默认值：`INFO`。

  可能的值：`DEBUG`、`INFO`、`WARN`、`ERROR`，以及极少使用的 `PANIC`、`FATAL`。

- **include** 指定要包含在此日志器中的日志名称。

  默认情况下，此列表为空（即包含所有日志）。

  例如，要仅包含管理 API 发出的日志，你可以包含 `admin.api`。

- **exclude** 指定要从此日志器中排除的日志名称。

  默认情况下，此列表为空（即不排除任何日志）。

  例如，要仅排除 HTTP 访问日志，你可以排除 `http.log.access`。

`include` 和 `exclude` 接受的日志器名称取决于所用的模块，发现它们的最简单方法是从先前的日志中查看。

以下是一个示例，将所有 HTTP 访问日志和管理日志以 JSON 格式记录到标准输出：

```caddy
{
	log default {
		output stdout
		format json
		include http.log.access admin.api
	}
}
```

##### `grace_period`
定义关闭 HTTP 服务器的宽限期（即在配置更改或 Caddy 停止时）。

在宽限期内，不接受新连接，关闭空闲连接，并耐心等待活动连接完成其请求。如果客户端未在宽限期内完成其请求，服务器将被强制终止以允许重新加载完成并释放资源。接受[时长值](/docs/conventions#durations)。

默认情况下，宽限期是无限的，这意味着永远不会强制关闭连接。

```caddy
{
	grace_period 10s
}
```


##### `shutdown_delay`
定义在[宽限期](#grace_period)之前的[时长](/docs/conventions#durations)，在此期间即将停止的服务器继续正常运行，但 `{http.shutting_down}` 占位符的值为 `true`，并且 `{http.time_until_shutdown}` 提供距离宽限期开始的时间。

如果作为配置更改的一部分关闭任何服务器，这会导致延迟，并实际上将更改安排在稍后时间。它对于向健康检查器宣告此服务器即将关闭，并给负载均衡器时间将其移出轮换非常有用；例如：

```caddy
{
	shutdown_delay 30s
}

example.com {
	handle /health-check {
		@goingDown vars {http.shutting_down} true
		respond @goingDown "Bye-bye in {http.time_until_shutdown}" 503
		respond 200
	}
	handle {
		respond "Hello, world!"
	}
}
```


## TLS 选项

##### `auto_https`
配置[自动 HTTPS](/docs/automatic-https)，这是使 Caddy 能够为站点自动化证书管理和 HTTP 到 HTTPS 重定向的功能。

有几个模式可供选择：

- `off`：同时禁用证书自动化和 HTTP 到 HTTPS 重定向。

- `disable_redirects`：仅禁用 HTTP 到 HTTPS 重定向。

- `disable_certs`：仅禁用证书自动化。

- `ignore_loaded_certs`：即使对于手动加载的证书中出现的名称，也自动化证书。如果您使用 [`tls` 指令](/docs/caddyfile/directives/tls) 指定了包含名称（或通配符）的证书，而您希望这些名称由 Caddy 自动管理，则此选项很有用。

<aside class="tip">

此选项不影响 Caddy 的默认协议，当站点地址具有有效域名时，默认协议始终为 HTTPS。这意味着 `auto_https off` 不会导致您的站点通过 HTTP 提供服务，它只会禁用自动证书管理和重定向。

这意味着，如果您希望通过 HTTP 提供站点服务，应将[站点地址](/docs/caddyfile/concepts#addresses) 更改为以 `http://` 开头或以 `:80` 结尾（或使用 [`http_port` 选项](#http_port)）。

</aside>

```caddy
{
	auto_https disable_redirects
}
```


##### `email`
您的电子邮件地址。主要用于向证书颁发机构 (CA) 创建 ACME 帐户，并且强烈建议提供，以防您的证书出现问题。

<aside class="tip">

请注意，Let's Encrypt 可能会向您发送有关证书即将过期的电子邮件，但这可能具有误导性，因为 Caddy 可能在续期时选择了不同的颁发者（例如 ZeroSSL）。请检查您的日志和/或证书本身（例如在浏览器中），以查看使用了哪个颁发者，以及其到期日期是否仍然有效；如果是，您可以安全地忽略 Let's Encrypt 的电子邮件。

</aside>

```caddy
{
	email admin@example.com
}
```


##### `default_sni`
设置默认的 TLS ServerName，用于客户端在其 ClientHello 中未使用 SNI 的情况。

```caddy
{
	default_sni example.com
}
```


##### `fallback_sni`
⚠️ <i>实验性</i>

如果配置，当原始 ServerName 与缓存中的任何证书都不匹配时，ClientHello 中的 TLS ServerName 会退回到此名称。

此用途非常小众；通常如果客户端是 CDN 并传递了下游握手的 ServerName，但可以接受使用源站主机名的证书，那么您可以将其设置为源站的主机名。请注意，Caddy 必须为此名称管理证书。

```caddy
{
	fallback_sni example.com
}
```


##### `local_certs`
导致**所有**证书默认在内部颁发，而不是通过（公共）ACME CA（如 Let's Encrypt）。这在开发环境中作为快速切换非常有用。

```caddy
{
	local_certs
}
```


##### `skip_install_trust`
跳过尝试将本地 CA 的根证书安装到系统信任存储以及 Java 和 Mozilla Firefox 信任存储。

```caddy
{
	skip_install_trust
}
```


##### `acme_ca`
指定 ACME CA 目录的 URL。强烈建议将其设置为 Let's Encrypt 的[临时端点 <img src="/old/resources/images/external-link.svg" class="external-link">](https://letsencrypt.org/docs/staging-environment/) 用于测试或开发。默认值：ZeroSSL 和 Let's Encrypt 的生产端点。

请注意，全局配置的 ACME CA 可能不适用于所有站点；请查看使用默认 ACME 颁发者的[主机名要求](/docs/automatic-https#hostname-requirements)。

```caddy
{
	acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}
```

##### `acme_ca_root`
指定包含 ACME CA 端点受信任根证书的 PEM 文件，如果系统信任存储中不存在的话。

```caddy
{
	acme_ca_root /path/to/ca/root.pem
}
```


##### `acme_eab`
指定用于所有 ACME 交易的外部帐户绑定 (EAB)。

例如，使用模拟的 ZeroSSL 凭据：

```caddy
{
	acme_eab {
		key_id GD-VvWydSVFuss_GhBwYQQ
		mac_key MjXU3MH-Z0WQ7piMAnVsCpD1shgMiWx6ggPWiTmydgUaj7dWWWfQfA
	}
}
```


##### `acme_dns`
配置要用于所有 ACME 交易的 [ACME DNS 挑战](/docs/automatic-https#dns-challenge) 提供商。

需要自定义构建的 Caddy，并包含你的 DNS 提供商的插件。

提供商名称后的令牌用于设置提供商，方式与在 [`tls` 指令的 `acme` 颁发者](/docs/caddyfile/directives/tls#acme) 中指定相同。

```caddy
{
	acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}
```


##### `dns`
配置在相关上下文中未指定其他本地 DNS 提供商时使用的默认 DNS 提供商。例如，如果启用 ACME DNS 挑战但未配置 DNS 提供商，则将使用此全局默认值。它也用于发布加密客户端问候 (ECH) 配置。

您的 Caddy 二进制文件必须使用指定的 DNS 提供商模块编译才能工作。

示例，使用环境变量中的凭据：

```caddy
{
	dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}
```

（需要 Caddy 2.10 beta 1 或更新版本。）


##### `ech`
通过使用指定的公共域名作为 TLS 握手中的明文服务器名称 (SNI) 来启用加密客户端问候 (ECH)。在条件合适的情况下，ECH 可以帮助保护连接过程中站点域名的在线隐私。Caddy 将为每个指定的公共名称生成并发布一个 ECH 配置。发布后，兼容的客户端（例如正确配置的现代浏览器）就会知道使用 ECH 访问您的站点。

为了正常工作，ECH 配置必须以客户端期望的方式发布。大多数浏览器（启用了 DNS-over-HTTPS 或 DNS-over-TLS）期望将 ECH 配置发布到 HTTPS 类型的 DNS 记录。Caddy 会自动进行此类发布，但您必须使用 `dns` 子选项或在全局 [`dns` 全局选项](#dns) 中指定 DNS 提供商，并且您的 Caddy 二进制文件必须使用指定的 DNS 提供商模块构建。（自定义构建可在此处获得：[我们的下载页面](/download)。）

**隐私声明：**

- 通常建议**最大化您的[匿名集](https://www.ietf.org/archive/id/draft-ietf-tls-esni-23.html#name-introduction)的大小**。因此，我们通常建议大多数用户仅配置**一个**公共域名来保护您所有的站点。
- **您的服务器应为您指定的公共域名负责**（即这些域名应指向您的服务器），因为 Caddy 将为其获取证书。这些证书对于帮助符合规范的客户端在某些情况下可靠且安全地使用 ECH 连接至关重要。它们仅用于促进正常的 ECH 握手，不用于应用数据（您的站点——除非您定义的站点与您的公共域名相同）。
- 每种情况都可能不同。如果风险较高，我们建议咨询专家以**审查您的威胁模型**，因为 ECH 并非“一刀切”的解决方案。

示例，使用环境变量中的凭据将配置发布到 Cloudflare 托管的名称服务器：

```caddy
{
	dns cloudflare {env.CLOUDFLARE_API_TOKEN}
	ech ech.example.net
}
```

这应该会导致兼容的客户端使用 `ech.example.net` 加载您的所有站点，而不是在明文中暴露各个站点名称。

成功发布要求您的站点域名已托管在已配置的 DNS 提供商处，并且可以使用给定的凭据/提供商配置修改记录。

（需要 Caddy 2.10 beta 1 或更新版本。）


##### `on_demand_tls`
配置[按需 TLS](/docs/automatic-https#on-demand-tls)（已启用），但并不启用它（要启用它，请使用 [`tls` 指令的 `on_demand` 子指令](/docs/caddyfile/directives/tls#syntax)）。在生产环境中使用以防止滥用，这是必需的。

- **ask** 会导致 Caddy 向给定 URL 发出 HTTP 请求，询问是否允许为某个域名颁发证书。

  请求的查询字符串为 `?domain=`，包含域名的值。
  
  如果端点返回 `2xx` 状态码，Caddy 将被授权为该名称获取证书。任何其他状态码将导致取消证书颁发并终止 TLS 握手。

<aside class="tip">

ask 端点应尽可能快地返回，在几毫秒内。理想情况下，您的端点应在具有域名索引的数据库中进行常量时间查找；避免循环。避免进行 DNS 查询或其他网络请求。

</aside>

- **permission** 允许使用自定义模块来确定是否应为特定名称颁发证书。该模块必须实现 [`caddytls.OnDemandPermission` 接口](https://pkg.go.dev/github.com/caddyserver/caddy/v2/modules/caddytls#OnDemandPermission)。包含了一个 `http` 权限模块，`ask` 选项正是使用的它，并且作为向后兼容的快捷方式保留。

- ⚠️ **interval** 和 **burst** 限流选项曾经可用，但**不推荐使用**。如果您的配置中仍有它们，请将其删除。

```caddy
{
	on_demand_tls {
		ask http://localhost:9123/ask
	}
}

https:// {
	tls {
		on_demand
	}
}
```


##### `key_type`
指定要为 TLS 证书生成的密钥类型；仅当有特定需求自定义时才更改。

可能的值：`ed25519`、`p256`、`p384`、`rsa2048`、`rsa4096`。

```caddy
{
	key_type ed25519
}
```


##### `cert_issuer`
定义 TLS 证书的颁发者（或来源）。

这允许全局配置颁发者，而不是像使用 [`tls` 指令的 `issuer` 子指令](/docs/caddyfile/directives/tls#issuer)那样按站点配置。

如果您希望配置多个颁发者进行尝试，可以重复此选项。它们将按定义顺序尝试。

```caddy
{
	cert_issuer acme {
		...
	}
	cert_issuer zerossl {
		...
	}
}
```


##### `renew_interval`
多久扫描一次所有已加载的托管证书以检查过期，如果已过期则触发续期。

默认值：`10m`

```caddy
{
	renew_interval 30m
}
```


##### `cert_lifetime`
请求 CA 为证书颁发的有效期。

此值用于计算 ACME 订单的 `notAfter` 字段；因此系统必须具有合理同步的时钟。注意：并非所有 CA 都支持此功能。请查阅您的 CA 的 ACME 文档以了解是否允许以及可以使用哪些值。

默认值：`0`（CA 选择有效期，通常为 90 天）

⚠️ 这是一个实验性功能。可能会更改或删除。

```caddy
{
	cert_lifetime 30d
}
```


##### `ocsp_interval`
多久检查一次 [OCSP 装订 <img src="/old/resources/images/external-link.svg" class="external-link">](https://en.wikipedia.org/wiki/OCSP_stapling) 是否需要更新。

默认值：`1h`

```caddy
{
	ocsp_interval 2h
}
```


##### `ocsp_stapling`
可以设置为 `off` 以禁用 OCSP 装订。在由于防火墙而无法访问应答器的环境中很有用。

```caddy
{
	ocsp_stapling off
}
```

##### `renewal_window_ratio`
在 Caddy 尝试续期证书之前必须剩余的证书有效期的比率（介于 0 和 1 之间）。例如，如果证书的有效期为 90 天，且此比率为 `0.3333`（默认值），则 Caddy 将在证书剩余 30 天或更少时持续尝试续期。也可以使用 [`tls` 指令的 `renewal_window_ratio` 子指令](/docs/caddyfile/directives/tls#renewal_window_ratio) 按站点设置。

您很少需要更改此选项，但如果您的 CA 颁发时间非常长，则提前续期可能有用。

请注意，这只是一个建议，因为 ACME 颁发者可能实现 [ARI 扩展](https://datatracker.ietf.org/doc/rfc9773/)，该扩展由颁发者指定一个窗口，ACME 客户端（此处为 Caddy）应在该窗口内尝试续期，并且该窗口可能与此比率不一致。

```caddy
{
	renewal_window_ratio 0.1
}
```


##### `preferred_chains`
如果您的 CA 提供多个证书链，您可以使用此选项指定 Caddy 应优先选择哪个链。设置以下选项之一：

- **smallest** 将告诉 Caddy 优先选择字节数最少的链。

- **root_common_name** 是一个或多个通用名称的列表；Caddy 将选择根证书与至少一个指定通用名称匹配的第一个链。

- **any_common_name** 是一个或多个通用名称的列表；Caddy 将选择颁发者与至少一个指定通用名称匹配的第一个链。

请注意，如果没有任何[覆盖的颁发者级别配置](/docs/caddyfile/directives/tls#acme)，将 `preferred_chains` 作为全局选项会影响所有颁发者。

```caddy
{
	preferred_chains smallest
}
```

```caddy
{
	preferred_chains {
		root_common_name "ISRG Root X2"
	}
}
```


## 服务器选项

自定义 [HTTP 服务器](/docs/json/apps/http/servers/) 的设置，这些设置可能跨越多个站点，因此无法在站点块中正确配置。这些选项影响监听器/套接字或 HTTP 层之下的其他设施。

可以通过使用不同的 `listener_address` 值多次指定，来为每个服务器配置不同的选项。例如，`servers :443` 仅适用于绑定到监听器地址 `:443` 的服务器。省略监听器地址将把选项应用于所有剩余的服务器。

<aside class="tip">

使用 [`caddy adapt`](/docs/command-line#caddy-adapt) 命令查找 Caddyfile 中服务器的监听地址。

</aside>


例如，要为端口 `:80` 和 `:443` 上的服务器配置不同的选项，您需要指定两个 `servers` 块：

```caddy
{
	servers :443 {
		listener_wrappers {
			http_redirect
			tls
		}
	}

	servers :80 {
		protocols h1 h2c
	}
}
```

使用 `servers` 时，它**仅**应用于**实际出现**在 Caddyfile 中的服务器（即由站点块产生的服务器）。请记住，[自动 HTTPS](/docs/automatic-https) 将创建一个监听端口 `80`（或 [`http_port` 选项](#http_port)）的服务器，以提供 HTTP 到 HTTPS 重定向并解决 ACME HTTP 挑战；这发生在运行时，即 _在_ Caddyfile 适配器应用 `servers` _之后_。换句话说，这意味着 `servers` **不会**应用于 `:80`，除非您显式声明一个类似 `http://` 或 `:80` 的站点块。


<aside class="tip">

如果您使用 [`bind` 指令](/docs/caddyfile/directives/bind) 或 [`default_bind` 全局选项](/docs/caddyfile/options#default_bind)，则 `listener_address` *必须*与站点块的绑定地址和端口组合匹配，否则这些设置将不会被应用。例如：

```caddy
{
	# 这不会匹配服务器，缺少绑定地址
	servers :8080 {
		name private
	}
	
	# 这将起作用，因为它是完全匹配
	servers 192.168.1.2:8080 {
		name public
	}
}

:8080 {
	bind 127.0.0.1
}

:8080 {
	bind 192.168.1.2
}
```

</aside>



##### `name`

分配给此服务器的自定义名称。通常有助于在日志和指标中按名称标识服务器。如果未设置，Caddy 将使用 `srvX` 模式动态定义它，其中 `X` 从 `0` 开始并根据配置中的服务器数量递增。

请记住，只有由配置文件中的站点块产生的服务器才会应用设置。[自动 HTTPS](/docs/automatic-https) 会在运行时创建一个 `:80` 服务器（或 [`http_port`](#http_port)），因此如果您想重命名它，您至少需要一个空的 `http://` 站点块。

例如：

```caddy
{
	servers :443 {
		name https
	}
	
	servers :80 {
		name http
	}
}

example.com {
}

http:// {
}
```

</aside>



##### `listener_wrappers`

允许配置[监听器包装器](/docs/json/apps/http/servers/listener_wrappers/)，它们可以修改套接字监听器的行为。它们按给定的顺序应用。

###### `tls`

`tls` 监听器包装器是一个无操作监听器包装器，用于标记 TLS 监听器在监听器包装器链中的位置。仅当另一个监听器包装器必须放在 TLS 握手之前时才应使用。

###### `http_redirect`

[`http_redirect`](/docs/json/apps/http/servers/listener_wrappers/http_redirect/) 为在 TLS 端口上作为 HTTP 请求传入的连接提供 HTTP 到 HTTPS 重定向，通过检测前几个字节判断不是 TLS 握手，而是 HTTP 请求。当在非标准端口（非 `443`）上提供 HTTPS 时最有用，因为浏览器会尝试 HTTP，除非指定了方案。它必须放在 `tls` 监听器包装器_之前_。示例如下：

```caddy
{
	servers {
		listener_wrappers {
			http_redirect
			tls
		}
	}
}
```

###### `proxy_protocol`

[`proxy_protocol`](/docs/json/apps/http/servers/listener_wrappers/proxy_protocol/) 监听器包装器（在 v2.7.0 之前仅通过插件提供）启用 [PROXY 协议](https://github.com/haproxy/haproxy/blob/master/doc/proxy-protocol.txt) 解析（由 HAProxy 推广）。这必须用于 `tls` 监听器包装器_之前_，因为它解析连接开始处的明文数据：

请注意，来自 PROXY 协议的元数据可能会在评估匹配器或 [`trusted_proxies`](/docs/caddyfile/options#trusted-proxies) 之前应用于连接。即时对端的 IP 地址将在进一步评估中丢失。

```caddy-d
proxy_protocol {
	timeout <时长>
	allow <CIDR...>
	deny <CIDR...>
	fallback_policy <策略>
}
```

- **timeout** 指定等待 PROXY 标头的最大持续时间。默认为 `5s`。

- **allow** 是受信任来源的 CIDR 范围列表，以接收 PROXY 标头。Unix 套接字默认受信任，不属于此选项。

- **deny** 是要拒绝接收 PROXY 标头的受信任来源的 CIDR 范围列表。

- **fallback_policy** 是当 PROXY 标头来自不在 allow/deny 任一列表中的地址时要采取的操作。默认的 fallback 策略是 `ignore`。`fallback_policy` 的接受值为：
	- `ignore`：使用 PROXY 标头中的地址，但接受连接
	- `use`：使用 PROXY 标头中的地址
	- `reject`：当发送 PROXY 标头时拒绝连接
	- `require`：要求连接发送 PROXY 标头，如果不存在则拒绝
	- `skip`：接受连接而不要求 PROXY 标头。


例如，对于一个 HTTPS 服务器（需要 `tls` 监听器包装器），它接受来自特定 IP 范围的 PROXY 标头，并拒绝来自不同范围的 PROXY 标头，超时时间为 2 秒：

```caddy
{
	servers {
		listener_wrappers {
			proxy_protocol {
				timeout 2s
				allow 192.168.86.1/24 192.168.86.1/24
				deny 10.0.0.0/8
				fallback_policy reject
			}
			tls
		}
	}
}
```


##### `timeouts`

- **read_body** 是一个[时长值](/docs/conventions#durations)，设置允许从客户端上传读取的时间。将其设置为较短的、非零值可以缓解 slowloris 攻击，但也可能影响合法慢速客户端。默认无超时。

- **read_header** 是一个[时长值](/docs/conventions#durations)，设置允许从客户端请求标头读取的时间。默认无超时。

- **write** 是一个[时长值](/docs/conventions#durations)，设置允许写入客户端的时间。请注意，在提供大文件时将其设置为较小的值可能会对合法慢速客户端产生负面影响。默认无超时。

- **idle** 是一个[时长值](/docs/conventions#durations)，设置启用 keep-alive 时等待下一个请求的最长时间。默认为 5 分钟，以帮助避免资源耗尽。

```caddy
{
	servers {
		timeouts {
			read_body   10s
			read_header 5s
			write       30s
			idle        10m
		}
	}
}
```


##### `keepalive_interval`

当没有其他数据传输时，发送 TCP keepalive 数据包以保持连接活跃的时间间隔。默认为 `15s`。

```caddy
{
	servers {
		keepalive_interval 30s
	}
}
```


##### `keepalive_idle`

在发送 TCP keepalive 数据包之前，连接必须处于空闲状态的时间。默认为 `15s`。

```caddy
{
	servers {
		keepalive_idle 1m
	}
}
```


##### `keepalive_count`

在认为连接断开之前发送的 TCP keepalive 数据包的最大数量。默认为 `9`。

```caddy
{
	servers {
		keepalive_count 5
	}
}
```


##### `0rtt`

默认情况下，QUIC 监听器（即 HTTP/3）启用 0-RTT（早期数据），以允许客户端在 TLS 握手的第一次往返中发送数据，这可以提高重复连接的性能。

您可以将其设置为 `off` 以禁用 QUIC 监听器的 0-RTT。禁用 0-RTT 的一个原因是如果使用了 [`remote_ip` 匹配器](/docs/caddyfile/matchers#remote-ip)，这会在 TLS 握手完成之前引入对远程地址验证的依赖。在这种情况下会写入 HTTP 425 响应，但某些客户端（浏览器）可能会行为异常且不执行重试，因此禁用 0-RTT 可以确保用户不会看到 425 响应，代价是失去 0-RTT 的性能优势。

```caddy
{
	servers {
		0rtt off
	}
}
```


##### `trusted_proxies`

允许配置代理服务器的 IP 范围（CIDR），这些服务器发出的请求应被信任。默认情况下，不信任任何代理。

启用后，受信任的请求将从 HTTP 标头中解析出_真实_客户端 IP（默认情况下，从 `X-Forwarded-For` 解析；请参见 [`client_ip_headers`](#client-ip-headers) 以配置其他标头）。如果受信任，客户端 IP 将被添加到[访问日志](/docs/caddyfile/directives/log)中，可作为 `{client_ip}` [占位符](/docs/caddyfile/concepts#placeholders)使用，并允许使用 [`client_ip` 匹配器](/docs/caddyfile/matchers#client-ip)。如果请求不是来自受信任的代理，则客户端 IP 设置为直接传入连接的远程 IP 地址，或设置为 PROXY 协议设置的地址（如果已使用）。默认情况下，从左到右解析标头中的 IP。请参见 [`trusted_proxies_strict`](#trusted-proxies-strict) 以更改此行为。

某些匹配器或处理程序可能会根据请求的信任状态做出决策。例如，如果受信任，[`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy#defaults) 处理程序将代理并增加敏感的 `X-Forwarded-*` 请求标头。

目前，Caddy 的标准发行版中仅包含 `static` [IP 源模块](/docs/json/apps/http/servers/trusted_proxies/)，但可以通过插件[扩展](/docs/extending-caddy)以维护动态的 IP 范围列表。


###### `static`

接受一个静态（不变）的 IP 范围（CIDR）列表以信任。

作为快捷方式，可以使用 `private_ranges` 来匹配所有私有 IPv4 和 IPv6 范围。这等同于指定所有这些范围：`192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 127.0.0.1/8 fd00::/8 ::1`。

语法如下：

```caddy-d
trusted_proxies static [private_ranges] <范围...>
```

这是一个完整示例，信任一个示例 IPv4 范围和一个 IPv6 范围：

```caddy
{
	servers {
		trusted_proxies static 12.34.56.0/24 1200:ab00::/32
	}
}
```

##### `trusted_proxies_strict`

当启用 [`trusted_proxies`](#trusted-proxies) 时，默认情况下，标头（由 [`client_ip_headers`](#client-ip-headers) 配置）中的 IP 从左到右解析。找到的第一个不受信任的 IP 地址成为真实客户端地址。从 v2.8 开始，您可以选择使用 `trusted_proxies_strict` 进行从右到左的标头解析。默认情况下，为向后兼容，此选项处于禁用状态。

上游代理（如 HAProxy、CloudFlare、AWS ALB、CloudFront 等）会将每个新连接的远程地址附加到 `X-Forwarded-For` 的右侧。建议在使用它们时启用 `trusted_proxies_strict`，因为最左边的 IP 地址可能被客户端伪造。

```caddy
{
	servers {
		trusted_proxies static private_ranges
		trusted_proxies_strict
	}
}
```

<aside class="tip">

特别是对于 AWS ALB，您肯定希望启用此选项。根据[其文档](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/x-forwarded-headers.html#w227aac13c27b9c15)，您只能通过将 XFF 模式设置为 `append` 来识别真实客户端 IP。此 IP 将附加到 `X-Forwarded-For` 的右侧，并且只能通过 `trusted_proxies_strict` 安全提取。

</aside>

##### `trusted_proxies_unix`

`trusted_proxies_unix` 选项允许信任所有来自 Unix 套接字的连接，当 Caddy 位于通过 Unix 套接字连接的反向代理（可能是另一个 Caddy 实例）之后时很有用（即 [`bind` 指令](/docs/caddyfile/directives/bind) 设置为 Unix 套接字）。默认情况下禁用。

```caddy
{
	servers {
		trusted_proxies_unix
	}
}
```

##### `client_ip_headers`

与 [`trusted_proxies`](#trusted-proxies) 配合使用，允许配置用于确定客户端 IP 地址的标头。默认情况下，仅考虑 `X-Forwarded-For`。可以指定多个标头字段，此时将使用第一个非空标头值。

```caddy
{
	servers {
		trusted_proxies static private_ranges
		client_ip_headers X-Forwarded-For X-Real-IP
	}
}
```


##### `metrics`

启用 Prometheus 指标收集；在抓取指标之前是必需的。请注意，指标会降低非常繁忙的服务器上的性能。（我们的社区正在努力改进这一点。请参与进来！）

```caddy
{
	metrics
}
```

您可以添加 `per_host` 选项，以用指标的主机名标记指标。

```caddy
{
	metrics {
		per_host
	}
}
```

由于观察所有可能的主机（客户端可能发送）具有无限的基数潜力，Caddy 将仅为配置的主机记录指标，而所有其他主机（例如 attacker.com）将聚合在“_other”标签下。要强制观察所有主机，并且如果潜在无限基数是可接受的风险，您可以添加 `observe_catchall_hosts`。请注意，添加 `observe_catchall_hosts` 不会启用 `per_host`。但是，对于 HTTPS 服务器会自动启用此功能（因为证书提供了一些防止无限基数的保护），而对于 HTTP 服务器默认禁用，以防止来自任意 Host 标头的基数攻击。

```caddy
{
	metrics {
		per_host
		observe_catchall_hosts
	}
}
```

##### `trace`

记录每个被调用的单独处理程序。要求日志以 `DEBUG` 级别发出（您可以使用 [`debug` 全局选项](#debug) 来实现）。

注意：这可能会记录您的 HTTP 处理程序模块的配置；如果在配置中存在敏感数据，请不要在不安全的环境中启用此选项。

⚠️ 这是一个实验性功能。可能会更改或删除。

```caddy
{
	servers {
		trace
	}
}
```


##### `max_header_size`

从客户端 HTTP 请求标头解析的最大大小。如果超过限制，服务器将以 HTTP 状态 `431 请求标头字段过大` 响应。它接受 [go-humanize](https://github.com/dustin/go-humanize/blob/master/bytes.go) 支持的所有格式。默认情况下，限制为 `1MB`。

```caddy
{
	servers {
		max_header_size 5MB
	}
}
```


##### `enable_full_duplex`

为 HTTP/1 请求启用全双工通信。

对于 HTTP/1 请求，Go HTTP 服务器默认在开始写入响应之前消耗请求正文的任何未读部分，从而阻止处理程序同时从请求中读取和写入响应。启用此选项将禁用此行为，并允许处理程序在并发写入响应时继续从请求中读取。

对于 HTTP/2+ 请求，Go HTTP 服务器始终允许并发读取和响应，因此此选项无效。

请与您的 HTTP 客户端进行彻底测试，因为某些较旧的客户端可能不支持全双工 HTTP/1，这可能导致它们死锁。有关更多信息，请参见 [golang/go#57786](https://github.com/golang/go/issues/57786)。

⚠️ 这是一个实验性功能。可能会更改或删除。

```caddy
{
	servers {
		enable_full_duplex
	}
}
```


##### `log_credentials`

默认情况下，包含潜在敏感信息（`Cookie`、`Set-Cookie`、`Authorization` 和 `Proxy-Authorization`）的标头的访问日志（通过 [`log` 指令](/docs/caddyfile/directives/log) 启用）将记录为 `REDACTED`。

如果您希望_不_遮蔽这些标头，可以启用 `log_credentials` 选项。

```caddy
{
	servers {
		log_credentials
	}
}
```



##### `protocols`

空格分隔的列表，指定要支持的 HTTP 协议。

默认值：`h1 h2 h3`

接受的值是：
- `h1` 用于 HTTP/1.1
- `h2` 用于 HTTP/2
- `h2c` 用于基于明文的 HTTP/2
- `h3` 用于 HTTP/3

目前，启用 HTTP/2（包括 H2C）必然意味着启用 HTTP/1.1，因为 Go 标准库不允许我们在使用其 HTTP 服务器时禁用 HTTP/1.1。但是，HTTP/1.1 或 HTTP/3 可以独立启用。

请注意，H2C（"基于明文的 HTTP/2" 或 "H2 over TCP"）和 HTTP/3 不是由 Go 标准库实现的，因此某些功能可能有限制。我们建议除非您的应用程序绝对必要，否则不要启用 H2C。

```caddy
{
	servers :80 {
		protocols h1 h2c
	}
}
```



##### `strict_sni_host`

启用此选项要求请求的 `Host` 标头与客户端 TLS ClientHello 发送的 `ServerName` 值匹配，这是使用 TLS 客户端身份验证时的必要安全措施。如果不匹配，将向客户端写入 HTTP 状态 `421 错误定向请求` 响应。

如果配置了[客户端身份验证](/docs/caddyfile/directives/tls#client_auth)，此选项将自动开启。这可以防止 TLS 客户端身份验证绕过（域名前置），否则攻击者可以在 TLS 握手中发送不受保护的 SNI 值，然后在建立连接后将受保护的域名放入 Host 标头中来利用此漏洞。此行为是安全的默认行为，但您可以明确将其关闭，使用 `insecure_off`；例如，在运行代理且希望使用域名前置且不基于主机名限制访问的情况下。

```caddy
{
	servers {
		strict_sni_host on
	}
}
```



## 文件系统

`filesystem` 全局选项允许声明一个或多个可用于文件 I/O 的文件系统。

这可以让您连接到云中运行的远程文件系统、具有类似文件接口的数据库，甚至从嵌入在 Caddy 二进制文件中的文件中读取。

文件系统使用名称来标识它们进行声明。这意味着您可以连接到同一类型的多个文件系统，如果需要的话。

默认情况下，Caddy 没有任何文件系统模块，因此您需要使用要使用的文件系统的插件构建 Caddy。

#### 示例

使用一个虚构的 `custom` 文件系统模块，您可以声明两个文件系统：

```caddy
{
	filesystem foo custom {
		...
	}

	filesystem bar custom {
		...
	}
}

foo.example.com {
	fs foo
	file_server
}

foo.example.com {
	fs bar
	file_server
}
```



## PKI 选项

PKI（公钥基础设施）应用是 Caddy [本地 HTTPS](/docs/automatic-https#local-https) 和 [ACME 服务器](/docs/caddyfile/directives/acme_server) 功能的基础。该应用定义了能够签署证书的证书颁发机构（CA）。

默认的 CA ID 是 `local`。如果在配置 `ca` 时省略了 ID，则假定为 `local`。

##### `name`
证书颁发机构的面向用户的名称。

默认值：`Caddy Local Authority`

```caddy
{
	pki {
		ca local {
			name "My Local CA"
		}
	}
}
```

##### `root_cn`
要放在根证书的 CommonName 字段中的名称。

默认值：`{pki.ca.name} - {time.now.year} ECC Root`

```caddy
{
	pki {
		ca local {
			root_cn "My Local CA - 2024 ECC Root"
		}
	}
}
```

##### `intermediate_cn`
要放在中间证书的 CommonName 字段中的名称。

默认值：`{pki.ca.name} - ECC Intermediate`

```caddy
{
	pki {
		ca local {
			intermediate_cn "My Local CA - ECC Intermediate"
		}
	}
}
```

##### `intermediate_lifetime`
中间证书的有效期[时长](/docs/conventions#durations)。此值**必须**小于根证书的有效期（`3600d` 或 10 年）。

默认值：`7d`。除非绝对必要，否则_不建议_更改。

```caddy
{
	pki {
		ca local {
			intermediate_lifetime 30d
		}
	}
}
```

##### `maintenance_interval`
检查中间证书（以及根证书，如果适用）是否需要续期的频率[时长](/docs/conventions#durations)。

默认值：`10m`。除非绝对必要，否则_不建议_更改。

```caddy
{
	pki {
		ca local {
			maintenance_interval 30m
		}
	}
}
```

##### `renewal_window_ratio`
在 Caddy 尝试续期证书之前必须剩余的证书有效期的比率（介于 0 和 1 之间）。例如，如果证书的有效期为 1 年，且此比率为 `0.2`（默认值），则 Caddy 将在证书剩余 73 天或更少时持续尝试续期。

```caddy
{
	pki {
		ca local {
			renewal_window_ratio 0.1
		}
	}
}
```


##### `root`
用作 CA 根证书的密钥对（证书和私钥）。如果未指定，将自动生成并管理一个。

- **format** 是提供证书和私钥的格式。目前仅支持 `pem_file`，这是默认值，因此此字段可选。
- **cert** 是证书。使用 `pem_file` 格式时，这应该是 PEM 文件的路径。
- **key** 是私钥。使用 `pem_file` 格式时，这应该是 PEM 文件的路径。

##### `intermediate`
用作 CA 中间证书的密钥对（证书和私钥）。如果未指定，将自动生成并管理一个。

- **format** 是提供证书和私钥的格式。目前仅支持 `pem_file`，这是默认值，因此此字段可选。
- **cert** 是证书。使用 `pem_file` 格式时，这应该是 PEM 文件的路径。
- **key** 是私钥。使用 `pem_file` 格式时，这应该是 PEM 文件的路径。

```caddy
{
	pki {
		ca local {
			root {
				format pem_file
				cert /path/to/root.pem
				key /path/to/root.key
			}
			intermediate {
				format pem_file
				cert /path/to/intermediate.pem
				key /path/to/intermediate.key
			}
		}
	}
}
```


## 事件选项

当有趣的事情发生时（或即将发生时），Caddy 模块会发出事件。

事件通常包含一个元数据负载。了解事件及其负载的最佳方法是通过每个模块的文档，但您也可以通过启用 [`debug` 全局选项](#debug) 并阅读日志来查看事件及其数据负载。

##### `on`

将事件处理程序绑定到命名事件。指定事件处理程序模块的名称，后跟其配置。

例如，在获取证书后运行一个命令（需要[第三方插件 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/mholt/caddy-events-exec)），使用占位符将部分事件负载传递给脚本：

```caddy
{
	events {
		on cert_obtained exec ./my-script.sh {event.data.certificate_path}
	}
}
```

### 事件

Caddy 发出以下标准事件：

- [`tls` 事件 <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddyserver/certmagic#events)
- [`reverse_proxy` 事件](/docs/caddyfile/directives/reverse_proxy#events)

插件也可能发出事件，因此请查看其文档获取详细信息。