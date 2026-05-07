---
title: Caddyfile 快速入门
---

# Caddyfile 快速入门

创建一个新的文本文件，命名为 `Caddyfile`（无扩展名）。

在 Caddyfile 中，首先要输入的是你网站的地址：

```caddy
localhost
```

<aside class="tip">

如果 HTTP 和 HTTPS 端口（分别是 80 和 443）在你的操作系统上是特权端口，那么你需要以提升的权限运行，或者使用更高的端口。要获取权限，可以使用 `sudo -E` 以 root 身份运行，或者使用 `sudo setcap cap_net_bind_service=+ep $(which caddy)`。或者，若要使用更高的端口，只需将地址改为类似 `localhost:2080` 的形式，并使用 Caddyfile 中的 [`http_port`](/docs/caddyfile/options) 选项更改 HTTP 端口。

</aside>

然后按回车键，输入你想要它执行的操作，使其看起来像这样：

```caddy
localhost

respond "Hello, world!"
```

保存此文件，并在包含 Caddyfile 的同一文件夹中运行 Caddy：

<pre><code class="cmd bash">caddy start</code></pre>

系统可能会要求你输入密码，因为默认情况下 Caddy 会通过 HTTPS 为所有站点（包括本地站点）提供服务。（密码提示应该只在第一次出现！）

<aside class="tip">

对于本地 HTTPS，Caddy 会自动为你生成证书和唯一的私钥。根证书会添加到系统的信任存储中，这就是为何需要密码提示的原因。这让你可以在 HTTPS 下进行本地开发，而不会出现证书错误。

</aside>

（如果出现权限错误，你可能需要以提升的权限运行，或者选择大于 1023 的端口。）

在浏览器中打开 [localhost](http://localhost) 或使用 `curl`：

<pre><code class="cmd"><span class="bash">curl https://localhost</span>
Hello, world!</code></pre>

你也可以通过使用花括号 `{ }` 在 Caddyfile 中定义多个站点。修改 Caddyfile 如下：

```caddy
localhost {
	respond "Hello, world!"
}

localhost:2016 {
	respond "Goodbye, world!"
}
```

你可以通过两种方式向 Caddy 提供更新后的配置：直接通过 API：

<pre><code class="cmd bash">curl localhost:2019/load \
	-H "Content-Type: text/caddyfile" \
	--data-binary @Caddyfile
</code></pre>

或者使用 reload 命令，该命令会为你执行相同的 API 请求：

<pre><code class="cmd bash">caddy reload</code></pre>

在[浏览器](https://localhost:2016)中或使用 `curl` 测试新的 "goodbye" 端点，确保其正常工作：

<pre><code class="cmd"><span class="bash">curl https://localhost:2016</span>
Goodbye, world!</code></pre>

当你完成 Caddy 操作后，请确保停止它：

<pre><code class="cmd bash">caddy stop</code></pre>

## 进一步阅读

- [Caddyfile 概念](/docs/caddyfile/concepts)
- [指令](/docs/caddyfile/directives)
- [常见模式](/docs/caddyfile/patterns)