---
title: "Caddyfile 支持"
---

# Caddyfile 支持

Caddy 模块在[注册](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#RegisterModule)后，会根据其命名空间自动添加到[原生 JSON 配置](/docs/json/)中，使其既可用又有文档可查。这使得 Caddyfile 支持成为可选项，但通常偏好 Caddyfile 的用户会提出这一需求。

## 解组器

要为你的模块添加 Caddyfile 支持，只需实现 [`caddyfile.Unmarshaler`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/caddyfile?tab=doc#Unmarshaler) 接口。你可以通过解析令牌的方式，自行决定模块的 Caddyfile 语法。

解组器的工作是利用传入的 [`caddyfile.Dispenser`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/caddyfile?tab=doc#Dispenser) 来设置模块类型，例如填充其字段。例如，一个名为 `Gizmo` 的模块类型可能有以下方法：

```go
// UnmarshalCaddyfile 实现 caddyfile.Unmarshaler。语法：
//
// gizmo <name> [<option>]
//
func (g *Gizmo) UnmarshalCaddyfile(d *caddyfile.Dispenser) error {
	d.Next() // 消费指令名称

	if !d.Args(&g.Name) {
		// 参数不足
		return d.ArgErr()
	}
	if d.NextArg() {
		// 可选参数
		g.Option = d.Val()
	}
	if d.NextArg() {
		// 参数过多
		return d.ArgErr()
	}

	return nil
}
```

建议在方法的 godoc 注释中记录语法。有关解析 Caddyfile 的更多信息，请参见 [`caddyfile` 包的 godoc](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/caddyfile?tab=doc)。

可以通过简单的 `d.Next()` 调用来消费/跳过指令名称令牌。

请确保使用 `d.NextArg()` 或 `d.RemainingArgs()` 检查缺失和/或多余的参数。使用 `d.ArgErr()` 返回简单的"无效情况"消息，或使用 `d.Errf("some message")` 构造包含问题说明（理想情况下还有建议的解决方案）的有用错误消息。

你还应该添加一个[接口守卫](/docs/extending-caddy#interface-guards)以确保接口正确实现：

```go
var _ caddyfile.Unmarshaler = (*Gizmo)(nil)
```

### 块

为了接受比单行更多的配置，你可能希望允许带有子指令的块。这可以通过 `d.NextBlock()` 并迭代直到返回到原始嵌套级别来实现：

```go
for nesting := d.Nesting(); d.NextBlock(nesting); {
	switch d.Val() {
		case "sub_directive_1":
		// ...
		case "sub_directive_2":
		// ...
	}
}
```

只要循环的每次迭代消耗了整个段（行或块），这就是处理块的一种优雅方式。

## HTTP 指令

HTTP Caddyfile 是 Caddy 默认的 Caddyfile 适配器语法（或"服务器类型"）。它是可扩展的，这意味着你可以为你的模块[注册](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/httpcaddyfile?tab=doc#RegisterDirective)自己的"顶级"指令：

```go
func init() {
	httpcaddyfile.RegisterDirective("gizmo", parseCaddyfile)
}
```

如果你的指令只返回单个 HTTP 处理器（常见情况），你可能会发现 [`RegisterHandlerDirective`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/httpcaddyfile?tab=doc#RegisterHandlerDirective) 更容易使用：

```go
func init() {
	httpcaddyfile.RegisterHandlerDirective("gizmo", parseCaddyfileHandler)
}
```

基本思路是：你与指令关联的[解析函数](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/httpcaddyfile?tab=doc#UnmarshalFunc)返回一个或多个 [`ConfigValue`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/httpcaddyfile?tab=doc#ConfigValue) 值。（或者，如果使用 `RegisterHandlerDirective`，它会直接返回填充好的 `caddyhttp.MiddlewareHandler` 值。）每个配置值都与一个["类"](#classes)相关联，这有助于 HTTP Caddyfile 适配器知道它可以用于最终 JSON 配置的哪些部分。所有配置值都会堆积在一起，适配器在构建最终 JSON 配置时从中提取。

这种设计允许你的指令为任何可识别的类返回任何配置值，这意味着它可以影响 HTTP Caddyfile 适配器为其指定了类的配置的任何部分。

如果你已经实现了 `UnmarshalCaddyfile()` 方法，那么你的解析函数可以像下面这样简单：

```go
// parseCaddyfileHandler 将 h 中的令牌解组为新的中间件处理器值。
func parseCaddyfileHandler(h httpcaddyfile.Helper) (caddyhttp.MiddlewareHandler, error) {
	var g Gizmo
	err := g.UnmarshalCaddyfile(h.Dispenser)
	return g, err
}
```

有关如何使用 `httpcaddyfile.Helper` 类型的更多信息，请参见 [`httpcaddyfile` 包的 godoc](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/httpcaddyfile?tab=doc)。

### 处理器顺序

所有返回 HTTP 中间件/处理器值的指令都需要按正确的顺序进行评估。例如，设置网站根目录的处理器必须在使用根目录的处理器之前出现，这样后者才能知道目录路径是什么。

HTTP Caddyfile [为标准指令定义了一个硬编码的顺序](/docs/caddyfile/directives#directive-order)。这确保用户不需要了解其 Web 服务器最常见功能的实现细节，并使其更容易编写正确的配置。考虑到 Caddyfile 的可扩展性，单一的硬编码列表也防止了非确定性。

**当你注册一个新的处理器指令时，它必须先添加到该列表中才能使用（在 `route` 块之外）。** 这可以通过以下三种方法之一完成：

- （推荐）插件作者可以在 `init()` 中调用 [`httpcaddyfile.RegisterDirectiveOrder`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/httpcaddyfile#RegisterDirectiveOrder)，将指令插入到相对于另一个[标准指令](/docs/caddyfile/directives#directive-order)的顺序中。这样，用户无需额外设置即可在站点中直接使用该指令。例如，要将你的指令 `gizmo` 插入到 `header` 处理器之后进行评估：

	```go
	httpcaddyfile.RegisterDirectiveOrder("gizmo", httpcaddyfile.After, "header")
	```

- 用户可以添加 [`order` 全局选项](/docs/caddyfile/options) 来修改其 Caddyfile 的标准顺序。例如：`order gizmo before respond` 会将新指令 `gizmo` 插入到 `respond` 处理器之前进行评估。然后指令可以正常使用。

- 用户可以将指令放在 [`route` 块](/docs/caddyfile/directives/route) 中。由于 `route` 块中的指令不会被重新排序，因此在 `route` 块中使用的指令不需要出现在列表中。

如果你选择后两种选项之一，请为用户记录建议，指明你的指令在列表中的正确位置，以便他们能够正确使用。

### 类

下表描述了 HTTP Caddyfile 适配器可识别的每个类及其导出类型：

类名 | 预期类型 | 描述
---------- | ------------- | -----------
bind | `[]string` | 服务器监听器绑定地址
route | `caddyhttp.Route` | HTTP 处理器路由
error_route | `*caddyhttp.Subroute` | HTTP 错误处理路由
tls.connection_policy | `*caddytls.ConnectionPolicy` | TLS 连接策略
tls.cert_issuer | `certmagic.Issuer` | TLS 证书颁发者
tls.cert_loader | `caddytls.CertificateLoader` | TLS 证书加载器

## 服务器类型

从结构上讲，Caddyfile 是一种简单的格式，因此可以有不同类型的 Caddyfile 格式（有时称为"服务器类型"）来满足不同的需求。

默认的 Caddyfile 格式是 HTTP Caddyfile，你可能对此很熟悉。这种格式主要配置 [`http` 应用](/docs/modules/http)，同时可能只在 Caddy 配置结构的其他部分（例如用于加载和自动化证书的 `tls` 应用）中零星地注入一些配置。

要配置除 HTTP 之外的应用，你可能希望实现自己的配置适配器，该适配器使用[你自己的服务器类型](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/caddyfile?tab=doc#Adapter)。Caddyfile 适配器实际上会为你解析输入，并为你提供服务器块和选项列表，然后由你的适配器来理解该结构并将其转换为 JSON 配置。