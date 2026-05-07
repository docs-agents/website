---
title: "入门指南"
---

# 入门指南

欢迎使用 Caddy！本教程将带您了解 Caddy 的基础用法，并从宏观层面帮助您熟悉它。

**学习目标：**
- 🔲 运行守护进程
- 🔲 尝试使用 API
- 🔲 为 Caddy 提供配置
- 🔲 测试配置
- 🔲 编写 Caddyfile
- 🔲 使用配置适配器
- 🔲 使用初始配置启动
- 🔲 比较 JSON 和 Caddyfile
- 🔲 比较 API 和配置文件
- 🔲 后台运行
- 🔲 零停机配置重载

**前置要求：**
- 基本的终端/命令行操作能力
- 基本的文本编辑器操作能力
- 系统中已安装 `caddy` 和 `curl` 命令（需在 PATH 中）

---

**如果您已通过包管理器[安装 Caddy](/docs/install)，Caddy 可能已作为服务运行。如果是这样，请先停止该服务再开始本教程。**

让我们从运行 Caddy 开始：

<pre><code class="cmd bash">caddy</code></pre>

哎呀；如果没有子命令，`caddy` 命令只会显示帮助文本。无论何时忘记该做什么，都可以使用这个命令。

要将 Caddy 作为守护进程启动，请使用 `run` 子命令：

<pre><code class="cmd bash">caddy run</code></pre>

<aside class="complete">运行守护进程</aside>

这个命令会一直阻塞，但它在做什么呢？目前……什么都没做。默认情况下，Caddy 的配置（"config"）是空的。我们可以通过另一个终端使用[管理 API](/docs/api) 来验证：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

<aside class="tip">

请注意，这**不是您的网站**：localhost:2019 的管理端点用于控制 Caddy，默认只允许本地访问。

</aside>

<aside class="complete">尝试使用 API</aside>

我们可以通过给 Caddy 提供一个配置来让它变得有用。这可以通过多种方式实现，但我们将从下一节中使用 `curl` 向 [/load](/docs/api#post-load) 端点发送 POST 请求开始。

## 您的第一个配置

为了准备我们的请求，我们需要创建一个配置。核心上，Caddy 的配置只是一个 [JSON 文档](/docs/json/)。

将其保存为一个 JSON 文件（例如 `caddy.json`）：

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

<aside class="tip">

您不必使用配置文件，但本教程中我们会这样做。Caddy 的[管理 API](/docs/api) 设计用于其他程序或脚本。

</aside>

然后上传它：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: application/json" \
	-d @caddy.json
</code></pre>

<aside class="complete">为 Caddy 提供配置</aside>

我们可以通过再次发送 GET 请求来验证 Caddy 是否应用了我们的新配置：

<pre><code class="cmd bash">curl localhost:2019/config/</code></pre>

通过在浏览器中访问 [localhost:2015](http://localhost:2015) 或使用 `curl` 来测试它是否工作：

<pre><code class="cmd"><span class="bash">curl localhost:2015</span>
Hello, world!</code></pre>

如果您看到 _Hello, world!_，那么恭喜您——它成功了！确保您的配置按预期工作总是一个好主意，尤其是在部署到生产环境之前。

<aside class="complete">测试配置</aside>

## 您的第一个 Caddyfile

为了一个 "Hello World" 程序，我们刚才做的**工作还挺多的**。

另一种配置 Caddy 的方式是使用 [**Caddyfile**](/docs/caddyfile)。上面我们用 JSON 编写的相同配置可以简单地表达为：

```caddy
:2015

respond "Hello, world!"
```

将其保存到当前目录下的一个名为 `Caddyfile`（无扩展名）的文件中。

<aside class="complete">编写 Caddyfile</aside>

如果 Caddy 已经在运行，请停止它（<kbd>Ctrl</kbd>+<kbd>C</kbd>），然后运行：

<pre><code class="cmd bash">caddy adapt</code></pre>

或者，如果您将 Caddyfile 存储在其他位置或命名不是 `Caddyfile`：

<pre><code class="cmd bash">caddy adapt --config /path/to/Caddyfile</code></pre>

您将看到 JSON 输出！这是怎么回事？

我们刚刚使用了一个[配置适配器](/docs/config-adapters)将 Caddyfile 转换成了 Caddy 的原生 JSON 结构。

<aside class="complete">使用配置适配器</aside>

虽然我们可以获取那个输出并再次发起 API 请求，但我们可以跳过所有这些步骤，因为 `caddy` 命令可以为我们完成这些。如果当前目录下有一个名为 Caddyfile 的文件，并且没有指定其他配置，Caddy 将加载 Caddyfile，自动适配它，并立即运行它。

现在当前文件夹中有了一个 Caddyfile，让我们再次运行 `caddy run`：

<pre><code class="cmd bash">caddy run</code></pre>

或者，如果您的 Caddyfile 在其他位置：

<pre><code class="cmd bash">caddy run --config /path/to/Caddyfile</code></pre>

（如果它命名不是以 "Caddyfile" 开头，您还需要指定 `--adapter caddyfile`。）

现在您可以再次尝试加载您的网站，您会看到它正在工作！

<aside class="complete">使用初始配置启动</aside>

如您所见，有几种方式可以通过初始配置启动 Caddy：

- 当前目录下名为 Caddyfile 的文件
- `--config` 标志（可选地配合 `--adapter` 标志）
- `--resume` 标志（如果之前加载过配置）

## JSON 与 Caddyfile 的比较

现在您知道 Caddyfile 只是为了方便而转换为 JSON。

Caddyfile 看起来比 JSON 更简单，但您应该总是使用它吗？每种方法都有利弊。答案取决于您的需求和使用场景。

JSON | Caddyfile
-----|----------
易于生成 | 手工编写简单
易于编程自动化 | 自动化起来比较笨拙
表达能力极强 | 表达能力中等
涵盖 Caddy 的全部功能 | 涵盖 Caddy 的大部分功能
支持配置遍历 | 无法在 Caddyfile 内部遍历
支持部分配置修改 | 仅支持整体配置替换
可以导出 | 无法导出
与所有 API 端点兼容 | 与部分 API 端点兼容
文档自动生成 | 文档需手工编写
通用常见 | 特定领域
更高效 | 计算开销更大
有点乏味 | 有点乐趣
**了解更多：[JSON 结构](/docs/json/)** | **了解更多：[Caddyfile 文档](/docs/caddyfile)**

您需要根据您的使用场景决定哪种方式最适合。

需要注意的是，JSON 和 Caddyfile（以及[任何其他支持的配置适配器](/docs/config-adapters)）都可以与 [Caddy 的 API](/docs/api) 配合使用。但是，只有使用 JSON 才能获得 Caddy 的全部功能和 API 特性。如果使用配置适配器，通过 API 加载或更改配置的唯一方法是使用 [/load 端点](/docs/api#post-load)。

<aside class="complete">比较 JSON 和 Caddyfile</aside>

## API 与配置文件的比较

<aside class="tip">

实际上，即使是配置文件也要经过 Caddy 的 API 端点；`caddy` 命令只是为您封装了这些 API 调用。

</aside>

您还需要决定您的工作流程是基于 API 还是基于 CLI。（您可以在同一台服务器上同时使用 API 和配置文件，但我不推荐这样做：最好只有一个真相来源。）

API | 配置文件
----|-------------
通过 HTTP 请求进行配置更改 | 通过 shell 命令进行配置更改
易于扩展 | 难以扩展
手工管理困难 | 手工管理简单
非常有趣 | 也很有趣
**了解更多：[API 教程](/docs/api-tutorial)** | **了解更多：[Caddyfile 教程](/docs/caddyfile-tutorial)**

<aside class="tip">
	通过 API 手动管理服务器配置是完全可以做到的，只需使用合适的工具，例如：任何 REST 客户端应用程序。
</aside>

选择 API 还是配置文件工作流程与是否使用配置适配器是正交的：您可以使用 JSON 但将其存储在文件中并通过命令行界面使用；反过来，您也可以将 Caddyfile 与 API 一起使用。

但大多数人会使用 JSON+API 或 Caddyfile+CLI 的组合。

如您所见，Caddy 适用于各种不同的使用场景和部署方式！

<aside class="complete">比较 API 和配置文件</aside>

## 启动、停止、运行

由于 Caddy 是一个服务器，它会无限期运行。这意味着执行 `caddy run` 后，您的终端不会解除阻塞，直到进程终止（通常通过 <kbd>Ctrl</kbd>+<kbd>C</kbd>）。

尽管 `caddy run` 是最常见的，并且通常推荐使用（尤其是在创建系统服务时），但您也可以使用 `caddy start` 来启动 Caddy 并在后台运行：

<pre><code class="cmd bash">caddy start</code></pre>

这将让您可以继续使用终端，这在某些交互式的无头环境中非常方便。

之后您需要自行停止该进程，因为 <kbd>Ctrl</kbd>+<kbd>C</kbd> 不会为您停止它：

<pre><code class="cmd bash">caddy stop</code></pre>

或者使用 API 的 [/stop 端点](/docs/api#post-stop)。

<aside class="complete">后台运行</aside>

## 重载配置

您的服务器可以执行零停机的配置重载/更改。

所有加载或更改配置的 [API 端点](/docs/api) 都是平滑且零停机的。

然而，在使用命令行时，您可能会倾向于使用 <kbd>Ctrl</kbd>+<kbd>C</kbd> 停止服务器，然后重新启动以应用新配置。不要这样做：停止和启动服务器与配置更改是正交的，并且会导致停机。

<aside class="tip">
	停止服务器会导致服务中断。
</aside>

相反，请使用 `[caddy reload](/docs/command-line#caddy-reload)` 命令进行平滑的配置更改：

<pre><code class="cmd bash">caddy reload</code></pre>

实际上，这个命令在底层也使用了 API。它会加载（并在需要时适配您的配置文件为 JSON），然后平滑地替换活动配置，而不会造成停机。

如果加载新配置时出现任何错误，Caddy 会回退到上一个有效的配置。

<aside class="tip">
	技术上来说，新配置会在旧配置停止之前启动，因此在一小段时间内，两个配置都会运行！如果新配置失败，它会中止并报错，而旧配置则不会被停止。
</aside>

<aside class="complete">零停机配置重载</aside>