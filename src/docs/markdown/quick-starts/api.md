---
title: API 快速入门
---

# API 快速入门

**前置要求：**
- 基本的终端/命令行操作技能
- 环境变量 `PATH` 中包含 `caddy` 和 `curl`

---

首先启动 Caddy：

<pre><code class="cmd bash">caddy start</code></pre>

Caddy 当前处于空闲运行状态（使用空白配置）。使用 `curl` 为其提供一个简单的配置：

<pre><code class="cmd bash">curl localhost:2019/load \
    -H "Content-Type: application/json" \
    -d @- << EOF
    {
        "apps": {
            "http": {
                "servers": {
                    "hello": {
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
EOF</code></pre>

使用 [Here文档](https://en.wikipedia.org/wiki/Here_document#Unix_shells) 传递 POST 请求体可能会比较繁琐，如果你更倾向于使用文件，请将 JSON 保存到名为 `caddy.json` 的文件中，然后改用以下命令：

<pre><code class="cmd bash">curl localhost:2019/load \
  -H "Content-Type: application/json" \
  -d @caddy.json
</code></pre>

现在在浏览器中打开 [localhost:2015](http://localhost:2015) 或使用 `curl`：

<pre><code class="cmd"><span class="bash">curl localhost:2015</span>
Hello, world!</code></pre>

我们还可以使用以下 JSON 在不同接口上定义多个站点：

```json
{
	"apps": {
		"http": {
			"servers": {
				"hello": {
					"listen": [":2015"],
					"routes": [
						{
							"handle": [{
								"handler": "static_response",
								"body": "Hello, world!"
							}]
						}
					]
				},
				"bye": {
					"listen": [":2016"],
					"routes": [
						{
							"handle": [{
								"handler": "static_response",
								"body": "Goodbye, world!"
							}]
						}
					]
				}
			}
		}
	}
}
```

更新你的 JSON 文件，然后再次执行 API 请求。

在浏览器中尝试新的“再见”端点 [localhost:2016](http://localhost:2016) 或使用 `curl` 来确认其正常工作：

<pre><code class="cmd"><span class="bash">curl localhost:2016</span>
Goodbye, world!</code></pre>

当你完成 Caddy 的使用后，请确保将其停止：

<pre><code class="cmd bash">caddy stop</code></pre>

API 还有更多功能，包括导出配置以及对配置进行精细调整（而不是整体更新）。请务必阅读 [完整 API 教程](/docs/api-tutorial) 来学习如何使用！

## 延伸阅读

- [完整 API 教程](/docs/api-tutorial)
- [API 文档](/docs/api)