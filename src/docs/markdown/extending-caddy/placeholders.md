---
title: "占位符支持"
---

# 占位符

在 Caddy 中，占位符由每个插件根据需要进行处理；它们不会自动在所有地方生效。

这意味着如果你希望自己的插件支持占位符，你必须显式地添加对它们的支持。

如果你对占位符还不熟悉，请从[这里](/docs/conventions#placeholders)开始阅读！

## 占位符概述

[占位符](/docs/conventions#placeholders)是一种格式为 `{foo.bar}` 的字符串，用作动态配置值，稍后在运行时进行评估。

Caddyfile 中的[环境变量替换](/docs/caddyfile/concepts#environment-variables)以美元符号开头，例如 `{$FOO}`，它们在 Caddyfile 解析时进行评估，不需要由你的插件处理。这些_不是_占位符，尽管它们使用相同的 `{ }` 语法。

因此，理解 `{env.HOST}`（一个[全局占位符](/docs/conventions#placeholders)）与 `{$HOST}`（Caddyfile 环境变量替换）本质上是不同的，这一点很重要。

例如，请参见以下 Caddyfile：
```caddy
:8080 {
	respond {$HOST} 200
}

:8081 {
	respond {env.HOST} 200
}
```

当你使用 `HOST=example caddy adapt` 将此 Caddyfile 适配为 JSON 时，你将得到：

```json
{
  "apps": {
    "http": {
      "servers": {
        "srv0": {
          "listen": [":8080"],
          "routes": [
            {
              "handle": [
                {
                  "body": "example",
                  "handler": "static_response",
                  "status_code": 200
                }
              ]
            }
          ]
        },
        "srv1": {
          "listen": [":8081"],
          "routes": [
            {
              "handle": [
                {
                  "body": "{env.HOST}",
                  "handler": "static_response",
                  "status_code": 200
                }
              ]
            }
          ]
        }
      }
    }
  }
}
```

特别要注意 `srv0` 和 `srv1` 中的 `"body"` 字段。

由于 `srv0` 使用了 `{$HOST}`（Caddyfile 环境变量替换），该值变成了 `example`，因为它是在生成 JSON 配置时，在 Caddyfile 解析期间处理的。

由于 `srv1` 使用了 `{env.HOST}`（一个全局占位符），在适配为 JSON 时它保持不变。

这意味着用户编写 JSON 配置（不通过 Caddyfile）时不能使用 `{$ENV}` 语法。因此，插件作者在配置被供应时实现占位符的替换支持非常重要。下文将对此进行解释。

## 实现占位符支持

你不应该在 [`UnmarshalCaddyfile()`](/docs/extending-caddy/caddyfile) 中处理占位符。相反，占位符应稍后被替换，要么在 [`Provision()`](/docs/extending-caddy#provisioning) 步骤中，要么在你的模块执行期间（例如 HTTP 处理器的 `ServeHTTP()`、匹配器的 `Match()` 等），使用 `caddy.Replacer`。

### 示例

这里，我们使用一个新构建的 replacer 来处理占位符。它可以访问[全局占位符](/docs/conventions#placeholders)，例如 `{env.HOST}`，但_不能_访问 HTTP 占位符，如 `{http.request.uri}`，因为供应发生在配置加载时，而不是请求期间。

```go
func (g *Gizmo) Provision(ctx caddy.Context) error {
	repl := caddy.NewReplacer()
	g.Name = repl.ReplaceAll(g.Name,"")
	return nil
}
```

这里，我们在 `ServeHTTP` 期间从请求上下文 `r.Context()` 中获取 replacer。这个 replacer 既可以访问全局占位符，也可以访问每个请求的 HTTP 占位符，例如 `{http.request.uri}`。

```go
func (g *Gizmo) ServeHTTP(w http.ResponseWriter, r *http.Request, next caddyhttp.Handler) error {
	repl := r.Context().Value(caddy.ReplacerCtxKey).(*caddy.Replacer)
	_, err := w.Write([]byte(repl.ReplaceAll(g.Name,"")))
	if err != nil {
		return err
	}
	return next.ServeHTTP(w, r)
}