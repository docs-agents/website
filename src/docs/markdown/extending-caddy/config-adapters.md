---
title: "编写配置适配器"
---

# 编写配置适配器

出于各种原因，你可能希望使用非 [JSON](/docs/json/) 格式来配置 Caddy。Caddy 通过[配置适配器](/docs/config-adapters)为此提供了原生支持。

如果你偏好的语言/语法/格式尚无对应的适配器，你可以自行编写！

## 模板

以下是一个可供您起步的模板：

```go
package myadapter

import (
	"fmt"

	"github.com/caddyserver/caddy/v2/caddyconfig"
)

func init() {
	caddyconfig.RegisterAdapter("adapter_name", MyAdapter{})
}

// MyAdapter 将 ____ 适配为 Caddy JSON。
type MyAdapter struct{
}

// Adapt 将主体内容适配为 Caddy JSON。
func (a MyAdapter) Adapt(body []byte, options map[string]interface{}) ([]byte, []caddyconfig.Warning, error) {
	// TODO: 解析 body 并将其转换为 JSON
	return nil, nil, fmt.Errorf("未实现")
}
```

- 请参阅 [`RegisterAdapter()`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig?tab=doc#RegisterAdapter) 的 godoc 文档
- 请参阅 ['Adapter'](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig?tab=doc#Adapter) 接口的 godoc 文档

返回的 JSON **不应**缩进；它应始终是紧凑格式。调用方如果需要，可以自行美化。

需要注意的是，虽然配置适配器是 Caddy _插件_，但它们并非 Caddy _模块_，因为它们不集成到配置的某个部分中（但为方便起见，它们会出现在 `list-modules` 中）。因此，它们没有 `Provision()` 或 `Validate()` 方法，也不遵循其余的模块生命周期。它们只需实现 `Adapter` 接口并注册为适配器即可。

在填充配置中类型为 `json.RawMessage`（即模块字段）的字段时，请使用 `JSON()` 和 `JSONModuleObject()` 函数：

- [`caddyconfig.JSON()`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig?tab=doc#JSON) 用于编组不包含嵌入模块名称的模块值。（常用于 ModuleMap 字段，其中模块名称是映射键。）
- [`caddyconfig.JSONModuleObject()`](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig?tab=doc#JSONModuleObject) 用于编组包含添加到对象中的模块名称的模块值。（几乎在所有其他地方使用。）

## Caddyfile 服务器类型

也可以实现自定义的 Caddyfile 格式。Caddyfile 适配器是一个单一的适配器实现，其默认的“服务器类型”是 HTTP，但它支持在注册时使用替代的“服务器类型”。例如，HTTP Caddyfile 的注册方式如下：

```go
func init() {
	caddyconfig.RegisterAdapter("caddyfile",  caddyfile.Adapter{ServerType: ServerType{}})
}
```

你需要实现 [`caddyfile.ServerType` 接口](https://pkg.go.dev/github.com/caddyserver/caddy/v2/caddyconfig/caddyfile?tab=doc#ServerType)并相应地注册你自己的适配器。