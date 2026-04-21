---
title: "API Tutorial"
---

# API 教程

本教程将向您展示如何使用 Caddy 的 [管理 API](/docs/api)，使其能够以可编程的方式自动化操作。

**目标：**
- 🔲 运行守护进程
- 🔲 为 Caddy 提供配置
- 🔲 测试配置
- 🔲 替换活动配置
- 🔲 遍历配置
- 🔲 使用 `@id` 标签

**前提条件：**
- 基本的终端/命令行技能
- 基本的 JSON 经验
- PATH 中包含 `caddy` 和 `curl`

---

要启动 Caddy 守护进程，请使用 `run` 子命令：

<pre><code class="cmd bash">caddy run</code></pre>

<aside class="complete">运行守护进程</aside>

这会无限期阻塞，但它正在做什么？此刻……什么都没做。默认情况下，Caddy 的配置（"配置"）是空白的。我们可以在另一个终端中使用 [管理 API](/docs/api) 验证这一点：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

我们可以通过向 [/load](/docs/api#post-load) 端点发出 POST 请求来使 Caddy 发挥作用。就像任何 HTTP 请求一样，有许多方法可以做到这一点，但在本教程中，我们将使用 `curl`。

## 您的第一个配置

为了准备我们的请求，我们需要制作一个配置。Caddy 的配置只是一个 [JSON 文档](/docs/json/)（或 [任何可以转换为 JSON 的内容](/docs/config-adapters)）。

<aside class="tip">
	配置文件不是必需的。配置 API 始终可以无需文件使用，这在自动化时很方便。本教程使用文件是因为它更便于手动编辑。
</aside>

将其保存到 JSON 文件：

```json
{
	"apps": {
		"http": {
			"servers": {
				"example": {
					"listen": [":2015"],
					"routes": [
						{
							"handle": [{
								"handler": "static_response",
								"body": "Hello, world!"
							}]
						}
					]
				}
			}
		}
	}
}
```

然后上传它：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: application/json" \
	-d @caddy.json
</code></pre>

<aside class="tip">
	确保不要忘记文件名前的 @；这告诉 curl 您正在发送一个文件。
</aside>

<aside class="complete">为 Caddy 提供配置</aside>

我们可以通过另一个 GET 请求验证 Caddy 已应用我们的新配置：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

通过在浏览器中访问 [localhost:2015](http://localhost:2015) 或使用 `curl` 测试它是否有效：

<pre><code class="cmd"><span class="bash">curl localhost:2015</span>
Hello, world!</code></pre>

<aside class="complete">测试配置</aside>

如果您看到 _Hello, world!_，那么恭喜——它正在工作！始终确保您的配置按预期工作，特别是在部署到生产环境之前，这是一件好事。

让我们将欢迎消息从 "Hello world!" 改为稍微更有激励性的一些话："I can do hard things."（我可以做艰难的事情）在我们的配置文件中做此更改，使处理器对象现在看起来像这样：

```json
{
	"handler": "static_response",
	"body": "I can do hard things."
}
```

保存配置文件，然后通过再次运行相同的 POST 请求来更新 Caddy 的活动配置：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: application/json" \
	-d @caddy.json
</code></pre>

<aside class="complete">替换活动配置</aside>

为了更好地衡量，验证配置是否已更新：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

通过在浏览器中刷新页面（或再次运行 `curl`）来测试它，您将看到一个鼓舞人心的消息！


## 配置遍历

对于小更改，不是上传整个配置文件，让我们使用 Caddy API 的强大功能进行更改，而无需触碰我们的配置文件。

<aside class="tip">
	通过用整个配置替换来对生产服务器进行小更改可能很危险；这就像拥有文件系统的 root 访问权限。Caddy 的 API 允许您将更改范围限制在可以保证配置其他部分不会被意外更改。
</aside>

使用请求 URI 的路径，我们可以遍历配置结构并仅更新消息字符串（如果截断，请向右滚动）：

<pre><code class="cmd bash">curl \
	localhost:2019/config/apps/http/servers/example/routes/0/handle/0/body \
	-H "Content-Type: application/json" \
	-d '"Work smarter, not harder."'
</code></pre>


<aside class="tip">

每次使用 API 更改配置时，Caddy 都会将新配置的副本持久化保存，以便您以后可以 [**--resume** 它](/docs/command-line#caddy-run)!

</aside>


您可以使用类似的 GET 请求验证它是否有效，例如：

<pre><code class="cmd bash">curl localhost:2019/config/apps/http/servers/example/routes</code></pre>

您应该看到：

```json
[{"handle":[{"body":"Work smarter, not harder.","handler":"static_response"}]}]
```


<aside class="tip">

您可以使用 [`jq` 命令 <img src="/old/resources/images/external-link.svg" class="external-link">](https://stedolan.github.io/jq/) 美化 JSON 输出：**`curl ... | jq`**

</aside>


<aside class="complete">遍历配置</aside>

**重要提示：**这应该很明显，但一旦您使用 API 进行配置文件中不存在的更改，您的配置文件就失效了。有几种方法来处理这个问题：

- 使用 [caddy run](/docs/command-line#caddy-run) 命令的 `--resume` 选项来使用上次活动的配置。
- 不要混用配置文件和通过 API 的更改；只有一个事实来源。
- [导出 Caddy 的新配置](/docs/api#get-configpath) 通过随后的 GET 请求（与前面两个选项相比不太推荐）。



## 在 JSON 中使用 `@id`

配置遍历确实有用，但路径有点长，不是吗？

我们可以为我们的处理器对象添加一个 [`@id` 标签](/docs/api#using-id-in-json) 以便更容易访问：

<pre><code class="cmd bash">curl \
	localhost:2019/config/apps/http/servers/example/routes/0/handle/0/@id \
	-H "Content-Type: application/json" \
	-d '"msg"'
</code></pre>

这为我们的处理器对象添加了一个属性：`"@id": "msg"`，所以它现在看起来像这样：

```json
{
	"@id": "msg",
	"body": "Work smarter, not harder.",
	"handler": "static_response"
}
```


<aside class="tip">

**@id** 标签可以放在任何对象中，可以有任何基本值（通常是字符串）。[了解更多](/docs/api#using-id-in-json)

</aside>


然后我们可以直接访问它：

<pre><code class="cmd bash">curl localhost:2019/id/msg</code></pre>

现在我们可以用更短的路径更改消息：

<pre><code class="cmd bash">curl \
	localhost:2019/id/msg/body \
	-H "Content-Type: application/json" \
	-d '"Some shortcuts are good."'
</code></pre>

再次检查它：

<pre><code class="cmd bash">curl localhost:2019/id/msg/body</code></pre>

<aside class="complete">使用 <code>@id</code> 标签</aside>
