---
title: templates (Caddyfile directive)
---

# templates

将响应主体作为 [template](/docs/modules/http.handlers.templates) 文档执行。Templates 提供了用于构建简单动态页面的功能性原语。功能包括 HTTP 子请求、HTML 文件包含、Markdown 渲染、JSON 解析、基本数据结构、随机性、时间处理等。


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

- **mime** 是 templates 中间件将作用的 MIME 类型；任何没有合格 `Content-Type` 的响应都不会作为模板进行评估。

  默认值：`text/html text/plain`。

- **between** 是模板操作的开闭分隔符。如果它们干扰了文档的其他部分，可以更改它们。

  默认值：`{{printf "{{ }}"}}`。

- **root** 是站点根目录，当使用访问文件系统的函数时。

  默认为由 [`root`](root) 指令设置的站点根目录，如果未设置，则为当前工作目录。

- **extensions** 允许您注册由 `http.handlers.templates.functions.*` 命名空间中的模块提供的自定义模板函数。

  块内的每个子指令对应一个模块名称。这些模块可以向模板函数映射添加自定义函数，通常用于实现可重用的组件。此功能主要用于插件。

内置模板函数的文档可在 [templates module](/docs/modules/http.handlers.templates#docs) 中找到。



## 示例

要查看使用模板提供 Markdown 的站点的完整示例，请查看 [本网站](https://github.com/caddyserver/website) 的源码！具体来说，请查看 [`Caddyfile`](https://github.com/caddyserver/website/blob/master/Caddyfile) 和 [`src/docs/index.html`](https://github.com/caddyserver/website/blob/master/src/docs/index.html)。

为静态站点启用模板：

```caddy
example.com {
	root /srv
	templates
	file_server
}
```

要使用模板提供简单的静态响应，请确保设置 `Content-Type`：

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
```
