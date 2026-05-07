---
title: 响应匹配器（Caddyfile）
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
			item.innerHTML = `<a href="#syntax" style="color: inherit;">${text}</a>`;
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

**响应匹配器**可用于根据特定条件过滤（或分类）响应。

它们通常仅作为配置出现在某些其他指令内部，用于在将响应写给客户端的过程中作出决策。

- [语法](#syntax)
- [匹配器](#matchers)
	- [status](#status)
	- [header](#header)

## 语法

如果一个指令接受响应匹配器，则在语法文档中以 `[<response_matcher>]` 或 `[<inline_response_matcher>]` 的形式表示。

- **<response_matcher>** 标记可以是先前声明的命名响应匹配器的名称。例如：`@name`。
- **<inline_response_matcher>** 标记可以是响应条件本身，无需事先声明。例如：`status 200`。

### 命名

```caddy-d
@name {
	status <code...>
	header <field> [<value>]
}
```
如果只有响应的一个方面与该指令相关，可以将名称和条件放在同一行：

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

按 HTTP 状态码。

- **&lt;code...&gt;** 是一个 HTTP 状态码列表。特殊形式如 `2xx` 和 `3xx`，分别匹配 `200`-`299` 和 `300`-`399` 范围内的所有状态码。

#### 示例：

```caddy-d
@success status 2xx
```

### header

```caddy-d
header <field> [<value>]
```

按响应头字段。

- `<field>` 是要检查的 HTTP 头字段名称。
	- 如果以 `!` 开头，则字段必须不存在才能匹配（省略值参数）。
- `<value>` 是字段必须具有的值才能匹配。
	- 如果以 `*` 开头，则执行快速后缀匹配（出现在末尾）。
	- 如果以 `*` 结尾，则执行快速前缀匹配（出现在开头）。
	- 如果被 `*` 包围，则执行快速子串匹配（出现在任意位置）。
	- 否则，为快速精确匹配。

同一集合中的不同头字段是“与”关系。每个字段的多个值是“或”关系。

请注意，头字段可能会重复并具有不同的值。后端应用程序必须考虑到头字段值是数组而非单一值，而 Caddy 不会在这种困境中解释含义。

#### 示例：

匹配 `Foo` 头包含值 `bar` 的响应：

```caddy-d
@upgrade header Foo *bar*
```

匹配 `Foo` 头值为 `bar` 或 `baz` 的响应：

```caddy-d
@foo {
	header Foo bar
	header Foo baz
}
```

匹配根本没有 `Foo` 头字段的响应：

```caddy-d
@not_foo header !Foo