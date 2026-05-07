---
title: import（Caddyfile 指令）
---

# import

包含一个[代码片段](/docs/caddyfile/concepts#snippets)或文件，并用该片段或文件的内容替换此指令。

这是一个特例：它在结构解析之前被评估，并且可以出现在 Caddyfile 中的任何位置。

## 语法

```caddy-d
import <pattern> [<args...>] [{block}]
```

- **&lt;pattern&gt;** 是要包含的文件名、glob 模式或[代码片段](/docs/caddyfile/concepts#snippets)的名称。其内容将替换此行，就像该文件的内容原本就出现在这里一样。

  如果找不到特定文件，则会报错，但空的 glob 模式不会报错。

  如果导入的是特定文件，且文件为空，则会发出警告。

  如果模式是文件名或 glob，则始终相对于 `import` 所在文件的路径。

  如果路径的最后一段使用了 glob 模式 `*`，则会忽略隐藏文件（即文件名以 `.` 开头的文件）。要导入隐藏文件，请使用 `.*` 作为最后一段。
- **&lt;args...&gt;** 是传递给导入令牌的可选参数列表。这个占位符是一个特例，在 Caddyfile 解析时（而非运行时）被评估。它们可以以多种形式使用，类似于 [Go 的切片语法](https://gobyexample.com/slices)：
  - `{args[n]}`，其中 `n` 是参数的基于 0 的位置索引
  - `{args[:]}`，插入所有参数
  - `{args[:m]}`，插入参数中 `m` 之前的部分
  - `{args[n:]}`，插入从 `n` 开始的参数
  - `{args[n:m]}`，插入 `n` 到 `m` 之间的参数

  对于插入多个令牌的形式，该占位符**必须**是一个独立的[令牌](/docs/caddyfile/concepts#tokens-and-quotes)，不能是另一个令牌的一部分。换句话说，它周围必须有空格，且不能放在引号内。

  请注意，在 v2.7.0 之前，语法为 `{args.N}`，但此形式已被弃用，推荐使用上述更灵活的语法。

⚠️ <i>实验性</i> <span style='white-space: pre;'> | </span> <span>v2.9.x+</span>
- **{block}** 是传递给导入令牌的可选块。这个占位符是一个特例，在 Caddyfile 解析时（而非运行时）递归评估。它可以以两种形式使用：
  - `{block}`，整个提供的块的内容将替换该占位符
  - `{blocks.key}`，其中 `key` 是提供的块中某个参数的第一个令牌

## 示例

导入相邻 `sites-enabled` 文件夹中的所有文件（隐藏文件除外）：

```caddy-d
import sites-enabled/*
```

导入一个使用导入参数设置 CORS 头的代码片段：

```caddy
(cors) {
	@origin header Origin {args[0]}
	header @origin Access-Control-Allow-Origin "{args[0]}"
	header @origin Access-Control-Allow-Methods "OPTIONS,HEAD,GET,POST,PUT,PATCH,DELETE"
}

example.com {
	import cors example.com
}
```

导入一个将代理上游列表作为参数的代码片段：

```caddy
(https-proxy) {
	reverse_proxy {args[:]} {
		transport http {
			tls
		}
	}
}

example.com {
	import https-proxy 10.0.0.1 10.0.0.2 10.0.0.3
}
```

导入一个创建代理并将前缀重写规则作为第一个参数的代码片段：

```caddy
(proxy-rewrite) {
	rewrite {args[0]}{uri}
	reverse_proxy {args[1:]}
}

example.com {
	import proxy-rewrite /api 10.0.0.1 10.0.0.2 10.0.0.3
}
```

⚠️ <i>实验性</i> <span style='white-space: pre;'> | </span> <span>v2.9.x+</span>

导入一个响应可配置的“hello world”消息和内容类型的代码片段：

```caddy
(hello-world) {
	header {
		Cache-Control max-age=3600
		X-Foo bar
		{blocks.content_type}
	}
	respond /hello-world 200 {
		{blocks.body}
	}
}

example.com {
	import hello-world {
		content_type {
			Content-Type text/html
		}
		body {
			body "<h1>hello world</h1>"
		}
	}
}
```

导入一个为反向代理提供可扩展选项的代码片段：

```caddy
(extendable-proxy) {
	reverse_proxy {
		{blocks.proxy_target}
		{blocks.proxy_options}
	}
}

example.com {
	import extendable-proxy {
		proxy_target {
			to 10.0.0.1
		}
		proxy_options {
			transport http {
				tls
			}
		}
	}
}
```

导入一个提供任意指令集但带有预加载中间件的代码片段：

```caddy
(instrumented-route) {
	header {
		Alt-Svc `h3="0.0.0.0:443"; ma=2592000`
	}
	tracing {
		span args[0]
	}
	{block}
}

example.com {
	import instrumented-route example-com {
		respond "OK"
	}
}