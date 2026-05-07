---
title: "扩展Caddy"
---

# 扩展Caddy

Caddy 因其模块化架构而易于扩展。大多数Caddy扩展（或插件）如果扩展或插入到Caddy的配置结构中，被称为**模块**。需要明确的是，Caddy模块与[Go模块](https://github.com/golang/go/wiki/Modules)不同（但它们也是Go模块）。

**先决条件：**
- 基本了解 [Caddy 的架构](/docs/architecture)
- 熟练掌握 Go 语言
- [`go` <img src="/old/resources/images/external-link.svg" class="external-link">](https://golang.org/doc/install)
- [`xcaddy` <img src="/old/resources/images/external-link.svg" class="external-link">](https://github.com/caddyserver/xcaddy)


## 快速开始

Caddy 模块是一种命名类型，当其包被导入时，它会将自己注册为Caddy模块。关键在于，模块总是实现 [`caddy.Module`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#Module) 接口，该接口提供了模块的名称和一个构造函数。

在一个新的Go模块中，将以下模板粘贴到一个Go文件中，并自定义你的包名、类型名和Caddy模块ID：

```go
package mymodule

import "github.com/caddyserver/caddy/v2"

func init() {
	caddy.RegisterModule(Gizmo{})
}

// Gizmo 是一个示例；在此处放入你自己的类型。
type Gizmo struct {
}

// CaddyModule 返回 Caddy 模块信息。
func (Gizmo) CaddyModule() caddy.ModuleInfo {
	return caddy.ModuleInfo{
		ID:  "foo.gizmo",
		New: func() caddy.Module { return new(Gizmo) },
	}
}
```

然后从你的项目目录运行此命令，你应该能在列表中看到你的模块：

<pre><code class="cmd bash">xcaddy list-modules
...
foo.gizmo
...</code></pre>

<aside class="tip">

[`xcaddy` 命令](https://github.com/caddyserver/xcaddy) 是每个模块开发者工作流程中的重要组成部分。它用你的插件编译 Caddy，然后使用给定的参数运行它。它会每次都丢弃临时二进制文件（类似于 `go run`）。

</aside>


恭喜，你的模块已注册到 Caddy，并且可以在 [Caddy 的配置文档](/docs/json/) 中的任何使用相同命名空间模块的地方使用它。

在底层，`xcaddy` 只是创建一个新的 Go 模块，该模块同时需要 Caddy 和你的插件（使用适当的 `replace` 来使用你的本地开发版本），然后添加一个导入以确保它被编译进来：

```go
import _ "github.com/example/mymodule"
```


## 模块基础

Caddy 模块：

1. 实现 `caddy.Module` 接口以提供 ID 和构造函数
2. 在正确的命名空间中具有唯一名称
3. 通常满足一些对该命名空间的主模块有意义的接口

**主模块**（或**父模块**）是加载/初始化其他模块的模块。它们通常为客模块定义命名空间。

**客模块**（或**子模块**）是被加载或初始化的模块。所有模块都是客模块。


## 模块 ID

每个 Caddy 模块都有一个唯一的 ID，由命名空间和名称组成：

- 完整的 ID 看起来像 `foo.bar.module_name`
- 命名空间是 `foo.bar`
- 名称是 `module_name`，必须在其命名空间内唯一

模块 ID 必须使用 `snake_case` 约定。

### 命名空间

命名空间类似于类，即命名空间定义了其内所有模块共有的某些功能。例如，我们可以预期 `http.handlers` 命名空间内的所有模块都是 HTTP 处理程序。因此，主模块可以将该命名空间中的客模块从 `interface{}` 类型断言为更具体、有用的类型，例如 `caddyhttp.MiddlewareHandler`。

客模块必须正确命名空间，才能被主模块识别，因为主模块会要求 Caddy 提供某个命名空间内的模块，以获得主模块所需的功能。例如，如果你要编写一个名为 `gizmo` 的 HTTP 处理程序模块，你的模块名称应为 `http.handlers.gizmo`，因为 `http` 应用会在 `http.handlers` 命名空间中查找处理程序。

换句话说，根据其模块命名空间，Caddy 模块应实现[特定接口](/docs/extending-caddy/namespaces)。根据这个约定，模块开发者可以说出诸如“`http.handlers` 命名空间中的所有模块都是 HTTP 处理程序”这样直观的说法。更技术性地说，这通常意味着“`http.handlers` 命名空间中的所有模块都实现了 `caddyhttp.MiddlewareHandler` 接口”。因为该方法集是已知的，所以可以断言并使用更具体的类型。

**[查看将标准 Caddy 命名空间映射到其 Go 类型的表格。](/docs/extending-caddy/namespaces)**

`caddy` 和 `admin` 命名空间是保留的，不能用作应用名称。

要编写插入第三方主模块的模块，请查阅这些模块的命名空间文档。

### 名称

命名空间内的名称对用户来说很重要且高度可见，但并不是特别重要，只要它唯一、简洁并且对其功能有意义即可。


## 应用模块

应用是命名空间为空的模块，并且按照惯例成为其自己的顶级命名空间。应用模块实现 [`caddy.App`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#App) 接口。

这些模块出现在 Caddy 配置顶层的 `"apps"` 属性中：

```json
{
	"apps": {}
}
```

示例[应用](/docs/json/apps/)包括 `http` 和 `tls`。它们的命名空间为空。

为这些应用编写的客模块应位于从应用名称派生的命名空间中。例如，HTTP 处理程序使用 `http.handlers` 命名空间，TLS 证书加载器使用 `tls.certificates` 命名空间。

## 模块实现

模块几乎可以是任何类型，但结构体是最常见的，因为它们可以保存用户配置。


### 配置

大多数模块需要一些配置。只要你的类型与 JSON 兼容，Caddy 会自动处理这些。因此，如果模块是结构体类型，则需要在其字段上使用结构体标签，这些标签应根据 Caddy 约定使用 `snake_case`：

```go
type Gizmo struct {
	MyField string `json:"my_field,omitempty"`
	Number  int    `json:"number,omitempty"`
}
```

在结构体标签中使用 `omitempty` 选项将在 JSON 输出中忽略该字段（如果它是其类型的零值）。这在输出 JSON 时（例如，从 Caddyfile 适配到 JSON）有助于保持配置简洁。

当模块初始化时，它已经填充了其配置。也可以在模块初始化后执行其他[配置步骤](#配置)和[验证](#验证)步骤。


### 模块生命周期

当一个模块被主模块加载时，它的生命周期开始。发生以下步骤：

1. 调用 [`New()`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#ModuleInfo.New) 获取模块值的实例。
2. 模块的配置被反序列化到该实例中。
3. 如果模块实现了 [`caddy.Provisioner`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#Provisioner)，则调用 `Provision()` 方法。
4. 如果模块实现了 [`caddy.Validator`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#Validator)，则调用 `Validate()` 方法。
5. 此时，主模块获得加载的客模块作为 `interface{}` 值，因此主模块通常会将客模块断言为更有用的类型。请查阅主模块的文档，了解其命名空间中客模块需要满足的条件，例如需要实现哪些方法。
6. 当不再需要模块时，如果它实现了 [`caddy.CleanerUpper`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#CleanerUpper)，则调用 `Cleanup()` 方法。

请注意，在给定时间内，多个已加载的模块实例可能会重叠！在配置更改期间，新模块会在旧模块停止之前启动。请谨慎使用全局状态。使用 [`caddy.UsagePool`](https://pkg.go.dev/github.com/caddyserver/caddy/v2#UsagePool) 类型来帮助管理跨模块加载的全局状态。如果你的模块监听套接字，使用 `caddy.Listen*()` 获取支持重叠使用的套接字。

### 配置

模块的配置将自动反序列化到其值中（在加载 JSON 配置时）。这意味着，例如，结构体字段将被为你填充。

但是，如果你的模块需要额外的配置步骤，你可以实现（可选）接口 [`caddy.Provisioner`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#Provisioner)：

```go
// Provision 设置模块。
func (g *Gizmo) Provision(ctx caddy.Context) error {
	// TODO: 设置模块
	return nil
}
```

在此处，你应该为用户未提供的字段（即非零值的字段）设置默认值。如果某个字段是必需的，并且在未设置时可以返回错误。对于零值具有含义的数字字段（例如，某些超时时间），你可能希望支持 `-1` 表示“关闭”而不是 `0`，这样可以在用户未配置时设置默认值。

这也是主模块通常会加载其客/子模块的地方。

模块可以通过调用 `ctx.App()` 访问其他应用，但模块不能有循环依赖关系。换句话说，如果 `tls` 应用加载的模块依赖于 `http` 应用，那么 `http` 应用加载的模块就不能依赖于 `tls` 应用。（与 Go 中禁止导入循环的规则非常相似。）

此外，你应该避免在 `Provision` 中执行昂贵的操作，因为即使配置仅被验证，配置也会执行。在配置阶段，不要期望模块会被实际使用。

#### 日志

请参阅 [Caddy 中的日志记录方式](/docs/logging)。如果你的模块需要日志记录，不要使用 Go 标准库中的 `log.Print*()`。换句话说，**不要使用 Go 的全局日志记录器**。Caddy 使用高性能、高度灵活的结构化日志记录，基于 [zap](https://github.com/uber-go/zap)。

要输出日志，在模块的 Provision 方法中获取一个日志记录器：

```go
func (g *Gizmo) Provision(ctx caddy.Context) error {
	g.logger = ctx.Logger() // g.logger 是一个 *zap.Logger
}
```

然后你可以使用 `g.logger` 输出结构化的分级日志。有关详细信息，请参阅 [zap 的 godoc](https://pkg.go.dev/go.uber.org/zap?tab=doc#Logger)。


### 验证

想要验证其配置的模块可以通过满足（可选）接口 [`caddy.Validator`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#Validator) 来实现：

```go
// Validate 验证模块是否具有可用的配置。
func (g Gizmo) Validate() error {
	// TODO: 验证模块的设置
	return nil
}
```

Validate 应为只读函数。它在 `Provision()` 方法之后运行。


### 接口守卫

Caddy 模块的行为是隐式的，因为 Go 接口是隐式满足的。只需向模块的类型添加正确的方法即可使模块正确或错误。因此，拼写错误或方法签名错误可能导致意外（缺乏）行为。

幸运的是，有一种简单、无开销的编译时检查方法，你可以将其添加到代码中，以确保添加了正确的方法。这些被称为接口守卫：

```go
var _ InterfaceName = (*YourType)(nil)
```

将 `InterfaceName` 替换为你打算满足的接口，将 `YourType` 替换为模块类型的名称。

例如，一个 HTTP 处理程序（如静态文件服务器）可能满足多个接口：

```go
// 接口守卫
var (
	_ caddy.Provisioner           = (*FileServer)(nil)
	_ caddyhttp.MiddlewareHandler = (*FileServer)(nil)
)
```

如果 `*FileServer` 不满足这些接口，这将阻止程序编译。

如果没有接口守卫，可能会混入令人困惑的错误。例如，如果你的模块在使用前必须配置自身，但你的 `Provision()` 方法有错误（例如，拼写错误或签名错误），则配置将永远不会发生，导致令人困惑的情况。接口守卫非常容易，可以防止这种情况。它们通常放在文件底部。


## 主模块

当一个模块加载自己的客模块时，它就成为主模块。如果模块功能的某个部分可以通过不同方式实现，这将非常有用。

主模块几乎总是结构体。通常，支持客模块需要两个结构体字段：一个用于保存其原始 JSON，另一个用于保存其解码后的值：

```go
type Gizmo struct {
	GadgetRaw json.RawMessage `json:"gadget,omitempty" caddy:"namespace=foo.gizmo.gadgets inline_key=gadgeter"`

	Gadget Gadgeter `json:"-"`
}
```

第一个字段（此示例中的 `GadgetRaw`）是客模块的原始、未配置的 JSON 形式的存放位置。

第二个字段（`Gadget`）是最终配置后的值最终存储的位置。由于第二个字段不是面向用户的，我们通过结构体标签将其从 JSON 中排除。（如果其他包不需要，你也可以将其取消导出，这样就不需要结构体标签了。）

### Caddy 结构体标签

原始模块字段上的 `caddy` 结构体标签帮助 Caddy 了解要加载的模块的命名空间和名称（构成完整 ID）。它还用于生成文档。

结构体标签具有非常简单的格式：`key1=val1 key2=val2 ...`

对于模块字段，结构体标签将如下所示：

```go
`caddy:"namespace=foo.bar inline_key=baz"`
```

`namespace=` 部分是必需的。它定义了在其中查找模块的命名空间。

`inline_key=` 部分仅在模块名称将与模块本身一起**内联**找到时使用；这意味着该值是一个对象，其中一个键是内联键，其值是模块的名称。如果省略，则字段类型必须为 [`caddy.ModuleMap`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#ModuleMap) 或 `[]caddy.ModuleMap`，其中映射键是模块名称。


### 加载客模块

要加载客模块，在配置阶段调用 [`ctx.LoadModule()`](https://pkg.go.dev/github.com/caddyserver/caddy/v2?tab=doc#Context.LoadModule)：

```go
// Provision 设置 g 并加载其 gadget。
func (g *Gizmo) Provision(ctx caddy.Context) error {
	if g.GadgetRaw != nil {
		val, err := ctx.LoadModule(g, "GadgetRaw")
		if err != nil {
			return fmt.Errorf("加载 gadget 模块：%v", err)
		}
		g.Gadget = val.(Gadgeter)
	}
	return nil
}
```

注意，`LoadModule()` 调用接受指向结构体的指针和字段名称作为字符串。这有点奇怪，对吧？为什么不直接传递结构体字段？这是因为根据配置的布局，有几种不同的加载模块的方式。这种方法签名允许 Caddy 使用反射来确定加载模块的最佳方式，最重要的是，读取其结构体标签。

如果客模块必须由用户显式设置，则在尝试加载之前，如果 Raw 字段为 nil 或为空，你应该返回错误。

注意加载的模块是如何被断言类型的：`g.Gadget = val.(Gadgeter)` - 这是因为返回的 `val` 是 `interface{}` 类型，不是很有用。但是，我们期望声明的命名空间（来自示例中结构体标签的 `foo.gizmo.gadgets`）中的所有模块都实现 `Gadgeter` 接口，因此这种类型断言是安全的，然后我们可以使用它！

如果你的主模块定义了一个新的命名空间，请务必记录该命名空间及其 Go 类型，供开发者参考，[就像我们在这里所做的那样](/docs/extending-caddy/namespaces)。

## 模块文档

注册模块使新的 Caddy 模块出现在模块文档中，并在 http://caddyserver.com/download 上可用。注册可在 http://caddyserver.com/account 进行。如果你还没有帐户，请创建一个新帐户，然后点击“注册包”。

## 完整示例

假设我们要编写一个 HTTP 处理程序模块。这将是一个为了演示目的而编造中间件，它在每个 HTTP 请求时将访问者的 IP 地址打印到流中。

我们还希望它可以通过 Caddyfile 进行配置，因为大多数人在非自动化情况下更喜欢使用 Caddyfile。我们通过注册一个 Caddyfile 处理程序指令来实现这一点，这是一种可以向 HTTP 路由添加处理程序的指令。我们还实现了 `caddyfile.Unmarshaler` 接口。通过添加这几行代码，该模块就可以使用 Caddyfile 进行配置！例如：`visitor_ip stdout`。

以下是此类模块的代码，并附有解释性注释：

```go
package visitorip

import (
	"fmt"
	"io"
	"net/http"
	"os"

	"github.com/caddyserver/caddy/v2"
	"github.com/caddyserver/caddy/v2/caddyconfig/caddyfile"
	"github.com/caddyserver/caddy/v2/caddyconfig/httpcaddyfile"
	"github.com/caddyserver/caddy/v2/modules/caddyhttp"
)

func init() {
	caddy.RegisterModule(Middleware{})
	httpcaddyfile.RegisterHandlerDirective("visitor_ip", parseCaddyfile)
}

// Middleware 实现了一个 HTTP 处理程序，它将访问者的 IP 地址写入文件或流。
type Middleware struct {
	// 要写入的文件或流。可以是 "stdout" 或 "stderr"。
	Output string `json:"output,omitempty"`

	w io.Writer
}

// CaddyModule 返回 Caddy 模块信息。
func (Middleware) CaddyModule() caddy.ModuleInfo {
	return caddy.ModuleInfo{
		ID:  "http.handlers.visitor_ip",
		New: func() caddy.Module { return new(Middleware) },
	}
}

// Provision 实现 caddy.Provisioner。
func (m *Middleware) Provision(ctx caddy.Context) error {
	switch m.Output {
	case "stdout":
		m.w = os.Stdout
	case "stderr":
		m.w = os.Stderr
	default:
		return fmt.Errorf("需要指定输出流")
	}
	return nil
}

// Validate 实现 caddy.Validator。
func (m *Middleware) Validate() error {
	if m.w == nil {
		return fmt.Errorf("没有写入器")
	}
	return nil
}

// ServeHTTP 实现 caddyhttp.MiddlewareHandler。
func (m Middleware) ServeHTTP(w http.ResponseWriter, r *http.Request, next caddyhttp.Handler) error {
	m.w.Write([]byte(r.RemoteAddr))
	return next.ServeHTTP(w, r)
}

// UnmarshalCaddyfile 实现 caddyfile.Unmarshaler。
func (m *Middleware) UnmarshalCaddyfile(d *caddyfile.Dispenser) error {
	d.Next() // 消耗指令名称

	// 需要一个参数
	if !d.NextArg() {
		return d.ArgErr()
	}

	// 存储参数
	m.Output = d.Val()
	return nil
}

// parseCaddyfile 将 h 中的标记解析为新的 Middleware。
func parseCaddyfile(h httpcaddyfile.Helper) (caddyhttp.MiddlewareHandler, error) {
	var m Middleware
	err := m.UnmarshalCaddyfile(h.Dispenser)
	return m, err
}

// 接口守卫
var (
	_ caddy.Provisioner           = (*Middleware)(nil)
	_ caddy.Validator             = (*Middleware)(nil)
	_ caddyhttp.MiddlewareHandler = (*Middleware)(nil)
	_ caddyfile.Unmarshaler       = (*Middleware)(nil)
)