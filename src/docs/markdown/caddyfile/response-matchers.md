---
title: 响应匹配器 (Caddyfile)
---

<script>
ready(function() {
	// Response matchers
	$$_('pre.chroma .nd').forEach(item => {
		if (item.innerText.includes('@')) {
			let text = item.innerText.replace(/</g, '&lt;').replace(/>/g, '&gt;');
			let url = '#' + item.innerText.replace(/_/g, "-");
			item.classList.add('nd');
			item.classList.remove('k');
			item.innerHTML = `<a href="#语法" style="color: inherit;">${text}</a>`;
		}
	});
	
	$$_('pre.chroma .k').forEach(item => {
		if (item.innerText.includes('status')) {
			item.innerHTML = '<a href="#status" style="color: inherit;">status</a>';
		}
	});
	
	$$_('pre.chroma .k').forEach(item => {
		if (item.innerText.includes('header')) {
			item.innerHTML = '<a href="#header" style="color: inherit;">header</a>';
		}
	});

	// We'll add links to all the subdirectives if a matching anchor tag is found on the page.
	addLinksToSubdirectives();
});
</script>

# 响应匹配器

**响应匹配器**可用于根据特定标准来过滤（或分类）响应。

这些通常仅作为某些其他指令内的配置出现，以便在响应写入客户端时做出决策。

- [语法](#语法)
- [匹配器](#匹配器)
	- [status](#status)
	- [header](#header)

## 语法

如果指令接受响应匹配器，用法在语法文档中表示为 `[<响应匹配器>]` 或 `[<内联响应匹配器>]`。

- **&lt;响应匹配器&gt;** 令牌可以是先前声明的命名响应匹配器的名称。例如：`@name`。
- **&lt;内联响应匹配器&gt;** 令牌可以是响应标准本身，无需先前声明。例如：`status 200`。

### 命名

```caddy-d
@name {
	status <code...>
	header <field> [<value>]
}
```
如果指令只与响应的一个方面相关，您可以将名称和标准放在同一行：

```caddy-d
@name status <code...>
```

### 内联

```caddy-d
... {
	status <code...>
	header <field> [<value>]
}
```
```caddy-d
... status <code...>
```
```caddy-d
... header <field> [<value>]
```

## 匹配器

### status

```caddy-d
status <code...>
```

按 HTTP 状态码匹配。

- **&lt;code...&gt;** 是 HTTP 状态码列表。特殊情况的字符串如 `2xx` 和 `3xx`，分别匹配 `200`-`299` 和 `300`-`399` 范围内的所有状态码。

#### 示例：

```caddy-d
@success status 2xx
```



### header

```caddy-d
header <field> [<value>]
```

按响应头字段匹配。

- `<field>` 是要检查的 HTTP 头字段名称。
	- 如果前缀为 `!`，则字段必须不存在才能匹配（省略值参数）。
- `<value>` 是字段必须具有的匹配值。
	- 如果前缀为 `*`，则执行快速后缀匹配（出现在末尾）。
	- 如果后缀为 `*`，则执行快速前缀匹配（出现在开头）。
	- 如果由 `*` 包围，则执行快速子字符串匹配（出现在任何位置）。
	- 否则，是快速精确匹配。

同一集合中的不同头字段是 AND（与）关系。每个字段的多个值使用 OR（或）连接。

请注意，头字段可以重复并具有不同的值。后端应用程序必须考虑头字段值是数组，而不是单个值，Caddy 不在此类情况中解释含义。

#### 示例：

匹配 `Foo` 头包含值 `bar` 的响应：

```caddy-d
@upgrade header Foo *bar*
```

匹配 `Foo` 头具有值 `bar` 或 `baz` 的响应：

```caddy-d
@foo {
	header Foo bar
	header Foo baz
}
```

匹配完全不具有 `Foo` 头字段的响应：

```caddy-d
@not_foo header !Foo
```