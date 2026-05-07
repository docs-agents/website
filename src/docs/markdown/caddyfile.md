---
title: Caddyfile
---

# Caddyfile

**Caddyfile** 是一种面向人类的便捷 Caddy 配置格式。它易于编写、易于理解，并且足以满足大多数使用场景，因此成为最受用户欢迎的 Caddy 使用方式。

其格式如下：

```caddy
example.com {
	root /var/www/wordpress
	encode
	php_fastcgi unix//run/php/php-version-fpm.sock
	file_server
}
```

（这是一个真实且可用于生产环境的 Caddyfile，它通过完全托管的 HTTPS 为 WordPress 提供服务。）

基本思路是：首先输入网站地址，然后输入网站所需的功能或特性。[查看更多常见模式](/docs/caddyfile/patterns)

## 菜单

- #### [快速入门指南](/docs/quick-starts/caddyfile)
  熟悉 Caddyfile 的良好起点。
- #### [Caddyfile 完整教程](/docs/caddyfile-tutorial)
  学习使用 Caddyfile 完成各种常见任务。
- #### [Caddyfile 概念](/docs/caddyfile/concepts)
  必读内容！涵盖结构、站点地址、匹配器、占位符等。
- #### [指令](/docs/caddyfile/directives)
  位于行首的关键词，用于为站点启用功能。
- #### [请求匹配器](/docs/caddyfile/matchers)
  通过将匹配器与指令结合使用来过滤请求。
- #### [全局选项](/docs/caddyfile/options)
  适用于整个服务器而非单个站点的设置。
- #### [常见模式](/docs/caddyfile/patterns)
  完成常见任务的简单方法。
<!-- - #### [Caddyfile 规范](/docs/caddyfile/spec) TODO: 完成此内容 -->

## 注意

Caddyfile 仅仅是 Caddy 的[配置适配器](/docs/config-adapters)。它通常更适合手动编写配置，但在表达能力、灵活性和可编程性方面不如 Caddy 的[原生 JSON 结构](/docs/json/)。如果你在自动化配置/部署 Caddy，可能更希望使用 JSON 结合 [Caddy 的 API](/docs/api)。（实际上，你也可以在有限范围内通过 API 使用 Caddyfile。）