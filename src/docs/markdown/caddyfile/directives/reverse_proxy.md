---
title: reverse_proxy（Caddyfile 指令）
---

<script>
ready(function() {
	// 修复响应匹配器，使其以正确颜色渲染，
	// 并链接到响应匹配器部分
	$$_('pre.chroma .k').forEach(item => {
		if (item.innerText.includes('@')) {
			let text = item.innerText.replace(/</g, '&lt;').replace(/>/g, '&gt;');
			let url = '#' + item.innerText.replace(/_/g, "-");
			item.classList.add('nd');
			item.classList.remove('k');
			item.innerHTML = `<a href="/docs/caddyfile/response-matchers" style="color: inherit;" title="响应匹配器">${text}</a>`;
		}
	});

	// 修复匹配器占位符
	const nameMatchers = $$_('pre.chroma .nd');
	for (let item of nameMatchers) {
		if (item.innerText.includes('@name')) {
			item.innerHTML = '<a href="/docs/caddyfile/response-matchers" style="color: inherit;" title="响应匹配器">@name</a>';
			break;
		}
	}
	
	const replaceStatusElements = $$_('pre.chroma .k');
	for (let item of replaceStatusElements) {
		if (item.innerText.includes('replace_status') && item.nextElementSibling) {
			const next = item.nextElementSibling;
			const span = document.createElement('span');
			span.className = 'nd';
			next.parentNode.insertBefore(span, next);
			span.appendChild(next);
			span.innerHTML = '<a href="/docs/caddyfile/response-matchers" style="color: inherit;" title="响应匹配器">[&lt;matcher&gt;]</a>';
			break;
		}
	}
	
	const handleResponseElements = $$_('pre.chroma .k');
	for (let item of handleResponseElements) {
		if (item.innerText.includes('handle_response') && item.nextElementSibling) {
			const next = item.nextElementSibling;
			const span = document.createElement('span');
			span.className = 'nd';
			next.parentNode.insertBefore(span, next);
			span.appendChild(next);
			span.innerHTML = '<a href="/docs/caddyfile/response-matchers" style="color: inherit;" title="响应匹配器">[&lt;matcher&gt;]</a>';
			break;
		}
	}

	// 如果在页面上找到匹配的锚标签，我们将为所有子指令添加链接。
	addLinksToSubdirectives();
});
</script>

# reverse_proxy

将请求代理到一个或多个后端，并支持可配置的传输、负载均衡、健康检查、请求操作和缓存选项。

- [语法](#syntax)
- [上游](#upstreams)
  - [上游地址](#upstream-addresses)
  - [动态上游](#dynamic-upstreams)
    - [SRV](#srv)
    - [A/AAAA](#aaaaa)
	- [Multi](#multi)
- [负载均衡](#load-balancing)
  - [主动健康检查](#active-health-checks)
  - [被动健康检查](#passive-health-checks)
  - [事件](#events)
- [流式传输](#streaming)
- [请求头](#headers)
- [重写](#rewrites)
- [传输](#transports)
  - [`http` 传输](#the-http-transport)
  - [`fastcgi` 传输](#the-fastcgi-transport)
- [拦截响应](#intercepting-responses)
- [示例](#examples)



## 语法

```caddy-d
reverse_proxy [<matcher>] [<upstreams...>] {
	# 后端
	to      <upstreams...>
	dynamic <module> ...

	# 负载均衡
	lb_policy       <name> [<options...>]
	lb_retries      <retries>
	lb_try_duration <duration>
	lb_try_interval <interval>
	lb_retry_match  <request-matcher>

	# 主动健康检查
	health_uri          <uri>
	health_upstream     <ip:port>
	health_port         <port>
	health_interval     <interval>
	health_passes       <num>
	health_fails	    <num>
	health_timeout      <duration>
	health_method       <method>
	health_status       <status>
	health_request_body <body>
	health_body         <regexp>
	health_follow_redirects
	health_headers {
		<field> [<values...>]
	}

	# 被动健康检查
	fail_duration     <duration>
	max_fails         <num>
	unhealthy_status  <status>
	unhealthy_latency <duration>
	unhealthy_request_count <num>

	# 流式传输
	flush_interval     <duration>
	request_buffers    <size>
	response_buffers   <size>
	stream_timeout     <duration>
	stream_close_delay <duration>

	# 请求/请求头操作
	trusted_proxies [private_ranges] <ranges...>
	header_up   [+|-]<field> [<value|regexp> [<replacement>]]
	header_down [+|-]<field> [<value|regexp> [<replacement>]]
	method <method>
	rewrite <to>

	# 往返
	transport <name> {
		...
	}

	# 可选：拦截来自上游的响应
	@name {
		status <code...>
		header <field> [<value>]
	}
	replace_status [<matcher>] <status_code>
	handle_response [<matcher>] {
		<directives...>

		# 仅在 handle_response 中可用的特殊指令
		copy_response [<matcher>] [<status>] {
			status <status>
		}
		copy_response_headers [<matcher>] {
			include <fields...>
			exclude <fields...>
		}
	}
}
```



## 上游

- **&lt;upstreams...&gt;** 是一个上游（后端）列表，请求将被代理到这些后端。
- **to** <span id="to"/> 是指定上游列表的另一种方式，每行一个（或多个）。
- **dynamic** <span id="dynamic"/> 配置一个 _动态上游_ 模块。这允许为每个请求动态获取上游列表。有关标准动态上游模块的说明，请参见下面的[动态上游](#dynamic-upstreams)。动态上游在每个代理循环迭代中获取（即，如果启用了负载均衡重试，则每个请求可能多次获取），并且优先于静态上游。如果发生错误，代理将回退到使用任何静态配置的上游。


### 上游地址

静态上游地址可以采用仅包含方案和主机/端口的 URL 形式，或传统的 [Caddy 网络地址](/docs/conventions#network-addresses)形式。有效示例：

- `localhost:4000`
- `127.0.0.1:4000`
- `[::1]:4000`
- `http://localhost:4000`
- `https://example.com`
- `h2c://127.0.0.1`
- `example.com`
- `unix//var/php.sock`
- `unix+h2c//var/grpc.sock`
- `localhost:8001-8006`
- `[fe80::ea9f:80ff:fe46:cbfd%eth0]:443`

默认情况下，通过明文 HTTP 连接到上游。使用 URL 形式时，方案可以用作设置某些 [`transport`](#transports) 默认值的简写。
- 使用 `https://` 作为方案将使用 [`http` 传输](#the-http-transport)并启用 [`tls`](#tls)。

  此外，您可能需要覆盖 `Host` 请求头，使其与 TLS SNI 值匹配，服务器使用该值进行路由和证书选择。有关更多详细信息，请参见下面的 [HTTPS](#https) 部分。

- 使用 `h2c://` 作为方案将使用 [`http` 传输](#the-http-transport)并将 [HTTP 版本](#versions)设置为允许明文 HTTP/2 连接。

- 使用 `http://` 作为方案与省略方案相同，因为 HTTP 已经是默认的。包含此语法是为了与其他方案快捷方式对称。

方案不能混合使用，因为它们会修改共同的传输配置（启用 TLS 的传输不能同时承载 HTTPS 和明文 HTTP）。任何显式的传输配置都不会被覆盖，省略方案或使用其他端口不会假定特定的传输。

当使用带有区域的 IPv6（例如，具有特定网络接口的链路本地地址）时，**不能**使用方案作为快捷方式，因为 `%` 会导致 URL 解析错误；请改用显式配置传输。

当使用 [网络地址](/docs/conventions#network-addresses) 形式时，网络类型作为前缀指定给上游地址。这不能与 URL 方案结合使用。作为一个特例，`unix+h2c/` 被支持作为 `unix/` 网络并结合 `h2c://` 方案相同效果的快捷方式。端口范围被支持作为快捷方式，它会扩展为具有相同主机的多个上游。

上游地址**不能**包含路径或查询字符串，因为这将意味着在代理时同时重写请求，此行为未定义或不受支持。如果需要，您可以使用 [`rewrite`](/docs/caddyfile/directives/rewrite) 指令。

如果地址不是 URL（即没有方案），则可以使用[占位符](/docs/caddyfile/concepts#placeholders)，但这会使上游成为 _动态静态_，意味着在健康检查和负载均衡方面，许多不同的后端充当单个静态上游。如果可能，我们建议使用[动态上游](#dynamic-upstreams)模块。使用占位符时，**必须**包含端口（通过占位符替换或作为地址的静态后缀）。


### 动态上游

Caddy 的反向代理内置了一些动态上游模块。请注意，使用动态上游会对负载均衡和健康检查产生影响，具体取决于特定的策略配置：主动健康检查不会为动态上游运行；如果上游列表相对稳定且一致（尤其是轮询），负载均衡和被动健康检查效果最佳。理想情况下，动态上游模块只返回健康、可用的后端。


#### SRV

从 SRV DNS 记录中检索上游。

```caddy-d
	dynamic srv [<full_name>] {
		service   <service>
		proto     <proto>
		name      <name>
		refresh   <interval>
		resolvers <ip...>
		dial_timeout        <duration>
		dial_fallback_delay <duration>
	}
```

- **&lt;full_name&gt;** 是待查询记录的全域名（即 `_service._proto.name`）。
- **service** 是全名的服务组件。
- **proto** 是全名的协议组件。可以是 `tcp` 或 `udp`。
- **name** 是名称组件。或者，如果 `service` 和 `proto` 为空，则为要查询的全域名。
- **refresh** 是刷新缓存结果的频率。默认值：`1m`
- **resolvers** 是用于覆盖系统解析器的 DNS 解析器列表。
- **dial_timeout** 是拨号查询的超时时间。
- **dial_fallback_delay** 是在生成 RFC 6555 快速回退连接之前等待的时间。默认值：`300ms`



#### A/AAAA

从 A/AAAA DNS 记录中检索上游。

```caddy-d
	dynamic a [<name> <port>] {
		name      <name>
		port      <port>
		refresh   <interval>
		resolvers <ip...>
		dial_timeout        <duration>
		dial_fallback_delay <duration>
		versions ipv4|ipv6
	}
```

- **name** 是要查询的域名。
- **port** 是要用于后端的端口。
- **refresh** 是刷新缓存结果的频率。默认值：`1m`
- **resolvers** 是用于覆盖系统解析器的 DNS 解析器列表。
- **dial_timeout** 是拨号查询的超时时间。
- **dial_fallback_delay** 是在生成 RFC 6555 快速回退连接之前等待的时间。默认值：`300ms`
- **versions** 是要解析的 IP 版本列表。默认值：`ipv4 ipv6`，分别对应 A 和 AAAA 记录。


#### Multi

追加多个动态上游模块的结果。如果您想要冗余的上游来源，例如：一个由次要 SRV 集群备份的主要 SRV 集群，则此功能非常有用。

```caddy-d
	dynamic multi {
		<source> [...]
	}
```

- **&lt;source&gt;** 是动态上游模块的名称，后跟其配置。可以指定多个。




## 负载均衡

负载均衡通常用于在多个上游之间分配流量。通过启用重试，也可以与一个或多个上游一起使用，以保持请求直到可以选择健康的上游（例如，在重启或重新部署上游时等待并减轻错误）。

默认情况下启用，使用 `random` 策略。默认情况下禁用重试。

- **lb_policy** <span id="lb_policy"/> 是负载均衡策略的名称，以及任何选项。默认值：`random`。

  对于涉及哈希的策略，使用[最高随机权重 (HRW)](https://en.wikipedia.org/wiki/Rendezvous_hashing) 算法来确保具有相同哈希键的客户端或请求映射到相同的上游，即使上游列表发生变化。

  某些策略支持回退作为选项（如果注明），在这种情况下，它们采用一个[块](/docs/caddyfile/concepts#blocks)，其中包含 `fallback <policy>`，该策略又采用另一个负载均衡策略。对于这些策略，默认回退是 `random`。配置回退允许在主策略未选择时使用辅助策略，从而实现强大的组合。如果需要，回退可以嵌套多次。
  
  例如，`header` 可以用作主策略，允许开发人员选择特定的上游，而所有其他连接则回退到 `first`，以实现主/从故障转移。
  ```caddy-d
  lb_policy header X-Upstream {
  	fallback first
  }
  ```

	- `random` 随机选择一个上游

	- `random_choose <n>` 随机选择两个或更多上游，然后选择负载最小的一个（`n` 通常为 2）

	- `first` 按照配置中定义的顺序选择第一个可用的上游，允许主/从故障转移；请记得同时启用健康检查，否则不会发生故障转移

	- `round_robin` 依次轮询每个上游

	- `weighted_round_robin <weights...>` 依次轮询每个上游，并尊重提供的权重。权重参数的数量应与配置的上游数量匹配。权重应为非负整数。例如，对于两个上游和权重 `5 1`，第一个上游将连续选择 5 次，然后第二个上游选择一次，然后重复循环。如果使用零作为权重，则将禁用为新请求选择该上游。

	- `least_conn` 选择当前请求数量最少的那个上游；如果有多个主机具有最少的请求数，则随机选择其中一个主机

	- `ip_hash` 将远程 IP（直接对等方）映射到粘性上游

	- `client_ip_hash` 将客户端 IP 映射到粘性上游；最好与 `servers > trusted_proxies` 全局选项配对使用，该选项启用真实客户端 IP 解析，否则其行为与 `ip_hash` 相同

	- `uri_hash` 将请求 URI（路径和查询）映射到粘性上游

	- `query [key]` 通过哈希查询值，将请求查询映射到粘性上游；如果指定的键不存在，则使用回退策略选择上游（默认 `random`）

	- `header [field]` 通过哈希标头值，将请求标头映射到粘性上游；如果指定的标头字段不存在，则使用回退策略选择上游（默认 `random`）

	- `cookie [<name> [<secret>]]` 在客户端的第一次请求（没有 cookie 时），使用回退策略选择上游（默认 `random`），并在响应中添加 `Set-Cookie` 标头（如果未指定，默认 cookie 名称为 `lb`）。cookie 值是所选上游的拨号地址，使用 HMAC-SHA256 哈希（使用 `<secret>` 作为共享密钥，如果未指定则为空字符串）。
	
	  在后续请求中，如果 cookie 存在，则 cookie 值将映射到相同的上游（如果可用）；如果不可用或未找到，则使用回退策略选择新上游，并将 cookie 添加到响应中。

	  如果您希望使用特定上游进行调试，可以使用密钥对上游地址进行哈希，并在 HTTP 客户端（浏览器或其他）中设置 cookie。例如，使用 PHP，您可以运行以下命令来计算 cookie 值，其中 `10.1.0.10:8080` 是您的一个上游地址，`secret` 是您配置的密钥。
	  ```php
	  echo hash_hmac('sha256', '10.1.0.10:8080', 'secret');
	  // cdd96966817dd14a99f47ee17451464f29998da170814a16b483e4c1ff4c48cf
	  ```
	
	  您可以通过 JavaScript 控制台在浏览器中设置 cookie，例如设置名为 `lb` 的 cookie：
	  ```js
	  document.cookie = "lb=cdd96966817dd14a99f47ee17451464f29998da170814a16b483e4c1ff4c48cf";
	  ```

- **lb_retries** <span id="lb_retries"/> 是当下一个可用的主机宕机时，为每个请求重试选择可用后端的次数。默认情况下，重试被禁用（零）。

  如果同时配置了 [`lb_try_duration`](#lb_try_duration)，则如果达到持续时间，重试可能会提前停止。换句话说，重试持续时间优先于重试计数。

- **lb_try_duration** <span id="lb_try_duration"/> 是一个[持续时间值](/docs/conventions#durations)，定义当下一个可用的主机宕机时，为每个请求尝试选择可用后端的持续时间。默认情况下，重试被禁用（持续时间为零）。

  客户端将等待最多这么长时间，同时负载均衡器尝试找到可用的上游主机。一个合理的起点可能是 `5s`，因为 HTTP 传输的默认拨号超时是 `3s`，所以如果第一个选定的上游无法访问，至少应该允许一次重试；但请随意试验以找到适合您用例的正确平衡。

- **lb_try_interval** <span id="lb_try_interval"/> 是一个[持续时间值](/docs/conventions#durations)，定义在从池中选择下一个主机之前等待的时间。默认值为 `250ms`。仅当对上游主机的请求失败时才相关。请注意，如果所有后端都宕机且延迟非常低，将此值设置为 `0` 且 `lb_try_duration` 非零可能会导致 CPU 空转。

- **lb_retry_match** <span id="lb_retry_match"/> 限制允许重试的请求。如果与上游的连接成功但后续往返失败，则请求必须匹配此条件才能被重试。如果与上游的连接失败，则始终允许重试。默认情况下，仅重试 `GET` 请求。

  此选项的语法与[命名的请求匹配器](/docs/caddyfile/matchers#named-matchers)相同，但没有 `@name`。如果您只需要一个匹配器，可以将其配置在同一行上。对于多个匹配器，则需要一个块。



### 主动健康检查

主动健康检查在后台按计时器执行健康检查。要启用此功能，需要 `health_uri` 或 `health_port`。

- **health_uri** <span id="health_uri"/> 是主动健康检查的 URI 路径（以及可选的查询）。

- **health_upstream** <span id="health_upstream"/> 是用于主动健康检查的 ip:port，如果与上游不同。应与 `health_header` 和 `{http.reverse_proxy.active.target_upstream}` 结合使用。

- **health_port** <span id="health_port"/> 是用于主动健康检查的端口，如果与上游的端口不同。如果使用 `health_upstream`，则忽略此选项。

- **health_interval** <span id="health_interval"/> 是一个[持续时间值](/docs/conventions#durations)，定义执行主动健康检查的频率。默认值：`30s`。

- **health_passes** <span id="health_passes"/> 是在将后端再次标记为健康之前所需的连续健康检查次数。默认值：`1`。

- **health_fails** <span id="health_fails"/> 是在将后端标记为不健康之前所需的连续健康检查次数。默认值：`1`。

- **health_timeout** <span id="health_timeout"/> 是一个[持续时间值](/docs/conventions#durations)，定义在将后端标记为宕机之前等待回复的时间。默认值：`5s`。

- **health_method** <span id="health_method"/> 是用于主动健康检查的 HTTP 方法。默认值：`GET`。

- **health_status** <span id="health_status"/> 是期望从健康后端获得的 HTTP 状态码。可以是 3 位状态码，或结尾为 `xx` 的状态码类。例如：`200`（默认值），或 `2xx`。

- **health_request_body** <span id="health_request_body"/> 是一个字符串，表示要随主动健康检查一起发送的请求正文。

- **health_body** <span id="health_body"/> 是一个子字符串或正则表达式，用于匹配主动健康检查的响应正文。如果后端未返回匹配的正文，则将其标记为宕机。

- **health_follow_redirects** <span id="health_follow_redirects"/> 将导致健康检查遵循上游提供的重定向。默认情况下，重定向响应会导致健康检查计为失败。

- **health_headers** <span id="health_headers"/> 允许指定在主动健康检查请求上设置的标头。如果您需要更改 `Host` 标头，或需要在健康检查中为后端提供某种身份验证，这将非常有用。



### 被动健康检查

被动健康检查与实际代理请求一起内联发生。要启用此功能，需要 `fail_duration`。

- **fail_duration** <span id="fail_duration"/> 是一个[持续时间值](/docs/conventions#durations)，定义记住失败请求的时间。持续时间 > `0` 启用被动健康检查；默认值为 `0`（关闭）。一个合理的起点可能是 `30s`，以平衡错误率与在将不健康的上游重新上线时的响应能力；但请随意试验以找到适合您用例的正确平衡。

- **max_fails** <span id="max_fails"/> 是在 `fail_duration` 内将后端视为宕机之前所需的最大失败请求数；必须 >= `1`；默认值为 `1`。

- **unhealthy_status** <span id="unhealthy_status"/> 如果响应返回这些状态码之一，则将请求视为失败。可以是 3 位状态码，或结尾为 `xx` 的状态码类，例如：`404` 或 `5xx`。

- **unhealthy_latency** <span id="unhealthy_latency"/> 是一个[持续时间值](/docs/conventions#durations)，如果获取响应花费了这么长时间，则将请求视为失败。

- **unhealthy_request_count** <span id="unhealthy_request_count"/> 是在将后端标记为宕机之前允许的并发请求数。换句话说，如果特定后端当前正在处理此数量的请求，则将其视为“过载”，并且将优先选择其他后端。

  这应该是一个相当大的数字；配置此选项意味着代理将限制总并发请求数为 `unhealthy_request_count × upstreams_count`，超过该点后的任何请求都将因为没有可用的上游而导致错误。


## 事件

当上游从健康变为不健康或反之亦然时，会触发[一个事件](/docs/caddyfile/options#event-options)。这些事件可用于触发其他操作，例如发送通知或记录消息。事件如下：

- `healthy` 在上游从不健康变为健康时触发
- `unhealthy` 在上游从健康变为不健康时触发

在这两种情况下，`host` 作为元数据包含在事件中，用于标识状态发生变化的上游。它可以作为占位符与 `exec` 事件处理程序一起使用，例如 `{event.data.host}`。



## 流式传输

默认情况下，代理部分缓冲响应以提高线路效率。

代理还支持 WebSocket 连接，执行 HTTP 升级请求，然后将连接转换为双向隧道。

<aside class="tip">

默认情况下，当配置重新加载时，WebSocket 连接会被强制关闭（向客户端和上游发送 Close 控制消息）。每个请求都持有对配置的引用，因此关闭旧连接对于控制内存使用是必要的。此关闭行为可以通过 [`stream_timeout`](#stream_timeout) 和 [`stream_close_delay`](#stream_close_delay) 选项进行自定义。

</aside>

- **flush_interval** <span id="flush_interval"/> 是一个[持续时间值](/docs/conventions#durations)，调整 Caddy 将响应缓冲区刷新到客户端的频率。默认情况下，不执行定期刷新。负值（通常为 -1）表示“低延迟模式”，该模式完全禁用响应缓冲，并在每次写入客户端后立即刷新，并且即使客户端提前断开连接，也不会取消对后端的请求。如果响应的以下任一条件适用，此选项将被忽略，并且响应会立即刷新到客户端：
	- `Content-Type: text/event-stream`
	- `Content-Length` 未知
	- 代理两侧均为 HTTP/2，`Content-Length` 未知，且 `Accept-Encoding` 未设置或为 "identity"

- **request_buffers** <span id="request_buffers"/> 将导致代理在将请求正文发送到上游之前，将多达 `<size>` 字节的请求正文读入缓冲区。这效率很低，仅当上游需要无延迟地读取请求正文时（这是上游应用程序应该修复的问题）才应使用。它接受 [go-humanize](https://github.com/dustin/go-humanize/blob/master/bytes.go) 支持的所有大小格式。

- **response_buffers** <span id="response_buffers"/> 将导致代理在将响应正文返回给客户端之前，将多达 `<size>` 字节的响应正文读入缓冲区。出于性能原因，应尽可能避免使用，但如果后端内存限制较严格，则可能有用。它接受 [go-humanize](https://github.com/dustin/go-humanize/blob/master/bytes.go) 支持的所有大小格式。

- **stream_timeout** <span id="stream_timeout"/> 是一个[持续时间值](/docs/conventions#durations)，在此之后流式请求（例如 WebSocket）将在超时结束时被强制关闭。这实质上取消了长时间保持打开的连接。一个合理的起点可能是 `24h`，以剔除超过一天的连接。默认值：无超时。

- **stream_close_delay** <span id="stream_close_delay"/> 是一个[持续时间值](/docs/conventions#durations)，延迟在配置卸载时强制关闭流式请求（例如 WebSocket）的操作；相反，流将保持打开状态，直到延迟结束。换句话说，启用此功能可以防止在 Caddy 配置重新加载时立即关闭流。启用此功能可能是一个好主意，以避免因先前配置关闭而断开连接的客户端重新连接造成惊群效应。一个合理的起点可能是 `5m`，以便用户在配置重新加载后有 5 分钟时间自然离开页面。默认值：无延迟。



## 请求头

代理可以在自身与后端之间**操作请求头**：

- **header_up** <span id="header_up"/> 设置、添加（使用 `+` 前缀）、删除（使用 `-` 前缀）或执行替换（通过使用两个参数：搜索和替换）发送到上游的后端请求标头。

- **header_down** <span id="header_down"/> 设置、添加（使用 `+` 前缀）、删除（使用 `-` 前缀）或执行替换（通过使用两个参数：搜索和替换）从后端传回的响应标头。

例如，设置一个请求标头，覆盖任何现有值：

```caddy-d
header_up Some-Header "the value"
```

添加一个响应标头；请注意，一个标头字段可以有多个值：

```caddy-d
header_down +Some-Header "first value"
header_down +Some-Header "second value"
```

删除一个请求标头，防止其到达后端：

```caddy-d
header_up -Some-Header
```

删除所有匹配的请求标头，使用后缀匹配：

```caddy-d
header_up -Some-*
```

删除 _所有_ 请求标头，以便单独添加您想要的（不推荐）：

```caddy-d
header_up -*
```

对请求标头执行正则表达式替换：

```caddy-d
header_up Some-Header "^prefix-([A-Za-z0-9]*)$" "replaced-$1-suffix"
```

使用的正则表达式语言是 RE2，包含在 Go 中。请参阅 [RE2 语法参考](https://github.com/google/re2/wiki/Syntax) 和 [Go regexp 语法概述](https://pkg.go.dev/regexp/syntax)。替换字符串被[展开](https://pkg.go.dev/regexp#Regexp.Expand)，允许使用捕获的值，例如 `$1` 表示第一个捕获组。


### 默认值

默认情况下，Caddy 将传入的标头（包括 `Host`）未经修改地传递到后端，但有以下三个例外：

- 它设置或增强 [`X-Forwarded-For`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For) 标头字段。
- 它设置 [`X-Forwarded-Proto`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Proto) 标头字段。
- 它设置 [`X-Forwarded-Host`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-Host) 标头字段。

<span id="trusted_proxies"/> 对于这些 `X-Forwarded-*` 标头，默认情况下，代理将忽略来自传入请求的值，以防止欺骗。

如果您的客户端连接到的第一个服务器不是 Caddy（例如，当 CDN 位于 Caddy 之前时），您可以将 `trusted_proxies` 配置为 IP 范围（CIDR）列表，从这些范围传入的请求被视为已发送了正确的标头值。

强烈建议您通过 `servers > trusted_proxies` [全局选项](/docs/caddyfile/options#trusted-proxies) 进行配置，而不是在代理中，以便这适用于服务器中的所有代理处理程序，并且这还有启用客户端 IP 解析的好处。

<aside class="tip">

如果您在 Caddy 前面使用 Cloudflare，请注意您可能容易受到 `X-Forwarded-For` 标头欺骗。我们的朋友在 [Authelia](https://www.authelia.com) 已经记录了一种[解决方法](https://www.authelia.com/integration/proxies/forwarded-headers/)，用于配置 Cloudflare 忽略此标头的传入值。

</aside>

此外，当使用 [`http` 传输](#the-http-transport)时，如果客户端请求中缺少 `Accept-Encoding: gzip` 标头，将设置该标头。这允许上游（如果支持）提供压缩内容。可以通过在传输上设置 [`compression off`](#compression) 来禁用此行为。


### HTTPS

由于（大多数）标头在代理时保留其原始值，因此在代理到 HTTPS 时，通常需要将 `Host` 标头覆盖为配置的上游地址，以使 `Host` 标头与 TLS ServerName 值匹配：

```caddy-d
reverse_proxy https://example.com {
	header_up Host {upstream_hostport}
}
```

自 Caddy v2.11.0 起，这是自动完成的，因此在代理到 HTTPS 时不再需要显式覆盖 `Host` 标头。如果您希望退出此行为，可以将 `Host` 标头设置为其原始值（但这几乎没意义）：

```caddy-d
reverse_proxy https://example.com {
	header_up Host {hostport}
}
```

`X-Forwarded-Host` 标头仍然[默认](#defaults)传递，因此上游如果需要知道原始 `Host` 标头值，仍可以使用它。

当在 Caddy 中终止 TLS 并通过 HTTP 进行代理（无论是到端口还是 Unix 套接字）时，同样适用。事实上，当 Caddy 本身是 `reverse_proxy` 的目标时，它必须接收正确的 Host。在 Unix 套接字情况下，`upstream_hostport` 将是套接字路径，并且必须显式设置 Host。



## 重写

默认情况下，Caddy 使用与传入请求相同的 HTTP 方法和 URI 执行上游请求，除非在中间件链中到达 `reverse_proxy` 之前执行了重写。

在代理之前，请求会被克隆；这确保在处理程序中对请求所做的任何修改都不会泄漏到其他处理程序。这在处理后需要在代理之后继续的情况下很有用。

除了[请求头操作](#headers)之外，请求的方法和 URI 可以在发送到上游之前进行更改：

- **method** <span id="method"/> 更改克隆请求的 HTTP 方法。如果方法更改为 `GET` 或 `HEAD`，则此处理程序**不会**将传入请求的正文发送到上游。如果您希望允许不同的处理程序使用请求正文，这将非常有用。
- **rewrite** <span id="rewrite"/> 更改克隆请求的 URI（路径和查询）。这与 [`rewrite` 指令](/docs/caddyfile/directives/rewrite)类似，不同之处在于它不会将重写保留在此处理程序的范围之外。

这些重写通常用于“预检查请求”模式，其中请求被发送到另一台服务器以帮助决定如何继续处理当前请求。

例如，可以将请求发送到身份验证网关，以决定请求是否来自经过身份验证的用户（例如，请求具有会话 cookie）并应继续，或者应重定向到登录页面。对于这种模式，Caddy 提供了一个快捷指令 [`forward_auth`](/docs/caddyfile/directives/forward_auth)，以跳过大多数配置模板。




## 传输

Caddy 的代理**传输**是可插拔的：

- **transport** <span id="transport"/> 定义如何与后端通信。默认值为 `http`。


### `http` 传输

```caddy-d
transport http {
	read_buffer             <size>
	write_buffer            <size>
	max_response_header     <size>
	proxy_protocol          v1|v2
	dial_timeout            <duration>
	dial_fallback_delay     <duration>
	response_header_timeout <duration>
	expect_continue_timeout <duration>
	resolvers <ip...>
	tls
	tls_client_auth <automate_name> | <cert_file> <key_file>
	tls_insecure_skip_verify
	tls_curves <curves...>
	tls_timeout <duration>
	tls_trust_pool <module>
	tls_server_name <server_name>
	tls_renegotiation <level>
	tls_except_ports <ports...>
	keepalive [off|<duration>]
	keepalive_interval <interval>
	keepalive_idle_conns <max_count>
	keepalive_idle_conns_per_host <count>
	versions <versions...>
	compression off
	max_conns_per_host <count>
	network_proxy <module>
}
```

- **read_buffer** <span id="read_buffer"/> 是读取缓冲区的大小（字节）。它接受 [go-humanize](https://github.com/dustin/go-humanize/blob/master/bytes.go) 支持的所有格式。默认值：`4KiB`。

- **write_buffer** <span id="write_buffer"/> 是写入缓冲区的大小（字节）。它接受 [go-humanize](https://github.com/dustin/go-humanize/blob/master/bytes.go) 支持的所有格式。默认值：`4KiB`。

- **max_response_header** <span id="max_response_header"/> 是从响应标头读取的最大字节数。它接受 [go-humanize](https://github.com/dustin/go-humanize/blob/master/bytes.go) 支持的所有格式。默认值：`10MiB`。

- **proxy_protocol** <span id="proxy_protocol"/> 在到上游的连接上启用 [PROXY 协议](https://github.com/haproxy/haproxy/blob/master/doc/proxy-protocol.txt)（由 HAProxy 推广），添加真实客户端 IP 数据。如果 Caddy 位于另一个代理后面，则最好与 `servers > trusted_proxies` [全局选项](/docs/caddyfile/options#trusted-proxies) 配对使用。支持 `v1` 和 `v2` 版本。仅当您知道上游服务器能够解析 PROXY 协议时才应使用。默认情况下，此功能被禁用。

- **dial_timeout** <span id="dial_timeout"/> 是连接到上游套接字时等待的最大[持续时间](/docs/conventions#durations)。默认值：`3s`。

- **dial_fallback_delay** <span id="dial_fallback_delay"/> 是在生成 RFC 6555 快速回退连接之前等待的最大[持续时间](/docs/conventions#durations)。负值禁用此功能。默认值：`300ms`。

- **response_header_timeout** <span id="response_header_timeout"/> 是等待从上游读取响应标头的最大[持续时间](/docs/conventions#durations)。默认值：无超时。

- **expect_continue_timeout** <span id="expect_continue_timeout"/> 是如果请求具有 `Expect: 100-continue` 标头，则在完全写入请求标头后等待上游第一个响应标头的最大[持续时间](/docs/conventions#durations)。默认值：无超时。

- **read_timeout** <span id="read_timeout"/> 是等待从后端进行下一次读取的最大[持续时间](/docs/conventions#durations)。默认值：无超时。

- **write_timeout** <span id="write_timeout"/> 是等待向后端进行下一次写入的最大[持续时间](/docs/conventions#durations)。默认值：无超时。

- **resolvers** <span id="resolvers"/> 是用于覆盖系统解析器的 DNS 解析器列表。

- **tls** <span id="tls"/> 使用 HTTPS 与后端通信。如果您使用 `https://` 方案指定后端，或者配置了以下任何 `tls_*` 选项，则会自动启用此功能。

- **tls_client_auth** <span id="tls_client_auth"/> 通过两种方式之一启用 TLS 客户端身份验证：(1) 指定一个域名，Caddy 应为其获取证书并保持更新，或者 (2) 指定一个证书和密钥文件，用于与后端进行 TLS 客户端身份验证。

- **tls_insecure_skip_verify** <span id="tls_insecure_skip_verify"/> 关闭 TLS 握手验证，使连接不安全且容易受到中间人攻击。_请勿在生产中使用。_

- **tls_curves** <span id="tls_curves"/> 是用于上游连接的椭圆曲线列表。Caddy 的默认值是现代且安全的，因此只有在有特定要求时才应配置此选项。

- **tls_timeout** <span id="tls_timeout"/> 是等待 TLS 握手完成的最大[持续时间](/docs/conventions#durations)。默认值：无超时。

- **tls_trust_pool** <span id="tls_trust_pool"/> 配置受信任证书颁发机构的来源，类似于 `tls` 指令文档中描述的 [`trust_pool` 子指令](/docs/caddyfile/directives/tls#trust_pool)。标准 Caddy 安装中可用的信任池来源列表可在[此处](/docs/caddyfile/directives/tls#trust-pool-providers)找到。

- **tls_server_name** <span id="tls_server_name"/> 设置在验证 TLS 握手期间收到的证书时使用的服务器名称。默认情况下，这将使用上游地址的主机部分。

  只有当您的上游地址与上游可能使用的证书不匹配时，才需要覆盖此值。例如，如果上游地址是 IP 地址，则需要将其配置为上游服务器提供的主机名。

  可以使用请求占位符，在这种情况下，每个请求都会使用 HTTP 传输配置的克隆，这可能会带来性能损失。

- **tls_renegotiation** <span id="tls_renegotiation"/> 设置 TLS 重新协商级别。TLS 重新协商是在第一次握手之后执行后续握手的行为。级别可以是以下之一：
  - `never`（默认）禁用重新协商。
  - `once` 允许远程服务器在每个连接上请求一次重新协商。
  - `freely` 允许远程服务器重复请求重新协商。

- **tls_except_ports** <span id="tls_except_ports"/> 当启用 TLS 时，如果上游目标使用给定的端口之一，则这些连接的 TLS 将被禁用。这在配置动态上游时可能很有用，其中一些上游期望 HTTP，而另一些期望 HTTPS 请求。

- **keepalive** <span id="keepalive"/> 可以是 `off` 或一个[持续时间值](/docs/conventions#durations)，指定连接保持打开的时间（超时）。默认值：`2m`。

  ⚠️ 如果 keepalive 持续时间超过上游服务器的 keepalive 超时，对 HTTP/1.1 上游的请求可能会因“对等方重置连接”错误而失败。幂等请求将由 Go 的 HTTP 传输重试，但在其他情况下，Caddy 将响应状态码 502。

- **keepalive_interval** <span id="keepalive_interval"/> 是活跃探测之间的[持续时间](/docs/conventions#durations)。默认值：`30s`。

- **keepalive_idle_conns** <span id="keepalive_idle_conns"/> 定义保持活动状态的最大连接数。默认值：无限制。

- **keepalive_idle_conns_per_host** <span id="keepalive_idle_conns_per_host"/> 如果非零，控制每个主机保留的最大空闲（keep-alive）连接数。默认值：`32`。

- **versions** <span id="versions"/> 允许自定义要支持的 HTTP 版本。
  
  有效选项为：`1.1`、`2`、`h2c`、`3`。

  默认值：`1.1 2`，或者如果[上游的方案](#upstream-addresses)是 `h2c://`，则默认值为 `h2c 2`。

  `h2c` 启用明文 HTTP/2 连接到上游。这是一个非标准功能，不使用 Go 的默认 HTTP 传输，因此与其他功能互斥。

  `3` 启用 HTTP/3 连接到上游。⚠️ 这是一个实验性功能，可能会发生变化。

- **compression** <span id="compression"/> 可以设置为 `off` 以禁用对后端的压缩。

- **max_conns_per_host** <span id="max_conns_per_host"/> 可选地限制每个主机的总连接数，包括拨号中、活动和空闲状态的连接。默认值：无限制。

- **network_proxy** <span id="network_proxy"/> 指定用于向上游服务器发送请求的网络代理模块的名称。如果未显式配置，Caddy 会按照 [Go 标准库](https://pkg.go.dev/golang.org/x/net/http/httpproxy#FromEnvironment) 的方式，通过环境变量（即 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY`）来使用已配置的代理。当为此参数提供值时，请求将通过反向代理按以下顺序流动：客户端（用户）→ `reverse_proxy` → `network_proxy` → 上游。内置模块有：
	- `none`，用于忽略 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY` 的环境设置。
	- `url <url>`，用于指定单个 URL 以覆盖环境配置。

### `fastcgi` 传输

```caddy-d
transport fastcgi {
	root  <path>
	split <at>
	env   <key> <value>
	resolve_root_symlink
	dial_timeout  <duration>
	read_timeout  <duration>
	write_timeout <duration>
	capture_stderr
}
```

- **root** <span id="root"/> 是站点根目录。默认值：`{http.vars.root}` 或当前工作目录。

- **split** <span id="split"/> 是在 URI 末尾分割路径以获取 PATH_INFO 的位置。

- **env** <span id="env"/> 设置一个额外的环境变量为给定值。可以多次指定以设置多个环境变量。

- **resolve_root_symlink** <span id="resolve_root_symlink"/> 通过评估符号链接（如果存在）来启用将 `root` 目录解析为其实际值。

- **dial_timeout** <span id="dial_timeout"/> 是连接到上游套接字时等待的时间。接受[持续时间值](/docs/conventions#durations)。默认值：`3s`。

- **read_timeout** <span id="read_timeout"/> 是从 FastCGI 服务器读取时等待的时间。接受[持续时间值](/docs/conventions#durations)。默认值：无超时。

- **write_timeout** <span id="write_timeout"/> 是向 FastCGI 服务器发送时等待的时间。接受[持续时间值](/docs/conventions#durations)。默认值：无超时。

- **capture_stderr** <span id="capture_stderr"/> 启用捕获和记录上游 FastCGI 服务器在 `stderr` 上发送的任何消息。默认情况下，以 `WARN` 级别记录。如果响应具有 `4xx` 或 `5xx` 状态，则将改用 `ERROR` 级别。默认情况下，`stderr` 被忽略。

<aside class="tip">

如果您尝试提供现代 PHP 应用程序服务，您可能需要的是 [`php_fastcgi` 指令](/docs/caddyfile/directives/php_fastcgi)，它是使用 `fastcgi` 指令进行代理的快捷方式，并包含使用 `index.php` 作为路由入口点所需的重写。

</aside>



## 拦截响应

反向代理可以配置为拦截来自后端的响应。为此，可以定义[响应匹配器](/docs/caddyfile/response-matchers)（类似于请求匹配器的语法），并且将调用第一个匹配的 `handle_response` 路由。

当调用响应处理程序时，来自后端的响应不会写入客户端，而是执行配置的 `handle_response` 路由，由该路由负责写入响应。如果路由**未**写入响应，则请求处理将继续进行[在此 `reverse_proxy` 之后排序](/docs/caddyfile/directives#directive-order)的任何处理程序。

- **@name** 是[响应匹配器](/docs/caddyfile/response-matchers)的名称。只要每个响应匹配器具有唯一名称，就可以定义多个匹配器。可以根据状态码以及响应标头的存在或值来匹配响应。

- **replace_status** <span id="replace_status"/> 在给定匹配器匹配时，简单地更改响应的状态码。

- **handle_response** <span id="handle_response"/> 定义在给定匹配器匹配时（或者，如果省略匹配器，则为所有响应）执行的路由。将应用第一个匹配的块。在 `handle_response` 块内，可以使用任何其他[指令](/docs/caddyfile/directives)。

此外，在 `handle_response` 内部，可以使用两个特殊的处理程序指令：

- **copy_response** <span id="copy_response"/> 将从后端接收的响应正文复制回客户端。可选地允许在这样做时更改响应的状态码。此指令[排在 `respond` 之前](/docs/caddyfile/directives#directive-order)。

- **copy_response_headers** <span id="copy_response_headers"/> 将从后端收到的响应标头复制到客户端，可选地包含 _或_ 排除标头字段列表（不能同时指定 `include` 和 `exclude`）。此指令[排在 `header` 之后](/docs/caddyfile/directives#directive-order)。

在 `handle_response` 路由中，将有三个占位符可用：

- `{rp.status_code}` 后端响应的状态码。

- `{rp.status_text}` 后端响应的状态文本。

- `{rp.header.*}` 后端响应的标头。

虽然反向代理响应处理程序可以将从代理收到的新响应复制回客户端，但它不能将该新响应传递给后续的反向代理。每次使用 `reverse_proxy` 都会从原始请求（或使用不同模块修改后的请求）接收正文。




## 示例

将所有请求反向代理到本地后端：

```caddy
example.com {
	reverse_proxy localhost:9005
}
```


[负载均衡](#load-balancing)所有请求[到 3 个后端](#upstreams)：

```caddy
example.com {
	reverse_proxy node1:80 node2:80 node3:80
}
```


相同，但仅针对 `/api` 内的请求，并通过使用 [`cookie` 策略](#lb_policy)实现粘性：

```caddy
example.com {
	reverse_proxy /api/* node1:80 node2:80 node3:80 {
		lb_policy cookie api_sticky
	}
}
```


使用[主动健康检查](#active-health-checks)确定哪些后端健康，并在失败连接上启用[重试](#lb_try_duration)，保持请求直到找到健康的后端：

```caddy
example.com {
	reverse_proxy node1:80 node2:80 node3:80 {
		health_uri /healthz
		lb_try_duration 5s
	}
}
```


配置一些[传输选项](#transports)：

```caddy
example.com {
	reverse_proxy localhost:8080 {
		transport http {
			dial_timeout 2s
			response_header_timeout 30s
		}
	}
}
```


反向代理到 [HTTPS 上游](#https)（自 v2.11.0 起，Caddy 会自动设置 `Host` 标头以匹配上游的主机，因此不再需要手动执行此操作）：

```caddy
example.com {
	reverse_proxy https://example.com
}
```


反向代理到 HTTPS 上游，但 [⚠️ 禁用 TLS 验证](#tls_insecure_skip_verify)。不推荐这样做，因为它禁用了 HTTPS 提供的所有安全检查；如果可能，首选在专用网络上通过 HTTP 进行代理，因为它可以避免虚假的安全感：

```caddy
example.com {
	reverse_proxy 10.0.0.1:443 {
		transport http {
			tls_insecure_skip_verify
		}
	}
}
```


相反，您可以通过显式[信任上游的证书](#tls_trust_pool)来与上游建立信任，并（可选）设置 TLS-SNI 以匹配上游证书中的主机名：

```caddy
example.com {
	reverse_proxy 10.0.0.1:443 {
		transport http {
			tls_trust_pool file /path/to/cert.pem
			tls_server_name app.example.com
		}
	}
}
```



在代理之前[去除路径前缀](handle_path)；但请注意[子文件夹问题 <img src="/old/resources/images/external-link.svg" class="external-link">](https://caddy.community/t/the-subfolder-problem-or-why-cant-i-reverse-proxy-my-app-into-a-subfolder/8575)：

```caddy
example.com {
	handle_path /prefix/* {
		reverse_proxy localhost:9000
	}
}
```


在代理之前替换路径前缀，使用 [`rewrite`](/docs/caddyfile/directives/rewrite)：

```caddy
example.com {
	handle_path /old-prefix/* {
		rewrite /new-prefix{path}
		reverse_proxy localhost:9000
	}
}
```


`X-Accel-Redirect` 支持，即通过[拦截响应](#intercepting-responses)按需提供静态文件：

```caddy
example.com {
	reverse_proxy localhost:8080 {
		@accel header X-Accel-Redirect *
		handle_response @accel {
			root /path/to/private/files
			rewrite {rp.header.X-Accel-Redirect}
			method GET
			file_server
		}
	}
}
```


根据状态码[拦截错误响应](#intercepting-responses)，为上游错误提供自定义错误页面：

```caddy
example.com {
	reverse_proxy localhost:8080 {
		@error status 500 503
		handle_response @error {
			root /path/to/error/pages
			rewrite /{rp.status_code}.html
			file_server
		}
	}
}
```


从 DNS 查询[动态](#dynamic-upstreams)获取后端[`A`/`AAAA` 记录](#aaaaa)：

```caddy
example.com {
	reverse_proxy {
		dynamic a example.com 9000
	}
}
```


从 DNS 查询[动态](#dynamic-upstreams)获取后端[`SRV` 记录](#srv)：

```caddy
example.com {
	reverse_proxy {
		dynamic srv _api._tcp.example.com
	}
}
```


使用[主动健康检查](#active-health-checks)和 `health_upstream` 在创建中间服务以进行更彻底的检查时可能很有帮助。然后可以使用 `{http.reverse_proxy.active.target_upstream}` 作为标头，将原始上游提供给健康检查服务。

```caddy
example.com {
	reverse_proxy node1:80 node2:80 node3:80 {
		health_uri /health
		health_upstream 127.0.0.1:53336
		health_headers {
			Full-Upstream {http.reverse_proxy.active.target_upstream}
		}
	}
}