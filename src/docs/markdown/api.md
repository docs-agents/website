---
title: "API"
---

# API

Caddy 通过管理端点进行配置，该端点可通过 HTTP 使用 [REST <img src="/old/resources/images/external-link.svg" class="external-link">](https://en.wikipedia.org/wiki/Representational_state_transfer) API 访问。您可以在 Caddy 配置中[配置此端点](/docs/json/admin/)。

**默认地址：`localhost:2019`**

默认地址可以通过设置 `CADDY_ADMIN` 环境变量来更改。某些安装方法可能会将其设置为不同的值。Caddy 配置中的地址始终优先于默认地址。

<aside class="tip">
	如果您在服务器上运行不受信任的代码（哎呀 😬），请确保通过隔离进程、修补易受攻击的程序以及将端点配置为绑定到受限的 Unix socket 来保护您的管理端点。
</aside>

任何更改后，最新配置都会保存到磁盘（除非[已禁用](/docs/json/admin/config/)）。您可以通过 [`caddy run --resume`](/docs/command-line#caddy-run) 恢复上次正常启动的配置，该命令可确保在断电或类似情况下配置持久化。

要开始使用 API，请尝试我们的 [API 教程](/docs/api-tutorial)，或者如果您只有一分钟时间，可以查看我们的 [API 快速入门指南](/docs/quick-starts/api)。

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
  创建新对象；插入数组

- **[PATCH /config/[path]](#patch-configpath)**
  替换现有对象或数组元素

- **[DELETE /config/[path]](#delete-configpath)**
  删除指定路径的值

- **[在 JSON 中使用 `@id`](#using-id-in-json)**
  轻松遍历配置结构

- **[并发配置更改](#concurrent-config-changes)**
  避免对配置进行未同步更改时的冲突

- **[POST /adapt](#post-adapt)**
  将配置适配为 JSON 但不运行它

- **[GET /pki/ca/&lt;id&gt;](#get-pkicaltidgt)**
  返回特定 [PKI 应用](/docs/json/apps/pki/) CA 的信息

- **[GET /pki/ca/&lt;id&gt;/certificates](#get-pkicaltidgtcertificates)**
  返回特定 [PKI 应用](/docs/json/apps/pki/) CA 的证书链

- **[GET /reverse_proxy/upstreams](#get-reverse-proxyupstreams)**
  返回已配置的反向代理上游端的当前状态


## POST /load

设置 Caddy 配置，覆盖之前的任何配置。它会在重新加载完成或失败之前一直阻塞。配置更改非常轻量、高效，且零停机时间。如果新配置由于任何原因失败，旧配置会自动回滚，不会造成停机时间。

此端点支持使用配置适配器使用不同的配置格式。请求的 Content-Type 头指示请求体中使用的配置格式。通常，这应该是 `application/json`，它表示 Caddy 的原生配置格式。对于另一种配置格式，请指定适当的 Content-Type，使得 `/` 之后的值是配置适配器的名称。例如，提交 Caddyfile 时，使用 `text/caddyfile`；对于 JSON 5，使用 `application/json5` 等。

如果新配置与当前配置相同，则不会重新加载。要强制重新加载，请在请求头中设置 `Cache-Control: must-revalidate`。

### 示例

设置新的活动配置：

<pre><code class="cmd bash">curl "http://localhost:2019/load" \
	-H "Content-Type: application/json" \
	-d @caddy.json</code></pre>

注意：curl 的 `-d` 标志会移除换行符，因此如果您的配置格式对换行符敏感（如 Caddyfile），请使用 `--data-binary`：

<pre><code class="cmd bash">curl "http://localhost:2019/load" \
	-H "Content-Type: text/caddyfile" \
	--data-binary @Caddyfile</code></pre>


## POST /stop

优雅地关闭服务器并退出进程。如果只想停止运行配置而不退出进程，请使用 [DELETE /config/](#delete-configpath)。

### 示例

停止进程：

<pre><code class="cmd bash">curl -X POST "http://localhost:2019/stop"</code></pre>


## GET /config/[path]

导出 Caddy 在指定路径的当前配置。返回 JSON 体。

### 示例

导出整个配置并以美观格式打印：

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

仅导出监听器地址：

<pre><code class="cmd"><span class="bash">curl "http://localhost:2019/config/apps/http/servers/myserver/listen"</span>
[":443"]</code></pre>



## POST /config/[path]

将 Caddy 在指定路径的配置更改为请求的 JSON 体。如果目标值是一个数组，POST 会追加；如果是一个对象，它会创建或替换。

作为特殊情况，许多项目可以添加到数组中，如果：

1. 路径以 `/...` 结尾
2. `/...` 之前的路径元素引用一个数组
3. 负载是一个数组

在这种情况下，负载数组中的元素将被扩展，每个元素将被追加到目标数组中。在 Go 术语中，这相当于：

```go
baseSlice = append(baseSlice, newElems...)
```

### 示例

添加监听器地址：

<pre><code class="cmd bash">curl \
	-H "Content-Type: application/json" \
	-d '":8080"' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen"</code></pre>

添加多个监听器地址：

<pre><code class="cmd bash">curl \
	-H "Content-Type: application/json" \
	-d '[":8080", ":5133"]' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen/..."</code></pre>

## PUT /config/[path]

将 Caddy 在指定路径的配置更改为请求的 JSON 体。如果目标值是数组中的位置（索引），PUT 会插入；如果是一个对象，它会严格创建新值。

### 示例

在第一个位置添加监听器地址：

<pre><code class="cmd bash">curl -X PUT \
	-H "Content-Type: application/json" \
	-d '":8080"' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen/0"</code></pre>


## PATCH /config/[path]

将 Caddy 在指定路径的配置更改为请求的 JSON 体。PATCH 严格替换现有值或数组元素。

### 示例

替换监听器地址：

<pre><code class="cmd bash">curl -X PATCH \
	-H "Content-Type: application/json" \
	-d '[":8081", ":8082"]' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen"</code></pre>



## DELETE /config/[path]

删除 Caddy 在指定路径的配置。DELETE 删除目标值。

### 示例

卸载当前整个配置但保持进程运行：

<pre><code class="cmd bash">curl -X DELETE "http://localhost:2019/config/"</code></pre>

仅停止您的一个 HTTP 服务器：

<pre><code class="cmd bash">curl -X DELETE "http://localhost:2019/config/apps/http/servers/myserver"</code></pre>


## 在 JSON 中使用 `@id`

您可以在 JSON 文档中嵌入 ID，以便更直接地访问这些部分。

只需将名为 `"@id"` 的字段添加到对象中，并给它一个唯一名称。例如，如果您有一个反向代理处理器，您想频繁访问它：

```json
{
	"@id": "my_proxy",
	"handler": "reverse_proxy"
}
```

要使用它，只需以与相应 `/config/` 端点相同的方式向 `/id/` API 端点发出请求，只是不需要完整路径。ID 会将请求直接带入该配置范围。

例如，不使用 ID 访问反向代理的上游端，路径可能是：

```
/config/apps/http/servers/myserver/routes/1/handle/0/upstreams
```

但使用 ID 后，路径变为：

```
/id/my_proxy/upstreams
```

这更容易记忆和手动编写。

## 并发配置更改

<aside class="tip">

本节适用于所有 `/config/` 端点。这是实验性的，可能会更改。

</aside>


Caddy 的配置 API 为单个请求提供 [ACID 保证 <img src="/old/resources/images/external-link.svg" class="external-link">](https://en.wikipedia.org/wiki/ACID)，但涉及多个请求的更改，如果没有适当同步，可能会发生冲突或数据丢失。

例如，两个客户端可能同时 `GET /config/foo`，在该范围内进行编辑（配置路径），然后同时调用 `POST|PUT|PATCH|DELETE /config/foo/...` 应用更改，导致冲突：要么一个会覆盖另一个，要么第二个可能将配置置于意外状态，因为它应用于与其准备不同的配置版本。这是因为更改彼此不感知。

Caddy 的 API 不支持跨多个请求的事务，HTTP 是无状态协议。但是，您可以使用 `Etag` 和 `If-Match` 头来检测和防止任何和所有更改的冲突，作为一种乐观并发控制。如果可能并未经过同步就并发使用 Caddy 的 `/config/...` 端点，这将很有用。所有 `GET /config/...` 请求的响应都有一个名为 `Etag` 的 HTTP 头，其中包含路径和该范围的哈希值（例如 `Etag: "/config/apps/http/servers 65760b8e"`）。只需在变更请求中将 `If-Match` 头设置为先前 `GET` 请求的 Etag 头的值。

基本算法如下：

1. 对配置内的任何范围 `S` 执行 `GET` 请求。保留响应的 `Etag` 头。
2. 对返回的配置进行所需更改。
3. 在范围 `S` 内执行 `POST|PUT|PATCH|DELETE` 请求，将 `If-Match` 请求头设置为存储的 `Etag` 值。
4. 如果响应是 HTTP 412（前置条件失败），从步骤 1 重复，或在尝试过多后放弃。

此算法允许对 Caddy 配置进行多个重叠更改，而无需显式同步。它的设计使得对配置不同部分的并发更改不需要重试：只有重叠配置同一范围的更改才可能引起冲突，因此需要重试。


## POST /adapt

将配置适配为 Caddy JSON，但不加载或运行它。如果成功，结果 JSON 文档将在响应体中返回。

Content-Type 头用于指定配置格式，方式与 [/load](#post-load) 相同。例如，要适配 Caddyfile，设置 `Content-Type: text/caddyfile`。

此端点将适配任何配置格式，只要相关的 [配置适配器](/docs/config-adapters) 已插入您的 Caddy 构建。

### 示例

将 Caddyfile 适配为 JSON：

<pre><code class="cmd bash">curl "http://localhost:2019/adapt" \
	-H "Content-Type: text/caddyfile" \
	--data-binary @Caddyfile</code></pre>


## GET /pki/ca/&lt;id&gt;

通过 ID 返回特定 [PKI 应用](/docs/json/apps/pki/) CA 的信息。如果请求的 CA ID 是默认的 (`local`)，则如果尚未配置，将配置该 CA。其他 CA ID 如果尚未预先配置将返回错误。

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

通过 ID 返回特定 [PKI 应用](/docs/json/apps/pki/) CA 的证书链。如果请求的 CA ID 是默认的 (`local`)，则如果尚未配置，将配置该 CA。其他 CA ID 如果尚未预先配置将返回错误。

此端点被 [`caddy trust`](/docs/command-line#caddy-trust) 命令内部使用，以允许将 CA 的根证书安装到系统信任存储区。

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

以 JSON 文档形式返回已配置的反向代理上游端（后端）的当前状态。

<pre><code class="cmd"><span class="bash">curl "http://localhost:2019/reverse_proxy/upstreams" | jq</span>
[
	{"address": "10.0.1.1:80", "num_requests": 4, "fails": 2},
	{"address": "10.0.1.2:80", "num_requests": 5, "fails": 4},
	{"address": "10.0.1.3:80", "num_requests": 3, "fails": 3}
]</code></pre>

JSON 数组中的每个条目是全局上游池中配置的 [上游端](/docs/json/apps/http/servers/routes/handle/reverse_proxy/upstreams/)。

- **address** 是上游端的拨号地址。
- **num_requests** 是当前由上游端处理的活跃请求数量。
- **fails** 是当前记住的失败请求数量，如通过被动健康检查配置的。

如果您的目标是确定后端的可用性，您需要将上游的相关属性与您使用的处理器配置进行交叉检查。例如，如果您已为代理启用 [被动健康检查](/docs/json/apps/http/servers/routes/handle/reverse_proxy/health_checks/passive/)，那么您还需要考虑 `fails` 和 `num_requests` 值来确定上游是否被视为可用：检查 `fails` 数量是否小于代理配置的失败最大数量（即 [`max_fails`](/docs/json/apps/http/servers/routes/handle/reverse_proxy/health_checks/passive/max_fails/)），并且 `num_requests` 小于或等于您配置的每个上游的最大请求数量（即整个代理的 [`unhealthy_request_count`](/docs/json/apps/http/servers/routes/handle/reverse_proxy/health_checks/passive/unhealthy_request_count/)，或单个上游端的 [`max_requests`](/docs/json/apps/http/servers/routes/handle/reverse_proxy/upstreams/max_requests/)）。
