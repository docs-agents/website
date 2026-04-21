---
title: 配置适配器
---

# 配置适配器

Caddy 的本地配置语言是 [JSON](https://www.json.org/json-en.html)，但手动编写 JSON 可能既繁琐又容易出错。这就是为什么 Caddy 支持通过**配置适配器**使用其他语言进行配置。它们是 Caddy 插件，使您能够使用首选格式的配置，并为您输出 [Caddy JSON](/docs/json/)。

例如，配置适配器可以 [将您的 NGINX 配置转换为 Caddy JSON](https://github.com/caddyserver/nginx-adapter)。

## 已知的配置适配器

以下配置适配器当前可用（其中一些是第三方项目）：

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

您可以通过在命令行上指定 `--adapter` 标志来使用配置适配器，该标志适用于大多数接受配置的子命令：

<pre><code class="cmd bash">caddy run --config caddy.yaml --adapter yaml</code></pre>

或通过 [`/load` 端点](/docs/api#post-load) 的 API 使用：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: application/yaml" \
	--data-binary @caddy.yaml</code></pre>

如果您只想获取输出 JSON 而不运行它，可以使用 [`caddy adapt`](/docs/command-line#caddy-adapt) 命令：

<pre><code class="cmd bash">caddy adapt --config caddy.yaml --adapter yaml</code></pre>

## 注意事项

并非所有配置语言都与 Caddy 100% 兼容；某些功能或行为只是不能很好地转换，或者尚未在适配器或 Caddy 本身中编程实现。

某些适配器进行 1-1 转换，例如 YAML→JSON 或 TOML→JSON。其他适配器是专为 Caddy 设计的，例如 Caddyfile。通常，这些适配器将始终有效。

然而，并非所有适配器在所有时候都有效。配置适配器尽最大努力将您的输入转换为 Caddy JSON，以最高的保真度和正确性。由于此转换过程不能保证始终完整和正确，我们不称它们为"转换器"或"翻译器"。它们是"适配器"，因为它们至少会为您提供一个良好的起点，以完成最终 JSON 配置的创建。

配置适配器可以输出结果 JSON、警告和错误。如果没有错误则输出 JSON 结果。当输入有问题时（例如语法错误）会发生错误。当适应过程中有问题但不一定是致命问题时，会发出警告（例如，不支持的功能）。如果使用的配置在适应过程中出现警告，请谨慎行事。
