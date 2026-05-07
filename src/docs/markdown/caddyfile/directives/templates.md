---
title: templates (Caddyfile 指令)
---

# templates

将响应体作为[模版](/docs/modules/http.handlers.templates)文档执行。模板提供了用于制作简单动态页面的功能基元。特性包括 HTTP 子请求、HTML 文件包含、Markdown 渲染、JSON 解析、基本数据结构、随机性、时间等。


## 语法

```caddy-d
templates [<matcher>] {
	mime    <types...>
	between <open_delim> <close_delim>
	root    <path>
	extensions {
		<name> {
			...
		}
	}
}
```

- **mime** 是模板中间件会处理的 MIME 类型；任何不具备合格 `Content-Type` 的响应都不会被当作模板来解析。

  默认值：`text/html text/plain`。

- **between** 是模板动作的起始和结束分隔符。如果它们与文档中的其他内容冲突，可以修改。

  默认值：`{{printf "{{ }}"}}`。

- **root** 是站点根目录，在使用访问文件系统的函数时生效。

  默认值为 [`root`](root) 指令设置的站点根目录，若未设置则为当前工作目录。

- **extensions** 允许您注册由 `http.handlers.templates.functions.*` 命名空间下的模块提供的自定义模板函数。

  区块内的每个子指令对应一个模块名称。这些模块可以为模板函数映射表添加自定义函数，通常用于实现可复用的组件。此特性主要面向插件开发者。

内置模板函数的文档可在[模板模块](/docs/modules/http.handlers.templates#docs)中找到。



## 示例

如需一个用模板服务 Markdown 的完整网站示例，请查看[本网站](https://github.com/caddyserver/website)的源码！具体可查看 [`Caddyfile`](https://github.com/caddyserver/website/blob/master/Caddyfile) 和 [`src/docs/index.html`](https://github.com/caddyserver/website/blob/master/src/docs/index.html)。

为静态站点启用模板：

```caddy
example.com {
	root /srv
	templates
	file_server
}
```

使用模板服务简单静态响应时，请确保设置 `Content-Type`：

```caddy
example.com {
	header Content-Type text/plain
	templates
	respond `Current year is: {{printf "{{"}}now | date "2006"{{printf "}}"}}`
}
```

使用模板扩展（插件）：

```caddy
example.com {
	root /srv
	templates {
		extensions {
			# 需要 caddy-hitcounter 插件：
			# https://github.com/mholt/caddy-hitcounter
			hitCounter {
				style bright_green
				pad_digits 6
			}
		}
	}
	file_server
}