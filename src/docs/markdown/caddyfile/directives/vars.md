---
title: vars (Caddyfile directive)
---

# vars

设置一个或多个变量为特定值，以便在请求处理链中后续使用。

访问变量的主要方式是通过占位符，其形式为 `{vars.variable_name}`，或者使用 [`vars`](/docs/caddyfile/matchers#vars) 和 [`vars_regexp`](/docs/caddyfile/matchers#vars_regexp) 请求匹配器。

您可以使用 `placeholder` 函数在 [`templates`](templates) 指令中使用变量，例如：`{{ "{{placeholder \"http.vars.variable_name\"}}" }}`

作为一种特殊情况，可以覆盖存储在替换器中的名为 `http.auth.user.id` 的变量，以更新 [访问日志](log) 中的 `user_id` 字段。


## Syntax

```caddy-d
vars [<matcher>] [<name> <value>] {
    <name> <value>
    ...
}
```

- **&lt;name&gt;** 是要设置的变量名。

- **&lt;value&gt;** 是变量的值。

  如果可能，值将进行类型转换；`true` 和 `false` 将转换为布尔类型，数值将相应地转换为整数或浮点数。若要避免此转换并保持为字符串，您可以使用 [引号](/docs/caddyfile/concepts#tokens-and-quotes) 包裹它们。

## Examples

设置单个变量，其值根据请求路径条件设置，然后用该值进行响应：

```caddy
example.com {
	vars /foo* isFoo "yep"
	vars isFoo "nope"

	respond {vars.isFoo}
}
```

设置多个变量，每个变量转换为适当的标量类型：

```caddy-d
vars {
	# 布尔值
	abc true

	# 整数
	def 1

	# 浮点数
	ghi 2.3

	# 字符串
	jkl "example"
}
```
