---
title: 配置适配器
---

# 配置适配器

Caddy 的原生配置语言是 [JSON](https://www.json.org/json-en.html)，但手动编写 JSON 既繁琐又容易出错。因此，Caddy 支持通过**配置适配器**使用其他语言进行配置。这些适配器是 Caddy 插件，通过输出 [Caddy JSON](/docs/json/)，让你能够以偏好的格式使用配置。

例如，配置适配器可以将 [你的 NGINX 配置转换为 Caddy JSON](https://github.com/caddyserver/nginx-adapter)。

## 已知的配置适配器

目前可用的配置适配器如下（部分为第三方项目）：

- [**caddyfile**](/docs/caddyfile)（标准）
- [**nginx**](https://github.com/caddyserver/nginx-adapter)
- [**jsonc**](https://github.com/caddyserver/jsonc-adapter)
- [**json5**](https://github.com/caddyserver/json5-adapter)
- [**yaml**](https://github.com/abiosoft/caddy-yaml)
- [**cue**](https://github.com/caddyserver/cue-adapter)
- [**toml**](https://github.com/awoodbeck/caddy-toml-adapter)
- [**hcl**](https://github.com/francislavoie/caddy-hcl)
- [**dhall**](https://github.com/mholt/dhall-adapter)
- [**mysql**](https://github.com/zhangjiayin/caddy-mysql-adapter)

## 使用配置适配器

你可以通过在大多数接受配置的子命令中使用 `--adapter` 标志来指定配置适配器：

<pre><code class="cmd bash">caddy run --config caddy.yaml --adapter yaml</code></pre>

或者通过 API 的 [`/load` 端点](/docs/api#post-load) 使用：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: application/yaml" \
	--data-binary @caddy.yaml</code></pre>

如果你只想获取输出的 JSON 而不实际运行，可以使用 [`caddy adapt`](/docs/command-line#caddy-adapt) 命令：

<pre><code class="cmd bash">caddy adapt --config caddy.yaml --adapter yaml</code></pre>

## 注意事项

并非所有配置语言都能与 Caddy 100% 兼容；某些特性或行为可能无法很好地转换，或者尚未在适配器或 Caddy 本身中实现。

某些适配器进行 1:1 的转换，例如 YAML→JSON 或 TOML→JSON。其他适配器则是专门为 Caddy 设计的，例如 Caddyfile。通常，这些适配器始终能正常工作。

然而，并非所有适配器在任何时候都有效。配置适配器会尽力将你的输入以最高保真度和正确性转换为 Caddy JSON。由于这种转换过程并不能保证始终完整和正确，我们不将其称为“转换器”或“翻译器”。它们属于“适配器”，因为至少能为你提供一个良好的起点，帮助你最终完善 JSON 配置。

配置适配器可以输出最终的 JSON、警告和错误。如果无错误发生，则输出 JSON；如果输入出现问题（例如语法错误），则输出错误；当适配过程出现问题但不一定是致命的（例如不支持的特性）时，会发出警告。如果使用带有警告的适配配置，建议谨慎操作。