---
title: "API"
---

# API

Caddy 通过一个管理端点进行配置，该端点可通过 HTTP 使用 [REST <img src="/old/resources/images/external-link.svg" class="external-link">](https://en.wikipedia.org/wiki/Representational_state_transfer) API 访问。你可以在 Caddy 配置中[配置该端点](/docs/json/admin/)。

**默认地址：`localhost:2019`**

可以通过设置 `CADDY_ADMIN` 环境变量来更改默认地址。某些安装方法可能会将其设为不同的值。Caddy 配置中的地址始终优先于默认地址。

<aside class="tip">
	如果你在服务器上运行不可信代码（天哪 😬），请务必通过隔离进程、修补有漏洞的程序以及将端点配置为绑定到有权限的 Unix 套接字来保护你的管理端点。
</aside>

最新配置会在任何更改后保存到磁盘（除非[禁用](/docs/json/admin/config/)）。重启后，你可以使用 [`caddy run --resume`](/docs/command-line#caddy-run) 恢复上次正常工作的配置，这可在断电或类似情况下保证配置的持久性。

若要开始使用 API，请尝试我们的 [API 教程](/docs/api-tutorial)；如果你时间有限，可以阅读我们的 [API 快速入门指南](/docs/quick-starts/api)。

---

- **[POST /load](#post-load)**
  设置或替换活动配置

- **[POST /stop](#post-stop)**
  停止活动配置并退出进程

- **[GET /config/[path]](#get-configpath)**
  导出指定路径的配置

- **[POST /config/[path]](#post-configpath)**
  设置或替换对象；追加到数组

- **[PUT /config/[path]](#put-configpath)**
  创建新对象；插入到数组

- **[PATCH /config/[path]](#patch-configpath)**
  替换现有对象或数组元素

- **[DELETE /config/[path]](#delete-configpath)**
  删除指定路径的值

- **[在 JSON 中使用 `@id`](#using-id-in-json)**
  轻松遍历配置结构

- **[并发配置更改](#concurrent-config-changes)**
  避免在非同步更改配置时发生冲突

- **[POST /adapt](#post-adapt)**
  将配置适配为 JSON 而不运行它

- **[GET /pki/ca/&lt;id&gt;](#get-pkicaltidgt)**
  返回特定 [PKI 应用](/docs/json/apps/pki/) CA 的信息

- **[GET /pki/ca/&lt;id&gt;/certificates](#get-pkicaltidgtcertificates)**
  返回特定 [PKI 应用](/docs/json/apps/pki/) CA 的证书链

- **[GET /reverse_proxy/upstreams](#get-reverse-proxyupstreams)**
  返回已配置代理上游的当前状态


## POST /load

设置 Caddy 的配置，覆盖任何之前的配置。它会阻塞直到重载完成或失败。配置更改轻量、高效且零停机。如果新配置因任何原因失败，旧配置会无停机地回滚。

此端点支持使用配置适配器的不同配置格式。请求的 Content-Type 头部指示请求体中使用的配置格式。通常，这应为 `application/json`，表示 Caddy 的原生配置格式。对于其他配置格式，请指定适当的 Content-Type，使得斜杠 `/` 后的值为要使用的配置适配器名称。例如，提交 Caddyfile 时，使用 `text/caddyfile`；对于 JSON 5，使用 `application/json5` 等。

如果新配置与当前配置相同，则不会进行重载。要强制重载，请在请求头中设置 `Cache-Control: must-revalidate`。

### 示例

设置新的活动配置：

<pre><code class="cmd bash">curl "http://localhost:2019/load" \
	-H "Content-Type: application/json" \
	-d @caddy.json</code></pre>

注意：curl 的 `-d` 标志会删除换行符，因此如果你的配置格式对换行敏感（例如 Caddyfile），请改用 `--data-binary`：

<pre><code class="cmd bash">curl "http://localhost:2019/load" \
	-H "Content-Type: text/caddyfile" \
	--data-binary @Caddyfile</code></pre>


## POST /stop

正常关闭服务器并退出进程。若只停止运行中的配置而不退出进程，请使用 [DELETE /config/](#delete-configpath)。

### 示例

停止进程：

<pre><code class="cmd bash">curl -X POST "http://localhost:2019/stop"</code></pre>


## GET /config/[path]

导出 Caddy 当前配置在指定路径下的内容。返回 JSON 格式的响应体。

### 示例

导出整个配置并美化输出：

<pre><code class="cmd"><span class="bash">curl "http://localhost:2019/config/" | jq</span>
{
	"apps": {
		"http": {
			"servers": {
				"myserver": {
					"listen": [
						":443"
					],
					"routes": [
						{
							"match": [
								{
									"host": [
										"example.com"
									]
								}
							],
							"handle": [
								{
									"handler": "file_server"
								}
							]
						}
					]
				}
			}
		}
	}
}</code></pre>

仅导出监听地址：

<pre><code class="cmd"><span class="bash">curl "http://localhost:2019/config/apps/http/servers/myserver/listen"</span>
[":443"]</code></pre>



## POST /config/[path]

将请求的 JSON 体修改 Caddy 配置中指定路径的内容。如果目标值是数组，POST 会追加；如果是对象，则创建或替换。

作为特殊情况，可以向数组添加多个条目，前提是：

1. 路径以 `/...` 结尾
2. `/...` 之前的路径元素指向一个数组
3. 载荷是一个数组

在这种情况下，载荷数组中的元素会被展开，每个元素都被追加到目标数组中。以 Go 术语来说，效果等同于：

```go
baseSlice = append(baseSlice, newElems...)
```

### 示例

添加一个监听地址：

<pre><code class="cmd bash">curl \
	-H "Content-Type: application/json" \
	-d '":8080"' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen"</code></pre>

添加多个监听地址：

<pre><code class="cmd bash">curl \
	-H "Content-Type: application/json" \
	-d '[":8080", ":5133"]' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen/..."</code></pre>

## PUT /config/[path]

将请求的 JSON 体修改 Caddy 配置中指定路径的内容。如果目标值是数组中的位置（索引），PUT 会插入；如果是对象，则严格创建新值。

### 示例

在第一个位置添加一个监听地址：

<pre><code class="cmd bash">curl -X PUT \
	-H "Content-Type: application/json" \
	-d '":8080"' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen/0"</code></pre>


## PATCH /config/[path]

将请求的 JSON 体修改 Caddy 配置中指定路径的内容。PATCH 严格替换现有值或数组元素。

### 示例

替换监听地址：

<pre><code class="cmd bash">curl -X PATCH \
	-H "Content-Type: application/json" \
	-d '[":8081", ":8082"]' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen"</code></pre>



## DELETE /config/[path]

删除 Caddy 配置中指定路径的内容。DELETE 用于移除目标值。

### 示例

卸载整个当前配置但保持进程运行：

<pre><code class="cmd bash">curl -X DELETE "http://localhost:2019/config/"</code></pre>

仅停止一个 HTTP 服务器：

<pre><code class="cmd bash">curl -X DELETE "http://localhost:2019/config/apps/http/servers/myserver"</code></pre>


## 在 JSON 中使用 `@id`

你可以在 JSON 文档中嵌入 ID，以便更轻松地直接访问 JSON 的相应部分。

只需在对象中添加一个名为 `"@id"` 的字段，并赋予其唯一名称。例如，如果你有一个需要频繁访问的反向代理处理器：

```json
{
	"@id": "my_proxy",
	"handler": "reverse_proxy"
}
```

要使用它，只需像使用对应的 `/config/` 端点那样向 `/id/` API 端点发出请求，但无需完整路径。ID 会将请求直接带入配置的那个作用域。

例如，要访问反向代理的上游服务器，如果没有 ID，路径可能是：

```
/config/apps/http/servers/myserver/routes/1/handle/0/upstreams
```

但使用 ID 后，路径变为：

```
/id/my_proxy/upstreams
```

这样更容易记忆和手动输入。

## 并发配置更改

<aside class="tip">

本节适用于所有 `/config/` 端点。该功能为实验性，可能发生变更。

</aside>


Caddy 的配置 API 为单个请求提供了 [ACID 保证 <img src="/old/resources/images/external-link.svg" class="external-link">](https://en.wikipedia.org/wiki/ACID)，但涉及多个请求的更改若未正确同步，则可能发生冲突或数据丢失。

例如，两个客户端可能同时 `GET /config/foo`，在该作用域内进行编辑，然后同时调用 `POST|PUT|PATCH|DELETE /config/foo/...` 以应用其更改，从而导致冲突：其中一个会覆盖另一个，或者第二个可能会使配置处于非预期状态，因为它应用更改时基于的配置版本与自身准备时不同。这是因为更改之间互不知晓。

Caddy 的 API 不支持跨多个请求的事务，且 HTTP 是无状态协议。但是，你可以使用 `Etag` 和 `If-Match` 头部来检测和阻止任何更改的冲突，作为一种乐观并发控制。如果你有可能在未同步的情况下并发使用 Caddy 的 `/config/...` 端点，这将非常有用。所有对 `GET /config/...` 请求的响应都包含一个名为 `Etag` 的 HTTP 头部，其中包含路径及该作用域内容的哈希值（例如 `Etag: "/config/apps/http/servers 65760b8e"`）。只需在变异请求中设置 `If-Match` 头部为之前 `GET` 请求返回的 `Etag` 头部值即可。

基本算法如下：

1. 对配置中任意作用域 `S` 执行 `GET` 请求。记录响应中的 `Etag` 头部。
2. 对返回的配置进行所需更改。
3. 在作用域 `S` 内执行 `POST|PUT|PATCH|DELETE` 请求，并将 `If-Match` 请求头部设置为存储的 `Etag` 值。
4. 如果响应为 HTTP 412（Precondition Failed），则从步骤 1 开始重试，若尝试次数过多则放弃。

该算法安全地允许对 Caddy 配置进行多个重叠的更改，而无需显式同步。它被设计为，对配置不同部分的同时更改不需要重试：只有重叠同一作用域的更改才可能引发冲突，从而需要重试。


## POST /adapt

将配置适配为 Caddy JSON，而不加载或运行它。如果成功，响应体将返回生成的 JSON 文档。

Content-Type 头部的使用方式与 [/load](#post-load) 相同，用于指定配置格式。例如，要适配 Caddyfile，设置 `Content-Type: text/caddyfile`。

只要相应的[配置适配器](/docs/config-adapters)已嵌入你的 Caddy 构建中，该端点便可适配任何配置格式。

### 示例

将 Caddyfile 适配为 JSON：

<pre><code class="cmd bash">curl "http://localhost:2019/adapt" \
	-H "Content-Type: text/caddyfile" \
	--data-binary @Caddyfile</code></pre>


## GET /pki/ca/&lt;id&gt;

返回特定 [PKI 应用](/docs/json/apps/pki/) CA（按其 ID）的信息。如果请求的 CA ID 是默认的（`local`），则 CA 会被自动预配（如果尚未预配）。其他 CA ID 若之前未预配，则会返回错误。

<pre><code class="cmd"><span class="bash">curl "http://localhost:2019/pki/ca/local" | jq</span>
{
	"id": "local",
	"name": "Caddy Local Authority",
	"root_common_name": "Caddy Local Authority - 2022 ECC Root",
	"intermediate_common_name": "Caddy Local Authority - ECC Intermediate",
	"root_certificate": "-----BEGIN CERTIFICATE-----\nMIIB ... gRw==\n-----END CERTIFICATE-----\n",
	"intermediate_certificate": "-----BEGIN CERTIFICATE-----\nMIIB ... FzQ==\n-----END CERTIFICATE-----\n"
}</code></pre>


## GET /pki/ca/&lt;id&gt;/certificates

返回特定 [PKI 应用](/docs/json/apps/pki/) CA（按其 ID）的证书链。如果请求的 CA ID 是默认的（`local`），则 CA 会被自动预配（如果尚未预配）。其他 CA ID 若之前未预配，则会返回错误。

该端点由 [`caddy trust`](/docs/command-line#caddy-trust) 命令内部使用，用于将 CA 的根证书安装到系统的信任存储中。

<pre><code class="cmd"><span class="bash">curl "http://localhost:2019/pki/ca/local/certificates"</span>
-----BEGIN CERTIFICATE-----
MIIByDCCAW2gAwIBAgIQViS12trTXBS/nyxy7Zg9JDAKBggqhkjOPQQDAjAwMS4w
...
By75JkP6C14OfU733oElfDUMa5ctbMY53rWFzQ==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIBpDCCAUmgAwIBAgIQTS5a+3LUKNxC6qN3ZDR8bDAKBggqhkjOPQQDAjAwMS4w
...
9M9t0FwCIQCAlUr4ZlFzHE/3K6dARYKusR1ck4A3MtucSSyar6lgRw==
-----END CERTIFICATE-----</code></pre>


## GET /reverse_proxy/upstreams

返回已配置的反向代理上游（后端）的当前状态，以 JSON 文档形式呈现。

<pre><code class="cmd"><span class="bash">curl "http://localhost:2019/reverse_proxy/upstreams" | jq</span>
[
	{"address": "10.0.1.1:80", "num_requests": 4, "fails": 2},
	{"address": "10.0.1.2:80", "num_requests": 5, "fails": 4},
	{"address": "10.0.1.3:80", "num_requests": 3, "fails": 3}
]</code></pre>

JSON 数组中的每个条目都是存储在全局上游池中的一个已配置的[上游](/docs/json/apps/http/servers/routes/handle/reverse_proxy/upstreams/)。

- **address** 是上游的拨号地址。
- **num_requests** 是当前正在由该上游处理的活动请求数。
- **fails** 是当前记住的失败请求数，由被动健康检查配置决定。

如果你的目标是确定后端的可用性，则需要根据所使用的处理器配置，交叉检查上游的相关属性。例如，如果你已为代理启用了[被动健康检查](/docs/json/apps/http/servers/routes/handle/reverse_proxy/health_checks/passive/)，则需要同时考虑 `fails` 和 `num_requests` 的值，以确定上游是否可用：检查 `fails` 数量是否小于你为代理配置的最大失败次数（即 [`max_fails`](/docs/json/apps/http/servers/routes/handle/reverse_proxy/health_checks/passive/max_fails/)），并且 `num_requests` 是否小于等于你为每个上游配置的最大请求数（即全局代理的 [`unhealthy_request_count`](/docs/json/apps/http/servers/routes/handle/reverse_proxy/health_checks/passive/unhealthy_request_count/) 或单个上游的 [`max_requests`](/docs/json/apps/http/servers/routes/handle/reverse_proxy/upstreams/max_requests/)）。