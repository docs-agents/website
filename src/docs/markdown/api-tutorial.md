---
title: "API 教程"
---

# API 教程

本教程将演示如何使用 Caddy 的[管理 API](/docs/api)，从而以可编程的方式实现自动化。

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
- `caddy` 和 `curl` 已加入 PATH 环境变量

---

要启动 Caddy 守护进程，请使用 `run` 子命令：

<pre><code class="cmd bash">caddy run</code></pre>

<aside class="complete">运行守护进程</aside>

此命令会一直阻塞，但它在做什么呢？目前……什么也没做。默认情况下，Caddy 的配置（"config"）是空的。我们可以通过另一个终端使用[管理 API](/docs/api) 来验证这一点：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

我们可以通过为 Caddy 提供配置来让它发挥作用。一种方法是向 [/load](/docs/api#post-load) 端点发送 POST 请求。与任何 HTTP 请求一样，有很多方法可以做到这一点，但在本教程中我们使用 `curl`。

## 你的第一个配置

为了准备请求，我们需要创建一个配置。Caddy 的配置只是一个 [JSON 文档](/docs/json/)（或任何[可转换为 JSON 的内容](/docs/config-adapters)）。

<aside class="tip">
	配置文件不是必需的。配置 API 始终可以不借助文件使用，这在自动化场景中非常方便。本教程使用文件是因为手动编辑更便捷。
</aside>

将以下内容保存到一个 JSON 文件中：

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

然后上传该文件：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: application/json" \
	-d @caddy.json
</code></pre>

<aside class="tip">
	请确保不要忘记文件名前的 @ 符号；这告诉 curl 你正在发送一个文件。
</aside>

<aside class="complete">为 Caddy 提供配置</aside>

我们可以通过另一个 GET 请求来验证 Caddy 是否应用了新配置：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

通过浏览器访问 [localhost:2015](http://localhost:2015) 或使用 `curl` 来测试是否正常工作：

<pre><code class="cmd"><span class="bash">curl localhost:2015</span>
Hello, world!</code></pre>

<aside class="complete">测试配置</aside>

如果你看到 _Hello, world!_，那么恭喜——它工作正常！在将配置部署到生产环境之前，确保其按预期工作总是一个好主意。

现在让我们将欢迎信息从 "Hello world!" 改为更鼓舞人心的内容："I can do hard things." 在配置文件中进行此修改，使 handler 对象现在看起来像这样：

```json
{
	"handler": "static_response",
	"body": "I can do hard things."
}
```

保存配置文件，然后再次运行相同的 POST 请求来更新 Caddy 的活动配置：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: application/json" \
	-d @caddy.json
</code></pre>

<aside class="complete">替换活动配置</aside>

为了确认，验证配置已更新：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

通过浏览器刷新页面（或再次运行 `curl`）来测试，你会看到一句鼓舞人心的话！


## 配置遍历

对于一个小改动，无需上传整个配置文件，我们可以使用 Caddy API 的一个强大功能，在不涉及配置文件的情况下直接修改。

<aside class="tip">
	像我们上面那样用整个配置替换来对生产服务器进行小修改可能很危险；这就像对文件系统拥有 root 权限一样。Caddy 的 API 允许你限制修改范围，从而保证配置的其他部分不会被意外更改。
</aside>

利用请求 URI 的路径，我们可以遍历配置结构并仅更新消息字符串（如果被截断请向右滚动）：

<pre><code class="cmd bash">curl \
	localhost:2019/config/apps/http/servers/example/routes/0/handle/0/body \
	-H "Content-Type: application/json" \
	-d '"Work smarter, not harder."'
</code></pre>


<aside class="tip">

每次使用 API 修改配置时，Caddy 都会持久化一份新配置的副本，以便后续可以使用 [**--resume** 恢复](/docs/command-line#caddy-run)！

</aside>


你可以通过类似的 GET 请求来验证效果，例如：

<pre><code class="cmd bash">curl localhost:2019/config/apps/http/servers/example/routes</code></pre>

你应该看到：

```json
[{"handle":[{"body":"Work smarter, not harder.","handler":"static_response"}]}]
```


<aside class="tip">

你可以使用 [`jq` 命令 <img src="/old/resources/images/external-link.svg" class="external-link">](https://stedolan.github.io/jq/) 来美化 JSON 输出：**`curl ... | jq`**

</aside>


<aside class="complete">遍历配置</aside>

**重要提示：** 这显而易见，但一旦你通过 API 进行了原始配置文件中不存在的修改，你的配置文件就会失效。有几种方法可以处理：

- 使用 [caddy run](/docs/command-line#caddy-run) 命令的 `--resume` 选项来使用上次的活动配置。
- 不要混合使用配置文件与通过 API 进行的修改；应只保留单一的真实来源。
- 通过后续的 GET 请求[导出 Caddy 的新配置](/docs/api#get-configpath)（不推荐，前两个选项更优）。



## 在 JSON 中使用 `@id`

配置遍历固然有用，但路径有点长，你不觉得吗？

我们可以为 handler 对象添加一个 [`@id` 标签](/docs/api#using-id-in-json) 来简化访问：

<pre><code class="cmd bash">curl \
	localhost:2019/config/apps/http/servers/example/routes/0/handle/0/@id \
	-H "Content-Type: application/json" \
	-d '"msg"'
</code></pre>

这会在 handler 对象中添加一个属性：`"@id": "msg"`，因此它现在看起来像这样：

```json
{
	"@id": "msg",
	"body": "Work smarter, not harder.",
	"handler": "static_response"
}
```


<aside class="tip">

**@id** 标签可以放在任何对象中，并可以具有任何原始值（通常是一个字符串）。[了解更多](/docs/api#using-id-in-json)

</aside>


然后我们可以直接访问它：

<pre><code class="cmd bash">curl localhost:2019/id/msg</code></pre>

现在，我们可以用更短的路径来修改消息：

<pre><code class="cmd bash">curl \
	localhost:2019/id/msg/body \
	-H "Content-Type: application/json" \
	-d '"Some shortcuts are good."'
</code></pre>

并再次检查：

<pre><code class="cmd bash">curl localhost:2019/id/msg/body</code></pre>

<aside class="complete">使用 <code>@id</code> 标签</aside>