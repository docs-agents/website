---
title: map（Caddyfile 指令）
---

# map

根据输入值设置自定义占位符的值。

它将源值与映射的输入端进行比较，对于匹配的值，将输出值应用到每个目标。目标成为占位符名称。也可以为每个目标指定默认输出值。

映射的占位符直到被使用时才会求值，因此即使对于非常大的映射，此指令也非常高效。

## 语法

```caddy-d
map [<matcher>] <source> <destinations...> {
	[~]<input> <outputs...>
	default    <defaults...>
}
```

- **&lt;source&gt;** 是要切换的输入值。通常是一个占位符。

- **&lt;destinations...&gt;** 是要创建的占位符，用于保存输出值。

- **&lt;input&gt;** 是要匹配的输入值。如果以前缀 `~` 开头，则按正则表达式处理。

- **&lt;outputs...&gt;** 是一个或多个要存储在关联占位符中的输出值。第一个输出写入第一个目标，第二个输出写入第二个目标，依此类推。
  
  作为一种特殊情况，Caddyfile 解析器将字面连字符（`-`）输出视为 null/nil 值。如果您希望在给定输入的情况下回退到该输出的默认值，但希望其他输出使用非默认值，这将非常有用。

  输出值将尽可能进行类型转换；`true` 和 `false` 将转换为布尔类型，数值将相应地转换为整数或浮点数。要避免这种转换，您可以使用 [引号](/docs/caddyfile/concepts#tokens-and-quotes) 包裹输出，它们将保持为字符串。

  每个映射的输出数量不得超过目标数量；但是，为了方便起见，输出可以少于目标数量，任何缺失的输出将隐式填充。
  
  如果输入使用了正则表达式，则捕获组可以通过 `${group}` 引用，其中 `group` 是表达式中捕获组的名称或编号。捕获组 `0` 是整个正则表达式匹配结果，`1` 是第一个捕获组，`2` 是第二个捕获组，依此类推。

- **&lt;default&gt;** 指定如果没有匹配任何输入时要存储的输出值。


## 示例

以下示例演示了此指令的大部分功能：

```caddy-d
map {host}                {my_placeholder}  {magic_number} {
	example.com           "some value"      3
	foo.example.com       "another value"
	~(.*)\.example\.com$  "${1} subdomain"  5

	~.*\.net$             -                 7
	~.*\.xyz$             -                 15

	default               "unknown domain"  42
}
```

此指令根据 `{host}` 的值进行切换，即请求的域名。

- 如果请求的是 `example.com`，将 `{my_placeholder}` 设置为 `some value`，将 `{magic_number}` 设置为 `3`。
- 否则，如果请求的是 `foo.example.com`，将 `{my_placeholder}` 设置为 `another value`，并让 `{magic_number}` 默认为 `42`。
- 否则，如果请求的是 `example.com` 的任何子域名，将 `{my_placeholder}` 设置为包含第一个正则表达式捕获组值的字符串，即整个子域名，并将 `{magic_number}` 设置为 5。
- 否则，如果请求的是以 `.net` 或 `.xyz` 结尾的任何主机，仅将 `{magic_number}` 分别设置为 `7` 或 `15`。保持 `{my_placeholder}` 未设置。
- 否则（对于所有其他主机），将应用默认值：`{my_placeholder}` 将设置为 `unknown domain`，`{magic_number}` 将设置为 `42`。
